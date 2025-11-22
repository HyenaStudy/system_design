# Chapter 4 : 처리율 제한 장치의 설계

## I. 처리율 제한 장치(Rate Limiter)란?
1. 정의
 - 클라이언트 또는 서비스가 보내는 트래픽의 처리율을 제어하기 위한 장치
 - 사용 목적 및 처리율 제한 대상에 따라 다양한 작동 방식 및 조건이 있지만, '제한 장치에 정의된 임계치(Threshold)를 초과하는 호출/요청은 모두 Block' 이라는 골자는 모두 공통적으로 가짐.

2. 사용 시 장점
 - DoS(Denial of Service) 공격에 의한 서버 자원 고갈 방지 가능
   - ex. 구글 독스의 경우 분당 300회의 Read 요청, 트위터는 3시간 이내 300개의 트윗 업로드(Write) 만 가능하도록 제한하고 있다.
 - 효율적 자원 사용을 가능케 하여 비용 절감 가능
> [!NOTE]
> _**DoS(Denial of Service)**_
> 
> 대상 기계/자원에 대해 막대한 양의 호출/요청을 보내, 장애를 유발하고자 하는 악의적 공격 행위

## II. 요구사항 정의
> 현재 학습 대상인 _가상 면접 사례로 배우는 대규모 시스템 설계 기초_ 의 해당 챕터 내에서 등장하는 가상 면접관과 피면접자의 대화 부분은 생략하고 요점만 정리하겠습니다.

**요구사항**
 - 설정된 임계치를 초과하는 요청은 정확하게 제한.
 - 낮은 응답시간 : 처리율 제한 장치의 활동이 서비스 비즈니스 로직의 HTTP 응답 시간을 저하시켜선 안된다.
 - 상기한 항목과 같은 맥락으로, 가능한 적은 메모리를 써야한다.
 - 분산 시스템에서의 사용을 전제한다. : 하나의 처리율 제한 장치를 여러 서버나 프로세스에서 공유할 수 있어야 한다.
 - 예외 처리 : 요청이 제한되었을 때는 그 사실을 사용자에게 분명하게 알려야 한다.
 - 높은 Fault-tolerancy : 제한 장치에 장애가 발생하더라도 복구 가능해야 하며, 전체 시스템에 영향을 주어서는 안된다.

## III. 설계 시 고려할 점
### 처리율 제한 장치의 위치

크게 세 가지 위치를 고려해볼 수 있다.

1. **Client-side**
<img width="988" height="411" alt="Screenshot 2025-11-22 at 10 57 24 PM" src="https://github.com/user-attachments/assets/b667432f-9414-4550-9809-75491abb0c2d" />

 - 책에서는 이 위치에 처리율 제한 장치를 두는 것이 바람직하지 않다고 한다. 이를 이해해보기 위해 다음의 상황을 가정해보았고, 그 결과 '우회 가능하다'는 단점으로 인해 부적절함을 이해하게 되었다.
   - Client-side 의 Javascript 코드로 '특정 버튼 1초 이내 10번 클릭'을 제한했다고 해보자.
   - 하지만 이는 1) 웹 브라우저 도구를 통해 Javascript 코드를 우회하거나, 2) API Endpoint 로 브라우저를 통하지 않고 직접 요청을 보내는 방식으로 우회 가능하다.
   
2. **Server-side**
<img width="867" height="285" alt="Screenshot 2025-11-22 at 10 58 20 PM" src="https://github.com/user-attachments/assets/5eb220bc-661b-48c9-928f-8445c775f140" />

 - 서버 측에 설치할 수도 있다. 코드 수준에서도 가능하고, 별도의 컴포넌트를 활용할 수도 있다.
 - 이 경우엔 적절한 위치에, 적절한 시점에 HTTP 요청이 Rate Limiter 를 통과하도록 설계한다면, 우회 가능성을 최소화할 수 있다는 점에서 Client-side 보다 우위를 가진다.
 - 하지만, Server 와 같은 자원을 공유하는 Machine 상에서 Rate Limiter 를 작동시키는 경우, Rate Limiter 가 받아내야 하는 전체 요청으로 인한 부하가 결국 서버로의 부하로 이어질 수 있으므로 주의해야 한다.

3. **Middleware**
<img width="1011" height="272" alt="Screenshot 2025-11-22 at 11 03 52 PM" src="https://github.com/user-attachments/assets/63e32dae-2e46-4b22-a30e-eee4aa924fb9" />

- Middleware 형태의 Rate limiter 를 활용할 경우, Client-side 와 Server-side 의 단점을 극복할 수 있다.
- Cloud 환경에서의 마이크로서비스 아키텍처로 설계된 시스템의 경우, API Gateway 에게 처리율 제한 책임을 부여하기도 한다.

처리율 제한 장치의 위치를 정할 땐 다음의 사항을 고려하는 것이 바람직하다.
 - 프로그래밍 언어, 캐시 서비스 등 현행 서비스가 활용 중인 기술 스택
 - 무엇보다도, 비즈니스 요구와 가장 잘 부합하는 것이 중요.

## IV. 대표적인 Rater Limiting Algorithms

### 토큰 버킷(Token Bucket)
### 누출 버킷(Leaky Bucket)
### 고정 윈도우 카운터(Fixed Window Counter)
### 슬라이딩 윈도우 카운터(Sliding Window Counter)
### 슬라이딩 윈도우 로그(Sliding Window Log)
