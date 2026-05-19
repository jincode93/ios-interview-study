---
layout: default
title: 01. 컴퓨터 과학 기초
nav_order: 1
---

# 01. 컴퓨터 과학 기초

> iOS 개발자에게 필요한 CS 기초 — 컴퓨터 구조, 운영체제, 자료구조, 알고리즘, 보안 기초.

---

## 1. CPU, RAM, 저장 장치의 역할과 상호작용을 설명해주세요. (Lv0)

### 💬 면접 답변

CPU는 명령어를 실제로 실행하는 연산 장치이고, RAM은 실행 중인 코드와 데이터를 빠르게 읽고 쓸 수 있는 휘발성 메모리, 저장 장치는 전원이 꺼져도 데이터를 유지하는 영속 저장소입니다. 앱이 실행되면 저장 장치에 있는 앱 바이너리와 리소스가 RAM으로 로드되고, CPU가 RAM의 명령어를 읽어 실행하면서 필요한 데이터를 RAM에서 가져오는 구조입니다. iOS에서는 RAM이 제한적이라서 메모리 부족 시 백그라운드 앱부터 종료되고, 이게 앱 개발에서 메모리 관리를 중요하게 만드는 이유입니다.

### 📚 보충 설명

**앱 실행 흐름**
1. 저장 장치(NAND Flash)에서 앱 바이너리 읽기
2. RAM에 코드 세그먼트, 데이터 세그먼트 적재
3. CPU의 PC(Program Counter)가 가리키는 RAM 주소의 명령어를 fetch
4. decode → execute → writeback
5. UI를 그릴 때는 GPU로 데이터 전달 (iOS는 Unified Memory)

**iOS의 Unified Memory Architecture**

A-시리즈 칩부터 적용된 UMA는 CPU와 GPU가 같은 메모리 풀을 공유합니다. 데스크톱 GPU처럼 별도 VRAM에 데이터를 복사할 필요가 없어, Metal 프레임워크를 사용할 때 zero-copy 같은 최적화가 가능합니다.

**iOS의 메모리 압력 단계**
- Normal → Warning → Urgent → Critical
- Critical 단계에서는 백그라운드 앱이 강제 종료(Jetsam). 포그라운드 앱도 `didReceiveMemoryWarning`을 받습니다.

### 🔗 참고 자료
- [Apple - About the iOS Memory Management](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html)
- [Apple - Metal Programming Guide](https://developer.apple.com/library/archive/documentation/Miscellaneous/Conceptual/MetalProgrammingGuide/Introduction/Introduction.html)

---

## 2. 캐시(Cache)의 개념과 캐시 지역성(Locality)에 대해 설명해주세요. (Lv0)

### 💬 면접 답변

캐시는 CPU와 RAM 사이의 속도 차이를 줄이기 위한 작고 빠른 메모리로, L1·L2·L3 단계로 구성됩니다. CPU가 데이터를 요청하면 캐시에서 먼저 찾고, 있으면 캐시 히트(빠름), 없으면 캐시 미스로 RAM에서 가져옵니다. 캐시가 효율적으로 동작하려면 두 가지 지역성을 잘 활용해야 합니다 — **시간적 지역성**(최근 접근한 데이터는 곧 다시 접근될 가능성이 높음)과 **공간적 지역성**(접근한 데이터 근처가 곧 접근될 가능성이 높음)입니다. 그래서 2차원 배열을 순회할 때 행 우선으로 도는 게 열 우선으로 도는 것보다 캐시 친화적입니다.

### 📚 보충 설명

**캐시 친화 vs 비친화 예제**

```swift
// 캐시 친화 (공간적 지역성 활용)
for i in 0..<rows {
    for j in 0..<cols {
        sum += matrix[i][j]   // 같은 행의 연속된 메모리
    }
}

// 캐시 비친화
for j in 0..<cols {
    for i in 0..<rows {
        sum += matrix[i][j]   // 매번 다른 행 → 캐시 미스 증가
    }
}
```

**계층 구조와 접근 시간 (대략)**

| 계층 | 크기 | 접근 시간 |
|------|------|----------|
| 레지스터 | 수십 B | < 1ns |
| L1 캐시 | 수십 KB | ~1ns |
| L2 캐시 | 수백 KB | ~3ns |
| L3 캐시 | 수 MB | ~10ns |
| RAM | GB | ~100ns |
| SSD | TB | ~100μs |

이 1000배 단위의 격차 때문에 캐시 미스를 줄이는 것이 성능에 결정적입니다.

### 🔗 참고 자료
- [Apple - Optimization Guide for Apple Silicon](https://developer.apple.com/documentation/apple-silicon/cpu-optimization-guide)

---

## 3. CPU 아키텍처 종류와 iOS가 사용하는 아키텍처는? (Lv0)

### 💬 면접 답변

CPU 아키텍처는 크게 명령어 집합이 단순하고 파이프라인 효율이 좋은 RISC 계열과, 복잡한 명령어를 가진 CISC 계열로 나뉩니다. iOS 기기는 **ARM 기반 RISC 아키텍처**, 더 정확히는 Apple Silicon(A-시리즈, M-시리즈)을 사용합니다. ARM은 전력 효율이 좋아 모바일에 적합하고, 데스크톱은 전통적으로 x86(CISC) 계열을 써왔습니다. iOS 시뮬레이터는 호스트 머신의 아키텍처에서 동작하기 때문에 Intel Mac이면 x86_64, Apple Silicon Mac이면 arm64로 빌드되며, 이로 인해 실제 기기와 미세한 동작 차이가 날 수 있습니다.

### 📚 보충 설명

**Apple SoC의 구성**

iOS 기기의 AP(Application Processor)는 단순한 CPU가 아니라 SoC(System on a Chip)입니다.
- CPU 클러스터 (성능 코어 + 효율 코어)
- GPU
- Neural Engine (ML 가속)
- ISP (이미지 시그널 프로세서)
- Secure Enclave (보안 전용 코프로세서)
- Memory Controller

SoC는 이 모든 것을 하나의 다이에 통합해 전력·공간·대역폭 측면에서 유리합니다.

**시뮬레이터 vs 실기기 차이**
- 시뮬레이터는 호스트 macOS의 라이브러리를 일부 공유. UIKit 동작은 같지만 GPU/Metal 동작, 메모리 압력, 위치/센서 기능은 다름.
- 시뮬레이터는 메모리 압력이 거의 없어 메모리 누수가 잘 안 드러남 → Instruments는 실기기에서 측정해야 정확.

### 🔗 참고 자료
- [Apple Silicon - Developer Documentation](https://developer.apple.com/documentation/apple-silicon)

---

## 4. 프로세스와 스레드의 차이는 무엇인가요? iOS에서는 어떻게 관리되나요? (Lv0)

### 💬 면접 답변

프로세스는 운영체제로부터 자원을 할당받은 실행 단위로, 자체 메모리 공간을 가집니다. 스레드는 그 프로세스 안에서 실행되는 흐름의 단위이고, 같은 프로세스의 스레드들은 메모리(특히 힙)를 공유합니다. iOS는 보안 모델상 한 앱이 하나의 프로세스로 격리되어 실행되므로, 앱 내 동시 작업은 거의 항상 스레드(또는 그 위의 GCD/Operation/Task)로 처리합니다. 메인 스레드는 UI 업데이트 전용이고, 무거운 작업은 백그라운드 스레드에서 처리한 뒤 결과를 메인으로 디스패치하는 패턴이 기본입니다.

### 📚 보충 설명

**프로세스 vs 스레드 비교**

| 항목 | 프로세스 | 스레드 |
|------|---------|--------|
| 메모리 공간 | 독립 | 공유 (코드/힙/데이터) |
| 컨텍스트 스위칭 비용 | 큼 | 작음 |
| 통신 방법 | IPC (파이프, 소켓, 공유 메모리) | 공유 변수 (동기화 필요) |
| 안전성 | 한 프로세스 죽어도 다른 프로세스 무관 | 한 스레드 죽으면 프로세스 전체 영향 |

**메인 스레드에서 무거운 작업을 하면**
- UI 이벤트 처리(터치, 스크롤)와 렌더링이 16.6ms(60fps) 또는 8.3ms(120fps ProMotion) 내에 끝나야 함
- 메인 스레드가 막히면 워치독(Watchdog)이 앱을 강제 종료(0x8badf00d 크래시)

**iOS의 스레드 추상화**
1. **NSThread / pthread**: 저수준, 직접 관리. 거의 안 씀.
2. **GCD (Grand Central Dispatch)**: 큐 기반의 디스패치, 시스템이 스레드 풀 관리.
3. **OperationQueue**: GCD 위의 객체 지향 추상. 의존성, 취소, 우선순위 지원.
4. **Swift Concurrency (async/await, Actor)**: iOS 13+, 가장 권장되는 현대적 방식.

### 🔗 참고 자료
- [Apple - Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)

---

## 5. iOS의 샌드박스(Sandbox)란 무엇인가요? 앱 간 데이터 공유는 어떻게 하나요? (Lv0)

### 💬 면접 답변

샌드박스는 각 앱이 자기 전용의 파일 시스템 공간과 자원에만 접근할 수 있도록 OS가 격리한 보안 메커니즘입니다. 한 앱이 다른 앱의 데이터를 임의로 읽거나 시스템 영역을 건드릴 수 없어 보안과 안정성이 보장됩니다. 앱 간 데이터를 공유해야 할 때는 **URL Scheme**으로 간단한 메시지를 전달하거나, 같은 개발자 팀의 앱끼리는 **App Group**을 통해 공용 컨테이너에 접근하는 방식을 씁니다. 대용량/구조화 데이터는 App Group의 공유 컨테이너에 파일이나 SQLite를 두고, 위젯·익스텐션과 메인 앱 사이 통신에도 같은 메커니즘이 쓰입니다.

### 📚 보충 설명

**샌드박스 디렉터리 구조**

```
<AppDir>/AppName.app          ← 앱 번들 (읽기 전용)
~/Documents                    ← 사용자 생성 데이터, iCloud 백업 ⭕
~/Library/Application Support  ← 앱이 만든 데이터, 백업 ⭕
~/Library/Caches               ← 캐시, 백업 ❌, OS가 임의 삭제 가능
~/tmp                          ← 임시, OS가 정리
```

**App Group으로 데이터 공유**

1. Apple Developer에서 App Group 식별자 생성 (예: `group.com.company.myapp`)
2. 메인 앱과 익스텐션의 Capabilities에 추가
3. 공유 컨테이너 URL로 접근

```swift
let url = FileManager.default
    .containerURL(forSecurityApplicationGroupIdentifier: "group.com.company.myapp")
// url 아래에 파일 저장 → 위젯/익스텐션에서 같은 url로 접근
```

**URL Scheme**

```swift
// Info.plist에 CFBundleURLSchemes 등록
let url = URL(string: "myapp://detail?id=123")!
UIApplication.shared.open(url)
```

iOS 9+부터는 `LSApplicationQueriesSchemes`에 등록해야 다른 앱이 설치되어 있는지 확인 가능합니다. URL Scheme은 같은 이름의 스킴을 가진 악성 앱이 가로챌 수 있어, 보안이 중요한 경우 **Universal Link**가 권장됩니다.

### 🔗 참고 자료
- [Apple - App Sandbox in Depth](https://developer.apple.com/documentation/security/app_sandbox)
- [Apple - Sharing Data Between Apps Using App Groups](https://developer.apple.com/documentation/xcode/configuring-app-groups)

---

## 6. 시간 복잡도(Big-O)에 대해 설명해주세요. (Lv0)

### 💬 면접 답변

Big-O는 입력 크기 n이 커질 때 알고리즘의 실행 시간이 어떤 비율로 증가하는지를 표현하는 점근 표기법입니다. 상수항과 낮은 차수의 항은 무시하고 가장 영향이 큰 항만 남깁니다. 자주 등장하는 것은 O(1) 상수, O(log n) 로그(이진 탐색), O(n) 선형, O(n log n) 효율적인 정렬, O(n²) 이중 루프, O(2ⁿ) 지수 — 입니다. 예를 들어 1만 명의 사용자 목록에서 정렬되지 않은 배열을 선형 탐색하면 평균 5000번, 정렬한 뒤 이진 탐색하면 약 14번이라 큰 차이가 납니다.

### 📚 보충 설명

**복잡도 비교 (n=1,000,000)**

| 복잡도 | 연산 횟수 (대략) |
|--------|-----------------|
| O(1) | 1 |
| O(log n) | 20 |
| O(n) | 1,000,000 |
| O(n log n) | 20,000,000 |
| O(n²) | 1,000,000,000,000 (1조) |

**최선/평균/최악**

같은 알고리즘이라도 입력에 따라 다릅니다.
- Quick Sort: 평균 O(n log n), 최악 O(n²) (이미 정렬된 입력 + 나쁜 피벗)
- Hash Lookup: 평균 O(1), 최악 O(n) (충돌이 모두 같은 버킷에 몰릴 때)

**공간 복잡도**

iOS는 메모리가 제한적이라 시간만큼 공간 복잡도도 중요합니다. 예를 들어 Merge Sort는 시간 O(n log n)이지만 추가 O(n) 공간이 필요해서, 메모리 압력이 있을 땐 in-place인 Heap Sort나 Quick Sort가 유리할 수 있습니다.

### 🔗 참고 자료
- [Algorithms, 4th Edition - Sedgewick](https://algs4.cs.princeton.edu/home/)

---

## 7. 자주 사용되는 정렬 알고리즘과 시간 복잡도를 설명해주세요. (Lv0)

### 💬 면접 답변

기본 정렬은 O(n²)인 버블·선택·삽입 정렬과, O(n log n)인 병합·퀵·힙 정렬로 나뉩니다. 삽입 정렬은 거의 정렬된 데이터에서 O(n)에 가까운 성능을 보여 부분 정렬 최적화에 자주 쓰이고, 병합 정렬은 안정 정렬이면서 항상 O(n log n)을 보장하지만 추가 메모리가 필요합니다. 퀵 정렬은 평균은 빠르지만 최악은 O(n²)이라 피벗 선택이 중요합니다. Swift 표준 라이브러리의 `sort()`는 **introsort**(퀵 + 힙 + 삽입)와 유사한 하이브리드 알고리즘을 씁니다.

### 📚 보충 설명

**정렬 알고리즘 비교**

| 알고리즘 | 평균 | 최악 | 공간 | 안정 |
|---------|------|------|------|------|
| 버블 정렬 | O(n²) | O(n²) | O(1) | ⭕ |
| 선택 정렬 | O(n²) | O(n²) | O(1) | ❌ |
| 삽입 정렬 | O(n²) | O(n²) | O(1) | ⭕ |
| 병합 정렬 | O(n log n) | O(n log n) | O(n) | ⭕ |
| 퀵 정렬 | O(n log n) | O(n²) | O(log n) | ❌ |
| 힙 정렬 | O(n log n) | O(n log n) | O(1) | ❌ |

**안정 정렬(Stable Sort)이란?**

같은 키 값을 가진 원소들의 상대적 순서가 정렬 후에도 유지되는 정렬입니다. 다중 기준 정렬에서 중요합니다 — 예를 들어 "나이로 정렬한 결과를 다시 이름으로 정렬"하면 같은 이름끼리는 나이 순서가 유지됩니다.

> Swift 표준 라이브러리의 `sorted()`/`sort()`는 Swift 5부터 [안정 정렬임이 보장](https://github.com/apple/swift-evolution/blob/main/proposals/0372-document-sorting-as-stable.md)됩니다. (SE-0372)

### 🔗 참고 자료
- [Swift Evolution: SE-0372 - Document sorting as stable](https://github.com/apple/swift-evolution/blob/main/proposals/0372-document-sorting-as-stable.md)
- [Swift Standard Library - sort](https://developer.apple.com/documentation/swift/array/sort())

---

## 8. 이진 탐색(Binary Search)의 원리와 시간 복잡도는? (Lv0)

### 💬 면접 답변

이진 탐색은 **정렬된 배열에서** 중간 원소를 보고 찾는 값이 더 큰지 작은지에 따라 탐색 범위를 절반씩 줄여나가는 알고리즘으로, 시간 복잡도는 O(log n)입니다. 반드시 정렬이 전제되어야 하므로, 정렬이 안 된 배열에서는 정렬에 O(n log n)이 들기 때문에 단발 검색이라면 그냥 선형 탐색 O(n)이 빠를 수 있습니다. 여러 번 검색해야 한다면 정렬해두고 이진 탐색하는 게 유리합니다.

### 📚 보충 설명

```swift
func binarySearch<T: Comparable>(_ arr: [T], target: T) -> Int? {
    var lo = 0, hi = arr.count - 1
    while lo <= hi {
        let mid = (lo + hi) / 2
        if arr[mid] == target { return mid }
        else if arr[mid] < target { lo = mid + 1 }
        else { hi = mid - 1 }
    }
    return nil
}
```

**Swift 표준 라이브러리 활용**

```swift
// 정렬된 컬렉션에서 삽입 위치 찾기
let sorted = [1, 3, 5, 7, 9]
let idx = sorted.firstIndex { $0 >= 5 }  // 2
```

`Collection`에 직접적인 binarySearch는 없지만, `partitioningIndex(where:)` (swift-algorithms) 같은 함수로 활용할 수 있습니다.

### 🔗 참고 자료
- [swift-algorithms - Apple Open Source](https://github.com/apple/swift-algorithms)

---

## 9. 동적 프로그래밍(Dynamic Programming)이란? (Lv0)

### 💬 면접 답변

동적 프로그래밍은 큰 문제를 작은 부분 문제로 나누고, 그 부분 문제의 답을 저장해두었다가 재활용하는 기법입니다. **최적 부분 구조**(전체 최적해가 부분 최적해의 조합)와 **중복되는 부분 문제**가 있어야 적용 가능합니다. 구현 방식은 재귀 + 메모이제이션(Top-down)과 반복문 + 테이블(Bottom-up, 타뷸레이션) 두 가지입니다. 피보나치 수열이 고전적인 예시인데, 순수 재귀는 O(2ⁿ)이지만 DP로 풀면 O(n)이 됩니다.

### 📚 보충 설명

**피보나치 — 세 가지 방식**

```swift
// 1. 순수 재귀 — O(2ⁿ)
func fib(_ n: Int) -> Int {
    if n < 2 { return n }
    return fib(n-1) + fib(n-2)
}

// 2. 메모이제이션 (Top-down) — O(n)
var memo: [Int: Int] = [:]
func fib(_ n: Int) -> Int {
    if n < 2 { return n }
    if let cached = memo[n] { return cached }
    let v = fib(n-1) + fib(n-2)
    memo[n] = v
    return v
}

// 3. 타뷸레이션 (Bottom-up) — O(n), O(1) 공간
func fib(_ n: Int) -> Int {
    if n < 2 { return n }
    var a = 0, b = 1
    for _ in 2...n { (a, b) = (b, a + b) }
    return b
}
```

**DP가 적용되는 대표 문제**
- 0/1 Knapsack
- LCS (최장 공통 부분 수열)
- LIS (최장 증가 부분 수열)
- Edit Distance
- Coin Change

---

## 10. 스택과 큐는 어떤 자료구조이며, iOS의 어디서 쓰이나요? (Lv0)

### 💬 면접 답변

스택은 **LIFO**(Last-In-First-Out), 큐는 **FIFO**(First-In-First-Out) 구조입니다. iOS에서 가장 대표적인 예가 `UINavigationController`인데, push/pop으로 뷰 컨트롤러를 스택처럼 쌓고 꺼내며 화면을 관리합니다. 음악 앱의 재생 대기열은 큐로 모델링하는 게 자연스럽고, 메시지 큐, GCD의 DispatchQueue도 큐 자료구조에 기반합니다. 셔플이나 우선순위 재생을 구현하려면 일반 큐 대신 우선순위 큐(Heap)나 deque 같은 변형이 필요합니다.

### 📚 보충 설명

**Stack 구현 예시**

```swift
struct Stack<T> {
    private var items: [T] = []
    var isEmpty: Bool { items.isEmpty }
    mutating func push(_ item: T) { items.append(item) }
    mutating func pop() -> T? { items.popLast() }
    func peek() -> T? { items.last }
}
```

**UITableView의 셀 재사용 풀**

`dequeueReusableCell(withIdentifier:)`는 식별자별로 셀을 관리해야 하는데, 빠른 조회를 위해 Dictionary 기반으로 구현됩니다. 화면 밖으로 사라진 셀을 풀에 다시 넣고, 새로 보일 셀이 필요할 때 풀에서 꺼냅니다.

**SwiftUI의 NavigationStack**

iOS 16+의 `NavigationStack`은 내부적으로 경로 배열을 스택처럼 관리합니다. `path` 바인딩으로 배열을 직접 조작해 여러 화면을 한 번에 push/pop 할 수 있어 딥링크 처리에 유리합니다.

### 🔗 참고 자료
- [Apple - NavigationStack](https://developer.apple.com/documentation/swiftui/navigationstack)

---

## 11. Array와 Linked List의 차이는? (Lv0)

### 💬 면접 답변

Array는 메모리에 연속적으로 할당되어 **인덱스 접근이 O(1)** 로 빠르지만, 중간 삽입/삭제는 뒤의 원소를 모두 이동시켜야 해서 O(n)입니다. Linked List는 노드를 포인터로 연결한 구조라 **중간 삽입/삭제가 O(1)** (노드 위치만 알면)이지만, 인덱스 접근은 처음부터 따라가야 해서 O(n)입니다. 또 Array는 캐시 친화적이고 메모리 오버헤드가 적은 반면, Linked List는 노드마다 포인터 메모리가 추가로 들고 캐시 미스가 잦습니다. Swift의 `Array`는 값 타입이지만 Copy-on-Write로 최적화되어 있어 실제로는 매우 효율적입니다.

### 📚 보충 설명

**시간 복잡도 비교**

| 연산 | Array | Linked List |
|------|-------|-------------|
| 인덱스 접근 | O(1) | O(n) |
| 끝에 추가 | O(1) amortized | O(1) (tail 포인터 있을 때) |
| 끝에서 제거 | O(1) | O(n) (singly) / O(1) (doubly + tail) |
| 중간 삽입/삭제 (위치 알 때) | O(n) | O(1) |
| 중간 삽입/삭제 (탐색 필요) | O(n) | O(n) |

**Copy-on-Write가 주는 효과**

```swift
var a = [Int](repeating: 0, count: 1_000_000)
var b = a              // 버퍼 공유 → O(1)
b[0] = 1               // 이때 실제 복사 발생 (b만 새 버퍼)
```

Swift `Array`는 사용 패턴에서 LinkedList의 장점을 대부분 흡수해, 실무에서는 LinkedList를 직접 구현할 일이 거의 없습니다. (예외: LRU 캐시처럼 자료구조 자체가 요구되는 경우)

### 🔗 참고 자료
- [Swift Docs - Array](https://developer.apple.com/documentation/swift/array)

---

## 12. 직렬화(Serialization)와 역직렬화(Deserialization)는? (Lv0)

### 💬 면접 답변

직렬화는 메모리 상의 객체를 파일·네트워크로 전송 가능한 바이트 형태(JSON, XML, Protocol Buffers 등)로 변환하는 과정이고, 역직렬화는 그 반대입니다. iOS에서는 보통 서버와 JSON으로 통신하기 때문에 `Codable`을 활용해 Swift 객체와 JSON 사이를 양방향 변환합니다. JSON은 사람이 읽기 쉽고 디버깅이 편리하지만, Protocol Buffers나 MessagePack 같은 바이너리 포맷이 크기나 파싱 속도 면에서 더 효율적이라 성능이 중요한 곳에서는 선택지가 됩니다.

### 📚 보충 설명

**JSON vs Protocol Buffers**

| 항목 | JSON | Protocol Buffers |
|------|------|------------------|
| 가독성 | ⭕ | ❌ (바이너리) |
| 크기 | 큼 | 작음 (3~10배) |
| 파싱 속도 | 느림 | 빠름 (2~10배) |
| 스키마 | 없음 | 있음 (.proto) |
| 호환성 | 유연 | 엄격하지만 버전 관리 명확 |

**JSON이 여전히 우세한 이유**
- 거의 모든 언어가 기본 지원
- 디버깅·로깅 쉬움
- 웹 표준
- 모바일 통신량 정도는 JSON으로도 충분히 빠름

성능이 critical(예: 실시간 통신, 대용량 메시지)한 경우에만 Protocol Buffers/Flatbuffers 검토.

### 🔗 참고 자료
- [Apple - Encoding and Decoding Custom Types](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)

---

## 13. 암호화의 종류와 iOS에서의 보안 모범 사례는? (Lv0)

### 💬 면접 답변

암호화는 크게 **대칭키 암호화**(같은 키로 암복호화, AES가 대표적, 빠름)와 **비대칭키 암호화**(공개키/개인키 쌍, RSA·ECC, 느리지만 키 교환에 적합)로 나뉩니다. HTTPS는 핸드셰이크에서 비대칭으로 안전하게 세션 키를 교환한 뒤, 실제 데이터는 대칭키로 암호화하는 하이브리드 방식입니다. iOS에서 민감한 데이터(토큰, 비밀번호)는 반드시 **Keychain**에 저장해야 하고, UserDefaults는 plist로 평문 저장되어 적합하지 않습니다. 또 비밀번호 자체는 절대 저장하지 않고 **해시 + Salt**로만 다뤄야 합니다.

### 📚 보충 설명

**해싱과 암호화의 차이**

- 암호화: 키를 알면 **복호화 가능** (양방향)
- 해싱: 결과로부터 원본을 **복원할 수 없음** (단방향)

비밀번호는 해싱(SHA-256, bcrypt, scrypt, Argon2) + Salt로 저장해서, 데이터베이스가 유출되어도 원본을 복원할 수 없게 합니다.

**Keychain의 장점**
- 디바이스의 Secure Enclave가 보호
- iCloud Keychain 동기화 지원
- 앱 삭제 후에도 데이터 유지(설정 가능)
- 접근 제어(Face ID, Touch ID, passcode 요구)

**HTTPS와 인증서 검증, Certificate Pinning**

기본 HTTPS는 시스템이 신뢰하는 CA를 통해 인증서 체인을 검증합니다. 다만 신뢰하는 CA 중 하나가 잘못된 인증서를 발급하면 중간자 공격이 가능하므로, 금융처럼 보안이 중요한 앱은 **Certificate Pinning**(특정 인증서나 공개키만 신뢰)을 추가로 적용합니다.

```swift
final class PinningDelegate: NSObject, URLSessionDelegate {
    func urlSession(_ session: URLSession,
                    didReceive challenge: URLAuthenticationChallenge,
                    completionHandler: @escaping (URLSession.AuthChallengeDisposition,
                                                  URLCredential?) -> Void) {
        guard let trust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil); return
        }
        // 핀 비교 후 결정
        completionHandler(.useCredential, URLCredential(trust: trust))
    }
}
```

### 🔗 참고 자료
- [Apple - Keychain Services](https://developer.apple.com/documentation/security/keychain_services)
- [Apple - Performing Manual Server Trust Authentication](https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge/performing_manual_server_trust_authentication)
- [Apple - CryptoKit](https://developer.apple.com/documentation/cryptokit)

---

## 14. OOP(객체지향 프로그래밍)의 핵심 개념을 설명해주세요. (Lv2)

### 💬 면접 답변

OOP의 네 가지 핵심은 **캡슐화, 상속, 다형성, 추상화**입니다. 캡슐화는 데이터와 동작을 한 단위로 묶고 외부에서는 인터페이스로만 다루게 하는 것이고, 상속은 공통 동작을 부모 클래스에서 정의해 자식이 재사용하는 것입니다. 다형성은 같은 메시지를 받았을 때 객체의 실제 타입에 따라 다르게 동작하는 것이고, 추상화는 복잡한 내부 구현을 숨기고 본질적인 인터페이스만 노출하는 것입니다. Swift에서는 클래스 상속보다 프로토콜과 컴포지션을 우선시하는 **POP(프로토콜 지향)** 스타일이 더 권장됩니다.

### 📚 보충 설명

**캡슐화 vs 정보 은닉**

비슷하지만 미묘하게 다릅니다.
- **캡슐화**: 관련된 데이터와 동작을 하나의 단위(객체)로 묶는 것
- **정보 은닉**: 외부에서 알 필요 없는 내부 구현을 감추는 것 (접근 제어자가 도구)

캡슐화는 설계 원칙, 정보 은닉은 그 결과의 한 측면이라고 볼 수 있습니다.

**상속의 단점**

- 강한 결합: 부모 변경이 자식에 영향
- 단일 상속 제약
- 깊은 상속 트리는 디버깅·이해 어려움
- "is-a"가 아닌 단순 코드 재사용 목적이면 컴포지션이 더 나음

**다형성 예시**

```swift
// 정적 다형성 (오버로딩, 제네릭)
func display<T: CustomStringConvertible>(_ item: T) { print(item) }

// 동적 다형성 (오버라이드, 프로토콜)
protocol Shape { func area() -> Double }
struct Circle: Shape { let r: Double; func area() -> Double { .pi * r * r } }
struct Rect: Shape   { let w, h: Double; func area() -> Double { w * h } }

let shapes: [Shape] = [Circle(r: 1), Rect(w: 2, h: 3)]
shapes.forEach { print($0.area()) }
```

### 🔗 참고 자료
- [Apple - Designing for Inheritance vs Composition](https://developer.apple.com/videos/play/wwdc2015/408/)

---

## 15. 동시성(Concurrency)과 병렬성(Parallelism)의 차이는? (Lv0)

### 💬 면접 답변

동시성은 여러 작업이 **시간을 나누어** 동시에 진행되는 것처럼 보이게 하는 개념입니다. 단일 코어에서도 컨텍스트 스위칭으로 동시성이 가능합니다. 병렬성은 실제로 **여러 코어에서 동시에** 작업이 실행되는 것이고, 멀티코어 하드웨어가 전제됩니다. iOS 기기는 멀티코어이므로 GCD나 Operation을 통해 동시성을 표현하면 시스템이 가용 코어로 분산해 자동으로 병렬 실행해줍니다. 따라서 개발자는 "어떻게 병렬화할 것인가"보다 "어떻게 동시성 모델을 안전하게 표현할 것인가"에 집중합니다.

### 📚 보충 설명

**구조 시각화**

```
동시성(Concurrency): 
  Task A: ▓▓░░░░▓▓░░▓▓
  Task B: ░░▓▓▓▓░░▓▓░░  (단일 코어가 시간 분할)

병렬성(Parallelism):
  Core 1, Task A: ▓▓▓▓▓▓▓▓▓▓
  Core 2, Task B: ▓▓▓▓▓▓▓▓▓▓  (실제 동시 실행)
```

Rob Pike의 유명한 문장: *"Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once."*

**동시성에서 발생할 수 있는 문제**
- **Race Condition**: 여러 스레드가 공유 자원을 동시에 읽고 써서 결과가 비결정적
- **Deadlock**: 서로가 잡고 있는 자원을 기다리며 영원히 멈춤
- **Priority Inversion**: 낮은 우선순위가 자원을 잡고 있어 높은 우선순위가 대기
- **Livelock**: 데드락은 아니지만 작업이 진척되지 않음

**해결 도구**: Serial Queue, Lock(NSLock, os_unfair_lock), Semaphore, Actor(Swift Concurrency)

자세한 내용은 [05_Concurrency.md](./05_Concurrency.md) 참고.

### 🔗 참고 자료
- [Apple - Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)
- [WWDC 2021 - Meet async/await in Swift](https://developer.apple.com/videos/play/wwdc2021/10132/)

---

> 메모리·ARC 관련은 [04_Memory_ARC.md](./04_Memory_ARC.md), 네트워크 기초는 [08_Networking.md](./08_Networking.md), 동시성 심화는 [05_Concurrency.md](./05_Concurrency.md)에서 다룹니다.
