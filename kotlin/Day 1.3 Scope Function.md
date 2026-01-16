### 0. Scope Function이란?
#### 한 줄 정의
> 객체를 잠깐 특정 스코프에 넣고, 그 안에서 처리하는 문법
   대표 5개:
   let / run / apply / also / with
   많아 보이지만, 기준은 단 2개.

### 1. Scope Function의 핵심 기준(이것만이라도 기억하기)
##### 기준 1. this vs it
##### 기준2: 반환값이 뭐냐

| 함수    | 객체접근 | 반환값   |
| ----- | ---- | ----- |
| let   | it   | 람다 결과 |
| run   | this | 람다 결과 |
| apply | this | 자기 자신 |
| also  | it   | 자기 자신 |
| with  | this | 람다 결과 |
아래 예제로 감각 잡기

### 2. let - "null 안전 처리"
**가장 많이 쓰는 용도**
```kotlin
val length = name?.let {
    it.length
}
//의미
//  name이 null이 아니면
//  it으로 받아서
//  결과를 반환
```

java식으로 풀면
```java
if(name != null){
    length = name.length();
}
```
**Null Safety + Scope 제한**

### 3. run - "객체 기준 계산"
```kotlin
val area = rectangle.run {
    width * height
}
//특징
//  this 사용
//  계산 결과 반환
```
**객체 기반 expression**

### 4. apply - "객체 설정(Builder 대체)"
```kotlin
val user = User().apply{
    name = "SeungMin"
    age = 30
}
//핵심
//  this 사용
//  자기 자신을 반환
//  Java Builder 패턴 제거
```

### 5. also - "부가 작업(logging 등)"
```kotlin
var result = calculate()
             .also{println("result = $it")}
             
//핵심
//  it 사용
//  자기 자신 반환
//  사이드 작업용
//  로깅, 디버깅에 최적
```

### 6. with - "외부 객체 묶기"
```kotlin
val result = with(user) {
    "$name ($age)"
}

//특징
//  확장 함수 아님
//  this 사용
//  결과 반환
//  여러 번 같은 객체 쓸 때
```

### 7. Java 개발자가 자주 하는 실수
```kotlin
user.apply {
    saveToDatebase()
}

//Scope Function은 가독성을 위한 도구이지 책임이동 도구가 아님!
```


### 8. 핵심요약
```kotlin
let   -> null 처리, it, 결과 반환
run   -> 계산, this, 결과 반환
apply -> 설정, this, 자기 자신
also  -> 부가 작업, it, 자기 자신
```

#### 추가) 왜 어떤 Scope Function은 this, 어떤 건 it일까?
> 이 함수가 어떤 의도로 만들어졌는가의차이.
> Kotlin팀은 의도를 문법으로 드러내기 위해 고의적으로 나눔.

##### this를 쓰는 쪽의 철학
대표: `run`, `apply`, `with`
```kotlin
user.apply {
    name = "SeungMin"
    age = 30
}
```
> 이 코드를 자연어로 읽으면
> user를 기준으로 이 블록 안에서는 user 자신이 주어다.
  그렇기 때문에, 
  `user.name`이 아니고,
  `name`
  `this.name`(필요시 명시)
>> 객체 중심(Oject-centric의 사고
>
>**Kotlin의 의도**
>이 블록 안에서는 이 객체가 세상의 중심이다.

##### it를 쓰는 쪽의 철학
대표: let, also
```kotlin
user?.let {
    println(it.name)
}
```
> 이 코드를 자연어로 읽으면
> user가 있으면 그걸 it이라는 임시 이름으로 넘겨서 처리한다.
> 객체는 주인공이 아니고, 잠깐 전달받은 값일 뿐임.
> > 값 전달(Value-centric) 사고

##### 왜 반환값이 같은데도 다를까?
**apply vs also 비교**
```kotlin
user.apply {
    name = "A"
}

user.also {
    it.name = "A"
}

```
> 두 소스 모두 자기 자신을 반환.
> 그런데도 나눈 이유는 **의도 차이**

##### apply의 의도
> 이 객체를 설정(configure) 한다.
> 그래서 this
> Builder 대체용

##### also의 의도
> 이 객체에 대해 **부가 행동을 덧붙인다.**
> 그래서 it
> 로깅, 디버깅, 추적용


```kotlin
user.also {log(it)}
    .also {metrics(it)}
```
객체는 계속 흐름 속에 있음.
중심은 흐름, 객체는 전달 값

##### Java 개발자로써 이렇게 봐보기
`this` -> 이 객체 안에서 작업한다.
`it` -> 이 객체를 잠깐 전달받았다.
