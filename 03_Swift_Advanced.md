---
layout: default
title: "03. Swift 고급 주제"
nav_order: 3
---

# 03. Swift 고급 주제

> Property Wrapper, Reflection, Result Builder, ABI 안정성, Macro, Unsafe Pointer.

---

## 1. Property Wrapper란 무엇인가요? (Lv3, Lv4)

### 💬 면접 답변

Property Wrapper는 프로퍼티의 **저장과 접근 로직을 별도 타입으로 캡슐화**해서 여러 프로퍼티에 재사용할 수 있게 해주는 기능입니다. `@propertyWrapper` 어노테이션을 붙인 타입이 `wrappedValue` 프로퍼티를 가지고 있으면, 그 타입을 다른 프로퍼티 앞에 `@MyWrapper`처럼 붙여 사용할 수 있습니다. SwiftUI의 `@State`, `@Binding`, `@Published`가 모두 Property Wrapper로 구현되어 있고, UserDefaults·Keychain 래핑, 검증 로직, 스레드 안전 접근 같은 패턴에 자주 활용됩니다.

### 📚 보충 설명

**가장 단순한 예: 범위 제한**

```swift
@propertyWrapper
struct Clamped<T: Comparable> {
    private var value: T
    let range: ClosedRange<T>

    init(wrappedValue: T, _ range: ClosedRange<T>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }

    var wrappedValue: T {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}

struct Volume {
    @Clamped(0...100) var level: Int = 50
}

var v = Volume()
v.level = 150         // 자동으로 100으로 클램프
print(v.level)        // 100
```

**`projectedValue`로 추가 인터페이스 제공**

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T

    var wrappedValue: T {
        get { UserDefaults.standard.object(forKey: key) as? T ?? defaultValue }
        set { UserDefaults.standard.set(newValue, forKey: key) }
    }

    var projectedValue: Self { self }   // $name으로 접근 가능
}
```

`@Published`의 `$name`도 같은 메커니즘입니다 (`projectedValue`가 Publisher).

**SwiftUI Property Wrapper 정리**

| 래퍼 | 역할 | 프로젝션 |
|------|------|---------|
| `@State` | View 지역 상태 | `$state` → `Binding<T>` |
| `@Binding` | 외부 상태 바인딩 | `$binding` → 재바인딩 |
| `@StateObject` | View 소유 ObservableObject | `$obj.prop` → Binding |
| `@ObservedObject` | 외부 ObservableObject | 동일 |
| `@Published` | Combine Publisher 발행 | `$published` → Publisher |
| `@Environment` | 환경 값 | – |
| `@AppStorage` | UserDefaults wrapper | – |
| `@SceneStorage` | Scene별 상태 보존 | – |
| `@FocusState` | 키보드 포커스 | – |

### 🔗 참고 자료
- [Swift Evolution: SE-0258 - Property Wrappers](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md)
- [Apple - Property Wrappers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Property-Wrappers)

---

## 2. Reflection(Mirror)은 무엇이며 언제 사용하나요? (Lv3, Lv4)

### 💬 면접 답변

Reflection은 런타임에 타입의 구조(프로퍼티, 값)를 검사할 수 있는 기능으로, Swift에서는 `Mirror` 타입을 통해 제공됩니다. `Mirror(reflecting:)`으로 어떤 인스턴스를 들여다보면 그 인스턴스의 프로퍼티 이름과 값을 순회할 수 있습니다. 디버깅 출력, 직렬화, 자동화 도구처럼 동적으로 구조를 알아야 할 때 유용한데, **성능 비용**이 있고 **타입 안전성**도 약해지므로 핵심 로직에서는 가급적 피하고 컴파일 타임 기능(Codable, KeyPath, Macro)으로 대체할 수 있으면 그쪽이 낫습니다.

### 📚 보충 설명

**기본 사용**

```swift
struct User {
    let id: Int
    let name: String
    let email: String?
}

let u = User(id: 1, name: "철수", email: nil)
let mirror = Mirror(reflecting: u)

for child in mirror.children {
    print("\(child.label ?? "?"): \(child.value)")
}
// id: 1
// name: 철수
// email: nil
```

**활용 예: 객체 로깅**

```swift
extension CustomStringConvertible {
    var debugDump: String {
        let m = Mirror(reflecting: self)
        let pairs = m.children.compactMap { c -> String? in
            guard let label = c.label else { return nil }
            return "\(label)=\(c.value)"
        }
        return "\(type(of: self))(\(pairs.joined(separator: ", ")))"
    }
}
```

**Mirror의 한계**
- 읽기 전용 (값 변경 불가)
- 정적 디스패치된 protocol witness 표 접근 불가
- 성능 비용 (런타임 metadata 조회)
- private/internal 프로퍼티도 보임 → 캡슐화 우회

**대안**
- 컴파일 타임: `Codable`, `KeyPath`, Swift Macro
- 디버깅: `dump(_:)` 표준 함수

```swift
dump(u)
// ▿ User
//   - id: 1
//   - name: "철수"
//   - email: nil
```

### 🔗 참고 자료
- [Apple - Mirror](https://developer.apple.com/documentation/swift/mirror)
- [Apple - dump](https://developer.apple.com/documentation/swift/dump(_:name:indent:maxdepth:maxitems:))

---

## 3. @dynamicMemberLookup과 @dynamicCallable은? (Lv3)

### 💬 면접 답변

`@dynamicMemberLookup`은 컴파일 타임에 존재하지 않는 멤버 접근을 런타임의 서브스크립트로 라우팅하게 해주는 기능입니다. JSON 같은 동적 데이터나 외부 API 응답을 타입 시스템과 자연스럽게 결합할 때 유용합니다. `@dynamicCallable`은 인스턴스를 함수처럼 호출할 수 있게 하는 기능으로, DSL이나 Python 같은 동적 언어와의 브리지에서 활용됩니다. 둘 다 강력하지만 타입 안전성을 일부 양보하기 때문에, 정적으로 표현 가능한 곳에서는 쓰지 말고 정말로 동적 접근이 필요한 곳에만 한정해서 사용하는 게 좋습니다.

### 📚 보충 설명

**@dynamicMemberLookup**

```swift
@dynamicMemberLookup
struct JSON {
    let value: Any?

    subscript(dynamicMember member: String) -> JSON {
        if let dict = value as? [String: Any] {
            return JSON(value: dict[member])
        }
        return JSON(value: nil)
    }

    var stringValue: String? { value as? String }
    var intValue: Int? { value as? Int }
}

let raw: [String: Any] = ["user": ["name": "철수", "age": 30]]
let json = JSON(value: raw)
print(json.user.name.stringValue ?? "")  // "철수"
print(json.user.age.intValue ?? 0)       // 30
```

**KeyPath와 결합한 type-safe 패턴**

```swift
@dynamicMemberLookup
struct Box<T> {
    var value: T

    subscript<U>(dynamicMember kp: KeyPath<T, U>) -> U {
        value[keyPath: kp]
    }
}

struct User { let name: String }
let b = Box(value: User(name: "철수"))
print(b.name)   // KeyPath로 접근 → 컴파일 타임에 검증됨
```

**@dynamicCallable**

```swift
@dynamicCallable
struct Adder {
    func dynamicallyCall(withArguments args: [Int]) -> Int {
        args.reduce(0, +)
    }
}

let adder = Adder()
let sum = adder(1, 2, 3, 4)   // 인스턴스를 함수처럼 호출
```

### 🔗 참고 자료
- [Swift Evolution: SE-0195 - dynamicMemberLookup](https://github.com/apple/swift-evolution/blob/main/proposals/0195-dynamic-member-lookup.md)
- [Swift Evolution: SE-0216 - dynamicCallable](https://github.com/apple/swift-evolution/blob/main/proposals/0216-dynamic-callable.md)

---

## 4. Result Builder(@resultBuilder)란? SwiftUI의 View는 어떻게 동작하나요? (Lv4)

### 💬 면접 답변

Result Builder는 함수의 본문에 나열된 표현식들을 컴파일러가 자동으로 하나의 결과 값으로 합성해주는 메타프로그래밍 기능입니다. SwiftUI의 `@ViewBuilder`가 가장 익숙한 예시인데, `VStack { Text(...); Button(...) { ... } }`처럼 여러 View를 그냥 나열하면 컴파일러가 `TupleView`로 묶어줍니다. 이를 통해 선언적인 DSL이 가능해지고, 클로저 안에서 if·switch·for-loop 같은 제어 구조도 자연스럽게 사용할 수 있습니다.

### 📚 보충 설명

**최소한의 Result Builder**

```swift
@resultBuilder
struct StringBuilder {
    static func buildBlock(_ components: String...) -> String {
        components.joined(separator: "\n")
    }
}

func makeText(@StringBuilder content: () -> String) -> String {
    content()
}

let msg = makeText {
    "Hello"
    "World"
}
// "Hello\nWorld"
```

**지원하는 buildXXX 메서드**

| 메서드 | 역할 |
|--------|------|
| `buildBlock(_:)` | 블록 안의 표현식들을 합침 |
| `buildIf(_:)` / `buildEither(first:)` / `buildEither(second:)` | if-else 처리 |
| `buildOptional(_:)` | 옵셔널 처리 |
| `buildArray(_:)` | for 루프 처리 |
| `buildExpression(_:)` | 각 표현식 전처리 |
| `buildFinalResult(_:)` | 최종 결과 후처리 |

**SwiftUI ViewBuilder 단순화 예시**

```swift
@resultBuilder
struct ViewBuilder {
    static func buildBlock<V: View>(_ v: V) -> V { v }
    static func buildBlock<V1: View, V2: View>(_ v1: V1, _ v2: V2) -> TupleView<(V1, V2)> {
        TupleView((v1, v2))
    }
    // ... 더 많은 오버로드
}
```

**DSL 활용 예: HTML 빌더**

```swift
@resultBuilder
struct HTMLBuilder { /* ... */ }

func div(@HTMLBuilder _ content: () -> HTML) -> HTML { ... }

let page = div {
    h1("제목")
    p("본문")
    if showFooter { footer("Copyright") }
}
```

### 🔗 참고 자료
- [Swift Evolution: SE-0289 - Result Builders](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md)
- [Apple - Result Builders](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/advancedoperators/#Result-Builders)

---

## 5. Swift Macro(매크로)란? (iOS 17+/Swift 5.9+) (Lv4)

### 💬 면접 답변

Swift Macro는 Swift 5.9에서 도입된 **컴파일 타임 코드 생성** 기능으로, 우리가 매크로를 호출하면 컴파일러가 그 자리에 자동으로 코드를 펼쳐 넣어줍니다. 예를 들어 `@Observable`은 클래스에 붙이면 observation 보일러플레이트를, `#Preview` 매크로는 SwiftUI 프리뷰 코드를 자동으로 생성합니다. 매크로는 별도의 SwiftSyntax 기반 프로세스로 실행되어 **타입 안전성을 보장**하며, 잘못된 매크로 결과는 컴파일 에러로 즉시 잡힙니다. Objective-C의 #define 매크로처럼 텍스트 치환이 아니라 **AST 수준의 변환**이라 훨씬 안전하고 IDE 친화적입니다.

### 📚 보충 설명

**매크로의 종류**

| 종류 | 역할 | 예 |
|------|------|------|
| Expression macro | 표현식 위치에서 코드 생성 | `#URL("https://...")` |
| Declaration macro | 새 선언 자체를 생성 | – |
| Attached macro | 선언에 붙어 변환 | `@Observable`, `@AddCompletionHandler` |
| Freestanding macro | 독립적으로 사용 | `#Preview { ... }` |

**Attached macro 세부 종류**
- `@attached(member)`: 멤버 추가
- `@attached(memberAttribute)`: 멤버에 어트리뷰트 추가
- `@attached(accessor)`: 접근자 추가
- `@attached(extension)`: extension 추가
- `@attached(peer)`: 같은 레벨에 새 선언 추가

**iOS 17 표준 매크로 예시**

```swift
// @Observable 매크로 펼침 전
@Observable
class UserVM {
    var name = ""
}

// 펼친 후 (대략)
class UserVM: Observation.Observable {
    @ObservationIgnored private let _$observationRegistrar = ObservationRegistrar()

    var name = "" {
        @storageRestrictions(initializes: _name) init { _name = newValue }
        get { access(keyPath: \.name); return _name }
        set { withMutation(keyPath: \.name) { _name = newValue } }
    }
    private var _name = ""
    // ... access, withMutation 메서드
}
```

**매크로의 장점**
- 보일러플레이트 제거
- 타입 안전 (AST 검증)
- "Expand Macro" 기능으로 펼친 코드 확인 가능 (Xcode 우클릭)
- 컴파일 타임 → 런타임 오버헤드 없음

### 🔗 참고 자료
- [Swift Evolution: SE-0382 - Expression Macros](https://github.com/apple/swift-evolution/blob/main/proposals/0382-expression-macros.md)
- [Swift Evolution: SE-0389 - Attached Macros](https://github.com/apple/swift-evolution/blob/main/proposals/0389-attached-macros.md)
- [WWDC 2023 - Expand on Swift macros](https://developer.apple.com/videos/play/wwdc2023/10167/)
- [WWDC 2023 - Write Swift macros](https://developer.apple.com/videos/play/wwdc2023/10166/)

---

## 6. Copy-on-Write(COW) 메커니즘은 어떻게 동작하나요? (Lv4)

### 💬 면접 답변

Copy-on-Write는 값 타입을 변수에 복사할 때 **실제 데이터는 공유**하다가 **수정이 발생하는 순간에만 복사**가 일어나도록 하는 최적화 기법입니다. Swift의 `Array`, `Dictionary`, `Set`, `String` 모두 이 메커니즘으로 구현되어 있어, 큰 배열을 함수에 넘기거나 변수에 할당해도 즉시 O(n) 복사가 발생하지 않습니다. 내부적으로는 참조 카운팅과 `isKnownUniquelyReferenced(_:)` 함수를 사용해 자신이 유일한 참조자인지 확인한 뒤 변경합니다. 우리가 직접 만든 타입에도 같은 패턴을 적용하면 값 시맨틱의 장점과 성능을 모두 가져올 수 있습니다.

### 📚 보충 설명

**커스텀 COW 구현**

```swift
final class _Storage {
    var items: [Int] = []
}

struct MyArray {
    private var storage = _Storage()

    private mutating func ensureUnique() {
        if !isKnownUniquelyReferenced(&storage) {
            let copy = _Storage()
            copy.items = storage.items
            storage = copy
        }
    }

    mutating func append(_ x: Int) {
        ensureUnique()
        storage.items.append(x)
    }
}
```

**관찰 가능한 동작**

```swift
var a: [Int] = Array(0..<1_000_000)
var b = a                  // O(1) - 버퍼 공유
a.append(1)                // 이 시점에 b가 새 버퍼로 복사 (또는 a가)
```

**클래스 vs COW 값 타입**
- 클래스: 항상 참조 시맨틱, 부주의한 공유로 인한 버그 가능성
- COW: 값 시맨틱(예측 가능한 독립성) + 성능

### 🔗 참고 자료
- [Apple - Choosing Between Structures and Classes](https://developer.apple.com/documentation/swift/choosing-between-structures-and-classes)
- [Swift Standard Library - isKnownUniquelyReferenced](https://developer.apple.com/documentation/swift/isknownuniquelyreferenced(_:))

---

## 7. ABI 안정성이란? (Lv3)

### 💬 면접 답변

ABI(Application Binary Interface) 안정성은 **컴파일된 바이너리 사이의 호환성**을 보장하는 규약입니다. Swift 5에서 ABI가 안정화되어, 이후 Swift로 빌드된 앱은 OS에 내장된 Swift 런타임을 사용하므로 앱 바이너리에 Swift 라이브러리를 같이 번들할 필요가 없어졌습니다. 그래서 앱 크기가 줄고, 시스템 업데이트로 런타임도 함께 개선됩니다. 라이브러리 입장에서는 **Module Stability**(다른 Swift 컴파일러 버전과의 모듈 호환성)도 중요한데, `BUILD_LIBRARY_FOR_DISTRIBUTION = YES`를 활성화한 바이너리 프레임워크가 이를 지원합니다.

### 📚 보충 설명

**ABI 안정성의 효과**
- 같은 OS에서 Swift 컴파일러 버전이 달라도 바이너리끼리 호환
- 시스템에 Swift 런타임이 포함 → 앱 번들에 Swift dylib 미포함
- iOS 12.2 이상부터 시스템 Swift 활용

**Library Evolution (Module Stability)**

라이브러리 측이 `BUILD_LIBRARY_FOR_DISTRIBUTION = YES`로 빌드하면, 클라이언트와 라이브러리의 Swift 버전이 달라도 호환됩니다. 다음 변경이 허용:
- 새 멤버 추가
- public 멤버 순서 변경
- 새 enum case 추가 (단, `@frozen` 아닐 때)

**라이브러리에서 주의할 점**
- 공개 API의 시그니처 변경 = breaking change
- `@frozen`은 컴파일 최적화 가능하지만 enum case 추가 불가
- `@inlinable`은 인라인 가능 (성능 ↑, 단 구현 공개 = 추후 변경 제약)

### 🔗 참고 자료
- [Swift.org - ABI Stability and More](https://www.swift.org/blog/abi-stability-and-more/)
- [Swift.org - Module Stability](https://www.swift.org/blog/library-evolution/)

---

## 8. Unsafe Pointer는 무엇이며 언제 쓰나요? (Lv3)

### 💬 면접 답변

Unsafe Pointer는 Swift의 안전성 검사를 우회해 **C 스타일로 메모리에 직접 접근**할 수 있게 해주는 타입들입니다. `UnsafePointer<T>`(읽기 전용), `UnsafeMutablePointer<T>`(읽기·쓰기), `UnsafeRawPointer`(타입이 지정되지 않은 원시 포인터)가 있고, 주로 C 라이브러리 인터롭이나 성능 critical 코드, 메모리 레이아웃 제어가 필요한 곳에 쓰입니다. **수명·정렬·타입 호환을 모두 개발자가 보장**해야 하므로 잘못 쓰면 segfault·UB·메모리 손상으로 이어집니다. 그래서 가능한 한 `withUnsafeBytes`처럼 스코프가 명확한 함수형 API를 사용하고, 포인터를 escape시키지 않는 게 안전합니다.

### 📚 보충 설명

**기본 사용**

```swift
var x: Int = 42

withUnsafePointer(to: &x) { ptr in
    print(ptr.pointee)    // 42
}

withUnsafeMutablePointer(to: &x) { ptr in
    ptr.pointee = 100
}

print(x)  // 100
```

**C API와 인터롭**

```swift
import Foundation

let str = "hello"
str.withCString { cstr in
    strlen(cstr)    // C 함수 호출
}

// 반대 방향: C가 준 포인터를 Swift에서 사용
func process(_ buffer: UnsafePointer<UInt8>, count: Int) {
    let bytes = UnsafeBufferPointer(start: buffer, count: count)
    for b in bytes { print(b) }
}
```

**메모리 직접 할당**

```swift
let p = UnsafeMutablePointer<Int>.allocate(capacity: 10)
p.initialize(repeating: 0, count: 10)
defer {
    p.deinitialize(count: 10)
    p.deallocate()
}
p[0] = 42
```

**규칙**
- 포인터가 가리키는 메모리는 호출 시점에 유효해야 함
- 타입을 함부로 캐스팅하면 strict aliasing 위반
- 라이프타임 추적은 전적으로 개발자 책임

### 🔗 참고 자료
- [Apple - Manual Memory Management](https://developer.apple.com/documentation/swift/manual-memory-management)
- [Apple - UnsafePointer](https://developer.apple.com/documentation/swift/unsafepointer)

---

## 9. Swift의 Ownership(소유권)과 Borrowing(빌림)에 대해 설명해주세요. (Lv4)

### 💬 면접 답변

Swift 5.9부터 단계적으로 도입된 소유권 모델은 함수가 인자를 **빌리는지(borrowing)** 아니면 **가져가는지(consuming)** 를 명시적으로 표현할 수 있게 해줍니다. `borrowing`은 인자의 소유권을 호출자에 그대로 두고 잠깐 빌려 쓰는 것이라 refcount 증감이 없고, `consuming`은 소유권을 함수가 가져가서 호출자는 더 이상 사용할 수 없게 됩니다. 이를 통해 ARC의 retain/release 비용을 줄이고, `~Copyable` 같은 비복사 가능 타입(파일 핸들, GPU 자원 등)도 안전하게 모델링할 수 있습니다. 일상 코드에서는 컴파일러가 잘 추론해서 직접 어노테이션하는 경우가 드물지만, 성능 critical하거나 단일 소유가 요구되는 자원에서는 큰 차이를 만듭니다.

### 📚 보충 설명

**borrowing / consuming**

```swift
struct Buffer {
    let data: [UInt8]
}

// borrowing: 인자를 잠깐 빌림 (refcount 증가 없음, 일반 함수 호출의 기본과 유사)
func sum(_ b: borrowing Buffer) -> Int {
    b.data.reduce(0, +)
}

// consuming: 인자의 소유권을 가져감 (호출 후 호출자는 사용 불가)
func consumeAndWrite(_ b: consuming Buffer) {
    write(b.data)
    // b는 이 함수가 책임
}

var buf = Buffer(data: [1, 2, 3])
sum(buf)              // 호출 후 buf 계속 사용 가능
consumeAndWrite(buf)  // 호출 후 buf 사용 불가 (컴파일 에러)
```

**~Copyable 타입**

복사가 금지되는 타입:

```swift
struct FileDescriptor: ~Copyable {
    let fd: Int32

    init(_ path: String) throws { ... }

    consuming func close() {
        // fclose(fd)
    }

    deinit {
        // close가 안 불렸으면 여기서 정리
    }
}

func write(_ fd: borrowing FileDescriptor, data: Data) { ... }

let fd = try FileDescriptor("foo.txt")
write(fd, data: data)
fd.close()       // 명시적으로 소유권을 close에 양도 → 이후 fd 사용 불가
```

**왜 중요한가**
- ARC retain/release 비용 제거
- 단일 소유 자원(파일 핸들, 락, 트랜잭션) 안전한 모델링
- 성능 critical 코드 최적화

### 🔗 참고 자료
- [Swift Evolution: SE-0377 - borrowing/consuming](https://github.com/apple/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)
- [Swift Evolution: SE-0390 - Noncopyable structs and enums](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
- [WWDC 2023 - Beyond the basics of structured concurrency](https://developer.apple.com/videos/play/wwdc2023/10170/)

---

## 10. Custom String Interpolation의 활용은? (Lv4)

### 💬 면접 답변

Swift 5+ 에서는 `ExpressibleByStringInterpolation` 프로토콜과 `StringInterpolation` 타입을 확장해서 **문자열 보간에 도메인 특화 동작**을 추가할 수 있습니다. 예를 들어 통화 포맷, HTML escape, SQL 파라미터 바인딩 같은 작업을 보간 표현 안에서 자연스럽게 표현할 수 있어, 가독성과 안전성을 동시에 높일 수 있습니다. 다만 너무 많은 커스텀 보간을 도입하면 다른 개발자가 이해하기 어려워질 수 있으니 정말 도움이 되는 곳에 한정해서 씁니다.

### 📚 보충 설명

```swift
extension String.StringInterpolation {
    mutating func appendInterpolation(currency value: Decimal) {
        let formatter = NumberFormatter()
        formatter.numberStyle = .currency
        formatter.currencyCode = "KRW"
        appendLiteral(formatter.string(for: value) ?? "")
    }

    mutating func appendInterpolation(escaping value: String) {
        let escaped = value
            .replacingOccurrences(of: "&", with: "&amp;")
            .replacingOccurrences(of: "<", with: "&lt;")
            .replacingOccurrences(of: ">", with: "&gt;")
        appendLiteral(escaped)
    }
}

let price = "총 \(currency: 12_345 as Decimal)"
print(price)
// "총 ₩12,345"

let safe = "댓글: \(escaping: "<script>alert(1)</script>")"
// "댓글: &lt;script&gt;alert(1)&lt;/script&gt;"
```

### 🔗 참고 자료
- [Swift Evolution: SE-0228 - Fix ExpressibleByStringInterpolation](https://github.com/apple/swift-evolution/blob/main/proposals/0228-fix-expressiblebystringinterpolation.md)

---

## 11. KeyPath의 다양한 활용은? (Lv2)

### 💬 면접 답변

KeyPath는 02_Swift_Basics에서 다룬 기본 개념 외에도 다양한 패턴에서 활용됩니다. `map(\.property)` 같은 단축 표기, SwiftUI의 `ForEach(items, id: \.id)`, Combine의 `assign(to: \.text, on: label)`, Core Data·SwiftData의 정렬·필터 표현, 의존성 주입 컨테이너에서 의존성 식별자로 사용 등 정적 안전성을 유지한 채 멤버를 값처럼 다루는 모든 곳에 등장합니다. 또 `@dynamicMemberLookup`과 결합하면 컴파일 타임 검증이 되는 유연한 인터페이스를 만들 수도 있습니다.

### 📚 보충 설명

**Combine에서의 KeyPath**

```swift
publisher
    .map(\.user.name)
    .assign(to: \.text, on: label)
    .store(in: &cancellables)
```

**SwiftData/Core Data의 KeyPath 정렬**

```swift
@Query(sort: \Book.publishedDate, order: .reverse) var books: [Book]
```

**KeyPath 합성**

```swift
struct User { let address: Address }
struct Address { let city: String }

let kp1: KeyPath<User, Address> = \User.address
let kp2: KeyPath<Address, String> = \Address.city
let combined: KeyPath<User, String> = kp1.appending(path: kp2)
```

### 🔗 참고 자료
- [Apple - KeyPath](https://developer.apple.com/documentation/swift/keypath)

---

## 12. Sequence와 Collection 프로토콜은 어떻게 다른가요? (Lv2)

### 💬 면접 답변

`Sequence`는 한 번 순회할 수 있는 가장 기본적인 프로토콜로, `makeIterator()`만 구현하면 됩니다. 무한 수열도 표현할 수 있고 같은 시퀀스를 두 번 순회한다는 보장은 없습니다. `Collection`은 `Sequence`를 상속하면서 **여러 번 순회 가능**, **인덱스로 접근 가능**, **count를 알 수 있다**는 추가 보장을 둡니다. 그래서 `Array`, `Set`, `Dictionary` 같은 표준 컨테이너는 모두 Collection을 채택합니다. 커스텀 컬렉션을 만들 때 Sequence부터 시작해서 점진적으로 BidirectionalCollection, RandomAccessCollection으로 올려가면 적절한 시간 복잡도 보장과 함께 표준 라이브러리의 풍부한 알고리즘을 자동으로 활용할 수 있습니다.

### 📚 보충 설명

**프로토콜 계층**

```
Sequence
  └─ Collection
       ├─ MutableCollection
       ├─ BidirectionalCollection
       │    └─ RandomAccessCollection
       └─ RangeReplaceableCollection
```

| 프로토콜 | 핵심 보장 |
|---------|---------|
| Sequence | 한 번 순회 |
| Collection | 다중 순회, 인덱스, count |
| MutableCollection | 원소 변경 |
| BidirectionalCollection | 역방향 순회 가능 |
| RandomAccessCollection | O(1) 인덱스 거리 계산 |
| RangeReplaceableCollection | 부분 교체 가능 (append, remove) |

**커스텀 Collection 예시: 순환 버퍼**

```swift
struct CircularBuffer<T>: Collection {
    private var storage: [T]
    var startIndex: Int { 0 }
    var endIndex: Int { storage.count }

    init(_ items: [T]) { storage = items }

    subscript(position: Int) -> T { storage[position] }

    func index(after i: Int) -> Int { (i + 1) % storage.count }
}
```

### 🔗 참고 자료
- [Apple - Sequence](https://developer.apple.com/documentation/swift/sequence)
- [Apple - Collection](https://developer.apple.com/documentation/swift/collection)

---

## 13. Swift 컴파일러의 동적 디스패치와 정적 디스패치는 어떻게 다른가요? (Lv3)

### 💬 면접 답변

정적(static) 디스패치는 어떤 메서드를 호출할지를 **컴파일 타임에 결정**하는 방식으로 호출 오버헤드가 거의 없고 인라이닝 같은 추가 최적화가 가능합니다. 동적(dynamic) 디스패치는 **런타임에 결정**하는 방식으로, 가상 함수 테이블(vtable)이나 witness table을 거치므로 비용이 추가됩니다. Swift는 기본적으로 **`final` 또는 값 타입의 메서드는 정적**, **클래스의 일반 메서드와 프로토콜 요구사항은 동적**으로 디스패치하며, `@objc dynamic`을 붙이면 ObjC 런타임을 통한 더 동적인 메시지 디스패치도 가능합니다. 성능 critical한 곳에서는 `final` 키워드, struct 활용, `@inlinable`로 정적 디스패치와 인라이닝을 유도할 수 있습니다.

### 📚 보충 설명

**디스패치 비교표**

| 타입 | 메서드 | 디스패치 |
|------|--------|---------|
| struct/enum | 모든 메서드 | 정적 |
| 프로토콜 (witness table) | 프로토콜 요구사항 | 동적 (witness table) |
| 프로토콜 extension의 비-요구사항 | 정적 (주의!) | – |
| class | 일반 메서드 | 동적 (vtable) |
| `final class` 또는 final method | – | 정적 |
| `@objc dynamic` | – | 메시지 디스패치 (가장 동적) |

**주의: 프로토콜 extension의 함정**

```swift
protocol P {
    func a()
}
extension P {
    func a() { print("P.a") }
    func b() { print("P.b") }   // 요구사항 아님
}
struct S: P {
    func a() { print("S.a") }
    func b() { print("S.b") }
}

let p: P = S()
p.a()  // "S.a" (witness table)
p.b()  // "P.b" (정적 디스패치)
```

**최적화 힌트**
- `final class` 또는 메서드 단위 `final`
- struct 활용
- 모듈 외부에서 사용하는 함수에 `@inlinable` (단 구현 공개 = breaking change 어려움)
- whole module optimization 활성화

### 🔗 참고 자료
- [WWDC 2016 - Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/)

---

> 메모리 관리(ARC, COW의 상세)는 [04_Memory_ARC.md](./04_Memory_ARC.md), 동시성 관련 고급 주제(Actor 재진입 등)는 [05_Concurrency.md](./05_Concurrency.md)를 참고하세요.
