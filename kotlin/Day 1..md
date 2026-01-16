### Step 1. Kotlin의 핵심 철학(개념)
1. Kotlin은 Statement 언어가 아니라 Expression언어
```Java
if (condition) {
    return a;
}
else {
    return b;
}
```
```kotlin
val result = if (condition) a else b
```

2. val이 기본, var는 최후의 수단```
```kotlin
val name = "SeungMin"
// name = "Yu" //Compile Error
```
=> Why?
- 불변성 = 추론 가능성
- 멀티스레드 / 코루틴 시대의 기본 값

=> Java에서의 대응:
- final이 Kotlin의 val
- 차이: Kotlin은 기본이 final

3. 클래스는 데이터 표현이 기본(data Class)
```java
class User {
    private final String id;
    private final String name;
}
```
``` Kotlin
data class User {
    val id: String;
    val name: String;
}
```
=> Why?
- Kotlin은 객체를 행동보다 데이터로 보는 경향.
- equals/hashCode는 실수의 온상 -> 컴파일러에 위임

### Step 2. 예제코드 (Java -> Kotlin 사고 전환)
예제: 사용자 상태 계산
```kotlin
fun userStatus(age: Int): String =
    when {
        age < 0 -> "INVALID"
        age < 20 -> "MINOR"
        else -> "ADULT"
    }
```
포인트
- when은 **expression**
- return 없음
- 중괄호보다 **의미 단위**가 중요

### Step 3. 흔한 실수(Anti-pattern)
```kotlin
// Java 스타일의 Kotlin
fun getStatus(age: Int): String {
    var result = "";
    if (age < 20) {
        result = "MINOR"
    }
    else {
        result = "ADULT"
    }
    
    return result
}

// Kotlin스러운 코드
fun getStatus(age: Int): String = 
    if(age < 20) "MINOR" else "ADULT"
```
체크 질문
- 이 함수는 상태를 바꾸는가? 아니면 값을 계산하는가?

### Step 4. 실습(직접 해보는 부분)
아래를 Java 코드를 Kotlin 스럽게 바꿔보세요.
```java
public class Order {
    private final String id;
    private final int price;
    
    public Order(String id, int price){
        this.id = id;
        this.price = price;
    }
    
    public boolean isExpensive(){
        if(price > 10000){
            return true;
        }
        return false;
    }
}
```
요구사항
- data class 사용
- isExpensive()는 expression 스타일
- var 사용 금지