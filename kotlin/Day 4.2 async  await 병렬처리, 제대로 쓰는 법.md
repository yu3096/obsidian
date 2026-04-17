#### 핵심 결론부터
>`async`는 "**병렬로 돌려도 이득이 있을 때만**" 쓴다.

- 비동기라서 쓰는게 아님
- suspend 함수라서 쓰는게 아님
- 서로 독립적인 작업이고
- 같이 기다려야 할 값이 있을 때

#### 잘못 쓰는 예(아주 흔함)
```kotlin
val user = async {fetchUser()}.await()
val profile = async {fetchProfile(user.id)}.await()
```
##### 문제:
- `fetchProfile`은 `user`가 있어야 함
- **병렬 불가**
- async/await쓴 의미 없음 (오히려 오버헤드)

##### 위의 케이스는 이렇게 쓰는게 정답
```kotlin
val user = fetchUser()
val profile = fetchProfile(user.id)
```

#### async가 진짜 필요한 순간
##### 서로 의존성이 없는 작업들
```kotlin
suspend fun loadScreen(): ScreenData = coroutinScope {
    val userDeferred = async {fetchUser()}
    val noticeDeferred = async {fetchNotices()}
    
    ScreenData(
        user = userDeferred.await(),
        notices = noticeDeferred.await()
    )
}
```
##### 여기서 포인트:
- `fetchUser`, `fetchNotices`는 **서로 독립**
- 둘 다 I/O
- **같이 기다림**
이게 async의 정석

#### Java CompletableFuture와 정확한 대응
##### Java
```java
CompletableFuture<User> userFuture = fetchUserAsync();
CompletableFuture<List<Notice>> noticeFuture = fetchNoticesAsync();

CompletableFuture.allOf(userFuture, noticeFuture).join();
```
##### Kotlin
```kotlin
val user = async{fetchUser()}
val notice = async{fetchNotices()}
```

Kotlin은
- API가 아니라
- **언어 레벨에서 병렬 구조를 표현**

#### Structured Concurrency가 중요한 이유
```kotlin
coroutinScope {
    val a = asnyc{taskA()}
    val b = async{taskB()}
}
```
- `taskA`실패 => `taskB` 자동 취소
- 부모 스코프가 책임짐
- 리소스 누수 없음
Java는 직접 다 처리하던 것
Kotlin은 기본값으로 제공

#### Java 개발자의 잦은 오해
>비동기니까 다 async로 감싸야지

##### Kotlin적 사고
>suspend = 비동기 가능
>async     = 병렬 계산

#### 여기까지 핵심 요약
>1. async는 병렬일 때만 의미가 있다.
>2. 의존성(데이터에 대한 dependency) 있으면 async 사용 안함
>3. suspend 함수 안에서는 순차 코드가 기본
>4. coroutineScope안에서 async/await 쓴다.

[[Day 4.3 Coroutines 예외 전파, 취소, supervisorScope]]