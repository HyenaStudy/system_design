# Chapter 4. 처리율 제한 장치의 설계

### Rate limiter

- client or service가 보내는 트래픽의 처리율을 제한하기 위한 장치
- 장점
    - Dos 공격에 의한 resource starvation을 방지
    - 우선 순위가 높은 api에 더 많은 자원을 할당 가능하도록 하여 비용 절감
    - 서버 과부하를 막을 수 있다.
- 고려 사항
    - rate limiter가 응답시간에 영향을 미치지 않도록 설계 필요
    - 가능한 적은 메모리 사용
    - 설정된 처리율을 초과하는 요청의 경우에만 정확하겍 제한해야함
    - 요청이 제한되었을 경우 그 사실을 사용자에게 명확히 고지
    - fault tolerance : 제한 장치에 장애가 생기더라도 전체 시스템에 영향을 미치면 안된다.
- 위치
    - 클라이언트 : 일반적으로 클라이언트 요청은 쉽게 위변조 가능하여 모든 케이스를 대응하는 것이 어려울 수 있음
    - 서버
        - api 서버에 rate limiter를 함께 구성
        - 클라이언트 ↔ 미들웨어(rate limiter 역할) ↔ api 서버
- 알고리즘
    - token bucket
        - token bucket은 지정된 용량을 갖는 컨테이너
        - 사전 설정된 양의 토큰이 주기적으로 채워지고 over flow된 토큰은 버려진다
        - 충분한 토큰이 있는지 조회 → 토큰을 하나 꺼내 시스템에 전달한다
        - 구현이 비교적 간단하고 메모리 사용 측면에서 효율적이다.
        - 짧은 시간 집중되는 트래픽도 처리 가능하다
        - 토큰 공급률과 버킷 크기를 조절하는 것이 까다로울 수 있음
    - leaky bucket
        - request가 도착하면 큐에 가득차있는 지 본다. 빈자리가 있는 경우 큐에 요청을 추가하고 없으면 새 요청을 버린다
        - 큐의 크기가 제한되어 있어 메모리 측면에서 효율적
        - 단 시간에 많은 트래핏이 몰리는 경우 큐에는 오래된 요청들이 쌓이게 되고 그 요청들을 제때 처리하지 못하면 최신 요청들은 버려지게 된다
    - fixed window counter
        - 타임라인을 고정된 간격의 윈도우로 분할하고 각 윈도우마다 counter를 부여
        - 요청이 접수될 때마다 counter의 값을 증가, counter가 맥스에 도달하면 새 윈도우 타임라인에 도달할 때까지 버려진다
        - 윈도우 boundary에 많은 트래픽이 몰린 경우 윈도우에 할당된 양보다 더 많은 요청이 처리 될 수 있음
        - == 트래픽이 일정 구간에 몰리는 경우 제한했던 양보다 많은 양을 처리하게 됨
    - sliding window log
        - fixed window log의 단점을 해소한 알고리즘
        - 요청의 timestamp를 캐시에 저장하여 추적한다.
        - 새 요청이 오면 만료된 타임스탬프를 제거, 새 요청의 타임 스탬프를 캐시에 저장한다. 저장된 타임스탬프의 크기가 허용치보다 크다면 해당 요청을 drop
        - 어느 순간의 처리율을 보더라도 시스템의 처리율 한도를 넘지 않음
        - 거부된 요청의 timestamp도 보관하여 다량의 메모리를 사용하게 됨
    - sliding window counter
        - sliding window log + fixed window counter

**x Token bucket + 분산 Rate Limit**

- x는 Token bucket 기반의 Rate Limit  사용
- https://docs.x.com/x-api/fundamentals/rate-limits
- fixed window는 boudary 시점에 트래픽 스파이크가 발생할 수 있음
- 완화한 Sliding window의 경우는 대규모 분산 시스템에서 저장과 연산에 의한 비용이 올라가고 구현 복잡도가 높아짐
- 분산 ratelimit
    - 사용자 1이 미국 리전의 서버에서 api 요청 후, 곧 바로 한국 리전에서 api 요청 시 두 요청 모두 카운트가 되어야함 → 전 리전이 같은 rate-limit 버킷 상태를 요청해야 함
    - 마스터 캐시가 모든 리전에서 복제되어야 하고 대규모 요청 속에서도 race condition없이 동작해야 한다.
    - 모든 서버가 rate limit을 처리하는 경우 같은 서버의 캐시를 사용하도록 하고 atomic하게 처리

**Lift rate limiting**

- go + grpc + redis
- domain과 desciptor를 기반으로 서비스별 구체적으로 rate limit 정책 구현 가능
