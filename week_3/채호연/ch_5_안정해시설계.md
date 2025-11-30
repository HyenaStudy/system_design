# Chapter 5. 안정 해시 설계

## I. 왜 안정 해싱이 필요한가?: Rehashing 시 Modulo 연산 기반 해싱의 한계를 중심으로
### 1. 가장 널리 활용되는 해싱 : Modulo 연산 기반 해싱
```java
serverIndex = hashFunction(key) % numberOfBucket
```
- 상기한 예제는 가장 널리 쓰이는 해싱 기법 중 하나인 '모듈러 연산 활용 해싱'.
- 해시 테이블에 활용할 경우, 각 엔트리가 어떤 버킷에 할당되어야 하는지 결정하는데에 활용할 수 있음.
- 각 버킷을 '캐시 서버'로 생각할 경우, *캐싱할 데이터를 어느 캐시 서버에 저장할 것인가?* 와 같은 문제 해결에 활용할 수 있음.
- 모듈러 기반의 해싱이 잘 작동하기 위해선 두 가지 조건이 필요하다.
  - 1) 서버 풀(버킷의 수)가 고정되어 있을 것
  - 2) 모듈러 연산 대상인 '키 값'의 분포가 균등함에 가까울 것. -> 그렇지 않을 경우, Hash Collision 도래 시점이 더 빨라짐.

### 2. Rehashing Problem : Bucket 감소 시 발생하는 문제를 중심으로

기존에 버킷 별로 저장된 엔트리의 분배가 지나치게 불균등하거나, 특정 버킷의 가용성에 문제가 생겨 엔트리를 다시 배분하는 행위를 Rehashing 이라고 한다.

캐시 서버와 캐시 데이터가 다음의 도식과 같이 존재한다고 해보자.

<img width="544" height="234" alt="Screenshot 2025-11-30 at 4 29 41 PM" src="https://github.com/user-attachments/assets/4a8e8daf-4e57-446c-9e50-e2e4b84e3df7" />

만약 이 상황에서 1번 캐시 서버(`server 1`)에 장애가 발생할 경우, 다음과 같은 상황이 발생한다.
- 1) Rehashing 수행 : `server 1` 에 저장된 캐시 데이터에 대한 접근 가능성을 보장하기 위해 Rehashing 을 하여 다른 캐시 서버로 옮겨 저장한다.
- 2) 필연적인 Cache Miss 발생 : 서버 1의 장애가 복구되지 않았음에도, 여전히 모듈러 해싱의 결과 중 `1` 이라는 serverIndex 가 도출가능하기 때문에, Cache Miss 가 발생한다.

<img width="556" height="294" alt="Screenshot 2025-11-30 at 4 55 11 PM" src="https://github.com/user-attachments/assets/d7cc8f80-9d9e-4327-bd30-53b9208990c1" />


## II. 안정 해싱: 정의와 작동 방식

### 1. 정의
> In computer science, **consistent hashing** is a special kind of hashing technique such that when a hash table is resized,
>  only **n/m** keys need to be remapped on average where _n_ is the number of keys and _m_ is the number of slots.
> 
> <small>Consistent Hashing, Wikipedia</small>

- 해시 테이블의 크기가 재조정될 때, 기존의 모듈러 해싱과는 다르게 평균적으로 'n/m' 개의 키만 재배치되도록 하는 특수한 해싱 기법.
  - _n_ : 키의 수
  - _m_ : 버킷의 수
- 특유의 작동 방식 덕분에, 리해싱으로 인해 키의 재배치가 발생하더라도 앞서 살펴본 Caceh Miss 와 같은 문제가 발생하지 않도록 할 수 있음.

### 2. 작동 방식

**핵심 구성 요소 - 해시 링(Hash Ring)**

키가 존재할 수 있는 범위를 선형으로 배열한 후, 이를 환형으로 구부려 시작과 끝이 맞닿도록 구성한 키 공간.
<img width="600" height="85" alt="Screenshot 2025-11-30 at 5 23 09 PM" src="https://github.com/user-attachments/assets/10c90366-0bc3-4390-9fd1-20395e404b4f" />

<img width="229" height="262" alt="Screenshot 2025-11-30 at 5 23 23 PM" src="https://github.com/user-attachments/assets/3921694a-209e-468c-85b9-57315bb9b06a" />

**기본적인 작동 절차 - MIT가 제안한 버전**
1. 서버(버킷)과 엔트리의 키를 '균등 분포(Uniform Distribution)' 해시 함수를 사용해 해시 링에 배치한다.
2. 키의 위치에서 링을 시계 방향으로 탐색하다 만나는 '최초의 서버'가 키가 저장될 서버이다.

<img width="620" height="348" alt="Screenshot 2025-11-30 at 6 01 31 PM" src="https://github.com/user-attachments/assets/2a57035c-49e1-4869-92bb-07ee3ba4460b" />

만약, 어떤 서버(노드, 버킷)이 제거되거나 가용 불가 상황이 되는 경우, 다음 인접 서버로 Rehashing 한다.

다음 도식은 서버 불능으로 인한 Rehashing 시 키의 저장 위치 변경을 보여주는 도식이다.

<img width="624" height="441" alt="Screenshot 2025-11-30 at 6 03 37 PM" src="https://github.com/user-attachments/assets/02ecc51d-2262-48e0-9c25-c52d2a08ae07" />

눈치 빠른 여러분은 이미 파악하셨겠지만, 각 파티션(인접 서버 사이의 해시 공간)의 크기가 `n/m` 이라는 것을 알 것이다.

즉, 어떤 서버의 불능으로 인해 Rehashing 필요한 경우, 불능이 된 서버의 Partition 내 키에 대해서만 Rehashing 을 하면 되기 때문에 '평균적으로 'n/m' 개의 키만 재배치하면 되는 것이다.

'평균'인 이유는, 그렇게 재배치된 키가 새로 저장될 서버의 저장 공간에 여유가 없는 경우, 일부는 다음 서버로 가야할 수 있기 때문. 이 경우엔 추가적인 재배치가 발생.

반대로 통상적으로는 'n/m' 이하의 키만 재배치가 발생할 것이므로, 점근적으로 '평균 n/m' 으로 표현할 수 있음.

**한계점**
1. 서버가 지속적으로 추가/삭제 되는 상황에서는, 파티션(인접 서버 사이의 해시 공간)의 크기를 균등하게 유지하는 것이 불가능.
2. 키의 균등 분포가 달성하기 매우 어렵다는 것 -> 따라서 키의 편중이 일반적일 것으로 가정하는 것이 적절.

> [!NOTE]
>  _카이 제곱 검정(Chi-squared Test)를 활용한 Hash Function 의 Uniform Distributability 평가_
> 
> <img width="190" height="78" alt="Screenshot 2025-11-30 at 6 10 50 PM" src="https://github.com/user-attachments/assets/93419ce1-bf60-4a7b-a259-68e902bcb4ff" />
>
> - n : 키의 수 / m : 버킷의 수 / bj : 각 버킷 별 키의 수
> 
> 카이제곱검정은 통상 '두 범주형 변인 간 관계'를 분석하기 위해 활용.
> 
> 이를 우리의 문제로 가져온다면, '해시 함수에 인자로 투입되는 원래 식별자'와 '해시 함수 시행 결과로 도출되는 키' 간 상관관계가 적으면 적을 수록 더 균등하게 분포된다고 해석할 수 있는 것.

**극복 방법 : 가상 노드**
- 가상 노드 : 실제 노드/서버를 가리키는 노드
- 이를 통해, 해시 키 공간 내에서 서버가 보유한 파티션 크기를 최대한 균등하게 배분함으로써 한계점 극복

<img width="623" height="432" alt="Screenshot 2025-11-30 at 6 22 00 PM" src="https://github.com/user-attachments/assets/7c97543b-eef5-45d9-a29a-a7f813c47f2d" />


### 3. 활용 시의 Trade-off
- 가상 노드가 많으면 많을 수록, 해시 링 내에 존재하는 서버/노드 별 파티션 크기 및 위치가 균등하게 배분되므로 해싱 효율이 증가함.
- 하지만 그 만큼 많은 수의 가상 노드를 관리하고 실제로 키를 할당해야 하기 때문에 공간 효율 문제 대두.
- 따라서, 두 지점 간 트레이드 오프를 잘 고려한 설계가 필요.
