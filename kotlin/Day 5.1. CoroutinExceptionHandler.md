### 1. CoroutineExceptionHandler란?
`CoroutineExceptionHandler`를 한 문장으로 말하면
>"부모에게 전파될 예외 중,
>최종적으로 처리되지 않은 예외를 가로채는 '마지막 안전망'"

#### 중요포인트:
- 예외를 복구(recover)하는 용도가 아님
- 로깅/크래시 처리/알림 같은 side-effect 용도
- 구조적 동시성 규칙을 바꾸지 않는다.

#### Java로 비유하면..
- `try/catch` ≈ 지역 처리
- `Thread.UncaughtExceptionHandler` ≈ 전역 처리
- `CoroutineExceptionHandler` ≈ 코루틴 세계의 UncaughtExceptionHandler

### 2. 언제 동작하나? (핵심규칙)
#### 2-1. `launch`에서만 의미 있게 동작
```kotlin
val handler = CoroutineExceptionHandler {_, throwable -> 
    println("caught by handler: $throwable")
}

CorotineScope(Dispatchers.Default + handler).launch {
    error("boom")
}

// 출력
caught by handler: java.lang.IllegalStateException: boom
```
##### 이유
- `launch`는 **결과를 반환하지 않음**
- 예외가 발생하면 **부모로 즉시 전파**
- try/catch로 안 잡히면 -> handler가 처리

#### 2-2. `async`에서는 거의 동작하지 않는다.(중요)
```kotlin
val handler = CoroutineExceptionHandler {_, throwable ->
    println("caught by handler: $throwable")
}

CoroutineScope(Dispatchers.Default + handler).async {
    error("boom")
}

// 출력
// 아무 것도 출력 안됨
```
##### 왜냐하면
- `async`의 예외는 결과의 일부
- 예외는 `Deferred`안에 저장 됨
- `await()`할 때 터짐
- 즉, "처리되지 않은 예외"가 아니다.

### 3. async 예외는 어디서 처리해야 하나
#### 3-1. 정석: `await()`에서 처리
```kotlin
val deferred = async {
    error("boom")
}

try{
    deferred.await()
}
catch(e: IllegalStateException) {
    println("handled locally")
}
```
이게 코루틴에서 **가장 권장되는 예외 처리 방식**
>async = 값 + 실패를 함께 반환하는 구조
>-> 호출자가 책임 지고 처리

Java의 `Future.get()` / `CompletionException`구조랑 거의 동일한 철학

#### 3-2. async + handler를 믿으면 안 되는 이유
```kotlin
async {
    error("boom")
}

// await 안함
```
##### 이 경우
- 예외는 내부에 남아있다가
- GC 시점이나
- Scope종료 시점에
- 예상 못한 타이밍에 터질 수 있음
그렇기 때문에 async는 반드시 await가 원칙

### 4. try/catch vs CoroutineExceptionHandler
#### 4-1. try/catch가 이긴다(항상 우선)
```kotlin
val handler = CoroutineExceptionHandler {_, _->
    println("handler")
}

launch(handler) {
    try{
        error("boom")
    }
    catch(e: Throwable){
        println("caught locally")
    }
}

// 출력
caught locally
```
##### 규칙
1. 코루틴 내부 try/catch
2. 부모로 전파
3. CoroutineExceptionHandler
handler는 마지막 단계

### 5. coroutineScope / supervisorScope와의 관계
#### 5-1. coroutineScope + launch + handler
```kotlin
val handler = CoroutineExceptionHandler{_, e ->
    println("handler: $e")
}

suspend fun example() = coroutineScope {
    launch(handler) {
        error("boom")
    }
}
```
- launch 실패
- coroutineScope 전체 실패
- handler 호출
- 예외 상위 전파
#### 5-2. supervisorScope에서는?
```kotlin
suspend fun example() = supervisorScope {
    launch {
        error("boom")
    }
    launch {
        delay(500)
        println("still alive")
    }
}
```
- 첫 번째 launch 실패
- 두 번째 launch는 취소되지 않음
- 첫 번째 예외는 부모로 전파
- 처리되지 않으면 handler로 감