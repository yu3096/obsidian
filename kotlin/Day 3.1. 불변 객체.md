## 불변 객체란 무엇인가?(왜 중요한가)
### Java에서 익숙한 방식 (가변 객체)
```java
order.setStatus(PAID);
order.setPaidAt(now);
```
##### 특징:
- 객체 자체가 변함
- 누가 언제 바꿨는지 추적 어려움

### Kotlin이 선호하는 방식
>객체는 바꾸지 않는다.
>대신 "새 객체"를 만든다.

이게 Kotlin이 `val`을 기본으로 두는 이유와 연결된다.

### data class + val 기본 형태
```Kotlin
data class Order(
    val id: String,
    val status: OrderStatus,
    val paidAt: LocalDateTime?
)
```
##### 포인트:
- 모든 프로퍼티 `val`
- 한 번 생성되면 **절대 안 변함**

##### 여기서 질문
>그러면 상태 변경은 어떻게해...?

### `copy()` - 상태 변경에 대한 Kotlin의 해답
```kotlin
val paidOrder = order.copy(
    status = OrderStatus.PAID,
    paidAt = LocalDateTime.now()
)
```
##### 이 코드의 의미를 풀면
- 기존 `order`는 그대로
- 일부 필드만 바꾼 **새 Order 생성**
- 나머지 자동 복사
**불변 + 변경의 양립**

### Java Builder 패턴과의 비교 (why?)
```java title:java(Builder Pattern)
Order newOrder = Order.builder()
                      .id(order.getId())
                      .status(PAID)
                      .paidAt(now)
                      .build();
```
##### 문제:
- 장황함
- 필드 추가 시 builder 수정 필요
- 실수의 여지가 많음.

```kotlin
order.copy(status = PAUD)
```
**컴파일러가 다 처리**

### sealed class와 결합하면 진짜 힘이 나옴
```kotlin
sealed class OrderState {
    object Created: OrderState()
    data class Paid(val paidAt: LocalDateTime): OrderState()
    object Canceled: OrderState()
}
```

```Kotlin
val paidOrder = order.copy(
    state = OrderState.Paid(LocalDateTime.now())
)
```
상태 + 데이터 + 의미가 모두 타입에 드러남

### Java 개발자가 자주 하는 Anti-pattern
```Kotlin
data class Order(
    var status: OrderStatus
)
```
##### 문제가 되는 이유:
- 불변성 붕괴
- copy의 의미 반감
- kotlin 장점 절반 상실
##### 실무 규칙
>data class에서 var가 보이면, 이유를 증명해야 한다.

### 그럼 가변 객체를 사용하는 경우는?
##### 허용되는 경우
- DTO(직렬화 / 역직렬화)
- JPA Entity(어쩔 수 없음)
- 성능이 극단적으로 중요한 핫 루프
**도메인 모델은 기본 불변**

### 요약
> [!NOTE]
    > 1. Kotlin은 불변 객체를 기본으로 한다.
    > 2. 변경은 copy()로 표현한다.
    > 3. data class + val이 기본 형태
    > 4. sealed class와 결합하면 상태 전이가 안전해진다.
    
### 추가
##### 컴파일 에러
```kotlin
val order = ...
val order = order.copy(status = PAID) // redeclaration 컴파일 에러
```

##### 흔하고 정상
```kotlin
val order = ...
val updatedOrder = order.copy(status = PAID) //아주 흔하고 정상
```

##### 함수 체인 / 새 스코프
```kotlin
fun pay(order: Order): Order = 
    order.copy(status = PAID)
```

##### shadowing(가려쓰기)가 허용되는 경우
```kotlin
fun process(order: Order){
    val order = order.copy(status = PAID) //새 스코프
}
```

##### Kotlin에서 이 패턴이 중요한 이유
- 상태 변경이 표현식이 됨
- 함수형 스타일과 잘 맞음
- 테스트/동시성에 안정
```kotlin
fun pay(order: Order): Order = 
    order.copy(
        status = PAID,
        paidAt = LocalDateTime.now()
    )
```