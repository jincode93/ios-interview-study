---
layout: default
title: "05. 동시성 (Concurrency)"
nav_order: 5
---

# 05. 동시성 (Concurrency)

> GCD, OperationQueue, async/await, Actor — iOS의 동시성 모델 전반.

---

## 1. iOS에서 멀티스레딩을 구현하는 방법은? (Lv1)

### 💬 면접 답변

iOS에는 세 가지 추상화가 있습니다. 첫째, **GCD(DispatchQueue)** — 큐 기반의 함수형 인터페이스로 가장 가볍고 폭넓게 쓰입니다. 둘째, **OperationQueue** — GCD 위의 객체 지향 추상으로 의존성, 취소, 우선순위, KVO 같은 기능이 필요할 때 적합합니다. 셋째, iOS 13+의 **Swift Concurrency(async/await, Task, Actor)** — 구조적 동시성과 컴파일러 보장 데이터 격리를 제공해서 현재는 가장 권장되는 방식입니다. 어떤 방식을 쓰든 UI 업데이트는 반드시 메인 스레드(또는 `@MainActor`)에서 해야 합니다.

### 📚 보충 설명

**선택 기준**

| 시나리오 | 추천 |
|---------|------|
| 간단한 백그라운드 작업, async/await 마이그레이션 전 | GCD |
| 의존성·취소·우선순위가 필요한 작업 그래프 | OperationQueue |
| 새 코드, 비동기 흐름이 복잡 | async/await + Task |
| 공유 상태 격리 | Actor |

**메인 스레드에서 UI 업데이트해야 하는 이유**

UIKit/AppKit은 메인 스레드 전용으로 설계되어 있어 다른 스레드에서 접근하면 비결정적인 동작·크래시가 발생합니다. SwiftUI도 마찬가지로 `@MainActor` 격리가 기본입니다.

### 🔗 참고 자료
- [Apple - Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)
- [Apple - Concurrency (Swift book)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)

---

## 2. GCD(Grand Central Dispatch)의 주요 개념과 사용법을 설명해주세요. (Lv1)

### 💬 면접 답변

GCD는 작업을 큐에 넣으면 시스템이 관리하는 스레드 풀에서 실행해주는 동시성 프레임워크입니다. 큐는 **Serial**(작업이 순서대로 하나씩)과 **Concurrent**(여러 작업 동시)로 나뉘고, 메인 큐(`DispatchQueue.main`)는 시리얼이며 메인 스레드에서 실행됩니다. 글로벌 큐(`DispatchQueue.global(qos:)`)는 콘커런트이며 QoS 등급별로 시스템 풀이 준비되어 있습니다. 일반적으로 무거운 작업은 글로벌 큐에서 실행하고 UI 갱신만 메인으로 디스패치하는 패턴이 가장 흔합니다.

### 📚 보충 설명

**기본 사용**

```swift
// 백그라운드에서 작업 후 메인에서 UI 업데이트
DispatchQueue.global(qos: .userInitiated).async {
    let data = heavyComputation()
    DispatchQueue.main.async {
        label.text = data
    }
}
```

**QoS 등급**

| QoS | 용도 |
|-----|------|
| `.userInteractive` | UI와 직결, 즉각 반응 (애니메이션) |
| `.userInitiated` | 사용자가 요청한 즉시 결과 필요 (탭 이후 데이터 로드) |
| `.default` | 기본값 |
| `.utility` | 진행 표시되는 장기 작업 (다운로드) |
| `.background` | 사용자가 안 보는 작업 (인덱싱, 동기화) |

**Serial vs Concurrent 큐**

```swift
let serial = DispatchQueue(label: "com.app.serial")              // serial
let concurrent = DispatchQueue(label: "com.app.con", attributes: .concurrent)

serial.async { /* task 1 */ }
serial.async { /* task 2 (1 끝난 후) */ }

concurrent.async { /* task A */ }
concurrent.async { /* task B (A와 동시 가능) */ }
```

**sync vs async**
- `async`: 작업을 큐에 넣고 즉시 반환
- `sync`: 작업이 끝날 때까지 호출자가 대기. **메인에서 메인 큐로 sync는 데드락**, 같은 시리얼 큐에서 sync도 데드락 위험.

**DispatchWorkItem**

작업을 객체로 만들어 취소·재실행·완료 노티를 받을 수 있게 합니다.

```swift
let item = DispatchWorkItem { print("hi") }
DispatchQueue.global().async(execute: item)
item.cancel()
item.notify(queue: .main) { print("done") }
```

**DispatchGroup, DispatchSemaphore, DispatchBarrier**

```swift
// Group: 여러 비동기 작업의 완료 동기화
let group = DispatchGroup()
urls.forEach { url in
    group.enter()
    download(url) { _ in group.leave() }
}
group.notify(queue: .main) { print("all done") }

// Semaphore: 동시 실행 수 제한 / 동기화
let sema = DispatchSemaphore(value: 1)
sema.wait();  /* critical section */;  sema.signal()

// Barrier: concurrent 큐에서 쓰기 락처럼
concurrent.async(flags: .barrier) { /* 다른 작업이 없을 때만 실행 */ }
```

### 🔗 참고 자료
- [Apple - DispatchQueue](https://developer.apple.com/documentation/dispatch/dispatchqueue)
- [Apple - Energy Efficiency Guide for iOS Apps - Prioritize Work with Quality of Service Classes](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/EnergyGuide-iOS/PrioritizeWorkWithQoS.html)

---

## 3. DispatchQueue와 OperationQueue의 차이는? (Lv1)

### 💬 면접 답변

DispatchQueue는 GCD의 큐로 함수형이고 가벼우며, 클로저를 던지면 시스템 스레드 풀이 실행하는 단순한 모델입니다. OperationQueue는 Operation(또는 BlockOperation)을 객체로 다루기 때문에 **의존성, 취소, 우선순위, 동시 실행 수 제한, KVO**를 지원해서 복잡한 작업 그래프를 표현하기 적합합니다. 단순한 비동기는 GCD가, 복잡한 워크플로우와 취소가 필요하면 OperationQueue가 어울리는데, 새 코드라면 두 가지를 합친 듯한 표현력의 **async/await + Task**를 우선 고려하는 게 좋습니다.

### 📚 보충 설명

**OperationQueue 예시**

```swift
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 3   // 동시 실행 수 제한

let op1 = BlockOperation { /* ... */ }
let op2 = BlockOperation { /* ... */ }
op2.addDependency(op1)                   // op1 완료 후 op2 실행

queue.addOperations([op1, op2], waitUntilFinished: false)

// 일괄 취소
queue.cancelAllOperations()
```

**커스텀 Operation**

오래된 패턴이지만 여전히 유용한 경우:

```swift
final class DownloadOperation: Operation {
    override func main() {
        guard !isCancelled else { return }
        // 작업 수행
    }
}
```

비동기 작업을 다루려면 `isExecuting`, `isFinished`를 KVO로 발행하며 직접 상태 머신을 관리해야 해서 다소 번거롭습니다. 이런 케이스가 많으면 async/await가 훨씬 깔끔합니다.

---

## 4. Race Condition을 방지하는 방법은? (Lv1)

### 💬 면접 답변

Race Condition은 여러 스레드가 공유 자원을 동시에 읽고 쓰면서 결과가 비결정적이 되는 상황입니다. 해결 도구는 여러 가지가 있는데, 가장 기본은 **시리얼 큐**로 모든 접근을 한 큐로 직렬화하는 것입니다. 더 세밀하게는 `NSLock`, `os_unfair_lock`, `DispatchSemaphore`로 임계 영역을 보호하거나, 읽기/쓰기를 분리하려면 concurrent 큐 + barrier 패턴을 씁니다. Swift Concurrency 환경에서는 **Actor**가 컴파일 타임에 데이터 격리를 강제해 가장 안전하고 권장됩니다. Race Condition 진단에는 Xcode의 **Thread Sanitizer**(TSan)가 매우 효과적입니다.

### 📚 보충 설명

**Serial Queue 패턴 (Reader-Writer)**

```swift
final class SafeStore<T> {
    private var value: T
    private let queue = DispatchQueue(label: "store", attributes: .concurrent)

    init(_ initial: T) { self.value = initial }

    func read() -> T {
        queue.sync { value }
    }
    func write(_ new: T) {
        queue.async(flags: .barrier) { self.value = new }
    }
}
```

**Lock**

```swift
final class Counter {
    private var count = 0
    private let lock = NSLock()

    func increment() {
        lock.lock(); defer { lock.unlock() }
        count += 1
    }
}
```

`os_unfair_lock`이 NSLock보다 더 빠르지만 Swift에서 직접 다루려면 `OSAllocatedUnfairLock` (iOS 16+)을 쓰는 것이 안전합니다.

**Actor (가장 권장)**

```swift
actor Counter {
    private var count = 0
    func increment() { count += 1 }
    func get() -> Int { count }
}

let c = Counter()
Task { await c.increment() }
```

**Sendable 검사**

값 타입이나 모든 멤버가 Sendable이면 자동으로 Sendable. 클래스는 final + 모든 멤버 immutable이거나 내부 동기화가 있어야 명시적으로 `@unchecked Sendable` 선언 가능.

### 🔗 참고 자료
- [Apple - Synchronization](https://developer.apple.com/documentation/swift/synchronization)
- [WWDC 2021 - Protect mutable state with Swift actors](https://developer.apple.com/videos/play/wwdc2021/10133/)

---

## 5. 동기(Sync)와 비동기(Async)의 차이는? (Lv1)

### 💬 면접 답변

동기는 어떤 작업을 호출하면 그 작업이 끝날 때까지 호출자가 멈춰서 기다리는 방식이고, 비동기는 작업을 시작만 시키고 즉시 반환해 호출자가 다른 일을 할 수 있게 하는 방식입니다. 네트워크나 디스크 I/O처럼 시간이 오래 걸리는 작업을 동기로 메인 스레드에서 실행하면 UI가 얼어붙기 때문에 거의 항상 비동기로 처리합니다. iOS에서 비동기는 콜백 클로저, GCD `async`, Combine, `async/await` 등 여러 형태로 표현됩니다.

### 📚 보충 설명

**비동기 표현 방식의 변천**

```swift
// 1세대: Delegate
URLSessionDelegate

// 2세대: Completion Handler
URLSession.shared.dataTask(with: url) { data, _, _ in
    DispatchQueue.main.async { ... }
}.resume()

// 3세대: Combine (iOS 13+)
URLSession.shared.dataTaskPublisher(for: url)
    .receive(on: DispatchQueue.main)
    .sink(receiveCompletion: { _ in }, receiveValue: { _ in })

// 4세대: async/await (iOS 15+)
let (data, _) = try await URLSession.shared.data(from: url)
```

**Semaphore와 Mutex**

엄밀한 OS 개념으로,
- **Mutex** (Mutual Exclusion): 동시 1개만 진입 가능 (소유권 개념)
- **Semaphore**: 카운트 기반, N개까지 진입 가능 (소유권 없음)

`DispatchSemaphore(value: 1)`은 mutex처럼 사용할 수 있지만 정확히는 binary semaphore입니다.

---

## 6. async/await에 대해 설명해주세요. 기존 completion handler 코드와 어떻게 다른가요? (Lv2~3)

### 💬 면접 답변

async/await는 비동기 코드를 동기 코드처럼 선형적으로 쓸 수 있게 해주는 Swift Concurrency의 핵심 문법입니다. `async` 함수는 일시 정지(suspend)될 수 있는 함수이고, `await`로 그 결과를 기다리며 호출자가 양보합니다. 콜백 지옥(Callback Hell)이 사라지고, 에러는 `throws`로 자연스럽게 전파됩니다. 또 **Task**가 비동기 작업의 단위이고 **구조적 동시성**(부모-자식 Task)이 보장돼, 부모가 취소되면 자식들도 자동으로 취소됩니다.

### 📚 보충 설명

**기본 패턴**

```swift
func fetchUser(id: Int) async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url(id))
    return try JSONDecoder().decode(User.self, from: data)
}

Task {
    do {
        let user = try await fetchUser(id: 1)
        await MainActor.run { label.text = user.name }
    } catch {
        print(error)
    }
}
```

**구조적 동시성**

```swift
// 병렬: async let
async let a = fetchA()
async let b = fetchB()
let (resultA, resultB) = try await (a, b)

// 동적 그룹
try await withThrowingTaskGroup(of: User.self) { group in
    for id in ids { group.addTask { try await fetchUser(id: id) } }
    var users: [User] = []
    for try await u in group { users.append(u) }
    return users
}
```

**Task 취소**

```swift
let task = Task {
    while !Task.isCancelled {
        await doWork()
    }
}
task.cancel()
```

부모 Task가 취소되면 자식들도 자동으로 취소되며, `Task.checkCancellation()`이나 `Task.isCancelled`로 응답합니다.

**완료 핸들러 → async 변환 (Bridging)**

```swift
func legacyFetch(completion: @escaping (Result<Data, Error>) -> Void) { ... }

func newFetch() async throws -> Data {
    try await withCheckedThrowingContinuation { cont in
        legacyFetch { result in cont.resume(with: result) }
    }
}
```

주의: `continuation.resume`은 **정확히 한 번**만 호출되어야 합니다. 두 번이면 크래시, 안 부르면 영원히 대기.

### 🔗 참고 자료
- [Swift Evolution: SE-0296 - async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [Swift Evolution: SE-0304 - Structured Concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [WWDC 2021 - Meet async/await](https://developer.apple.com/videos/play/wwdc2021/10132/)
- [WWDC 2021 - Explore structured concurrency](https://developer.apple.com/videos/play/wwdc2021/10134/)

---

## 7. Actor란 무엇이며 언제 사용하나요? (Lv3, Lv4)

### 💬 면접 답변

Actor는 내부 상태에 대한 동시 접근을 컴파일러가 막아주는 참조 타입입니다. 클래스와 비슷하지만 내부 mutable 상태에 외부에서 접근할 때는 반드시 `await`을 통해 직렬화되므로, 락 없이도 데이터 경쟁이 원천적으로 차단됩니다. 캐시·세션 매니저·연결 상태 관리처럼 공유 mutable 상태를 다루는 곳에 적합하고, 메인 스레드 전용 코드는 글로벌 actor인 `@MainActor`로 표현합니다.

### 📚 보충 설명

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]

    func image(for url: URL) -> UIImage? { cache[url] }
    func set(_ image: UIImage, for url: URL) { cache[url] = image }
}

let cache = ImageCache()

Task {
    if let cached = await cache.image(for: url) {
        // 사용
    } else {
        let img = try await download(url)
        await cache.set(img, for: url)
    }
}
```

**Actor의 핵심 특징**
- **격리(Isolation)**: 외부에서 actor의 mutable 상태에 접근하려면 `await` 필요 (직렬화됨)
- **재진입(Re-entrancy)**: actor 메서드가 `await`로 일시 정지하면 다른 호출이 들어올 수 있음 → 일관성 가정 주의
- **Sendable**: actor를 가로지르는 값은 Sendable이어야 함

**@MainActor**

UI 갱신은 메인 스레드에서만 가능하다는 제약을 타입 시스템으로 표현:

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var items: [Item] = []

    func load() async {
        let new = await api.fetch()
        items = new            // 자동으로 메인 스레드 보장
    }
}
```

전체 클래스 외에 개별 메서드/프로퍼티에도 적용 가능. SwiftUI의 `View.body`는 자동으로 `@MainActor`입니다.

**재진입 함정 예시**

```swift
actor Counter {
    var count = 0
    func incrementAndPrint() async {
        let old = count
        await Task.sleep(...)        // 이 사이에 다른 호출이 들어와 count 변경 가능
        count = old + 1              // ❌ 잘못된 가정
    }
}
```

→ `await` 전후에는 상태 가정이 깨질 수 있다고 생각하기.

### 🔗 참고 자료
- [Swift Evolution: SE-0306 - Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [WWDC 2021 - Protect mutable state with Swift actors](https://developer.apple.com/videos/play/wwdc2021/10133/)
- [WWDC 2022 - Visualize and optimize Swift concurrency](https://developer.apple.com/videos/play/wwdc2022/110350/)

---

## 8. completion handler 기반 코드를 async/await로 마이그레이션할 때 어떤 기준으로 우선순위를 정하나요? (Lv3)

### 💬 면접 답변

우선순위는 **변경 비용 대비 가독성·안정성 개선의 효과**로 정합니다. 중첩된 콜백이 많거나 에러 처리가 분기적인 코드, 그리고 화면 전환·인증 같은 핵심 흐름이 가장 큰 효과를 봅니다. 반대로 단순한 콜백 한 단계는 굳이 변환할 필요가 없어 후순위입니다. 마이그레이션 동안 두 스타일이 공존하므로, `withCheckedContinuation`으로 브리지를 만들어 점진적으로 전환하는 게 안전합니다. 팀원의 학습 곡선을 고려해 페어 리뷰·내부 가이드 작성도 같이 진행합니다.

### 📚 보충 설명

**우선순위 매트릭스**

| 우선순위 | 대상 | 이유 |
|---------|------|------|
| 높음 | 중첩 콜백 3단계 이상 | 가독성·유지보수성 폭발적 개선 |
| 높음 | 에러 처리가 분기되는 곳 | throws로 단순화 |
| 중간 | 사용자 액션과 직접 연결된 흐름 | 취소·재시도 표현 자연스러움 |
| 낮음 | 단발성 콜백 (성공/실패만) | 변경 효과 적음 |
| 낮음 | 외부 라이브러리 깊숙한 곳 | 위험 대비 효과 |

**브리징 패턴**

```swift
// 레거시
func legacyFetch(completion: @escaping (Result<Data, Error>) -> Void) { ... }

// async 래퍼
func fetch() async throws -> Data {
    try await withCheckedThrowingContinuation { cont in
        legacyFetch { cont.resume(with: $0) }
    }
}
```

**@MainActor 사용의 비용**

`@MainActor`로 표시한 함수를 백그라운드에서 부르면 메인 큐로 hop이 발생합니다. 자주 호출되는 경로에서는 비용이 누적될 수 있어, 진짜 UI 갱신만 `@MainActor`로 좁히는 게 좋습니다.

```swift
// 비효율
@MainActor func computeAndShow() async { /* 무거운 계산 + UI */ }

// 분리
func compute() async -> Result { ... }
@MainActor func show(_ r: Result) { ... }
```

### 🔗 참고 자료
- [WWDC 2021 - Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)

---

## 9. Combine 프레임워크란 무엇인가요? Publisher/Subscriber 모델을 설명해주세요. (Lv2)

### 💬 면접 답변

Combine은 iOS 13+에서 도입된 Apple의 **선언적 비동기 이벤트 스트림** 처리 프레임워크입니다. 시간에 따라 발생하는 값들을 **Publisher**가 발행하고, **Subscriber**가 구독해서 받습니다. 그 사이에 `map`, `filter`, `combineLatest` 같은 **Operator**가 들어가 변환·결합·에러 처리를 표현합니다. 콜백이나 Notification, KVO처럼 분산된 비동기 메커니즘을 하나의 통일된 모델로 다룰 수 있고, 특히 SwiftUI의 `@Published`, `ObservableObject`가 Combine 위에 만들어져 있습니다. 다만 iOS 13+에서만 사용 가능하고, 이후 async/await + AsyncSequence가 등장하면서 Apple의 권장 방향은 이쪽으로 옮겨가는 분위기입니다.

### 📚 보충 설명

**기본 구조**

```swift
let publisher = [1, 2, 3].publisher

let cancellable = publisher
    .map { $0 * 2 }
    .filter { $0 > 2 }
    .sink { value in
        print(value)
    }
// cancellable을 보유하지 않으면 즉시 해제되어 구독 종료
```

**URLSession Publisher**

```swift
URLSession.shared.dataTaskPublisher(for: url)
    .map(\.data)
    .decode(type: User.self, decoder: JSONDecoder())
    .receive(on: DispatchQueue.main)
    .sink(
        receiveCompletion: { print($0) },
        receiveValue: { user in print(user) }
    )
    .store(in: &cancellables)
```

**Subject (수동 발행)**

- `PassthroughSubject<Value, Failure>`: 발행 시점에 구독 중인 사람만 받음
- `CurrentValueSubject<Value, Failure>`: 현재 값을 보유, 구독 시 즉시 발행

```swift
let subject = PassthroughSubject<String, Never>()
subject.sink { print($0) }
subject.send("hello")
```

**@Published**

```swift
class ViewModel: ObservableObject {
    @Published var name = ""
}

let vm = ViewModel()
vm.$name.sink { print($0) }   // $로 publisher 접근
vm.name = "test"              // "test" 출력
```

**Operator 카테고리 일부**
- 변환: `map`, `flatMap`, `scan`
- 필터: `filter`, `removeDuplicates`, `compactMap`
- 결합: `combineLatest`, `zip`, `merge`
- 시간: `debounce`, `throttle`, `delay`
- 에러: `catch`, `retry`, `replaceError`
- 스레드: `subscribe(on:)`, `receive(on:)`

### 🔗 참고 자료
- [Apple - Combine](https://developer.apple.com/documentation/combine)
- [WWDC 2019 - Introducing Combine](https://developer.apple.com/videos/play/wwdc2019/722/)
- [WWDC 2019 - Combine in Practice](https://developer.apple.com/videos/play/wwdc2019/721/)

---

## 10. Combine과 RxSwift의 차이는? (Lv2)

### 💬 면접 답변

둘 다 반응형 프로그래밍 라이브러리지만, RxSwift는 외부 오픈소스이고 iOS 9 이상에서 동작하며 다양한 플랫폼에 공통된 API를 가집니다. Combine은 Apple의 1st-party 프레임워크로 iOS 13+ 전용이고, Swift의 타입 시스템과 시스템 프레임워크들(URLSession, NotificationCenter, KVO)에 깊게 통합되어 있습니다. Combine은 에러 타입이 제네릭(`Failure`)이고 SwiftUI와 자연스럽게 연동되는 게 강점이고, RxSwift는 생태계가 풍부하고 학습 자료가 많은 게 강점입니다. 새 프로젝트에서 iOS 13+ 타깃이라면 Combine이 일반적으로 더 권장됩니다.

### 📚 보충 설명

**용어 매핑**

| RxSwift | Combine |
|---------|---------|
| `Observable` | `Publisher` |
| `Observer` | `Subscriber` |
| `Disposable` | `Cancellable` |
| `DisposeBag` | `Set<AnyCancellable>` |
| `BehaviorSubject` | `CurrentValueSubject` |
| `PublishSubject` | `PassthroughSubject` |
| `Driver`, `Signal` | – (직접 구현하거나 SwiftUI로 대체) |

**Combine의 한계**
- iOS 13+ 전용
- 일부 연산자가 RxSwift보다 적음 (예: `flatMapLatest`는 `switchToLatest`로 대체)
- 학습 자료가 RxSwift만큼 풍부하지 않음

### 🔗 참고 자료
- [Apple - Combine](https://developer.apple.com/documentation/combine)

---

## 11. Combine의 Scheduler란? 백그라운드 → 메인 디스패치 패턴은 어떻게 구현하나요? (Lv2)

### 💬 면접 답변

Scheduler는 작업을 **언제, 어디서 실행할지를 정의하는 추상**입니다. `subscribe(on:)`은 업스트림 작업(구독, 값 생성)이 실행될 컨텍스트를, `receive(on:)`은 다운스트림(맵·싱크 등)이 실행될 컨텍스트를 지정합니다. 가장 흔한 패턴은 네트워크는 백그라운드 큐에서 받고 UI 업데이트만 메인으로 전환하는 것으로, 보통 `.subscribe(on: DispatchQueue.global())`과 `.receive(on: DispatchQueue.main)`을 함께 씁니다.

### 📚 보충 설명

```swift
publisher
    .subscribe(on: DispatchQueue.global(qos: .userInitiated))  // 업스트림은 백그라운드
    .map { /* heavy transform */ }
    .receive(on: DispatchQueue.main)                            // UI 업데이트는 메인
    .sink { /* update UI */ }
    .store(in: &cancellables)
```

**Scheduler 종류**
- `DispatchQueue`: GCD 큐
- `OperationQueue`: NSOperation 큐
- `RunLoop`: 일반적인 메인 런루프 (UI 이벤트와 같은 사이클)
- `ImmediateScheduler`: 즉시 실행 (테스트용)

**`subscribe(on:)`은 한 번, `receive(on:)`은 마지막 사용 권장**

`subscribe(on:)`은 체인 어디에 있어도 가장 처음 한 번만 적용되므로 앞쪽에 둡니다. `receive(on:)`은 그 이후 모든 단계의 실행 컨텍스트를 바꾸므로, 일반적으로 sink 직전에 둡니다.

### 🔗 참고 자료
- [Apple - Scheduler](https://developer.apple.com/documentation/combine/scheduler)

---

## 12. Combine의 에러 처리는 어떻게 하나요? (Lv3)

### 💬 면접 답변

Combine은 Publisher가 값(`Output`)과 함께 에러 타입(`Failure`)을 제네릭으로 가지기 때문에, 에러도 스트림의 한 종류 이벤트로 다룹니다. 가장 흔한 처리 연산자는 `catch`(에러 시 대체 publisher 반환), `retry`(N번 재시도), `replaceError(with:)`(에러 시 기본값), `mapError`(에러 변환)입니다. 에러가 발생하면 그 시점에 스트림이 종료되므로, 종료시키고 싶지 않다면 `catch`로 대체 스트림을 이어 붙입니다. Combine과 Result 타입을 함께 쓰면 에러를 값으로 들고 다니며 더 유연하게 처리할 수도 있습니다.

### 📚 보충 설명

```swift
URLSession.shared.dataTaskPublisher(for: url)
    .map(\.data)
    .decode(type: User.self, decoder: JSONDecoder())
    .retry(2)                                          // 2번 재시도
    .catch { _ in Just(User.empty) }                   // 실패 시 기본값으로 대체
    .receive(on: DispatchQueue.main)
    .sink { user in ... }
    .store(in: &cancellables)
```

**`retry` vs `catch`**

- `retry(n)`: 같은 publisher를 다시 구독. 네트워크 일시적 실패에 좋음.
- `catch`: 에러를 받아 새로운 publisher 반환. 폴백 값, 대체 데이터 소스.

**에러 시 자동으로 구독 취소**

Combine은 에러가 발생하면 스트림이 종료되고, `sink`의 `receiveCompletion`이 `.failure`로 호출된 뒤 구독이 끝납니다. 추가 액션 없이도 자원이 정리됩니다.

**Result 타입과의 결합**

```swift
publisher
    .map(Result.success)
    .catch { Just(Result.failure($0)) }
    // 이제 Output = Result<...>, Failure = Never
```

### 🔗 참고 자료
- [Apple - Handling Errors](https://developer.apple.com/documentation/combine/handling-errors)

---

## 13. Thread Sanitizer는 무엇이며 어떻게 활용하나요? (Lv2)

### 💬 면접 답변

Thread Sanitizer(TSan)는 Clang/LLVM이 제공하는 동적 분석 도구로, 런타임에 메모리 접근을 추적해서 **데이터 경쟁(data race)** 을 감지해줍니다. Xcode의 Scheme → Diagnostics에서 활성화한 뒤 앱을 실행하면, 동기화되지 않은 공유 자원에 대한 동시 접근이 발견되면 즉시 로그로 알려줍니다. 디버깅 빌드의 성능은 5~15배 느려지고 메모리도 더 쓰지만, 동시성 버그는 재현이 어렵기 때문에 CI나 큰 배포 전 검증에 매우 유용합니다. Swift Concurrency의 `Sendable` 검사가 컴파일 타임 가드라면, TSan은 런타임 가드라고 볼 수 있습니다.

### 📚 보충 설명

**활성화 방법**
1. Scheme 편집 → Run → Diagnostics
2. **Thread Sanitizer** 체크
3. 실행하면 race 발견 시 콘솔에 상세 스택 트레이스 출력

**TSan이 잡는 것**
- 동시에 같은 메모리에 쓰기/읽기 (적어도 하나가 쓰기)
- 적절한 동기화(락, 큐, atomic) 없음

**TSan이 못 잡는 것**
- 로직 오류 (race가 발생하지 않는 순서로 실행된다면)
- 외부 라이브러리 내부의 race (심볼 없으면 진단 어려움)

**한계**
- 시뮬레이터에서만 동작 (실기기 불가)
- 32비트 미지원
- 메모리 사용량 5~10배

**Static 분석 보조**
- Swift Concurrency의 `-strict-concurrency=complete` 빌드 옵션은 컴파일 타임에 데이터 격리 위반을 잡아냄

### 🔗 참고 자료
- [Apple - Diagnosing Memory, Thread, and Crash Issues Early](https://developer.apple.com/documentation/xcode/diagnosing-memory-thread-and-crash-issues-early)

---

## 14. 새로운 동시성 모델(Actor, async/await)을 팀에 도입할 때 어떻게 평가·대응하시겠습니까? (Lv4)

### 💬 면접 답변

기술 도입은 기술 자체의 우수성보다 **팀의 준비도와 도입 비용**이 더 큰 변수입니다. 먼저 팀원들이 GCD/Combine을 어떻게 다루는지, 동시성 버그 경험이 얼마나 있는지를 페어 코드 리뷰로 가늠합니다. 다음으로 **실패해도 영향이 적은 신규 기능이나 화면**부터 파일럿 적용해 성공 사례를 만들고, 이를 팀 세미나·내부 가이드로 공유합니다. 마이그레이션 시에는 레거시 콜백과 공존하는 시기가 길어지므로 `withCheckedContinuation` 같은 브리지 사용 규칙, `@MainActor` 적용 범위 같은 컨벤션을 코드 리뷰 체크리스트에 포함합니다. 성공 지표는 동시성 관련 크래시·이슈 감소, 신규 PR의 리뷰 소요 시간 감소 등으로 정량화합니다.

### 📚 보충 설명

**단계별 도입 로드맵 예시**

| 단계 | 목표 | 산출물 |
|------|------|--------|
| 0. 학습 | 팀 세미나 2~3회 (async/await, Actor, Sendable) | 내부 위키 |
| 1. 파일럿 | 영향 적은 신규 모듈에 적용 | PR + 회고 |
| 2. 가이드 | 코드 리뷰 체크리스트, 모범 패턴 | 가이드 문서 |
| 3. 점진 확대 | 핵심 모듈로 단계적 마이그레이션 | 마이그레이션 트래커 |
| 4. 일상화 | Strict concurrency checking 활성화 | 빌드 옵션 변경 |

**자주 마주치는 함정**
- `@MainActor` 무분별한 사용 → 메인 큐 부하
- `Sendable` 경고 무시 → 빌드는 되지만 런타임 race
- Continuation 누락/이중 호출 → 무한 대기 또는 크래시
- Actor 재진입 → 상태 가정 위반

### 🔗 참고 자료
- [Swift Evolution: SE-0337 - Incremental migration to concurrency checking](https://github.com/apple/swift-evolution/blob/main/proposals/0337-support-incremental-migration-to-concurrency-checking.md)
- [WWDC 2022 - Eliminate data races using Swift Concurrency](https://developer.apple.com/videos/play/wwdc2022/110351/)

---

## 15. Distributed Actor란 무엇인가요? (Lv4)

### 💬 면접 답변

Distributed Actor는 Swift 5.7에서 도입된, **프로세스나 네트워크 경계를 가로지를 수 있는 actor**입니다. 일반 actor가 같은 프로세스 안에서 데이터 격리를 보장한다면, distributed actor는 그 추상을 분산 시스템으로 확장해 원격 호출도 컴파일러가 인지할 수 있게 합니다. 모든 원격 호출은 `try await`이 필수이고 인자/반환은 Codable이어야 하며, 실제 전송은 개발자가 구현하는 `DistributedActorSystem`이 담당합니다. 모바일 자체에서 많이 쓰이지는 않지만, 서버 사이드 Swift나 멀티 디바이스(휴대폰 ↔ Watch ↔ TV) 협업에서 가능성이 큰 추상입니다.

### 📚 보충 설명

```swift
import Distributed

distributed actor Player {
    let name: String
    var score: Int = 0

    distributed func updateScore(by delta: Int) {
        score += delta
    }
}
```

**실무 적용 사례 (현재까지)**
- 서버 사이드 Swift (Apple의 Hummingbird 등)
- iCloud 협업 도큐먼트
- 게임의 실시간 멀티플레이어 백엔드

### 🔗 참고 자료
- [Swift Evolution: SE-0336 - Distributed Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0336-distributed-actor-isolation.md)
- [WWDC 2022 - Meet distributed actors in Swift](https://developer.apple.com/videos/play/wwdc2022/110356/)

---

---

## 16. Thread, Queue, Task의 차이를 설명해주세요. (Lv2)

### 💬 면접 답변

세 개념은 추상화 레벨이 다릅니다. **스레드**는 CPU가 코드를 실행하는 물리적 작업자입니다. **큐(DispatchQueue)** 는 할 일 목록으로, 작업을 받아 스레드 풀에 배분하는 스케줄러 역할을 합니다. 큐와 스레드는 1:1이 아니며, Serial 큐도 실행할 때마다 다른 스레드를 사용할 수 있습니다. **Task**는 Swift Concurrency의 논리적 작업 단위로, `await` 지점에서 스레드를 반납하고(suspend) 나중에 다시 재개되는(resume) 방식으로 동작합니다. Task는 스레드를 블로킹하지 않기 때문에, 하나의 스레드가 여러 Task를 번갈아 처리할 수 있습니다.

### 📚 보충 설명

**스레드(Thread)**

CPU가 코드를 한 줄씩 순서대로 실행하는 실체입니다. 앱이 실행되면 기본적으로 메인 스레드 1개가 생기고, 시간이 걸리는 작업은 보조 스레드(백그라운드 스레드)에서 처리합니다. 스레드를 만들고 유지하는 것 자체가 메모리(기본 512KB/스레드)와 CPU를 소비합니다.

**큐(DispatchQueue)**

작업을 직렬(Serial) 또는 동시(Concurrent) 방식으로 스레드에 배분하는 접수대입니다. Serial 큐는 한 번에 하나씩만 처리하여 순서를 보장하고, Concurrent 큐는 여러 작업을 동시에 처리합니다.

- `sync`: 작업이 끝날 때까지 호출한 스레드를 블로킹
- `async`: 작업을 큐에 넣고 즉시 반환 (논블로킹)

**Task (Swift Concurrency)**

async/await 세계의 논리적 작업 단위입니다. `await`를 만나면 Task가 suspend되어 스레드를 반납하고, 기다리던 결과가 준비되면 resume되어 스레드를 다시 받아 실행을 이어갑니다. Swift의 cooperative thread pool(CPU 코어 수만큼만 스레드를 유지하는 체계)이 가능한 이유입니다.

| 개념 | 역할 | 특징 |
|---|---|---|
| Thread | 코드를 실제 실행하는 물리적 작업자 | OS가 관리, 생성 비용 있음 |
| DispatchQueue | 작업을 스레드에 배분하는 스케줄러 | sync(블로킹) / async(논블로킹) |
| Task | async/await의 논리적 작업 단위 | await에서 스레드 반납 후 재개 |

### 🔗 참고 자료
- [Apple - Concurrency (Swift book)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [WWDC 2021 - Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)

---

## 17. actor가 data race를 방지하는 원리를 설명해주세요. (Lv3)

### 💬 면접 답변

actor는 내부적으로 **Serial Executor(직렬 실행기)** 를 가지고 있어, 자신의 상태에 대한 접근을 직렬화합니다. 외부에서 actor의 메서드나 프로퍼티에 접근하려면 반드시 `await`을 사용해야 하고, Swift 컴파일러가 이를 강제합니다. 이 덕분에 동시에 여러 Task가 actor 내부 상태를 읽거나 쓰는 것이 원천적으로 불가능합니다. NSLock이나 DispatchQueue와 달리 잠금을 잘못 걸 여지가 없고, 컴파일 에러로 실수를 조기에 발견할 수 있습니다.

### 📚 보충 설명

**Serial Executor 동작 원리**

5개의 Task가 동시에 actor 메서드를 호출하면, executor가 이를 큐에 줄 세워 하나씩만 처리합니다. 두 번째 Task는 첫 번째가 끝날 때까지 대기합니다. 이 대기는 스레드 블로킹이 아니라 Task suspend로 처리되므로, 스레드는 다른 Task를 처리할 수 있습니다.

**GCD와의 핵심 차이**

| | actor | DispatchQueue (serial) |
|---|---|---|
| 안전 강제 방식 | 컴파일러 (await 없으면 에러) | 개발자 직접 관리 |
| 잘못 쓰면 | 컴파일 에러 | 런타임 크래시 or 무증상 버그 |
| 대기 방식 | Task suspend (스레드 반납) | sync면 스레드 블로킹 |

**TokenRefreshService 핵심 패턴**

5개의 API 요청이 동시에 401을 받는 상황에서, actor를 사용하면 첫 번째 Task만 갱신 Task를 생성하고 나머지 4개는 같은 Task의 결과를 기다립니다. `await` 이전에 `activeTask`를 먼저 set하는 것이 핵심입니다.

### 🔗 참고 자료
- [Swift Evolution: SE-0306 - Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [WWDC 2021 - Protect mutable state with Swift actors](https://developer.apple.com/videos/play/wwdc2021/10133/)

---

## 18. actor의 Reentrancy란 무엇이고, 어떻게 대응하나요? (Lv3)

### 💬 면접 답변

**Reentrancy(재진입)** 란, actor 내부에서 `await`을 만나 Task가 suspend(스레드를 반납하고 일시 중단)되는 순간 다른 Task가 actor에 진입할 수 있는 특성입니다. Swift의 actor는 기본적으로 reentrant하기 때문에, `await` 전후로 actor 내부 상태가 다른 Task에 의해 변경될 수 있습니다. 이를 방지하는 핵심 원칙은 **상태 변경을 await 이전에 배치**하는 것입니다. await 전에 상태를 확정지어두면, suspend 이후 진입한 Task도 올바른 상태를 보게 됩니다.

### 📚 보충 설명

**TokenRefreshService에서의 해결 전략**

```swift
actor TokenRefreshService {
    private var activeTask: Task<String, Error>?

    func refreshIfNeeded() async throws -> String {
        if let task = activeTask {
            return try await task.value  // 기존 갱신 완료 대기
        }
        // await 이전에 상태를 먼저 확정
        let task = Task { try await fetchNewToken() }
        activeTask = task                // ← await 전에 set
        defer { activeTask = nil }
        return try await task.value      // ← 여기서 suspend 가능
    }
}
```

await 이전에 `activeTask`를 set했기 때문에, suspend 이후 진입한 Task도 "이미 갱신 중"을 인식하고 중복 갱신을 하지 않습니다.

**Reentrancy 방지 대안 방법들**

| 방법 | 구현 난이도 | 안전성 | 적합한 상황 |
|---|---|---|---|
| await 전 상태 확정 | 낮음 | 충분함 | 대부분의 중복 실행 방지 |
| Continuation 큐 | 높음 | 완전함 | 순서·격리가 엄격히 필요한 경우 |
| AsyncChannel | 중간 | 완전함 | 요청량이 많고 흐름 제어 필요 |

**Continuation 큐 방식:** 진행 중인 작업이 있으면 새 Task를 실행하지 않고 `withCheckedContinuation`으로 만든 "재개 티켓"을 대기열에 저장합니다. 기존 작업이 끝나면 대기 티켓을 전부 꺼내 결과를 전달합니다. 완전하지만 구현이 복잡하고 Continuation 누수 위험이 있습니다.

### 🔗 참고 자료
- [Swift Evolution: SE-0306 - Actors (Reentrancy 섹션)](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)

---

## 19. @MainActor는 일반 actor와 무엇이 다른가요? (Lv3)

### 💬 면접 답변

`@MainActor`는 앱 전체에 단 하나 존재하는 **글로벌 actor**로, executor가 메인 스레드에 고정되어 있습니다. 일반 actor는 작업이 들어올 때마다 Swift의 cooperative thread pool에서 임의의 스레드를 할당받아 실행되지만, `@MainActor`는 항상 메인 스레드에서만 실행됩니다. 또한 일반 actor는 인스턴스마다 독립적인 executor를 가지지만, `@MainActor`는 앱 전체에서 하나의 executor를 공유합니다.

### 📚 보충 설명

**Executor(실행기)란?**

actor가 받은 작업을 실제로 어떤 스레드에서 실행할지 결정하는 스케줄러입니다. actor가 "줄 세우는 규칙"이라면, executor는 "그 줄을 어느 창구에서 처리할지"를 결정합니다.

| | 일반 actor | @MainActor |
|---|---|---|
| 실행 스레드 | cooperative thread pool의 임의 스레드 | 항상 Main Thread |
| 인스턴스 수 | 각 인스턴스마다 별도 executor | 앱 전체에 단 하나 |
| 주 용도 | 백그라운드 상태 보호 | UI 상태 보호 |

**메인 스레드로 전환하는 방법**

```swift
// 방법 1: await MainActor.run
Task {
    let data = await fetchData()
    await MainActor.run {
        self.label.text = data
    }
}

// 방법 2: @MainActor 함수 await 호출
@MainActor func updateUI(with data: String) {
    label.text = data
}
```

백그라운드 Task에서 `@MainActor` 함수를 `await`으로 호출하면, 현재 Task가 suspend되고 메인 스레드 executor 큐에 작업이 enqueue됩니다.

### 🔗 참고 자료
- [Swift Evolution: SE-0316 - Global Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md)
- [WWDC 2021 - Swift concurrency: Update a sample app](https://developer.apple.com/videos/play/wwdc2021/10194/)

---

## 20. iOS의 동기화 도구들을 비교해주세요. (NSLock, os_unfair_lock, Mutex, actor 등) (Lv3)

### 💬 면접 답변

iOS에서 공유 상태를 보호하는 도구는 추상화 수준과 성능 특성이 다릅니다. **NSLock**은 pthread_mutex를 래핑한 범용 잠금으로, 경합 시 스레드를 블로킹하고 커널 컨텍스트 스위칭이 발생합니다. **os_unfair_lock**은 atomic 연산 기반의 저수준 잠금으로 가장 빠르지만 Swift에서 직접 쓰기 불편하고, **OSAllocatedUnfairLock(iOS 16+)** 또는 **Mutex(Swift 6)** 가 안전한 래퍼입니다. **actor**는 컴파일러가 안전성을 강제하고 스레드를 블로킹하지 않으며, async 환경에서 가장 권장되는 방식입니다.

### 📚 보충 설명

**잠금 계열**

| 도구 | 특징 | 적합한 상황 |
|---|---|---|
| NSLock | pthread_mutex 래핑, 커널 개입 | 레거시, 가독성 우선 |
| NSRecursiveLock | 같은 스레드의 중복 lock 허용 | 재귀 호출 구간 |
| NSCondition | 조건 만족 시 신호 전달 | 생산자-소비자 패턴 |
| os_unfair_lock | atomic 기반, 1~5ns, 우선순위 역전 방지 | 성능 극한, 짧은 임계구역 |
| OSAllocatedUnfairLock | os_unfair_lock의 Swift 안전 래퍼 (iOS 16+) | Swift 코드에서 고성능 잠금 |
| Mutex (Swift 6) | 표준 라이브러리 공식 Mutex | Swift 6 이상 신규 코드 |

**신호/원자 계열**

| 도구 | 특징 | 적합한 상황 |
|---|---|---|
| DispatchSemaphore | 카운팅 신호, 동시 진입 수 제한 | 최대 N개 동시 실행 제어 |
| pthread_rwlock | 다중 읽기 허용, 쓰기만 단독 | 읽기 많고 쓰기 드문 데이터 |
| Swift Atomics | 락-프리 원자 연산 | 단순 카운터·플래그 |

**성능 비교 (uncontended 기준)**

| 도구 | 대략적 호출 비용 | 경합 시 |
|---|---|---|
| os_unfair_lock | 1~5ns | 스레드 블로킹 |
| NSLock | 20~50ns | 스레드 블로킹 + 커널 개입 |
| DispatchQueue.async | 50~200ns | thread explosion 위험 |
| actor | 100~200ns | Task suspend (스레드 반납) |

단순 호출 비용은 os_unfair_lock이 가장 낮지만, 경합이 많아질수록 actor가 스레드를 낭비하지 않아 **시스템 전체 처리량**이 더 높아집니다. 성능 병목이 확인된 경우에만 프로파일링 후 교체하는 것이 원칙입니다.

**Swift Concurrency에서의 주의사항**

잠금을 쥔 채로 `await`을 만나면 위험합니다. suspend된 사이 다른 스레드가 같은 잠금을 획득하려다 데드락이 발생할 수 있습니다. async 코드와 잠금을 섞는다면 잠금 구간을 아주 좁게, await 없이 유지해야 합니다.

### 🔗 참고 자료
- [Apple - OSAllocatedUnfairLock](https://developer.apple.com/documentation/os/osallocatedunfairlock)
- [Swift Evolution: SE-0433 - Synchronization Module (Mutex)](https://github.com/apple/swift-evolution/blob/main/proposals/0433-mutex.md)
- [Swift Atomics GitHub](https://github.com/apple/swift-atomics)

---

## 21. withThrowingTaskGroup으로 병렬 작업의 결과 순서를 보존하는 방법은? (Lv3)

### 💬 면접 답변

`withThrowingTaskGroup`은 Task가 완료되는 순서대로 결과를 반환하기 때문에, 원본 순서와 다를 수 있습니다. **각 Task를 추가할 때 원본 배열의 인덱스를 함께 튜플로 넘기고**, 결과를 `(index, result)` 쌍으로 수집한 뒤 인덱스 기준으로 정렬하면 원래 순서를 복원할 수 있습니다. 에러가 하나라도 발생하면 그룹 전체가 취소되고 에러가 던져지므로, 부분 실패를 허용하려면 각 Task의 반환 타입을 `Result<T, Error>`로 감싸서 `withTaskGroup`(non-throwing)을 사용하면 됩니다.

### 📚 보충 설명

**순서 보존 패턴**

```swift
func uploadFiles(_ files: [File]) async throws -> [UploadResult] {
    try await withThrowingTaskGroup(of: (Int, UploadResult).self) { group in
        for (index, file) in files.enumerated() {
            group.addTask {
                let result = try await upload(file)
                return (index, result)   // 인덱스를 함께 반환
            }
        }
        var results = [(Int, UploadResult)]()
        for try await pair in group {
            results.append(pair)
        }
        return results
            .sorted { $0.0 < $1.0 }     // 인덱스 기준 정렬
            .map { $0.1 }
    }
}
```

**withTaskGroup vs withThrowingTaskGroup**

| | withTaskGroup | withThrowingTaskGroup |
|---|---|---|
| 에러 처리 | 각 Task가 에러를 던질 수 없음 | 각 Task가 throws 가능 |
| 에러 발생 시 | 해당 없음 | 나머지 Task 취소 후 에러 전파 |
| 부분 실패 허용 | Result 타입으로 감싸서 사용 | 기본 불가 (Result 래핑 필요) |

**에러 발생 시 내부 동작**

그룹 내 어느 Task 하나라도 에러를 던지는 순간, 그룹이 나머지 모든 Task에 취소 신호를 보내고 해당 에러를 호출부로 전파합니다. 취소는 협력적(cooperative, 강제 종료가 아니라 Task 스스로 취소 여부를 확인하고 멈추는 방식)으로 이루어집니다.

### 🔗 참고 자료
- [Apple - withThrowingTaskGroup](https://developer.apple.com/documentation/swift/withthrowingtaskgroup(of:returning:body:))
- [WWDC 2021 - Explore structured concurrency in Swift](https://developer.apple.com/videos/play/wwdc2021/10134/)

---

## 22. Structured Concurrency가 GCD 대비 갖는 이점은 무엇인가요? (Lv3)

### 💬 면접 답변

**구조적 동시성(Structured Concurrency)** 이란 Task의 생명주기가 코드 블록의 범위(scope)에 묶여있는 것을 의미합니다. `withThrowingTaskGroup` 블록이 끝나면 그 안에서 만들어진 모든 Task가 반드시 완료되거나 취소됩니다. 핵심 이점은 세 가지입니다. 첫째, **취소 전파가 자동**입니다. 부모 Task가 취소되면 자식 Task도 자동으로 취소 신호를 받습니다. 둘째, **에러 전파가 명확**합니다. 자식 Task의 에러가 `try await`을 통해 부모로 자연스럽게 전달됩니다. 셋째, **리소스 누수가 없습니다.** 블록이 끝나는 시점에 모든 자식 Task가 정리됩니다.

### 📚 보충 설명

**GCD의 비구조적 문제**

`DispatchQueue.async`로 던진 작업은 언제 끝나는지, 에러가 났는지 추적하려면 콜백, 플래그, 세마포어 등을 직접 관리해야 합니다. 작업이 살아있는 동안 관련 객체도 메모리에 유지해야 하며, 이를 깜빡하면 크래시나 메모리 누수가 발생합니다.

| 특성 | GCD | Structured Concurrency |
|---|---|---|
| 취소 전파 | 수동 (OperationQueue, 플래그) | 자동 (부모 → 자식) |
| 에러 전파 | 콜백·외부 변수 필요 | try await으로 자연 전파 |
| 리소스 정리 | 개발자가 추적 | 블록 종료 시 자동 보장 |
| Task 누출 | 가능 | 구조적으로 불가 |

### 🔗 참고 자료
- [Swift Evolution: SE-0304 - Structured Concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [WWDC 2021 - Explore structured concurrency in Swift](https://developer.apple.com/videos/play/wwdc2021/10134/)

---

## 23. Task 취소 메커니즘을 설명해주세요. (Task.isCancelled, CancellationError) (Lv3)

### 💬 면접 답변

Task 취소는 **협력적(cooperative) 메커니즘**입니다. 취소 신호가 오면 Task의 내부 취소 플래그가 `true`로 설정됩니다. Task 스스로 `Task.isCancelled`를 확인하거나 `try Task.checkCancellation()`을 호출해 `CancellationError`를 던지는 방식으로 취소에 응답합니다. 강제 종료가 아니라 Task 스스로 안전한 시점에 멈추기 때문에, 파일 핸들 닫기나 네트워크 연결 해제 같은 리소스 정리를 안전하게 할 수 있습니다. 취소는 부모에서 자식 Task로 전파되지만, 반대 방향(자식 → 부모)은 전파되지 않습니다.

### 📚 보충 설명

**두 가지 확인 방법의 차이**

`Task.isCancelled`는 취소 여부를 `Bool`로 반환합니다. 취소됐을 때 에러를 던지지 않고 부분 결과를 반환하거나 조용히 종료하고 싶을 때 씁니다.

`try Task.checkCancellation()`은 취소됐을 때 즉시 `CancellationError`를 던집니다. 취소 시 에러로 처리해야 할 때 씁니다.

Swift의 표준 async API들(`URLSession`, `Task.sleep` 등)은 내부적으로 취소 플래그를 확인해 자동으로 `CancellationError`를 던집니다. 직접 만든 오래 걸리는 함수에서도 중간중간 취소 여부를 확인해야 빠른 취소 반영이 가능합니다.

**CancellationError 구분 처리**

```swift
do {
    let result = try await longRunningTask()
} catch is CancellationError {
    // 사용자에게 "취소됨" 표시
} catch {
    // 실제 에러 처리
}
```

취소는 부모 → 자식 방향으로만 전파됩니다. 자식 하나가 취소됐다고 부모 전체가 취소되지는 않습니다.

### 🔗 참고 자료
- [Apple - Task.isCancelled](https://developer.apple.com/documentation/swift/task/iscancelled-swift.type.property)
- [Swift Evolution: SE-0304 - Structured Concurrency (Cancellation 섹션)](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)

---

## 24. Swift Concurrency 시대에도 DispatchQueue를 쓰는 이유는 무엇인가요? (Lv2)

### 💬 면접 답변

DispatchQueue가 Task보다 기술적으로 우월해서 쓰는 경우는 사실상 없습니다. 대부분은 세 가지 이유입니다. 첫째, 이미 GCD로 작성된 **레거시 코드베이스**에서 전면 교체가 현실적으로 어렵습니다. 둘째, **Combine의 스케줄러 인터페이스**(`receive(on:)`, `subscribe(on:)`)가 DispatchQueue를 받습니다. 셋째, **서드파티 라이브러리**의 콜백 실행 큐가 DispatchQueue를 요구하는 경우가 있습니다. 새로 작성하는 코드라면 Swift Concurrency를 쓰는 것이 Apple의 권장 방향이고, iOS 15+ 타겟에서는 DispatchQueue를 새로 쓸 이유가 거의 없습니다.

### 📚 보충 설명

**DispatchQueue가 여전히 쓰이는 구체적 상황**

레거시 호환: 기존 GCD 코드를 점진적으로 마이그레이션하는 과정에서 `withCheckedContinuation`으로 GCD 콜백을 async 함수로 감싸는 브릿지 패턴을 씁니다.

Combine 통합: `receive(on: DispatchQueue.main)`, `subscribe(on: DispatchQueue.global())` 같은 스케줄러 지정은 Combine 파이프라인에서 자연스럽게 DispatchQueue를 사용합니다.

런루프 사이클 타이밍 제어: `DispatchQueue.main.async`로 현재 런루프 사이클이 끝난 뒤 UI 업데이트를 예약하는 패턴은 UIKit에서 여전히 쓰입니다.

**GCD → async 마이그레이션 브릿지**

```swift
func legacyFetch(completion: @escaping (Result<Data, Error>) -> Void) { ... }

func fetch() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetch { result in
            continuation.resume(with: result)  // 정확히 한 번만 호출
        }
    }
}
```

주의: `continuation.resume`은 **정확히 한 번**만 호출해야 합니다. 두 번 호출하면 크래시, 호출하지 않으면 Task가 영원히 suspend됩니다.

### 🔗 참고 자료
- [WWDC 2021 - Swift concurrency: Update a sample app](https://developer.apple.com/videos/play/wwdc2021/10194/)
- [Apple - withCheckedContinuation](https://developer.apple.com/documentation/swift/withcheckedcontinuation(function:_:))

---

---

## 25. async/await이 컴파일러 레벨에서 어떻게 변환되나요? 기존 콜백과 어떻게 다른가요? (Lv3)

### 💬 면접 답변

`async` 함수는 컴파일러에 의해 **상태 머신(state machine)** 으로 변환됩니다. 각 `await` 지점이 상태 전환 포인트가 되어, 함수가 suspend될 때 현재 지역 변수와 실행 위치를 힙 메모리에 저장합니다. 이후 resume될 때 저장된 상태에서 이어서 실행합니다. 기존 콜백 방식은 클로저가 중첩되면서 스택 프레임이 쌓이고 캡처 리스트가 복잡해지는 반면, async/await은 상태를 힙에 보관하고 스택은 suspend 시점에 해제하므로 스택 오버플로우 위험이 없고 메모리 사용이 예측 가능합니다.

### 📚 보충 설명

**상태 머신 변환 원리**

`await` 2개가 있는 함수는 내부적으로 3단계 상태를 가집니다. 첫 번째 `await` 전, 첫 번째와 두 번째 `await` 사이, 두 번째 `await` 이후입니다. 함수가 suspend될 때 "현재 단계 번호"와 "그 시점의 지역 변수들"을 힙에 저장하고 스택을 비웁니다. resume될 때 저장된 단계부터 다시 시작합니다.

**콜백 방식과의 비교**

| | 콜백(completion handler) | async/await |
|---|---|---|
| 코드 구조 | 중첩 클로저 | 선형 코드 |
| 에러 전파 | 콜백마다 처리 | throws로 자연 전파 |
| 상태 저장 위치 | 스택 + 클로저 캡처 | 힙 (상태 머신) |
| 스택 오버플로우 위험 | 중첩 깊을수록 증가 | 없음 |
| 취소 처리 | 수동 플래그 관리 | Task.isCancelled |

수천 개의 Task가 동시에 suspend 상태로 존재해도 스택이 아닌 힙에 상태를 보관하므로 메모리 효율이 높습니다.

### 🔗 참고 자료
- [Swift Evolution: SE-0296 - async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [WWDC 2021 - Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)

---

## 26. Continuation이란 무엇이고, withCheckedContinuation은 언제 쓰나요? (Lv3)

### 💬 면접 답변

**Continuation(연속)** 이란 "지금 이 지점 이후에 실행될 나머지 코드 전체"를 하나의 값으로 캡처한 것입니다. async 함수가 suspend될 때 런타임은 "재개될 때 어디서부터 무엇을 실행해야 하는지"를 continuation 객체로 저장합니다. `withCheckedContinuation`은 콜백 기반의 레거시 API를 async 함수로 감쌀 때 씁니다. 콜백 안에서 `continuation.resume(returning:)` 또는 `continuation.resume(throwing:)`을 호출하면 suspended Task가 그 시점에 resume됩니다. "Checked"라는 이름은 resume이 정확히 한 번 호출됐는지를 런타임이 검사한다는 의미입니다.

### 📚 보충 설명

**Continuation 비유**

영화를 일시정지하면 TV가 "몇 분 몇 초에서 멈췄는지"를 기억합니다. 나중에 재생하면 정확히 그 지점부터 이어집니다. Continuation이 바로 이 "멈춘 지점 정보"입니다.

**withCheckedContinuation 사용 패턴**

```swift
// 콜백 기반 레거시 API
func legacyFetch(completion: @escaping (Result<Data, Error>) -> Void) { ... }

// async 래퍼
func fetch() async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        legacyFetch { result in
            continuation.resume(with: result)  // 정확히 한 번만 호출
        }
    }
}
```

**Checked vs Unsafe 버전**

| | withCheckedContinuation | withUnsafeContinuation |
|---|---|---|
| resume 횟수 검사 | 런타임이 감시 | 검사 없음 |
| 위반 시 | 경고/크래시 | 미정의 동작 |
| 성능 | 약간의 오버헤드 | 더 빠름 |
| 권장 상황 | 일반적인 사용 | 성능이 극히 중요한 구간 |

**주의사항**

`continuation.resume`은 **정확히 한 번**만 호출해야 합니다. 한 번도 안 부르면 Task가 영원히 suspend 상태로 메모리에 남습니다. 두 번 부르면 크래시가 납니다.

### 🔗 참고 자료
- [Apple - withCheckedContinuation](https://developer.apple.com/documentation/swift/withcheckedcontinuation(function:_:))
- [Swift Evolution: SE-0300 - Continuations](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md)

---

## 27. Swift Concurrency의 cooperative thread pool은 몇 개의 스레드를 사용하고, 왜 그 수로 제한하나요? (Lv3)

### 💬 면접 답변

Swift의 cooperative thread pool은 기본적으로 **CPU 활성 코어 수**만큼만 스레드를 생성합니다. 제한하는 이유는 **컨텍스트 스위칭 비용**을 없애기 위해서입니다. 스레드가 코어 수보다 많아지면 OS가 스레드를 번갈아 실행하면서 컨텍스트 스위칭(현재 스레드 상태를 저장하고 다른 스레드로 전환하는 작업)이 발생하고, 이 비용이 누적되면 오히려 성능이 떨어집니다. Task는 `await` 지점에서 스레드를 반납하기 때문에, 스레드 수가 코어 수와 같아도 수천 개의 Task를 효율적으로 처리할 수 있습니다.

### 📚 보충 설명

**GCD와의 비교**

GCD에서는 작업이 블로킹되면 OS가 새 스레드를 생성해 대응합니다. 이것이 반복되면 스레드가 수십~수백 개로 불어나는 thread explosion이 발생합니다. 스레드 하나당 512KB 스택이 소비되므로 메모리 낭비도 심해집니다.

Swift Concurrency는 Task가 블로킹 없이 `await`으로 스레드를 반납하기 때문에, 코어 수만큼의 스레드로 충분합니다.

| | GCD | Swift Concurrency |
|---|---|---|
| 스레드 수 | 작업 증가 시 동적으로 증가 | CPU 코어 수로 고정 |
| 블로킹 대기 | 스레드가 잠들어 낭비 | Task suspend, 스레드 반납 |
| 컨텍스트 스위칭 | 스레드 많을수록 증가 | 최소화 |

**Thread pool 고갈 주의**

async 코드 안에서 `DispatchSemaphore.wait()`이나 `NSLock.lock()`처럼 스레드를 블로킹하는 코드를 쓰면, cooperative thread pool의 스레드가 잠들어 버립니다. 나머지 Task들이 실행될 스레드가 없어지는 **thread pool 고갈** 문제가 발생합니다. async 환경에서 블로킹 API를 피해야 하는 이유입니다.

### 🔗 참고 자료
- [WWDC 2021 - Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)

---

## 28. async 함수가 suspend되는 시점은 정확히 언제인가요? (Lv3)

### 💬 면접 답변

async 함수는 `await` 키워드를 만났다고 해서 항상 suspend되지는 않습니다. **실제로 기다릴 필요가 있을 때만** suspend됩니다. 구체적으로 세 가지 시점입니다. 첫째, `await`으로 호출한 async 함수가 즉시 결과를 반환할 수 없을 때입니다. 둘째, actor의 executor가 현재 다른 작업을 처리 중이어서 진입을 기다려야 할 때입니다. 셋째, 현재 컨텍스트와 호출 대상의 컨텍스트가 달라 전환이 필요할 때(예: 일반 Task에서 `@MainActor` 함수 호출)입니다. 결과가 이미 준비되어 있다면 `await`이 있어도 suspend 없이 바로 실행을 이어갑니다.

### 📚 보충 설명

**`await`은 "멈출 수도 있다"는 표시**

`await`은 무조건 suspend되는 명령이 아니라, 런타임에게 "여기서 suspend해도 된다"는 허가를 주는 것입니다. 실제로 멈출지는 런타임이 결정합니다.

**세 가지 suspend 시점 상세**

결과가 아직 없을 때: 네트워크 응답처럼 시간이 걸리는 작업을 `await`하면 응답이 올 때까지 suspend됩니다. 반면 캐시에 결과가 있어 즉시 반환 가능하다면 suspend 없이 진행됩니다.

actor 진입 대기: 다른 actor의 메서드를 `await`으로 호출할 때, 해당 actor가 현재 다른 Task를 처리 중이면 suspend됩니다. actor가 비어있으면 바로 진입하므로 suspend되지 않습니다.

컨텍스트 전환: 일반 Task에서 `@MainActor` 함수를 `await`하면 메인 스레드로 전환하는 과정에서 suspend됩니다.

**suspend는 비용**

상태를 힙에 저장하고, 스케줄링하고, resume하는 과정이 필요합니다. Swift 런타임이 "정말 기다릴 필요가 있을 때만" suspend하도록 최적화하는 이유이며, 개발자도 불필요한 `await`을 남발하지 않는 것이 좋습니다.

### 🔗 참고 자료
- [Swift Evolution: SE-0296 - async/await](https://github.com/apple/swift-evolution/blob/main/proposals/0296-async-await.md)
- [WWDC 2021 - Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)

---

## 29. Exponential Backoff + Jitter (Lv2)

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

## 30. Full Jitter, Equal Jitter, Decorrelated Jitter 비교 (Lv3)

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

## 31. Thundering Herd Problem과 Jitter의 완화 원리 (Lv3)

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

> Combine의 SwiftUI 결합 패턴은 [06_UIKit_SwiftUI.md](./06_UIKit_SwiftUI.md), 동시성 원리는 [01_CS_Fundamentals.md](./01_CS_Fundamentals.md)를 참고하세요.
