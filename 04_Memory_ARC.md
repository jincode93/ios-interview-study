---
layout: default
title: "04. 메모리 관리와 ARC"
nav_order: 4
---

# 04. 메모리 관리와 ARC

> iOS 면접에서 가장 자주 나오는 영역 중 하나. ARC 동작, 순환 참조, 메모리 압력, 이미지 메모리 등을 다룹니다.

---

## 1. iOS의 메모리 구조(Heap, Stack)에 대해 설명해주세요. (Lv0)

### 💬 면접 답변

프로세스 메모리는 보통 **코드(Text), 데이터(Data), 힙(Heap), 스택(Stack)** 영역으로 나뉩니다. 스택은 함수 호출 단위로 LIFO로 자동 관리되며, 지역 변수와 함수 호출 정보가 들어가고 매우 빠르지만 크기가 작습니다. 힙은 동적으로 할당되는 영역이고, 클래스 인스턴스처럼 수명이 함수 스코프를 벗어나는 객체가 여기 들어갑니다. Swift에서 **값 타입은 보통 스택**(상황에 따라 힙에 박싱), **참조 타입(클래스)은 항상 힙**에 할당되고, 스택에는 그 힙 객체를 가리키는 참조만 들어갑니다.

### 📚 보충 설명

**메모리 영역 다이어그램**

```
높은 주소
┌─────────────────┐
│   Stack         │ ← 함수 호출, 지역 변수 (아래로 자람)
├─────────────────┤
│       ↓         │
│                 │
│       ↑         │
├─────────────────┤
│   Heap          │ ← 동적 할당 (위로 자람)
├─────────────────┤
│   BSS / Data    │ ← 전역, static
├─────────────────┤
│   Text          │ ← 코드(실행 명령어)
└─────────────────┘
낮은 주소
```

**Stack의 특징**
- 함수 진입 시 stack frame이 push, 종료 시 pop
- 할당/해제 비용 거의 0 (스택 포인터 이동만)
- 크기 제한 (보통 메인 스레드 1MB, 백그라운드 스레드 512KB)
- 깊은 재귀 → Stack Overflow

**Heap의 특징**
- 시스템 호출(`malloc` 등) 통한 할당
- 단편화 가능
- 멀티스레드에서 동시 접근을 위해 락 등의 동기화 필요 → 느림
- 정확한 해제 필요 (Swift에서는 ARC가 처리)

**Swift의 미묘한 점**

값 타입도 무조건 스택에 있는 건 아닙니다. 다음 경우 힙에 박싱됩니다.
- 사이즈가 크거나 동적
- 프로토콜 타입(existential)으로 저장될 때 (`any P`)
- 클로저에 캡처되어 escape할 때
- `[any P]` 같은 컨테이너에 담길 때

### 🔗 참고 자료
- [WWDC 2016 - Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/)

---

## 2. ARC(Automatic Reference Counting)의 동작 원리를 설명해주세요. (Lv1, Lv2)

### 💬 면접 답변

ARC는 클래스 인스턴스가 몇 개의 강한 참조를 받고 있는지를 컴파일 타임에 자동으로 카운트해서, 0이 되는 순간 인스턴스를 해제하는 메모리 관리 방식입니다. 컴파일러가 코드 분석 후 적절한 위치에 `retain`(+1)과 `release`(-1)를 자동 삽입하기 때문에 개발자가 직접 호출할 필요가 없습니다. ARC는 컴파일 타임에 결정되어 즉시 해제가 가능한 게 장점이고, 대신 **순환 참조**를 자동으로 감지하지 못하므로 `weak`/`unowned`로 직접 끊어줘야 합니다.

### 📚 보충 설명

**ARC vs GC 비교**

| 항목 | ARC | GC (Java/JS) |
|------|-----|--------------|
| 결정 시점 | 컴파일 타임 | 런타임 |
| 해제 타이밍 | 즉시 (참조 0) | 비결정적 (GC 사이클) |
| 오버헤드 | 카운트 증감 비용 | 주기적 마킹/스위핑 (stop-the-world) |
| 순환 참조 | 자동 처리 ❌ | 자동 감지 ⭕ |
| 메모리 사용량 | 낮음 | 높음 (GC 마진) |
| 일관된 응답성 | 좋음 | GC 동안 일시 정지 |

**컴파일 후 어떻게 보이는가**

```swift
class Animal {}
func test() {
    let a = Animal()
    print(a)
}

// 컴파일러가 대략 다음과 같이 변환
func test() {
    let a = Animal()       // retainCount = 1
    print(a)
    // ARC가 release 삽입 → retainCount = 0 → deinit
}
```

ARC는 단순 카운팅이 아니라 코드 흐름 분석을 통해 가능한 한 빨리 release를 삽입하는 최적화도 합니다.

**deinit**

```swift
class FileHandle {
    let path: String
    init(_ path: String) { self.path = path; print("open \(path)") }
    deinit { print("close \(path)") }  // 참조 카운트 0이 될 때 호출
}
```

`deinit`은 소유자가 사라질 때 리소스 정리(파일 닫기, 옵저버 제거, 타이머 invalidate)에 적합한 자리입니다.

### 🔗 참고 자료
- [The Swift Programming Language - ARC](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- [WWDC 2021 - ARC in Swift: Basics and beyond](https://developer.apple.com/videos/play/wwdc2021/10216/)

---

## 3. 강한 참조, 약한 참조, 미소유 참조의 차이는? (Lv1, Lv2)

### 💬 면접 답변

강한 참조(strong)는 ARC가 카운팅하는 기본 참조이고, 객체의 수명을 연장합니다. 약한 참조(weak)는 카운팅하지 않으며, 대상이 해제되면 자동으로 nil이 됩니다. 그래서 항상 옵셔널이고 var여야 합니다. 미소유 참조(unowned)는 카운팅하지 않지만 옵셔널이 아니며, 대상의 수명이 자신보다 길다는 게 보장될 때 사용합니다. 보장이 깨지면 해제된 메모리에 접근해 크래시가 납니다. 의심스러우면 weak이 안전한 기본 선택입니다.

### 📚 보충 설명

**셋 비교**

| 종류 | 카운팅 | 해제 시 동작 | 옵셔널? | 사용 시점 |
|------|--------|-------------|---------|----------|
| `strong` | ⭕ | – | 자유 | 기본값 |
| `weak` | ❌ | 자동 nil | 반드시 옵셔널 var | 수명이 짧을 수 있는 참조 |
| `unowned` | ❌ | dangling (크래시) | 비-옵셔널 | 수명이 보장된 참조 (부모→자식의 자식→부모 등) |

**unowned의 변종 (현대 Swift)**

- `unowned`: 기본. 해제 후 접근 시 크래시.
- `unowned(unsafe)`: 안전 검증도 안 하는 unowned. 절대 권장 안 함.
- `weak`이지만 옵셔널이 싫을 때: `unowned`이 자연스럽지만, "수명이 정말 보장되는가?" 다시 검토.

**전형적인 사용 사례**

```swift
class Parent {
    var child: Child?           // strong (소유 관계)
}

class Child {
    weak var parent: Parent?    // 부모를 다시 참조 (역참조는 weak로 끊음)
}

class Customer {
    let card: CreditCard
    init() { card = CreditCard(owner: self) }
}

class CreditCard {
    unowned let owner: Customer   // 카드는 항상 customer보다 먼저 사라짐
    init(owner: Customer) { self.owner = owner }
}
```

---

## 4. 순환 참조(Retain Cycle)는 무엇이며 어떻게 해결하나요? (Lv1, Lv2)

### 💬 면접 답변

두 객체(또는 그 이상)가 서로를 강한 참조로 잡고 있어서 ARC 카운트가 0이 되지 않아 메모리에서 해제되지 않는 상황입니다. 가장 흔한 경우가 **클래스 인스턴스끼리의 양방향 strong 참조**와 **클로저가 self를 강하게 캡처**할 때입니다. 해결은 두 참조 중 한쪽을 `weak` 또는 `unowned`로 바꾸거나, 클로저에서는 `[weak self]` 캡처 리스트로 끊습니다. Xcode의 **Memory Graph Debugger**가 보라색 느낌표로 누수를 시각화해주므로 정기적으로 확인하는 게 좋습니다.

### 📚 보충 설명

**클래스 ↔ 클래스 순환**

```swift
class A {
    var b: B?
    deinit { print("A deinit") }
}
class B {
    var a: A?       // 양쪽 다 strong → 순환
    deinit { print("B deinit") }
}

var a: A? = A()
var b: B? = B()
a?.b = b
b?.a = a            // 순환 형성
a = nil
b = nil             // deinit 호출 ❌ → 누수
```

해결:
```swift
class B { weak var a: A? }   // 한쪽을 weak로
```

**클로저 ↔ self 순환**

```swift
class ViewController: UIViewController {
    var onTap: (() -> Void)?

    func setup() {
        onTap = { self.handle() }   // self가 클로저를 보유, 클로저가 self를 캡처
    }
}
```

해결:
```swift
onTap = { [weak self] in
    self?.handle()
}
```

**클로저에서 weak vs unowned 선택 기준**

| 상황 | 선택 |
|------|------|
| 비동기 콜백, self가 클로저 호출 전에 해제될 수 있음 | `weak` |
| self가 클로저 수명 동안 살아있는 게 보장 | `unowned` |
| 확신 없음 | `weak` (안전 우선) |

**iOS 17+ `guard let self` 단축**

```swift
fetch { [weak self] data in
    guard let self else { return }
    // 이후 self는 비-옵셔널
    self.update(data)
    refresh()                // self. 생략도 가능 (SE-0269)
}
```

**자주 발생하는 미세한 함정**
- `Timer.scheduledTimer(...) { ... }` — Timer가 target을 strong 참조
- `NotificationCenter.default.addObserver(forName:object:queue:using:)` — 클로저 캡처
- `DispatchSource`, `KVO` 옵저버 토큰

→ deinit에서 invalidate/remove 잊지 않기.

### 🔗 참고 자료
- [The Swift Programming Language - Strong Reference Cycles for Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/#Strong-Reference-Cycles-for-Closures)
- [WWDC 2021 - ARC in Swift: Basics and beyond](https://developer.apple.com/videos/play/wwdc2021/10216/)

---

## 5. `deinit` 메서드는 언제 호출되며 무엇을 합니까? (Lv1)

### 💬 면접 답변

`deinit`은 클래스 인스턴스의 마지막 강한 참조가 사라져 ARC가 해제하는 직전에 호출됩니다. 인스턴스의 수명이 끝날 때 한 번만 호출되며, 옵저버 제거, 타이머 invalidate, 파일 핸들 닫기, 큰 리소스 해제 같은 정리 작업을 여기서 합니다. 구조체와 열거형(값 타입)에는 `deinit`이 없고, 클래스에만 존재합니다. 수동으로 호출할 수 없고, 호출 시점도 ARC에 의해 결정됩니다.

### 📚 보충 설명

```swift
class Observer {
    var token: NSObjectProtocol?

    init() {
        token = NotificationCenter.default.addObserver(
            forName: .someEvent, object: nil, queue: .main
        ) { _ in print("event") }
    }

    deinit {
        if let token { NotificationCenter.default.removeObserver(token) }
        print("Observer deinit")
    }
}
```

**deinit이 호출되지 않으면**

- 순환 참조가 있을 가능성
- 클로저나 옵저버에 self가 잡혀 있을 가능성
- 부모 뷰컨이나 코디네이터가 strong으로 보유 중

Xcode의 Debug Memory Graph에서 인스턴스가 있는데 deinit 안 찍히면 누수입니다.

### 🔗 참고 자료
- [The Swift Programming Language - Deinitialization](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/deinitialization/)

---

## 6. iOS 앱이 백그라운드로 갔을 때 메모리 부족으로 종료되는 이유는? (Lv0)

### 💬 면접 답변

iOS는 RAM이 제한적이라서, 메모리 압력이 발생하면 **Jetsam**이라는 시스템 메커니즘이 우선순위가 낮은 프로세스부터 강제 종료합니다. 백그라운드로 간 앱은 우선순위가 낮아지고, 메모리 사용량이 많거나 오래 사용되지 않은 앱부터 정리 대상이 됩니다. 그래서 백그라운드 진입 직전에 캐시를 비우고 큰 메모리를 해제하는 게 중요한데, `applicationDidEnterBackground`나 `didReceiveMemoryWarning`이 그 기회를 제공합니다. 또 백그라운드에서도 계속 실행되어야 한다면 Background Mode capability나 BGTaskScheduler를 활용해 시스템이 적절한 시점에 깨워주는 방식을 써야 합니다.

### 📚 보충 설명

**메모리 압력 단계**
- Normal → Warning → Urgent → Critical
- Critical일수록 적극적으로 백그라운드 앱 종료(Jetsam)

**개발자가 받을 수 있는 신호**

```swift
override func didReceiveMemoryWarning() {
    super.didReceiveMemoryWarning()
    imageCache.removeAllObjects()
    // 큰 리소스 해제
}
```

NotificationCenter도 활용 가능:
```swift
NotificationCenter.default.addObserver(
    forName: UIApplication.didReceiveMemoryWarningNotification,
    object: nil, queue: .main
) { _ in /* 캐시 비우기 */ }
```

**Jetsam 우선순위 결정 요소**
- 포그라운드/백그라운드
- 마지막 활성 시간
- 메모리 사용량
- 앱 카테고리 (예: VoIP, 음악, 네비게이션 등 특수 권한)

**Background Mode (계속 실행이 필요한 앱)**

Info.plist에 등록할 수 있는 모드:
- Audio (음악 재생)
- Location updates (네비게이션)
- VoIP
- Bluetooth Central/Peripheral
- Background fetch (주기적 데이터 갱신)
- Background processing (BGTaskScheduler)

### 🔗 참고 자료
- [Apple - Background Execution](https://developer.apple.com/documentation/uikit/app_and_environment/scenes/preparing_your_ui_to_run_in_the_background)
- [Apple - BGTaskScheduler](https://developer.apple.com/documentation/backgroundtasks)

---

## 7. 메모리 정렬(Alignment)이 성능에 미치는 영향은? (Lv0)

### 💬 면접 답변

CPU는 메모리를 워드 단위(보통 8바이트)로 읽고 쓰는데, 데이터가 워드 경계에 정렬되어 있으면 한 번의 접근으로 끝나지만, 정렬되지 않으면 여러 번 읽어 합치는 추가 작업이 필요합니다. ARM 같은 일부 아키텍처에서는 정렬되지 않은 접근이 아예 크래시를 일으키기도 합니다. Swift 구조체는 컴파일러가 자동으로 정렬과 패딩을 처리해 줍니다. 그래서 성능에 민감한 영역에서는 `MemoryLayout`으로 size·alignment·stride를 확인해서 메모리 레이아웃을 최적화할 수 있습니다.

### 📚 보충 설명

```swift
struct Foo {
    let a: UInt8    // 1 바이트
    let b: UInt32   // 4 바이트
}
print(MemoryLayout<Foo>.size)       // 5 (논리적)
print(MemoryLayout<Foo>.stride)     // 8 (실제 메모리 차지 — alignment padding 포함)
print(MemoryLayout<Foo>.alignment)  // 4
```

**필드 순서로 메모리 절약**

```swift
struct Bad {
    let a: UInt8
    let b: UInt64
    let c: UInt8
}  // stride 24 (padding 많음)

struct Good {
    let b: UInt64
    let a: UInt8
    let c: UInt8
}  // stride 16
```

큰 필드를 먼저 배치하면 padding이 줄어듭니다.

---

## 8. iOS에서 이미지 메모리 사용량은 어떻게 계산하나요? 어떻게 줄이나요? (Lv0)

### 💬 면접 답변

이미지는 디스크에 있는 파일 크기와 메모리에서 차지하는 크기가 전혀 다릅니다. 화면에 표시될 때는 **width × height × bytesPerPixel** 만큼의 비트맵으로 펼쳐지므로, 예를 들어 4032×3024 사진은 압축된 JPEG가 3MB라도 메모리에서는 약 49MB(4032 × 3024 × 4바이트)를 차지합니다. 그래서 큰 이미지를 그대로 UIImageView에 띄우면 메모리 압력이 폭증할 수 있고, 해결 방법은 **다운샘플링**(필요한 표시 크기에 맞춰 디코딩)입니다. `CGImageSourceCreateThumbnailAtIndex`나 `ImageIO`로 표시 사이즈에 맞춰 디코딩하면 메모리를 수십 배 절약할 수 있습니다.

### 📚 보충 설명

**다운샘플링 함수**

```swift
import ImageIO
import UIKit

func downsample(imageAt url: URL, to pointSize: CGSize, scale: CGFloat) -> UIImage? {
    let opts: CFDictionary = [
        kCGImageSourceShouldCache: false      // 디코딩 캐싱 끔
    ] as CFDictionary

    guard let src = CGImageSourceCreateWithURL(url as CFURL, opts) else { return nil }

    let maxPixel = max(pointSize.width, pointSize.height) * scale
    let downOpts: CFDictionary = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxPixel
    ] as CFDictionary

    guard let cg = CGImageSourceCreateThumbnailAtIndex(src, 0, downOpts) else { return nil }
    return UIImage(cgImage: cg)
}
```

**Asset Catalog의 @1x/@2x/@3x**

기기 픽셀 밀도에 따라 다른 해상도를 자동 선택해 메모리·디스크 효율을 모두 잡습니다.
- iPhone SE 등: @2x
- 일반 iPhone: @2x 또는 @3x
- iPhone Pro Max 류: @3x

**캐싱 전략**
- `NSCache`: 자동 제거, 메모리 압력에 반응
- 디스크 캐시: 큰 이미지는 디스크에 두고 LRU 정책으로 관리
- SDWebImage / Kingfisher 같은 라이브러리가 디스크+메모리 캐시를 잘 구현

### 🔗 참고 자료
- [WWDC 2018 - Image and Graphics Best Practices](https://developer.apple.com/videos/play/wwdc2018/219/)
- [Apple - ImageIO](https://developer.apple.com/documentation/imageio)

---

## 9. 가상 메모리(Virtual Memory)란 무엇인가요? (Lv0)

### 💬 면접 답변

가상 메모리는 각 프로세스에게 **실제 물리 메모리와 독립된 자체 주소 공간**을 제공하는 OS 메커니즘입니다. 덕분에 프로세스 간 메모리가 격리되고, 실제 물리 메모리보다 큰 주소 공간을 다룰 수 있으며, 페이지 단위로 디스크와 스왑하면서 동작할 수 있습니다. 다만 iOS는 데스크톱과 달리 디스크로의 일반 스왑을 사용하지 않습니다. 대신 메모리 압력이 발생하면 **압축 메모리(Compressed Memory)** 와 **purgeable memory** 메커니즘으로 대처하고, 그래도 부족하면 Jetsam이 앱을 종료합니다.

### 📚 보충 설명

**페이지와 페이지 폴트**
- 가상 메모리는 일정 크기(보통 4KB 또는 16KB)의 **페이지** 단위로 관리
- CPU가 접근한 가상 주소를 MMU가 페이지 테이블로 물리 주소로 변환
- 해당 페이지가 메모리에 없으면 **페이지 폴트** 발생 → OS가 디스크에서 가져옴 (또는 압축 해제)

**iOS의 메모리 처리 특징**
1. 일반 스왑 없음 (디스크 수명 보호, 응답성 보장)
2. **Memory Compression** (iOS 7+): 페이지를 압축해 RAM에 보관
3. **Purgeable Memory**: 시스템이 필요 시 버려도 되는 데이터 (CGImage 디코딩 결과 등)
4. **Memory Mapping (mmap)**: 큰 파일을 파일 시스템에 매핑해 필요한 페이지만 로드 (Asset Catalog가 활용)

### 🔗 참고 자료
- [Apple - Memory Usage Performance Guidelines](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/AboutMemory.html)
- [WWDC 2018 - iOS Memory Deep Dive](https://developer.apple.com/videos/play/wwdc2018/416/)

---

## 10. 사진 편집 앱이 메모리 경고를 자주 받는다면 어떻게 진단·해결하시겠습니까? (Lv3)

### 💬 면접 답변

먼저 Instruments의 **Allocations**로 어떤 객체가 비정상적으로 많이 살아있는지 보고, **Leaks**로 명백한 누수를, **Memory Graph Debugger**로 순환 참조를 확인합니다. 사진 편집 같은 도메인에서는 이미지를 풀 해상도로 디코딩해 잡고 있는 경우가 대부분 원인이라, 표시 단계에서는 다운샘플링하고, 편집 중에는 타일 렌더링이나 GPU 기반 처리(Metal/Core Image)로 전환합니다. 또 백그라운드 진입 시 큰 버퍼와 캐시는 명시적으로 해제하고, `autoreleasepool`로 단기 객체의 정리를 강제해 피크 메모리를 줄입니다.

### 📚 보충 설명

**진단 순서**

1. **Instruments - Allocations**
   - "Persistent Bytes"가 시간에 따라 증가만 한다면 누수 의심
   - "Mark Generation" 기능으로 특정 작업 사이의 메모리 증가량 측정
2. **Leaks**
   - 명백한 누수(주인 없는 객체) 자동 감지
3. **Memory Graph Debugger** (Xcode → 디버그 메뉴)
   - 순환 참조(보라색 느낌표) 시각화
   - "Show Only Leaked Blocks"

**이미지 처리 최적화 전략**

- **다운샘플링**: 표시 크기에 맞춰 디코딩 (위 8번 항목 참조)
- **타일 렌더링**: 큰 이미지를 격자로 나눠 보이는 부분만 처리 (CATiledLayer)
- **GPU 처리 (Core Image, Metal)**: CPU 메모리 부담 ↓, 성능 ↑
- **`autoreleasepool`** 로 명시적 정리

```swift
for url in imageURLs {
    autoreleasepool {
        let img = downsample(imageAt: url, to: thumbSize, scale: scale)
        process(img)
    }
    // 루프 매 iteration마다 autorelease된 임시 객체 즉시 해제
}
```

**캐시 전략**
- `NSCache`는 메모리 압력에 자동 반응 (Dictionary와 다름)
- `countLimit`, `totalCostLimit` 설정으로 상한 두기
- 디스크 캐시를 LRU로 관리, 캐시 디렉터리는 `~/Library/Caches`

### 🔗 참고 자료
- [WWDC 2018 - iOS Memory Deep Dive](https://developer.apple.com/videos/play/wwdc2018/416/)
- [WWDC 2018 - Image and Graphics Best Practices](https://developer.apple.com/videos/play/wwdc2018/219/)
- [Apple - Instruments](https://help.apple.com/instruments/mac/current/)

---

## 11. 60fps 실시간 비디오 처리에서 메모리 안전성과 성능의 트레이드오프는? (Lv4)

### 💬 면접 답변

60fps라면 한 프레임당 16.6ms 안에 캡처, 처리, 출력을 모두 끝내야 하므로 ARC의 retain/release 오버헤드와 메모리 할당 자체가 병목이 됩니다. 그래서 가능한 한 **값 타입으로 스택에 머무르게** 하거나, **버퍼 풀**로 메모리 할당을 피합니다. Metal과 Core Video의 `CVPixelBufferPool`이 대표적인 예시고, GPU 텍스처는 미리 할당해두고 재사용합니다. Swift 5.9+에서는 `~Copyable`, `consuming`/`borrowing` 같은 **소유권(Ownership) 모델**로 불필요한 복사·참조 카운트 증감을 명시적으로 제거할 수 있어, 성능 critical 코드에서 안전성과 성능을 동시에 잡을 수 있습니다.

### 📚 보충 설명

**Swift 5.9+ Ownership**

```swift
// 일반 함수: 인자가 보통 borrowing처럼 동작
func process(_ buffer: PixelBuffer) { ... }

// borrowing: 명시적으로 빌려서 변경하지 않음 (refcount 변화 없음)
func process(_ buffer: borrowing PixelBuffer) { ... }

// consuming: 인자의 소유권을 가져가서 호출자는 더 이상 사용 못함
func consume(_ buffer: consuming PixelBuffer) { ... }
```

**~Copyable 타입**

복사가 절대 일어나서는 안 되는 자원 (파일 디스크립터, GPU 핸들 등):

```swift
struct FileHandle: ~Copyable {
    let fd: Int32
    init(_ path: String) throws { ... }
    consuming func close() { /* fclose(fd) */ }
    deinit { /* fclose if not consumed */ }
}
```

**Buffer Pool 패턴**

```swift
// CoreVideo의 CVPixelBufferPool: 같은 포맷의 픽셀 버퍼를 재사용
let attrs: [String: Any] = [
    kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA,
    kCVPixelBufferWidthKey as String: 1920,
    kCVPixelBufferHeightKey as String: 1080
]
var pool: CVPixelBufferPool?
CVPixelBufferPoolCreate(nil, nil, attrs as CFDictionary, &pool)

var buffer: CVPixelBuffer?
CVPixelBufferPoolCreatePixelBuffer(nil, pool!, &buffer)
// 사용 후 자동으로 풀에 반납
```

**`unsafe` API**

성능 critical 영역에서는 `UnsafeMutablePointer`, `withUnsafeBytes` 등을 써서 ARC를 우회하기도 합니다. 단, 검증 책임은 전적으로 개발자에게.

### 🔗 참고 자료
- [Swift Evolution: SE-0390 - Noncopyable structs and enums](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [WWDC 2023 - Beyond the basics of structured concurrency](https://developer.apple.com/videos/play/wwdc2023/10170/)
- [WWDC 2023 - Generalize APIs with parameter packs](https://developer.apple.com/videos/play/wwdc2023/10168/)
- [Apple - Core Video](https://developer.apple.com/documentation/corevideo)

---

> 동시성과 관련된 메모리 문제(actor 격리, Sendable)는 [05_Concurrency.md](./05_Concurrency.md)를 참고하세요.
