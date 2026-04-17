### 개념설명
#### 1-1. 코루틴의 실패는 "구조적 동시성(Structured Concurrency)" 규칙을 따른다.
`coroutineScope{...}` 내부에서 자식 코루틴 중 하나라도 실패(예외) 하면
- **스코프 전체가 실패**로 간주
- **나머지 자식들은 모두 취소(cancel)**
- 그리고 그 예외가 **상위로 전파**
##### 이게 기본 규칙인 이유
- "부분 성공" 상태로 남기지 않고
- 실패 시 관련 작업을 한꺼번에 정리해서
- 리소스/일관성 문제를 줄이기 위해서.
##### Java로 비유하면
```java
try{
    병렬작업;
}
catch(...){
    전체실패처리; //여기서 "전체" 개념이 강제되는 느낌
}
```


#### 1-2. 취소(Cancellation)는 "예외"처럼 전달된다
코루틴에서 취소는 내부적으로 `CancellationException`을 통해 전달
##### 중요 포인트
- 취소는 **정상적인 제어 흐름**(expected)으로 취급
	-> 보통 로그에 "에러"처럼 남기지 않음
- 취소는 **협력적(cooperative)**
	-> 코루틴이 **중단 지점(suspension point)** 에서 주로 취소를 인지
	-> CPU 바운드 루프는 직접 체크해야 함(`ensureActive()`, `isActive()`)

### 1-3. `async`와 `launch`의 예외 전파 차이(실무에서 매우 중요)
#### `launch`
- 예외가 발생하면 즉시 부모로 전파(부모 스코프를 실패로 만듦)
#### `async` 
- 예외가 지연되어 있다가 `await()`시점에 터짐
- 하지만 구조적 동시성 규칙상, 같은 스코프에서 자식 async가 실패하면 **형제도 취소**되는건 동일
#### 왜 필요하나면
- "A"가 실패해도 B/C는 계속해야 한다. 같은 부분 실패 허용 시나리오가 너무 많음
	- 여러 API 호출 중 일부만 실패해도 나머지 결과는 사용 가능
	- 여러 캐시 워밍업 중 하나 실패해도 나머지는 진행
	- 여러 알림 채널 중 하나 실패해도 다른 채널은 발송

### 예제 코드
#### 2-1. coroutineScope: 한 명 실패하면 다 같이 종료
``` kotlin
import kotlinx.coroutines.*

suspend fun exampleCoroutineScope() = coroutineScope{
    val a = async{
        error("A failed")
    }
    
    val b = async{
        try{
	        delay(1_000) //suspension point -> 취소가 잘 먹음
	        "B done"
        }
        finally {
            //취소 / 예외로 종료될 때 리소스 정리 지점
            println("B: finally(cleanup)")
        }
    }
    
    a.await()        //여기서 예외 던짐
    print(b.await()) //여기까지 보통 못 옴(b는 취소 됨)
}
```

##### 포인트: 
- `a`가 실패하면 `b` **자동 취소**
- `b`의 `finally`에서 **정리(cleanup)** 수행 가능

#### 2-2. DB/트랜잭션은 "취소되더라도 정리 보장"이 핵심
- 트랜잭션을 열었다면 **반드시** `try/finally`로 닫아야하고
- 취소 중에도 반드시 닫아야 하는 정리는 `NonCancellable`을 써서 보장
```kotlin
import kotlinx.coroutines.*

suspend fun<T> withTx(block: suspend() -> T): T {
    //실제로는 DataSource/TransactionManager를 사용
    print("tx begin")
    try{
        return block()
    }
    catch(e: Throwable){
        println("tx rollback: ${e::class.simpleName}")
        throw e
    }
    finally{
        withContext(NonCancellable){
            println("tx close (always)")
        }
    }
}
```
- `finally`는 취소돼도 실행 되지만,
- `finally` 내부에서 `delay()`같은 suspend를 하면 취소 때문에 중단될 수 있음
	->그래서 반드시 실행돼야 하는 정리에는 `NonCancellable`이 의미가 있음

#### 2-3. supervisorScope: A가 실패해도 B는 살아남는다.
```kotlin
import kotlinx.coroutine.*

suspend fun exampleSupervisorScope(): Pair<Result<String>, Result<String>> = supervisorScope {
    val a = async {
        error("A failed")
    }
    
    val b = async {
        delay(500)
        "B done"
    }
    
    val ra = runCatching {a.await()} //여기서 예외 던짐
    val rb = runCatching {b.await()} //B는 취소되지 않음
    
    ra to rb
}
```
##### 포인트
- 실패를 전체실패로 만들지 않고, **결과를 부분적으로 수집**가능
- `sealed class`결과 모델링과 찰떡궁합
```kotlin
sealed interface Outcome<out T> {
    data class Ok<T>(val value: T): Outcome<T>
    data class Err(val error: Throwable): Outcome<Nothing>
}

private fun <T> Result<T>.toOutcome(): Outcome<T> =
    fold(onSuccess = {Outcome.Ok(it)}, onFailure = {Outcome.Err(it)})
```

#### 2-4. CPU 바운드 작업 취소 체크(협력적 취소)
```kotlin
import kotlinx.coroutines.*

suspend fun hashLikeWork() = coroutinScope{
    launch(Dispatchers.Default) {
        repeat(10_000_000) { i-> 
            if(i%100_000 == 0) ensureActive() //취소 지점 명시
            //heavy compute...
        }
        print("done")
    }
}
```

#### 3. Anti-pattern
- `async`를 await하지 않는 fire-and-forget 용도로 사용하지 말 것
- `Exception`을 넓게 잡아 `CancellationException`까지 삼키지 말 것.
- `finally`안의 suspend 호출은 취소로 중단될 수 있으니 정리가 반드시 필요하면 `NonCancellable` 사용 고려

[[Day 5.1. CoroutinExceptionHandler]]