### 0. Kotlin 문법의 큰 그림(Java와의 결정적 차이)
Kotlin 코드의 특징 요약
- 세미콜론(;) 없음
- 타입은 뒤에 온다.
- 중괄호보다 "식(Expression)"이 중요
- 파일 단위 함수/변수 가능

#### 1. 변수 선언 문법
```kotlin title:kotlin
val name: String = "SeungMin"
```

```java title:java
String name = "SeungMin"
```

| Java   | Kotlin     |
| ------ | ---------- |
| 타입이 앞  | 타입이 뒤      |
| 변수 = 값 | 변수: 타입 = 값 |
##### 타입추론(매우 중요)
```Kotlin title:kotlin
val name = "SeungMin"
```
- 컴파일러가 "SeungMin" -> String으로 추론
- Java의 var와 다름(Kotlin은 정적 타입 유지)
##### val vs var
```kotlin title:kotlin
val a = 10 //불변(final)
var b = 10 //가변
```

Kotlin 관례
- 기본은 val
- 정말 필요할 때만 var
>Kotlin에서 var가 보이면 "상태가 필요한가?"를 의심해야 함

### 2. 함수 선언 문법
```Kotlin title:kotlin
//가장 기본적인 함수
fun sum(a: Int, b: Int): Int {
    return a + b
}

//Kotlin다운 표현식 함수
fun sum(a: Int, b: Int): Int = a + b
  //{} + return 제거
  //함수 자체가 값을 반환하는 식 = javascript랑 비슷하네?
  
//반환 타입 생략
fun sum(a: Int, b: Int) = a + b
  //컴파일러가 반환 타입을 추론합니다.
  //Java에는 없는 감각.. 이 함수는 값 하나를 계산한다.
```

### 3. if문법(중요)
```java title:java
if (x > 0){
    return "POSITIVE";
}
else {
    return "NEGATIVE";
}
```
```Kotlin title:kotlin
val result = if (x > 0) "POSITIVE" else "NEGATIVE"
  //if는 문(statement)이 아니라 식(expression)
  //반드시 else가 필요(값을 반환해야해서.)
```

### 4. when 문법(switch의 상위 호환)
```Kotlin title:kotilin
fun grade(score: Int): String = 
    when {
        score >= 90 -> "A"
        score >= 80 -> "B"
        else -> "F"
    }
```
##### 특징 정리
 - break 없음
 - 조건식 사용 가능
 - 값 반환 가능
 
 #### Java switch와 가장 큰 차이:
 > Kotlin when은 로직 + 결과 계산을 동시에 함

### 5. 클래스 문법 (가장 낯선 부분)
```kotlin title:"kotlin(일반 클래스)"
class User(
    val id: String,
    val name: String
)
//이 한 줄이 의미하는 것
//  생성자, 필드, getter 자동 생성, 불변 객체
```

```kotlin title:"Data Class"
data class User(
    val id: String,
    val name: String
)
//이 한 줄이 의미하는 것
//  equals, hashcode, toString, copy 자동 생성(Java에서 가장 많이 실수하는 부분을 컴파일러가 담당)
```


---
### 6. 패키지 & 파일 구조
```kotlin title:kotlin
package com.example.demo
//Kotlin 특징
// - 파일 이름 != 클래스 이름
// 한 파일에 여러 클래스 / 함수 가능

// Top-level 함수 가능(클래스 안에 들어있지 않은 함수)
fun hello() = "Hello Kotlin" //Java에서는 불가능한 구조
```
Top-level 함수 가능

[[Day 1.2. 코틀린의 진짜 무기(확장 함수)]]