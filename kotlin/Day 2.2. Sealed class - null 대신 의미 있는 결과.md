### 1. 먼저 문제부터 짚기(Java / 기존 방식)
```java
User user = findUser(id); //실패하면 null
if(user == null){
    //실패처리
}
```
문제점:
- 왜 null인지 알 수 없음
- 성공 / 실패가 **타입에 드러나지 않음**
- 호출자가 매번 추측해야 함

##### Kotlin의 질문
>뭐야 성공할 수도 있고 실패 할 수도 있는거잖아.
>아 씨 귀찮으니까 둘 다 타입으로 표현해!!!
>해서 나온게 sealed class

### 2. sealed class란?
##### 한 줄 정의
>가능한 하위 타입을 컴파일 시점에 "봉인(seal)"한 클래스

즉, 이 결과는 이 경우들 밖에 없다.
컴파일러가 **모든 경우를 알고 있음**

### 3. 가장 기본적인 Result 모델
```kotlin
sealed class UserResult {
    data class Success(val user: User) : UserResult()
    data class NotFound(val id: String) : UserResult()
    data class Error(val message: String) : UserResult()
}
```
* null 필요 없음
* 실패 이유 명확
* 경우의 수가 타입으로 고정

```kotlin
fun handle(result: UserResult) {
    when (result) {
        is UserResult.Success -> println(result.user)
        is UserResult.NotFound -> println("없음: ${result.id}")
        is UserResult.Error -> println(result.message)
    }
}
```
>else 없음
>모든 경우를 다 처리
>하나 빠지면 컴파일 에러 [^1]
>이걸 Exhaustive when

### 4. 왜 Enum이 아니라 sealed class?
##### enum의 한계
```kotlin
enum class Status {
    SUCCESS, ERROR
}
```
- 경우마다 다른 데이터 못 가짐
 ```java
 enum LoginResult {
     SUCCESS(User user),
     INVALID_PASSWORD(int retryCount),
     LOCKED(LocalDateTime unlockAt)
 }
 // 이거 안됨
 ```

- 경우별 의미 표현이 약함
##### sealed class의 장점
- 각 경우마다 **다른 데이터**
- 계층 구조 가능
- 실제 도메인 표현에 적합
- **상태(state)**보다 **결과(result)**에 잘 어울림



### 5. Kotlin 표준 `Result<T>`를 사용하는 방법
```kotlin
fun load(): Result<User>
```
- 단순 성공 / 실패
- 예외 래핑용
- 실무에서는 **도메인 실패 이유**가 중요하므로 직접 sealed class 추천

### 6. Java 개발자가 흔히 하는 Anti-pattern
```kotlin
fun findUser(id: String): User?
```
>왜 null인지 모름
>호출자가 추측해야함
  null = 정보 부족
  sealed class = 정보 명시

Kotlin은 모호함(null)을 줄이고 의미를 타입으로 끌어올린다.

### 여기까지 공부하고 ?가 생긴 내용
[^1]: 그렇다면, sealed class를 사용해서 처리 한 경우에, 유지보수를 하다 케이스가 하나 더 늘면 관련 소스를 전부 수정해야되네?

### 추가
##### sealed class를 쓸 때 중요한 질문 2개.
1) 이 타입을 누가 쓰는가?
- **나(내 모듈/내 팀)만 쓴다** - 깨져도 고치면 됨 -> sealed 적극 추천
- **다른 팀/외부 사용자도 쓴다** - 깨지면 큰일 - sealed "확장 전략"이 필요

2) 케이스가 앞으로 늘어날 가능성이 큰가?
- **거의 안늘어남(정말 고정)** - enum 또는 sealed class 둘 다 OK
- 늘어날 가능성이 있음 - "서브 클래스 추가"가 깨짐을 유발하므로 설계를 조심
---
### 실전 규칙 5개 (이대로 쓰면 큰 사고가 거의 안남)
##### 규칙 1) 내부 도메인 결과는 sealed + exhausive when을 기본으로
- 장점: 새 케이스 추가 시 "놓친 곳"을 컴파일러가 다 찾아줌
- 추천 상황: UseCase 결과, 비즈니스 규칙 결과, 파싱 결과 등

##### 규칙 2) 외부에 노출하는 타입은 "늘어나는 축"을 subclass로 만들지 말기
* 늘어나는 건 보통 **에러 종류 / 이벤트 종류**
* 이걸 subclass로 계속 늘리면 외부 사용자의 `when`이 다 깨질 수 있음.
* 대신, "데이터로 확장": `Error(code, message, details)` 처럼.
##### 규칙3) 공개 API는 sealed의 "골격"만 고정하고, 디테일은 데이터로
- `Success` / `Failure` 정도만 sealed로 고정
- Failure의 세부 사유는 code로 확장

##### 규칙 4) 내부 모델과 외부 모델은 분리하면 가장 안전
- 내부: 풍부한 sealed(케이스 마음껏)
- 외부: 안정적인 DTO / 인터페이스
- 경계에서 mapper로 변환 (한 곳에서만 when)

##### 규칙 5) "기본 처리로 뭉개도 되는 곳" 에서만 else를 허용
- else가 있으면 새 케이스를 놓쳐도 컴파일이 통과함 (안정성 하락)
- 그래서 else는 "진짜 기본 처리로 충분한 곳"에서만

[[Day 3.1. 불변 객체]]