---
layout: default
title: "13. 포트폴리오 관련"
nav_order: 13
---

# 13. 포트폴리오 관련

> 실제 프로젝트 경험 기반 질문 — 피피(PPFriends), MiniPlace POS, PeakTime 등에서 구현한 기능에 대한 심층 질문과 답변.

---

## 1. Exponential Backoff + Jitter (Lv2)

> "RetryInterceptor에 Exponential Backoff + Jitter를 적용하셨는데, 왜 단순 Exponential Backoff만으로는 부족하고 Jitter가 필요한가요?"

### 💬 면접 답변

단순 Exponential Backoff는 재시도 대기 시간을 지수로 늘려 서버 부하를 줄이지만, 동시에 에러를 받은 클라이언트가 모두 같은 공식을 쓰면 **동일한 시점에 동시 재시도**가 몰립니다. Jitter는 각 클라이언트의 대기 시간에 **무작위 노이즈**를 더해 요청을 시간축 위에 분산시킵니다. 피피 프로젝트의 `RetryInterceptor`에서는 `exponentialDelay + random(0...exponentialDelay)` 방식으로 구현했으며, `maxDelay = 5.0초`로 상한을 뒀습니다.

### 📚 보충 설명

**왜 동시 재시도가 문제인가**

1,000개 클라이언트가 같은 시점에 에러를 받으면, 모두 동일한 backoff 공식을 적용해 T=1s, T=2s, T=4s에 **1,000개씩** 몰립니다. 서버가 복구된 직후 다시 과부하가 걸립니다. 이를 **thundering herd problem**이라 합니다.

**실제 코드 (RetryInterceptor.swift)**

```swift
public init(
    maxRetries: Int = 3,
    baseDelay: TimeInterval = 1.0,
    maxDelay: TimeInterval = 5.0  // 최대 5초 cap
) { ... }

let exponentialDelay = baseDelay * pow(2.0, Double(retryCount))
let jitter = Double.random(in: 0...exponentialDelay)
let delay = min(maxDelay, exponentialDelay + jitter)
```

각 회차별 실제 대기 시간:
- 1회차: 1 + random(0~1) → 최소 1초, 최대 2초
- 2회차: 2 + random(0~2) → 최소 2초, 최대 4초
- 3회차: 4 + random(0~4) → 최소 4초, 최대 **5초 (cap 적용)**

### 🔗 참고 자료
- [AWS: Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

---

## 2. Full Jitter, Equal Jitter, Decorrelated Jitter 비교 (Lv3)

> "Full Jitter, Equal Jitter, Decorrelated Jitter의 차이를 아시나요?"

### 💬 면접 답변

세 방식 모두 Exponential Backoff에 무작위성을 더하지만 범위와 성격이 다릅니다. **Full Jitter**는 `random(0, cap)`으로 지연을 0에 가깝게 만들 수 있어 클라이언트 재시도가 가장 빠르지만, 서버 관점에서는 요청이 초반에 몰릴 수 있습니다. **Equal Jitter**는 `cap/2 + random(0, cap/2)`으로 최소 대기를 보장해 트래픽이 더 고르게 분산됩니다. **Decorrelated Jitter**는 `random(baseDelay, prevDelay × 3)`으로 이전 대기 시간에 연동되어 결과적으로 AWS 실험에서 서버 전체 처리량이 가장 높게 나왔습니다.

### 📚 보충 설명

**세 방식 공식 비교**

| 방식 | 공식 | 특징 |
|------|------|------|
| Full Jitter | `random(0, min(cap, baseDelay × 2^n))` | 0에 가까운 재시도 가능, 클라이언트 입장 유리 |
| Equal Jitter | `cap/2 + random(0, cap/2)` | 최소 대기 보장, 트래픽 분산 고름 |
| Decorrelated Jitter | `random(baseDelay, prevDelay × 3)` | 이전 대기와 연동, 서버 처리량 최고 |

**Decorrelated Jitter 트레이드오프**

AWS 실험에서 서버 전체 처리량이 가장 높게 나온 이유는, 이전 대기가 짧았다면 다음도 짧고 길었다면 다음도 길어지는 방식이어서 **클라이언트 간 재시도 타이밍이 자연스럽게 분리**됩니다. 다만 클라이언트 개별 대기는 길어질 수 있어 사용자 경험 측면에서는 불리합니다. 서버 안정성과 클라이언트 UX 중 무엇을 우선하느냐에 따라 선택이 달라집니다.

**재시도 횟수/상한 결정 기준**

피피 프로젝트에서는 `maxRetries = 3`, `maxDelay = 5.0초`로 설정했습니다. 네트워크 일시 불안정은 보통 3초 이내에 복구되므로 3회면 충분하고, 5초 이상 기다리면 사용자가 앱이 멈췄다고 느낍니다. 최대 총 대기는 약 11초(2 + 4 + 5)이며, 이를 초과하면 재시도를 포기하고 사용자에게 에러를 노출하는 것이 UX상 더 낫습니다.

### 🔗 참고 자료
- [AWS: Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

---

## 3. Thundering Herd Problem과 Jitter의 완화 원리 (Lv3)

> "서버 입장에서 thundering herd problem이 무엇이고, Jitter가 이를 어떻게 완화하나요?"

### 💬 면접 답변

Thundering herd problem은 서버 장애 복구 직후 대기 중이던 수천 개의 클라이언트가 **동시에** 재시도 요청을 보내 서버가 또다시 과부하로 쓰러지는 현상입니다. 단순 Exponential Backoff만 적용하면 모든 클라이언트가 같은 공식을 써서 같은 시점에 몰립니다. Jitter는 각 클라이언트의 대기 시간에 무작위 노이즈를 더해 요청을 시간축 위에 분산시킴으로써 순간 트래픽 급증을 완화합니다.

### 📚 보충 설명

**Thundering Herd 발생 과정**

```
T=0   서버 장애 → 1,000개 클라이언트가 에러 수신
T=1s  1,000개 동시 재시도 → 서버 또 다운
T=2s  1,000개 동시 재시도 → 서버 또 다운
```

서버가 복구되더라도 동시 요청이 밀려와 다시 과부하가 걸리는 악순환이 반복됩니다.

**Jitter 적용 후**

```
T=0.3s  80개 요청
T=0.7s  160개 요청
T=1.1s  90개 요청
T=1.5s  120개 요청
...  (시간축 위에 분산)
```

순간 최대 RPS가 급감하고, 서버는 복구 직후 점진적으로 트래픽을 받으면서 캐시, DB 커넥션 풀 등이 워밍업될 시간을 벌 수 있습니다.

### 🔗 참고 자료
- [AWS: Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

---

## 4. 401 토큰 갱신 플로우 — actor 기반 코어레싱 (Lv2)

> "동시에 5개의 API 요청이 401을 받았을 때, 토큰 갱신이 정확히 1번만 일어나도록 보장하는 전체 플로우를 처음부터 끝까지 설명해주세요."

### 💬 면접 답변

`actor` 기반 `TokenRefreshService`로 구현했습니다. actor의 serial executor가 동시 접근을 직렬화하기 때문에, 5개의 요청이 동시에 401을 받아도 refreshToken 호출은 정확히 1번만 일어납니다.

전체 플로우는 다음과 같습니다. 첫째, 5개 Task가 동시에 401을 받고 `TokenRefreshService.refresh()`를 호출합니다. actor이므로 한 번에 하나씩만 진입합니다. 둘째, 첫 번째 Task가 진입하면 `activeTask`가 nil인 것을 확인하고 실제 갱신 Task를 생성해 `activeTask`에 저장합니다. 셋째, 2~5번째 Task는 진입 시 `activeTask`가 이미 있으므로, 새 갱신을 하지 않고 **기존 Task의 결과를 await**합니다. 이것이 코어레싱(coalescing)입니다. 넷째, 갱신이 완료되면 새 accessToken을 반환하고 `activeTask`를 nil로 초기화합니다. 다섯째, 5개 요청 모두 새 토큰을 받아 원래 요청을 재시도합니다.

### 📚 보충 설명

**코어레싱(Coalescing)이란**

"같은 목적의 요청 여러 개를 하나로 합쳐서 실제 작업은 1번만 실행하는 것"입니다. `Task.value`는 Task가 완료될 때까지 await하고, 완료 후에는 결과값을 캐싱합니다. 덕분에 req2~5가 각각 다른 시점에 `existing.value`를 await해도 같은 결과를 돌려받습니다.

```swift
actor TokenRefreshService {
    private var activeTask: Task<String, Error>?

    func refresh() async throws -> String {
        if let existing = activeTask {
            return try await existing.value  // 코어레싱: 기존 Task 결과 대기
        }

        let task = Task<String, Error> {
            try await requestNewToken()
        }
        activeTask = task
        defer { activeTask = nil }
        return try await task.value
    }
}
```

**타임라인**

```
T=0ms   req1~5 모두 401 수신
T=1ms   req1이 actor 진입 → activeTask 생성
T=1ms   req2~5가 actor 직렬 대기
T=2ms   req2 진입 → activeTask 있음 → await existing.value
T=3ms   req3~5 동일
T=500ms 갱신 완료 → req1~5 모두 새 토큰 수신
T=501ms req1~5 원래 요청 재시도
```

### 🔗 참고 자료
- [Swift Documentation - Actors](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/#Actors)

---

## 5. 로그아웃 시 activeTask 취소 (Lv3)

> "토큰 갱신 중에 reset(로그아웃)이 호출되면 어떻게 되나요? activeTask를 어떻게 취소하나요?"

### 💬 면접 답변

`reset()` 메서드도 같은 actor 안에 두면, 로그아웃 호출이 들어왔을 때 `activeTask?.cancel()`을 호출해 진행 중인 갱신 Task를 취소할 수 있습니다. Task 취소는 협력적(cooperative)이므로, 실제 네트워크 요청이 중단되려면 내부에서 `try Task.checkCancellation()`이 실행되는 시점까지 기다려야 합니다. 취소 후 `activeTask = nil`로 초기화하고 토큰도 삭제하면 깔끔하게 로그아웃 상태가 됩니다.

### 📚 보충 설명

```swift
actor TokenRefreshService {
    private var activeTask: Task<String, Error>?

    func reset() {
        activeTask?.cancel()       // 취소 신호 전송
        activeTask = nil
        LoginUsersData.shared.clearTokens()
    }

    func refresh() async throws -> String {
        if let existing = activeTask {
            return try await existing.value  // 취소되면 CancellationError throw
        }
        // ...
    }
}
```

`existing.value`를 await 중이던 req2~5는 `activeTask`가 취소되는 순간 모두 `CancellationError`를 받아 재시도 없이 실패 처리됩니다.

### 🔗 참고 자료
- [Swift Documentation - Task Cancellation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/#Task-Cancellation)

---

## 6. 갱신된 토큰을 요청에 주입하는 방식 (Lv2)

> "갱신된 accessToken을 원래 실패한 5개 요청에 각각 주입할 때 어떤 방식을 사용했나요?"

### 💬 면접 답변

Alamofire의 `RequestRetrier` 프로토콜을 활용했습니다. 401 응답이 오면 `retry()`에서 `TokenRefreshService.refresh()`로 새 토큰을 받아 `LoginUsersData.shared`에 저장하고, `.retry`를 반환합니다. Alamofire가 원래 `URLRequest`를 처음부터 다시 만들 때 `adapt()`가 저장된 새 토큰을 Authorization 헤더에 자동으로 주입합니다.

### 📚 보충 설명

**Retrier는 갱신 후 저장만, 주입은 Adapter가 담당**

```swift
// Retrier: 토큰 갱신 후 저장만 하고 .retry 반환
let newToken = try await TokenRefreshService.shared.refresh()
LoginUsersData.shared.accessToken = newToken
completion(.retry)  // Alamofire가 adapt() → 재시도

// Adapter: 재시도 요청 생성 시 새 토큰을 자동으로 읽음
request.setValue(
    "Bearer \(LoginUsersData.shared.accessToken)",
    forHTTPHeaderField: "Authorization"
)
```

`.retry`는 "이 요청을 처음부터 다시 만들어서 보내라"는 의미이므로, `adapt()`를 다시 거치면서 새 토큰이 자동 반영됩니다.

### 🔗 참고 자료
- [Alamofire - RequestInterceptor](https://github.com/Alamofire/Alamofire/blob/master/Documentation/AdvancedUsage.md#requestinterceptor)

---

## 7. Adapter와 Retrier의 역할 차이 (Lv2)

> "Interceptor 체인에서 adapter와 retrier의 역할 차이는 무엇인가요? (Alamofire 기준)"

### 💬 면접 답변

Adapter는 요청을 **보내기 전**에 URLRequest를 가공하는 역할이고, Retrier는 응답을 **받은 후** 재시도 여부를 결정하는 역할입니다. Alamofire의 `Interceptor`는 이 둘을 하나의 객체에 묶은 편의 타입입니다.

### 📚 보충 설명

**실행 시점 비교**

```
요청 발송 전          응답 수신 후
────────────         ──────────────
adapt() 실행    →    서버 응답    →    retry() 실행
헤더 삽입             (200/401 등)      재시도 결정
```

**피피 프로젝트에서의 분리**

| | RefreshTokenInterceptor | RetryInterceptor |
|---|---|---|
| 프로토콜 | RequestInterceptor | RequestRetrier |
| adapt | 토큰 헤더 삽입 | — |
| retry | 401 감지 → 토큰 갱신 | 네트워크 에러 → Exponential Backoff |

역할이 다르니 프로토콜 채택도 다르게 가져갔습니다. `RetryInterceptor`는 요청 전 가공이 전혀 없으므로 `RequestRetrier`만 채택했습니다.

### 🔗 참고 자료
- [Alamofire - RequestInterceptor](https://github.com/Alamofire/Alamofire/blob/master/Documentation/AdvancedUsage.md#requestinterceptor)

---
