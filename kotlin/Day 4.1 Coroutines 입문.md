### 코루틴을 한 문장으로 설명하면
>"중단(suspend)할 수 있는 함수:

멀티쓰레드, 이벤트루프, 비동기...

#### Java 비동기 사고부터 짚고 넘어가기
```Java title:"Java(CompletableFuture)"
CompletableFuture<User> feture = 
    fetchUserAsync()
        .thenApply(user -> enrich(user))
        .thenCompose(user -> saveAsync(user))
```
문제점:
- 흐름이 콜백체인
- 예외 전파가 복잡
- "값을 계산한다"는 느낌이 약함

#### Kotlin의 질문
>"왜 비동기라고 해서 **코드를 다르게 써야해?**"

그래서 나온 개념이 `suspend`

#### `suspend` 함수란?
```kotlin
suspend fun fetchUser(): User
```
이 함수의 의미:
>"User를 반환한다. **다만, 중간에 잠깐 멈출 수 있다.**"

##### 중요한 점
* 반환 타입은 여전히 `User`
* `Future<User>` X
* `Defrerred<User>` X
**비동기 흔적이 type에 남지 않음**

#### 이게 혁명적인 이유
```kotlin
suspend fun process(): User {
    val user = fetchUser() //여기서 멈출 수 있음
    val saved = save(user) //다시 멈출 수 있음
    return saved
}
```
##### 읽는 느낌
> "순차코드"

##### 실제 동작
- 스레드 블로킹 X
- 필요할 때만 중단/재개
**동기 코드처럼 읽히는 비동기**

#### `launch` vs `async` (핵심차이)
##### `launch`
```kotlin title:kotlin(launch)
launch {
    doSomething()
}
```
- 결과 없음(`Job`)
- "불을 붙인다"
- 로그, 이벤트, fire-and-forget
------
##### `async`
```kotlin title:"kotlin(async)"
val deferred = async {
    fetchUser()
}
```
- 결과 있음 (`Deferred<T>`)
- 나중에 `await()`로 값 받음
- 병렬 계산

#### Java 개발자가 가장 많이 하는 오해
```kotlin
launch{
    val user = fetchUser()
}
```
그리고 밖에서
```kotlin
println(user) // X
```
`launch`는 **값을 돌려주지 않음**

Kotlin식 질문:
>"이 코드는 **값을 계산하는가**, 
>아니면 **작업을 실행하는가**?"

#### Structured Concurrency (아주 중요)
kotlin의 철학:
>"**비동기는 반드시 부모-자식 관계를 가진다**."

```kotlin
coroutinScope{
    launch{taskA()}
    launch{taskB()}
}
```
- 부모 스코프가 끝나면
- 자식 코루틴도 함께 정리
**유령 코루틴 없음**

#### 여기까지 핵심 요약
>1. suspend = 중단 가능한 함수
>2. 반환 타입은 여전히 값이다.
>3. launch = 작업, async = 값
>4. 비동기도 구조를 가진다.

[[Day 4.2 async  await 병렬처리, 제대로 쓰는 법]]