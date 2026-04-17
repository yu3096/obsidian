### Java에서 익숙 한 사고부터 짚기
Java에서 List를 가공할 때 보통 아래 2가지 방법을 사용함
##### 1) 전통적인 for-loop
```java
List<String> result = new ArrayList<>();
for(User user: users){
    if(user.isActive()){
        result.add(user.getName());
    }
}
```

##### 2) Stream(Java 8+)
```java
List<String> result = 
    users.stream()
         .filter(User::isActive)
         .map(User::getName)
         .toList();
```

##### Java에서 Stream은?:
- 필요하면 쓰는 것
- 여전히 loop 사고가 기본

### Kotlin의 기본 전제
>컬렉션은 '변환'한다.
>순회(loop)는 구현 디테일이다.

그래서 Kotlin에서는 **Collection API가 기본 스타일**이다.

### 가장 기본적인 3총사
##### map - 형태를 바꾼다
```kotlin
val names = users.map{it.name}
```

##### filter - 조건으로 거른다
```kotlin
val activeUsers = users.filter{it.isActive}
```
##### map + filter 조합
```kotlin
val names = users.filter{it.isActive}
                 .map{it.name}
```

### Java Stream과의 결정적 차이
##### Java
```java
users.stream()...
```

##### Kotlin
```kotlin
users.map {...}
```
##### 차이점
- Stream 객체 X
- stream() 호출 X
- **컬렉션 자체가 바로 함수 제공**

스트림으로 바꾼다 라는 개념이 없다.

### Kotlin코드에서 자주 보게 되는 패턴
##### find - 하나 찾기
```kotlin
val user = users.find{it.id == targetId}
```
반환타입: `User?`
##### any / all / none
```kotlin
users.any { it.isActive }
users.all { it.isActive }
users.none {it.isActive}
```

### Java 개발자가 자주 하는 Anti-pattern
```kotlin
val result = mutableListOf<String>()
for(user in users){
    if(user.isActive) {
        result.add(user.name)
    }
}
```
##### Kotlin에서는
- loop 자체가 나쁜 건 아니지만,
- "변환" 상황에서는 map/filter가 정답

### 불변 컬렉션 사고(중요)
```kotlin
val list = listOf(1,2,3)
// list.add(4) // 에러
```
##### Kotlin 기본
- `List`는 읽기 전용
- 변경하려면 `MutableList`
**불변이 기본, 변경은 의식적으로**

### 핵심 요약
> [!NOTE]
    > 1. 컬렉션은 변환한다.
    > 2. map / filter가 기본 언어
    > 3. Stream은 필요 없다.
    > 4. 불변 컬렉션이 기본이다.
    
### 추가
#### Java에서 `()`를 쓰는 이유
```java title:"java(Stream)"
users.stream()
     .filter(u -> u.isActive())
     .map(u -> u.getName())
```
##### 여기서,
- `filter(...)`안에 들어 가는 건
- 사실상 **함수 객체(Predicate, Function)**

##### Java는
- 함수가 **1급 시민이 아님**
- 그래서 결국 "람다"도 **객체**
- 객체를 전달 하니까 `()`를 사용

#### Kotlin은 함수가 "진짜 값"이다.
##### Kotlin에서는
- 함수 = 값
- 람다 = 값
- 함수 타입이 언어에 내장
```kotlin
(val) (User) -> String
```
그래서 Kotlin은 이렇게 말합니다.
>"여기엔 **코드 블록 자체**를 넘긴다"

그래서 `{}`를 사용

#### `{}`는 "실행 코드"라는 명확한 시각적 신호
```kotlin
user.filter {it.isActive}
```
이걸 읽으면
>users를 filter하는데, **이 조건 코드**로 걸러라

반면 `()`는
```kotlin
foo(bar)
```
>bar라는 **값**을 넘긴다.

Kotlin은 이 차이를 **문법으로 분리**

#### trailing lambda 문법이 핵심
kotlin의 결정적 설계
```kotlin
fun filter(predicate: (T) -> Boolean)
```
이렇게 생긴 함수에서 **마지막 인자가 람다**라면
```kotlin
filter {...} 
```
로 쓸 수 있게 허용
##### 즉, 
```kotlin
filter({it.isActive}) //원래 형태
filter{it.isActive}   //축약형태
```

#### 그래서 `()` + `{}`가 같이 나올 수 있음.
```kotlin
users.mapIndexed(0) {index, user -> 
    "$index: ${user.name}"
}
```
- `()` -> 일반 값 인자
- `{}` -> 람다 인자
**역할이 다르기 때문에 같이 공존 가능**

#### Java였다면 불가능한 가독성
##### Kotlin
```kotlin
users.filter{it.isActive}
     .map{it.name}
```

#### Java
```java
users.stream()
     .filter(user -> user.isActive())
     .map(user -> user.getName())
```

kotlin쪽이 노이즈도 적고, 흐름이 눈에 바로 들어옴

[[Day 3.3. Sequence vs List (지연 평가 vs 즉시 평가)]]