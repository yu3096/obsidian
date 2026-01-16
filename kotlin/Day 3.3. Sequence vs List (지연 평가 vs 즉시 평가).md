>map/filter를 많이 붙여도 어떤 코드는 빠르고 어떤 코드는 느린걸까?
### List 컬렉션 체인에서 실제로 일어나는 일
#### 기본 컬렉션(List)의 동작: 즉시 평가(Eager)
```kotlin
val result = 
    users.filter{it.isActive} //여기에서 새 List 하나 만들고(중간 결과)
         .map{it.name}        //여기에서 또 새 List 만듬(최종 결과)
```

#### Sequence의 동작: 지연 평가(Lazy)
```kotlin
val result = 
    users.asSequence()
         .filter{it.isActive}
         .map{it.name}
         .toList()
```
##### 여기서는
- 중간 List를 만들지 않고
- 요소를 **하나씩 흘려보내며** filter -> map을 적용
- 마지막 `toList()`에서 한 번만 결과를 모음
유사 파이프라인

### Sequence가 유리한 경우
- 컬렉션이 큼(대량 데이터)
- map/filter 체인이 길다
- 중간 결과가 필요 없다
- 특히 `take(n)` 같이 **앞에서 끊을 때** 엄청 유리
```kotlin
val top10Names =
    users.asSequence()
         .filter{it.isActive}
         .map{it.name}
         .take(10)
         .toList()
```
##### List 방식이면
- filter / map을 전부 다 수행한 뒤
- 마지막에 10개만 자름

##### Sequence는
- 10개를 얻는 순간 바로 멈춤

### Sequence가 손해인 경우
- 컬렉션이 작을 때
- 체인이 짧을 때(1~2단)
- 람다 호출 오버헤드가 더 큼
- 결과를 어차피 전부 다 만들어야 함

### 핵심 요약
> [!NOTE]
    > List 체인 = 즉시 평가 = 중간 List 생성
    > Sequence = 지연 평가 = 요소 단위 파이프라인
    > 큰 데이터/긴 체인/조기 종료(take)면 sequecne 유리
    
### 추가
>Sequence는 요소 하나마다 람다를 여러 번 '나눠서' 호출
>List는 각 단계에서 람다를 '묶어서' 호출

#### "람다 호출 오버헤드"란?
람다는 그냥 코드로 보이지만 실제로는
- 함수 객체(또는 인라인된 호출)
- 호출 스택
- 파라미터 전달
- 분기 처리
같은 **호출 비용**이 존재.
한 번은 미미하지만, 수천 ~ 수만 번 반복되면 체감됨


### List체인 vs Sequence 체인의 호출 방식 차이
#### List (즉시 평가)
```kotlin
users.filter{it.isActive}
     .map{it.name}
```
- `filter`: 
	- for-loop 한 번
	- 람다 isActive N번 호출
- `map`: 
	- for-loop 한 번
	- 람다 name N번 호출

#### Sequence(지연 평가)
```kotlin
users.asSequence()
     .filter{it.isActive}
     .map{it.name}
     .toList()
```
