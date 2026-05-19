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

> Combine의 SwiftUI 결합 패턴은 [06_UIKit_SwiftUI.md](./06_UIKit_SwiftUI.md), 동시성 원리는 [01_CS_Fundamentals.md](./01_CS_Fundamentals.md)를 참고하세요.
