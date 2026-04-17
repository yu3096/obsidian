**코틀린의 진짜 무기는 바로 확장 함수.**
>Extension Function(확장 함수)
> 여기서부터 Kotlin의 진가가 발휘됨

### 0. 확장 함수란?
기존 클래스에 함수를 '추가한 것처럼' 보이게 하는 문법
* 상속도 아니고, 실제로 클래스가 바뀌는 것도 아님

### 1. Java에서 하던 방식 떠올리기
```java title:java ln:false
// java에서 흔한 코드
StringUtils.isEmpty(str);
  //문제점
  //  항상 Utils 클래스
  //  읽을 때 "주어-동사"가 어색함
  //  자동 완성도 불편
```

### 2. Kotlin 확장 함수 문법
```kotlin title:"kotlin(기본 형태)"
fun String.isEmptyOrBlank(): Boolean {
    return this.isEmpty() || this.isBlank()
}

// 아래와 같이 사용
val result = "    ".isEmptyOrBlank()
  // 마치 String의 멤버 함수처럼 보임
```

### 3. 문법을 아주 정확히 뜯어보기
```kotlin
fun String.isEmptyOrBlank(): Boolean {
    return this.isEmpty() || this.isBlank()
}
```

| 부분             | 의미                 |
| -------------- | ------------------ |
| fun            | 함수 선언              |
| String         | 확장 대상 타입(receiver) |
| .              | 이 타입에 붙인다.         |
| isEmptyOrBlank | 함수 이름              |
| this           | 호출한 객체             |

### 4. 이걸 왜 만들었을까?
##### 설계 의도
> 유틸 함수는 객체의 행동처럼 읽혀야 한다.

##### 비교해보기
`StringUtils.isEmptyOrBlank(str) //Java 스타일`
`str.isEmptyOrBlank() //Kotlin 스타일`
읽는 순서가 자연어와 같음!

### 5. Top-level 함수와의 관계(중요)
확장 함수의 정체는 사실 아래와 같음
> 확장 함수 = Top-level 함수 + 문법적인 설탕
> ```Kotlin
> fun String.foo() {...}
> fun foo(receiver: String) {...}
> // 객체에 메서드를 추가한게 아님.
> //정적 바인딩
> ```

### 6. Java 개발자가 가장 많이 하는 오해 포인트
##### 오해 1: 오버라이딩 된다?
```kotlin
open class A {
    fun hello() = "A" //멤버함수
}

fun A.hello() = "EXT" //확장함수

val a = A();
a.hello() // 결과는 "A"

//이유
//  멤버 함수 > 확장함수
//  확장 함수는 정적 결정
```

### 7. 언제 쓰는게 Kotlin스러운가?
##### 확장 함수가 어울리는 경우
> - 변환 로직
> - 검증
> - 포맷팅
> - 도메인 규칙 보조

>`fun Int.isAdult() = this >= 20`

##### 쓰면 안 되는 경우
> - 상태 변경
> - 외부 시스템 접근
> - 핵심 비즈니스 로직

```Kotlin title:"Kotlin 나쁜 예"
fun User.saveToDatabase(){...}
```

### 요약
> 1. 확장 함수는 실제로 클래스를 바꾸지 않는다.
> 2. Top-level 함수의 문법적 확장이다.
> 3. 읽기 좋은 코드를 만드는 게 목적이다.
> 4. 상태/부작용 로직에는 쓰지 않는다.

[[Day 1.3 Scope Function]]