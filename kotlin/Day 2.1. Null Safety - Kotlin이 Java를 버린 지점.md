### 1. Kotlin의 가장 중요한 규칙 (단 하나)
> Null은 "값"이 아니라 "타입의 일부"다.

```java title:java
String name = null; //가능
```

```kotlin title:kotlin
val name: String = null //컴파일 에러
```

Kotlin에서는 **"이 변수는 null이 될 수 있는가?"**를 타입으로 선언해야함. 
(DB Nullable로 생각하면 되겠네)

### 2. Nullable 타입 문법
```kotlin
val name: String? = null
```
`String` = null 불가
`String?` = null 가능
`?` 하나가 언어 레벨 계약(contract).

### 3. 왜 이렇게까지 강제할까?
java의 문제: 
```java title:java
user.getName().length(); //런타임 NPE
```

Kotlin의 선택:
> "null 가능성을 컴파일 시점에 드러내자"
> 런타임버그에서 컴파일 에러로 이동

### 4. Safe Call `?.`
```kotlin
val length = name?.length
```
의미를 풀면
* name이 null이면 결과도 null
* name이 null이 아니면 length 계산
(javascript의 옵셔널 체이닝이랑 비슷하네)

반환 타입은?
```kotlin
Int?
```
> null 전파(null propagation)

### 5. Elvis 연산자 `?:`
```kotlin
val length = name?.length ?: 0
```
> name의 길이를 쓰되, 없으면 0
> 
> java의 삼항연산자 + null 체크 제거

### 6. 절대 남용하면 안되는 `!!`
```kotlin
val length = name!!.length
```
>여기서는 null이 아니라고 **내가 보증한다.**
>틀리면 즉시 NPE

Kotlin 철항 상 `!!`는 최후의 수단이며, 보이면 코드 리뷰 대상

### 7. Smart Cast(Java에는 없는 개념)
```kotlin
fun printLength(value: String?){
    if(value != null){
        println(value.length)
    }
}
```
>컴파일러가
>if 안에서는 null이 아님을 보장
>`String`으로 취급
```Kotlin title:"Smart Cast가 체감되는 순간(Kotlin)"
if(value is String){
    value.length //캐스팅이 따로 필요 없음.
}
```
```Java title:"Smart Cast가 체감되는 순간(Java)"
if(value instaceof String){
    ((String) value).length();
}
```

##### Java 였다면,
```java
if(value != null){
    value.length(); //컴파일러는 여전히 의심
}
```

### 8. lateinit vs by lazy(상태 초기화)
##### `lateinit`
```kotlin
lateinit var service: UserService
```
* var만 가능
* null이 아님 보장
* 초기화 전 접근 시 예외
* DI, 테스트용

`by lazy`
``` kotlin
val config by lazy{ loadConfig() }
```
- val 가능
- 최초 접근 시 초기화
- thread-safe 기본 제공
- 비용이 큰 객체에 적합

### 9. Java 연동 시 가장 위험한것: platform Type
```kotlin title:kotlin
val name = javaUser.name
```
타입: String!
>null일 수도 있고 아닐 수도 있음.
>Kotlin이 책임 안짐.

**실무 규칙**
* Java API 경계에서 즉시 Nullable 처리
```kotlin title:kotlin
val name: String? = javaUser.name
```

[[Day 2.2. Sealed class - null 대신 의미 있는 결과]]