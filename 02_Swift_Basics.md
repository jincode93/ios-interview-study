---
layout: default
title: "02. Swift 언어 기본"
nav_order: 2
---

# 02. Swift 언어 기본

> Swift 언어의 핵심 문법과 자주 사용되는 기능들. 2년차 면접에서 가장 많이 나오는 주제입니다.

---

## 1. 서버 API가 사용자 이름을 때로는 보내고 때로는 보내지 않을 때 어떻게 처리하나요? (Lv1)

### 💬 면접 답변

저는 이런 경우 해당 프로퍼티를 `String?` 같은 옵셔널 타입으로 모델링합니다. `Codable`에서는 옵셔널 프로퍼티는 키가 없거나 값이 `null`이면 자동으로 `nil`이 되므로 별도 처리 없이 디코딩되고, 사용하는 쪽에서는 `if let`이나 `guard let`으로 안전하게 언래핑하거나 `??` 연산자로 기본값을 제공합니다. 절대로 `!`(강제 언래핑)은 쓰지 않는데, 서버 응답은 언제든 변할 수 있어서 런타임 크래시의 원인이 되기 때문입니다.

### 📚 보충 설명

**옵셔널의 동작 원리**

`Optional`은 단순한 키워드가 아니라 다음과 같이 정의된 열거형입니다.

```swift
@frozen public enum Optional<Wrapped> {
    case none
    case some(Wrapped)
}
```

즉 `String?`은 `Optional<String>`의 단축 문법입니다. `nil`은 `.none` 케이스이고, 값이 있을 때는 `.some(value)`로 감싸져 있습니다.

**안전하게 다루는 패턴**

```swift
struct User: Codable {
    let id: Int
    let name: String?      // 서버가 안 보낼 수도 있음
}

// 1. Optional Binding (가장 일반적)
if let name = user.name {
    titleLabel.text = name
}

// 2. guard let (early exit)
guard let name = user.name else { return }
titleLabel.text = name

// 3. Nil-Coalescing
titleLabel.text = user.name ?? "Unknown"

// 4. Optional Chaining (체인 중간에 nil이면 전체가 nil)
let firstChar = user.name?.first
```

**중첩 JSON 안전 파싱 예시**

```swift
// {"user": {"profile": {"avatar": "url"}}}
let avatar = response.user?.profile?.avatar  // 어디서든 nil이면 그냥 nil
```

**강제 언래핑이 위험한 이유**

`!`는 컴파일러에게 "절대 nil이 아니라고 약속한다"고 말하는 것이라, nil인데 강제 언래핑하면 즉시 `Fatal error: Unexpectedly found nil` 크래시가 납니다. 서버 응답·외부 입력·옵셔널 캐스팅 결과에는 절대 쓰지 않습니다.

### 🔗 참고 자료
- [The Swift Programming Language - Optional Chaining](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/optionalchaining/)
- [The Swift Programming Language - The Basics (Optionals)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/thebasics/#Optionals)

---

## 2. Swift에서 클로저(Closure)란 무엇이며 어떻게 사용하나요? (Lv1)

### 💬 면접 답변

클로저는 이름 없는 함수, 즉 일급 객체로서 변수에 담거나 인자로 넘길 수 있는 코드 블록입니다. 가장 큰 특징은 자신이 정의된 컨텍스트의 변수와 상수를 **캡처(capture)** 해서 본문에서 사용할 수 있다는 점입니다. iOS에서는 비동기 콜백, 고차 함수, 애니메이션 블록 등 거의 모든 비동기·함수형 API에서 사용됩니다.

### 📚 보충 설명

**기본 문법**

```swift
let add: (Int, Int) -> Int = { a, b in a + b }
add(1, 2)  // 3
```

**트레일링 클로저(Trailing Closure)**

함수의 마지막 인자가 클로저면, 괄호 밖으로 빼낼 수 있습니다.

```swift
// 일반
UIView.animate(withDuration: 0.3, animations: { view.alpha = 0 })

// Trailing
UIView.animate(withDuration: 0.3) { view.alpha = 0 }
```

Swift 5.3부터는 **다중 트레일링 클로저**도 지원합니다.

```swift
UIView.animate(withDuration: 0.3) {
    view.alpha = 0
} completion: { _ in
    view.removeFromSuperview()
}
```

**`@escaping`과 non-escaping**

- **non-escaping (기본값)**: 클로저가 함수가 리턴하기 전에 호출되고 끝남. 함수 스택 밖으로 나갈 일이 없으므로 컴파일러가 `self` 캡처도 더 자유롭게 허용.
- **`@escaping`**: 클로저가 함수 리턴 후에도 살아남아 호출될 수 있음(저장되거나 비동기 큐로 넘어감). `self`를 강하게 캡처하므로 명시적으로 `[weak self]` 같은 캡처 리스트를 써야 순환 참조를 피할 수 있습니다.

```swift
class ViewModel {
    var onComplete: (() -> Void)?  // 저장되므로 @escaping

    func fetch(completion: @escaping (Data) -> Void) {
        URLSession.shared.dataTask(with: url) { data, _, _ in
            completion(data ?? Data())   // 비동기 호출 → @escaping 필요
        }.resume()
    }
}
```

**캡처(Capture)와 캡처 리스트**

기본적으로 참조 타입은 **강한 참조**로 캡처됩니다. 클로저가 self를 참조하고 self가 클로저를 보유하면 순환 참조가 발생하니, 캡처 리스트로 명시합니다.

```swift
fetch { [weak self] data in
    guard let self else { return }
    self.update(data)
}
```

값 타입은 **값이 복사되어 캡처**됩니다. 단, 클로저 안에서 변경하려면 캡처 리스트에 명시하지 않고 그냥 변수로 잡으면 참조 시맨틱처럼 동작(공유)합니다 — 정확히는 컴파일러가 박스를 만들어 공유합니다.

### 🔗 참고 자료
- [The Swift Programming Language - Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)
- [Swift Evolution: SE-0231 - Optional Iteration](https://github.com/apple/swift-evolution/blob/main/proposals/0231-optional-iteration.md)
- [Swift Evolution: SE-0279 - Multiple Trailing Closures](https://github.com/apple/swift-evolution/blob/main/proposals/0279-multiple-trailing-closures.md)

---

## 3. 값 타입(Value Type)과 참조 타입(Reference Type)의 차이는 무엇인가요? (Lv1)

### 💬 면접 답변

값 타입은 변수에 대입되거나 함수로 전달될 때 **값이 복사**됩니다. `struct`, `enum`, 그리고 `Int`·`String`·`Array`·`Dictionary` 같은 표준 컬렉션이 모두 값 타입입니다. 반면 참조 타입은 같은 인스턴스에 대한 참조를 공유하므로, 한 곳에서 변경하면 다른 참조에도 보입니다. `class`, 클로저, 액터(actor)가 참조 타입입니다. 이런 차이 때문에 **스레드 안전성, 예측 가능성**을 원하면 값 타입을, **identity(동일성)나 상속이 필요**하면 참조 타입을 선택합니다.

### 📚 보충 설명

**저장 위치의 일반화된 설명 (정확하게 말하자면)**

흔히 "값 타입은 스택, 참조 타입은 힙"이라고 단순화해서 말하지만 엄밀히는 다릅니다.

- **값 타입의 *인스턴스 자체*** 는 자신을 담는 컨테이너가 있는 곳에 저장됩니다. 지역 변수면 스택, 클래스의 프로퍼티면 그 클래스가 있는 힙. 또 사이즈가 크거나 escape하면 힙에 boxing될 수 있습니다.
- **참조 타입(클래스 인스턴스)** 은 **힙**에 할당되고, 그 참조(포인터)가 변수에 저장됩니다.

**Copy-on-Write (COW)**

`Array`, `Dictionary`, `Set`, `String` 같은 표준 컬렉션은 값 타입이지만 매번 통째로 복사하면 비효율적이므로 **수정될 때만 실제 복사**가 일어납니다. 내부 버퍼는 참조 카운팅으로 공유되다가, `isKnownUniquelyReferenced(_:)`로 단독 참조 여부를 확인해 필요 시에만 새 버퍼를 만듭니다.

```swift
var a = [1, 2, 3]
var b = a              // 아직 버퍼 공유
b.append(4)            // 이때 비로소 실제 복사 발생
```

**struct를 써야 하는가, class를 써야 하는가**

Apple은 [공식 가이드](https://developer.apple.com/documentation/swift/choosing-between-structures-and-classes)에서 **기본은 struct**, 다음 경우에 class를 선택하라고 권합니다.
- Objective-C 호환성이 필요한 경우
- 인스턴스의 identity를 비교해야 하는 경우 (`===`)
- 상속을 통한 다형성이 필요한 경우
- 인스턴스를 메모리에서 관리해야 하는 경우 (예: `deinit`이 필요한 리소스)

### 🔗 참고 자료
- [The Swift Programming Language - Structures and Classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)
- [Apple Docs - Choosing Between Structures and Classes](https://developer.apple.com/documentation/swift/choosing-between-structures-and-classes)
- [WWDC 2016 - Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/)

---

## 4. Swift의 기본 컬렉션 타입에는 어떤 것들이 있나요? (Lv1)

### 💬 면접 답변

Swift 표준 라이브러리의 핵심 컬렉션은 **`Array`, `Dictionary`, `Set`** 세 가지이며, 모두 값 타입이고 Copy-on-Write로 최적화되어 있습니다. `Array`는 순서가 있고 인덱스로 접근, `Dictionary`는 해시 기반의 키-값 매핑, `Set`은 순서 없고 중복 없는 해시 집합입니다. 추가로 문자열을 다룰 때는 `String`도 컬렉션 프로토콜을 따르고, 시퀀스 처리에는 `Sequence`/`Collection` 프로토콜이 기반이 됩니다.

### 📚 보충 설명

**시간 복잡도 요약 (평균)**

| 연산 | Array | Set | Dictionary |
|------|-------|-----|------------|
| 조회 (`contains`/lookup) | O(n) | O(1) | O(1) |
| 인덱스 접근 | O(1) | – | – |
| 삽입 | O(1) amortized (끝) / O(n) (중간) | O(1) | O(1) |
| 삭제 | O(n) (중간), O(1) (끝) | O(1) | O(1) |

**원시값(Raw Value)과 연관값(Associated Value)을 가진 Enum**

```swift
// 원시값
enum HTTPStatus: Int {
    case ok = 200, notFound = 404, serverError = 500
}

// 연관값
enum NetworkResult {
    case success(Data)
    case failure(Error)
    case retry(after: TimeInterval)
}
```

원시값은 모든 케이스가 같은 타입의 정해진 값을 가지는 반면, 연관값은 케이스별로 다른 타입의 데이터를 함께 묶을 수 있어 훨씬 표현력이 강합니다. `switch`에서 패턴 매칭으로 분해합니다.

### 🔗 참고 자료
- [Swift Standard Library - Collections](https://developer.apple.com/documentation/swift/swift-standard-library/collections)
- [The Swift Programming Language - Collection Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/collectiontypes/)

---

## 5. 구조체(Struct)와 클래스(Class)는 어떻게 구분해서 사용하나요? (Lv1)

### 💬 면접 답변

기본은 struct로 시작하고, 다음 네 가지 경우에만 class를 선택합니다. 첫째, **identity(동일성)** 가 중요할 때 — 예를 들어 같은 인스턴스인지 `===`로 비교해야 하는 경우. 둘째, **상속**이 필요할 때 (UIKit의 UIView 같은 경우). 셋째, **Objective-C 인터롭**이 필요할 때. 넷째, **`deinit`을 통해 리소스 해제**가 필요할 때. 그 외에는 값 타입이 스레드 안전성, 예측 가능성, 성능 면에서 유리합니다.

### 📚 보충 설명

**비교 표**

| 항목 | Struct | Class |
|------|--------|-------|
| 메모리 시맨틱 | 값 (복사) | 참조 |
| 저장 위치 | 일반적으로 스택 (포함된 곳에) | 힙 |
| 상속 | ❌ | ⭕ |
| `deinit` | ❌ | ⭕ |
| ARC | ❌ (필요 없음) | ⭕ |
| `let` 인스턴스의 프로퍼티 | 변경 불가 | 참조는 못 바꾸지만 프로퍼티는 변경 가능 |
| Mutating 메서드 | `mutating` 키워드 필요 | 불필요 |

**`let` 인스턴스의 차이**

```swift
struct Point { var x: Int; var y: Int }
class CPoint { var x: Int; var y: Int; init(...) {...} }

let s = Point(x: 0, y: 0)
s.x = 1   // ❌ 컴파일 에러

let c = CPoint(x: 0, y: 0)
c.x = 1   // ✅ OK (참조는 변경 안 했음)
```

### 🔗 참고 자료
- [Apple Docs - Choosing Between Structures and Classes](https://developer.apple.com/documentation/swift/choosing-between-structures-and-classes)

---

## 6. 프로토콜(Protocol)이란 무엇이며 어떻게 활용하나요? (Lv1)

### 💬 면접 답변

프로토콜은 어떤 타입이 가져야 할 프로퍼티, 메서드, 이니셜라이저 등의 **계약(요구사항)을 정의한 청사진**입니다. 클래스, 구조체, 열거형 모두 채택할 수 있고 다중 채택이 가능해서 다중 상속의 대안이 됩니다. 프로토콜 확장으로 기본 구현을 제공하면 여러 타입에 공통 로직을 주입할 수 있고, 이런 방식이 Swift의 표준 라이브러리 설계 철학인 **프로토콜 지향 프로그래밍(POP)** 의 핵심입니다.

### 📚 보충 설명

**기본 사용**

```swift
protocol Identifiable {
    associatedtype ID: Hashable
    var id: ID { get }
}

struct User: Identifiable {
    let id: UUID
}
```

**프로토콜 확장으로 기본 구현 제공**

```swift
protocol Greetable {
    var name: String { get }
    func greet() -> String
}

extension Greetable {
    func greet() -> String { "Hello, \(name)!" }  // 기본 구현
}

struct Person: Greetable { let name: String }
Person(name: "철수").greet()  // "Hello, 철수!"
```

**메서드 디스패치 주의**

프로토콜 확장에서 정의한 메서드가 **프로토콜 요구사항이 아니면** 정적 디스패치되어, 채택 타입에서 같은 이름으로 재정의해도 프로토콜 타입으로 호출하면 확장 구현이 불립니다.

```swift
protocol P { func a() }              // 요구사항
extension P {
    func a() { print("P.a") }        // 동적 디스패치
    func b() { print("P.b") }        // 요구사항 아님 → 정적 디스패치
}

struct S: P {
    func a() { print("S.a") }
    func b() { print("S.b") }
}

let p: P = S()
p.a()   // "S.a" (요구사항이므로 witness table을 통해 동적 디스패치)
p.b()   // "P.b" ← 주의!
```

**POP의 장점**

- 값 타입에도 다형성을 부여할 수 있음
- 수평적 코드 재사용 (믹스인)
- 상속 트리에 묶이지 않음
- 테스트 용이성 (Mock 객체를 프로토콜 채택으로 쉽게 작성)

### 🔗 참고 자료
- [The Swift Programming Language - Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- [WWDC 2015 - Protocol-Oriented Programming in Swift](https://developer.apple.com/videos/play/wwdc2015/408/)
- [WWDC 2016 - Protocol and Value Oriented Programming in UIKit Apps](https://developer.apple.com/videos/play/wwdc2016/419/)

---

## 7. 접근 제어자(Access Control)에 대해 설명해주세요. (Lv1)

### 💬 면접 답변

Swift는 다섯 단계 접근 제어를 제공합니다 — `open`, `public`, `internal`, `fileprivate`, `private`. 핵심은 모듈(target) 경계와 파일 경계입니다. 라이브러리 외부에 노출할 때는 `public`, 그중 상속까지 허용하려면 `open`을 씁니다. 모듈 내부에서만 쓰면 기본값인 `internal`, 같은 파일 안에서만이면 `fileprivate`, 같은 선언(또는 같은 파일의 확장) 안에서만이면 `private`입니다.

### 📚 보충 설명

**5단계 요약**

| 제어자 | 접근 범위 | 상속/오버라이드 가능 (다른 모듈에서) |
|--------|----------|-------------------------------------|
| `open` | 모든 모듈 | ⭕ |
| `public` | 모든 모듈 | ❌ |
| `internal` (기본값) | 같은 모듈 | – |
| `fileprivate` | 같은 파일 | – |
| `private` | 같은 선언(+ 같은 파일의 extension) | – |

**`open` vs `public` 차이**

- `public class A`: 외부 모듈에서 사용은 가능하지만 **상속과 메서드 오버라이드는 불가**.
- `open class A`: 외부 모듈에서 상속·오버라이드 모두 가능.

라이브러리 API를 설계할 때 의도적으로 상속을 막아 안정성을 보장하려면 `public`이 적절합니다.

**`private`의 미묘함 (Swift 4 이후)**

같은 파일 내 같은 타입의 extension에서는 `private` 멤버 접근이 허용됩니다.

```swift
// File1.swift
struct A {
    private var x = 0
}
extension A {
    func use() { print(x) }   // ✅ 같은 파일이라 접근 가능
}
```

### 🔗 참고 자료
- [The Swift Programming Language - Access Control](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/accesscontrol/)

---

## 8. Swift의 고차 함수(Higher-Order Functions)에 대해 설명해주세요. (Lv1)

### 💬 면접 답변

함수를 인자로 받거나 반환하는 함수를 고차 함수라고 하고, Swift 표준 라이브러리에서는 `map`, `filter`, `reduce`, `compactMap`, `flatMap`이 대표적입니다. 컬렉션의 각 요소를 선언적으로 변환하거나 필터링할 때 사용해 가독성과 안전성을 높입니다.

### 📚 보충 설명

```swift
let numbers = [1, 2, 3, 4, 5]

// map: 각 요소를 변환
numbers.map { $0 * 2 }              // [2, 4, 6, 8, 10]

// filter: 조건을 만족하는 요소만
numbers.filter { $0 % 2 == 0 }      // [2, 4]

// reduce: 누적
numbers.reduce(0, +)                // 15

// compactMap: nil 제거
let strs = ["1", "abc", "3"]
strs.compactMap { Int($0) }         // [1, 3]

// flatMap: 중첩 컬렉션 평탄화
let nested = [[1, 2], [3, 4]]
nested.flatMap { $0 }               // [1, 2, 3, 4]
```

**`map` vs `flatMap` vs `compactMap` 헷갈리지 않게**

- `map`: 1:1 변환. 결과 컨테이너 차원 유지.
- `flatMap`: 결과가 컬렉션이면 한 단계 펼쳐줌. (옵셔널 펼침은 deprecated → `compactMap` 사용)
- `compactMap`: 결과가 옵셔널일 때 `nil`을 제거하고 펼쳐줌.

```swift
let strs = ["1", "abc", "3"]
strs.map { Int($0) }         // [Optional(1), nil, Optional(3)]
strs.compactMap { Int($0) }  // [1, 3]
```

### 🔗 참고 자료
- [Swift Standard Library - Collection (Functional)](https://developer.apple.com/documentation/swift/array)

---

## 9. Swift의 에러 처리 방법에 대해 설명해주세요. (Lv1)

### 💬 면접 답변

Swift는 `Error` 프로토콜을 채택한 타입을 던지고(`throw`), 받는 쪽에서 `do-catch`로 잡거나 `try?`/`try!`로 변환합니다. `throws` 키워드가 붙은 함수는 호출 시 반드시 `try`를 명시해야 하며, 이는 호출자가 에러 가능성을 인지하도록 만드는 명시적 흐름입니다. 비동기 환경에서는 `async throws`와 `try await`를, 결과만 필요한 경우는 `Result` 타입을 활용합니다.

### 📚 보충 설명

```swift
enum NetworkError: Error {
    case invalidURL
    case timeout
    case server(code: Int)
}

func fetch(from urlString: String) throws -> Data {
    guard let url = URL(string: urlString) else { throw NetworkError.invalidURL }
    // ...
    return Data()
}

// 1. do-catch
do {
    let data = try fetch(from: "https://...")
} catch NetworkError.invalidURL {
    print("URL 잘못됨")
} catch NetworkError.server(let code) {
    print("서버 에러 \(code)")
} catch {
    print("기타 에러: \(error)")
}

// 2. try? → 결과를 옵셔널로
let data = try? fetch(from: "https://...")  // 실패하면 nil

// 3. try! → 실패하면 크래시 (확신할 때만)
let data2 = try! fetch(from: "https://...")
```

**`Result` 타입과의 비교**

```swift
// 콜백 기반에서 에러와 성공을 한 타입으로 표현
func fetch(completion: (Result<Data, NetworkError>) -> Void) { ... }

fetch { result in
    switch result {
    case .success(let data): ...
    case .failure(let error): ...
    }
}
```

`async/await`가 도입된 이후로는 `throws` 함수가 더 자연스럽지만, 콜백 API를 다룰 때나 에러를 값으로 들고 다녀야 할 때는 `Result`가 여전히 유용합니다.

**`rethrows`**

인자로 받은 클로저가 throw하는 경우에만 자신도 throw하는 함수를 표현할 때 씁니다. `map`, `filter` 같은 고차 함수가 대표적입니다.

```swift
func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]
```

### 🔗 참고 자료
- [The Swift Programming Language - Error Handling](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/)
- [Swift Evolution: SE-0235 - Result Type](https://github.com/apple/swift-evolution/blob/main/proposals/0235-add-result.md)

---

## 10. 상속(Inheritance)과 프로토콜(Protocol)의 차이점은 무엇인가요? (Lv1)

### 💬 면접 답변

상속은 클래스에서만 가능한 **수직적·is-a 관계**로, 부모의 구현까지 그대로 물려받습니다. 반면 프로토콜은 **수평적 계약**으로, 클래스·구조체·열거형이 모두 채택할 수 있고 다중 채택도 가능합니다. 클래스 상속은 강력하지만 강한 결합과 단일 상속의 제약이 있고, 프로토콜은 느슨한 결합과 다형성을 제공해 테스트와 모듈화에 유리합니다. Swift는 그래서 "프로토콜을 먼저 고려하고, 정말 필요할 때만 상속"이라는 POP 철학을 권장합니다.

### 📚 보충 설명

**다중 상속이 불가능한 이유: 다이아몬드 문제**

C++의 다중 상속에서는 같은 메서드를 여러 부모가 가지고 있을 때 어떤 구현을 쓸지 모호한 다이아몬드 상속 문제가 생깁니다. Swift는 클래스 다중 상속을 금지하는 대신 프로토콜 다중 채택으로 이 문제를 해결합니다. 프로토콜은 (요구사항만 정의하므로) 구현 충돌을 채택하는 타입이 명시적으로 해결합니다.

**다형성 구현**

```swift
protocol Animal {
    func sound() -> String
}

struct Dog: Animal { func sound() -> String { "멍" } }
struct Cat: Animal { func sound() -> String { "야옹" } }

let animals: [Animal] = [Dog(), Cat()]
animals.forEach { print($0.sound()) }
```

### 🔗 참고 자료
- [The Swift Programming Language - Inheritance](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/inheritance/)

---

## 11. 제네릭(Generic)에 대해 설명해주세요. (Lv1)

### 💬 면접 답변

제네릭은 타입을 일반화해서 같은 로직을 다양한 타입에 안전하게 재사용할 수 있게 해주는 기능입니다. 컴파일 타임에 타입이 결정되므로 `Any`와 달리 타입 안전성을 유지하면서 캐스팅 비용도 없습니다. 표준 라이브러리의 `Array<Element>`, `Optional<Wrapped>`, `Result<Success, Failure>`가 모두 제네릭입니다. 제약 조건(`where`, 프로토콜 제약)을 통해 어떤 타입이 와도 되는지 좁힐 수 있습니다.

### 📚 보충 설명

```swift
// 제네릭 함수
func swapValues<T>(_ a: inout T, _ b: inout T) {
    let tmp = a; a = b; b = tmp
}

// 제네릭 타입
struct Stack<Element> {
    private var items: [Element] = []
    mutating func push(_ item: Element) { items.append(item) }
    mutating func pop() -> Element? { items.popLast() }
}

// 제약 조건
func max<T: Comparable>(_ a: T, _ b: T) -> T {
    a > b ? a : b
}

// where 절
extension Array where Element: Numeric {
    func sum() -> Element { reduce(0, +) }
}
```

**Associated Type을 가진 프로토콜(PAT)과의 차이**

`some` 타입과 `any` 타입(Swift 5.7+)으로 PAT를 다루는 방식이 바뀌었습니다.

```swift
protocol Container {
    associatedtype Item
    var count: Int { get }
    subscript(i: Int) -> Item { get }
}

// some: 컴파일러가 구체 타입을 추론 (Opaque type)
func makeContainer() -> some Container { ... }

// any: 런타임 다형성을 위한 존재 타입 (Existential)
let c: any Container = ...
```

`some`은 효율이 좋고(타입이 정해진 것처럼 동작), `any`는 런타임에 다양한 구체 타입을 담을 수 있지만 박싱 비용이 듭니다.

### 🔗 참고 자료
- [The Swift Programming Language - Generics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)
- [Swift Evolution: SE-0244 - Opaque Result Types](https://github.com/apple/swift-evolution/blob/main/proposals/0244-opaque-result-types.md)
- [Swift Evolution: SE-0335 - Existential `any`](https://github.com/apple/swift-evolution/blob/main/proposals/0335-existential-any.md)
- [WWDC 2022 - Embrace Swift generics](https://developer.apple.com/videos/play/wwdc2022/110352/)

---

## 12. 옵셔널을 사용할 때 주의할 점은 무엇인가요? (Lv2)

### 💬 면접 답변

가장 중요한 원칙은 **강제 언래핑(`!`)을 쓰지 않는 것**입니다. `nil` 가능성이 있는 모든 곳에서는 `if let`, `guard let`, 옵셔널 체이닝, 또는 `??`로 안전하게 처리해야 합니다. 또한 옵셔널 바인딩과 옵셔널 체이닝을 구분해서 사용해야 하는데, 바인딩은 값을 꺼내 사용하는 용도이고 체이닝은 nil이면 전체 표현식이 nil이 되는 단축 평가입니다. IBOutlet처럼 라이프사이클상 나중에 초기화되는 경우에만 `IUO`(`!`)를 제한적으로 사용합니다.

### 📚 보충 설명

**옵셔널 체이닝 vs 바인딩**

```swift
// 체이닝: 중간에 nil이면 전체가 nil. 옵셔널 결과 반환.
let length: Int? = user?.address?.city.count

// 바인딩: 값을 꺼내 변수에 담음. 비-옵셔널로 사용.
if let city = user?.address?.city {
    print(city.count)
}
```

**Implicitly Unwrapped Optional (IUO)**

- 선언: `var label: UILabel!`
- 사용 시 자동으로 언래핑되지만, 여전히 옵셔널이라 nil이면 크래시.
- 사용 정당화: IBOutlet, 의존성 주입 컨테이너에서 초기화가 보장되는 경우.

**`if let` 단축 문법(Swift 5.7+)**

```swift
let name: String? = "..."
if let name {           // if let name = name 의 단축
    print(name)
}
```

### 🔗 참고 자료
- [Swift Evolution: SE-0345 - if let shorthand](https://github.com/apple/swift-evolution/blob/main/proposals/0345-if-let-shorthand.md)
- [The Swift Programming Language - Optional Chaining](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/optionalchaining/)

---

## 13. Codable 프로토콜은 무엇이며 어떻게 사용하나요? (Lv2)

### 💬 면접 답변

`Codable`은 `Encodable & Decodable`의 type alias로, 타입을 외부 표현(JSON, plist 등)으로 인코딩하고 다시 복원할 수 있게 해주는 프로토콜입니다. 표준 라이브러리의 기본 타입과 컬렉션은 이미 채택되어 있어서, 우리가 만든 모델이 단지 `Codable`을 채택하기만 하면 `JSONEncoder`/`JSONDecoder`로 양방향 변환이 됩니다. 키 매핑은 `CodingKeys`로, 복잡한 변환은 커스텀 `init(from:)`이나 `encode(to:)`로 처리합니다.

### 📚 보충 설명

**기본 사용**

```swift
struct User: Codable {
    let id: Int
    let name: String
    let email: String?
}

let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase  // user_name → userName
decoder.dateDecodingStrategy = .iso8601

let user = try decoder.decode(User.self, from: data)
```

**`CodingKeys`로 커스텀 매핑**

```swift
struct User: Codable {
    let id: Int
    let userName: String

    enum CodingKeys: String, CodingKey {
        case id
        case userName = "user_name"   // JSON 키와 매핑
    }
}
```

**복잡한 구조에서 커스텀 디코더**

```swift
struct Price: Decodable {
    let amount: Decimal
    let currency: String

    enum CodingKeys: String, CodingKey { case amount, currency }

    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        // 서버가 문자열로 보낼 경우 대응
        if let s = try? c.decode(String.self, forKey: .amount),
           let d = Decimal(string: s) {
            self.amount = d
        } else {
            self.amount = try c.decode(Decimal.self, forKey: .amount)
        }
        self.currency = try c.decode(String.self, forKey: .currency)
    }
}
```

**자주 쓰는 디코딩 전략**

- `keyDecodingStrategy = .convertFromSnakeCase`: 서버가 snake_case일 때
- `dateDecodingStrategy = .iso8601` / `.millisecondsSince1970`
- `dataDecodingStrategy = .base64`

### 🔗 참고 자료
- [Apple Docs - Encoding and Decoding Custom Types](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)
- [Apple Docs - JSONDecoder](https://developer.apple.com/documentation/foundation/jsondecoder)

---

## 14. Result 타입의 사용 이유와 장점은 무엇인가요? (Lv2)

### 💬 면접 답변

`Result<Success, Failure: Error>`는 성공값과 실패 에러를 하나의 값으로 표현하는 열거형입니다. 콜백 기반의 비동기 API에서 성공 시 값을, 실패 시 에러를 동시에 받아야 할 때 옵셔널 두 개나 `(Value?, Error?)` 같은 모호한 형태 대신 명확하게 다룰 수 있습니다. `map`, `flatMap`, `mapError` 같은 함수형 연산이 가능해 변환 파이프라인을 깔끔하게 작성할 수 있다는 장점도 있습니다.

### 📚 보충 설명

```swift
public enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}
```

**do-catch와 Result 변환**

```swift
// throws → Result
let result = Result { try fetch() }   // Result<Data, Error>

// Result → throws
do {
    let data = try result.get()
} catch {
    print(error)
}
```

**언제 어떤 걸 쓸까**

- **`async throws`** 가 가능한 환경: 보통 throws가 더 자연스럽고 간결.
- **콜백 API, 값으로 들고 다녀야 할 때, 에러 변환이 잦을 때**: `Result`가 유리.

### 🔗 참고 자료
- [Swift Evolution: SE-0235 - Result](https://github.com/apple/swift-evolution/blob/main/proposals/0235-add-result.md)
- [Apple Docs - Result](https://developer.apple.com/documentation/swift/result)

---

## 15. Swift의 문자열(String)을 다룰 때 알아야 할 것들은? (Lv2)

### 💬 면접 답변

Swift의 `String`은 **유니코드 정확성**을 위해 인덱스를 `Int`가 아닌 `String.Index`로 다룹니다. 즉 `s[5]`처럼 정수 인덱싱이 불가능하고 `s.index(s.startIndex, offsetBy: 5)`로 접근합니다. 이는 한 글자가 여러 바이트(grapheme cluster)일 수 있어 O(1) 정수 인덱싱이 정확성을 보장하지 못하기 때문입니다. 부분 문자열은 메모리 효율을 위해 별도의 `Substring` 타입을 쓰며, 장기 보관하려면 `String(substring)`으로 복사해야 합니다.

### 📚 보충 설명

**인덱스 다루기**

```swift
let s = "Hello, 안녕"
let i = s.index(s.startIndex, offsetBy: 7)
print(s[i])   // "안"

// 범위
let range = s.index(s.startIndex, offsetBy: 0)..<s.index(s.startIndex, offsetBy: 5)
print(s[range])  // "Hello"
```

**Substring**

```swift
let original = "Hello, World"
let sub = original.prefix(5)        // Substring("Hello")
// sub은 original의 메모리를 참조 → original이 살아있는 동안만 유효
let copied = String(sub)            // 별도 메모리로 복사
```

**문자열 보간법(String Interpolation)**

```swift
let name = "철수"
let age = 30
let msg = "이름: \(name), 나이: \(age)"

// 커스텀 보간 (Swift 5+)
extension String.StringInterpolation {
    mutating func appendInterpolation(currency value: Decimal) {
        appendLiteral(value.formatted(.currency(code: "KRW")))
    }
}
print("가격: \(currency: 12345)")  // "가격: ₩12,345"
```

**정규식 (Swift 5.7+)**

```swift
import Foundation
let s = "전화번호: 010-1234-5678"
if let match = s.firstMatch(of: /(\d{3})-(\d{4})-(\d{4})/) {
    print(match.1, match.2, match.3)  // 010 1234 5678
}
```

### 🔗 참고 자료
- [Apple Docs - String](https://developer.apple.com/documentation/swift/string)
- [Swift.org - Strings and Characters](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/stringsandcharacters/)
- [Swift Evolution: SE-0354 - Regex Literals](https://github.com/apple/swift-evolution/blob/main/proposals/0354-regex-literals.md)
- [WWDC 2022 - Meet Swift Regex](https://developer.apple.com/videos/play/wwdc2022/110357/)

---

## 16. 클로저와 함수의 차이점은? 클로저가 일급 객체라는 게 무슨 의미인가요? (Lv1)

### 💬 면접 답변

문법적으로 함수는 클로저의 특수한 형태이고, 본질적으로 같은 것입니다. 둘 다 일급 객체이기 때문에 변수에 담거나, 인자로 전달하거나, 반환값으로 돌려줄 수 있습니다. 차이는 함수는 이름이 있고 모듈 수준에서 선언될 수 있는 반면, 클로저는 보통 익명이고 주변 컨텍스트를 캡처할 수 있다는 점입니다. 일급 객체라는 것은 다른 데이터 타입(Int, String 같은)과 동등하게 다룰 수 있다는 의미이고, 이 덕분에 `map`, `filter` 같은 함수형 패턴이 가능합니다.

### 📚 보충 설명

세 가지 일급 객체의 조건을 만족합니다.
1. 변수에 할당 가능
2. 함수의 인자로 전달 가능
3. 함수의 반환값으로 사용 가능

```swift
let f: (Int) -> Int = { $0 * 2 }   // ① 변수 할당
[1, 2, 3].map(f)                   // ② 인자 전달

func makeAdder(_ x: Int) -> (Int) -> Int {
    return { $0 + x }              // ③ 반환값
}
```

### 🔗 참고 자료
- [The Swift Programming Language - Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)

---

## 17. 옵셔널을 캡처하는 클로저에서 `[weak self]`와 `[unowned self]` 차이는? (Lv1)

### 💬 면접 답변

둘 다 클로저가 `self`를 캡처할 때 순환 참조를 막기 위한 캡처 방식인데, `weak`은 `self`가 해제될 수 있다고 보고 옵셔널로 캡처합니다. 그래서 클로저 안에서 `guard let self else { return }`처럼 안전하게 다뤄야 합니다. 반면 `unowned`는 클로저 실행 시점에 `self`가 반드시 살아있다는 보장이 있을 때만 쓰며, 옵셔널이 아닌 형태로 더 가볍게 접근합니다. 다만 해제된 후 접근하면 즉시 크래시이므로 라이프사이클에 100% 확신이 있을 때만 씁니다. 의심스러우면 `weak`이 안전한 기본 선택입니다.

### 📚 보충 설명

**선택 기준**

| 상황 | 선택 |
|------|------|
| self가 클로저보다 먼저 해제될 수 있음 | `weak self` |
| self의 수명이 클로저 수명 이상으로 보장됨 | `unowned self` |
| 확신이 없음 | `weak self` (안전 우선) |

**`guard let self` 패턴(Swift 5.7+)**

```swift
fetch { [weak self] data in
    guard let self else { return }
    self.update(data)        // 이후 self는 비-옵셔널
    self.refresh()
}
```

Swift 5.8부터는 `guard let self` 자체 단축 문법(`guard let self else { return }`)이 지원되어 더 간결하게 쓸 수 있습니다.

**unowned의 함정**

```swift
class A {
    var closure: (() -> Void)?
    deinit { print("A deinit") }
}

var a: A? = A()
a?.closure = { [unowned self] in print(self) }  // self == a
// a = nil 이후 closure 호출 시 → 크래시
```

### 🔗 참고 자료
- [The Swift Programming Language - Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- [Swift Evolution: SE-0365 - Implicit `self` for weak self](https://github.com/apple/swift-evolution/blob/main/proposals/0365-implicit-self-weak-capture.md)

---

## 18. Key Path란 무엇이며 어떻게 사용하나요? (Lv2)

### 💬 면접 답변

Key Path는 어떤 타입의 특정 프로퍼티에 접근하는 경로를 **값**으로 표현한 것입니다. 즉 "타입 `Person`의 `name` 프로퍼티" 같은 정보를 `\Person.name`처럼 적어 변수에 담을 수 있습니다. 덕분에 함수의 인자로 프로퍼티 자체를 넘기거나, 컬렉션의 정렬·매핑을 선언적으로 표현할 수 있습니다. SwiftUI의 `ForEach`, Combine의 `assign(to:)`, KVO 등이 모두 Key Path를 사용합니다.

### 📚 보충 설명

```swift
struct Person {
    var name: String
    var age: Int
}

let nameKP: KeyPath<Person, String> = \Person.name
let p = Person(name: "철수", age: 30)
print(p[keyPath: nameKP])   // "철수"

// 활용 1: map의 단축
let people = [Person(name: "A", age: 1), Person(name: "B", age: 2)]
let names = people.map(\.name)             // ["A", "B"]

// 활용 2: 정렬
let sorted = people.sorted { $0.age < $1.age }
```

**`WritableKeyPath`, `ReferenceWritableKeyPath`**

- `KeyPath`: 읽기 전용
- `WritableKeyPath`: 값 타입(struct)의 mutable 프로퍼티에 쓰기 가능
- `ReferenceWritableKeyPath`: 참조 타입(class)의 mutable 프로퍼티에 쓰기 가능

**KVO와의 관계**

KVO에서 옵저버를 추가할 때 키 경로 표현식을 사용합니다. (Objective-C 런타임이 필요해서 클래스가 `NSObject`를 상속하고 `@objc dynamic` 프로퍼티여야 합니다.)

```swift
class Observable: NSObject {
    @objc dynamic var count: Int = 0
}

let o = Observable()
let token = o.observe(\.count, options: [.new]) { _, change in
    print(change.newValue ?? 0)
}
```

### 🔗 참고 자료
- [Apple Docs - KeyPath](https://developer.apple.com/documentation/swift/keypath)
- [Swift Evolution: SE-0161 - Smart KeyPaths](https://github.com/apple/swift-evolution/blob/main/proposals/0161-key-paths.md)

---

## 19. 사용자 정의 연산자(Custom Operator)는 어떻게 만들고, 언제 쓸까요? (Lv2)

### 💬 면접 답변

사용자 정의 연산자는 `prefix`, `infix`, `postfix` 키워드로 선언하고, 우선순위 그룹(`precedencegroup`)에 속하게 한 뒤 함수로 구현합니다. 수학 라이브러리나 도메인 특화 코드처럼 표현력을 명확히 높일 때 유용하지만, 남용하면 가독성이 떨어지므로 팀 컨벤션 안에서 신중히 도입해야 합니다.

### 📚 보충 설명

```swift
// 1) 연산자 선언
infix operator +++ : AdditionPrecedence

// 2) 구현
func +++ (lhs: String, rhs: String) -> String {
    "\(lhs) ✨ \(rhs)"
}

print("Hello" +++ "World")  // "Hello ✨ World"
```

**Precedence Group**

```swift
precedencegroup MyPrecedence {
    associativity: left
    higherThan: AdditionPrecedence
}
infix operator ** : MyPrecedence

func ** (lhs: Int, rhs: Int) -> Int { Int(pow(Double(lhs), Double(rhs))) }
```

**주의점**

- 이미 있는 연산자를 무의미하게 오버로드하지 않기 (예: `+`를 더하기가 아닌 연결로)
- 팀이 이해할 수 있는 도메인 의미가 있을 때만
- SwiftUI의 `~=`, Combine의 파이프 같은 표준 컨벤션을 우선

### 🔗 참고 자료
- [The Swift Programming Language - Advanced Operators](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/advancedoperators/)

---

## 20. Initializer의 종류와 차이점은? (Lv2)

### 💬 면접 답변

클래스의 이니셜라이저는 크게 **지정 이니셜라이저(designated)** 와 **편의 이니셜라이저(convenience)** 로 나뉩니다. 지정은 모든 저장 프로퍼티를 초기화하는 메인 진입점이고, 편의는 같은 클래스의 다른 이니셜라이저를 호출하는 보조 역할입니다. 또한 실패할 수 있는 상황은 **failable initializer(`init?`)**, 서브클래스가 반드시 구현해야 하는 경우는 **required initializer(`required init`)** 로 표현합니다.

### 📚 보충 설명

**Designated vs Convenience**

```swift
class Animal {
    var name: String
    var age: Int

    // designated: 모든 저장 프로퍼티 초기화
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }

    // convenience: 같은 클래스의 다른 init을 호출 (self.init)
    convenience init(name: String) {
        self.init(name: name, age: 0)
    }
}
```

**상속 관계의 이니셜라이저 규칙**

1. 서브클래스의 **designated**는 부모의 **designated**를 반드시 호출.
2. 서브클래스의 **convenience**는 같은 클래스의 다른 init만 호출 가능.
3. 두 단계 초기화(Two-Phase Initialization): 모든 저장 프로퍼티 초기화 → 그 후 메서드/self 사용.

**Failable Initializer**

```swift
struct Age {
    let value: Int
    init?(_ v: Int) {
        guard v >= 0 else { return nil }
        self.value = v
    }
}
```

**Required Initializer**

```swift
class Base {
    required init() {}     // 모든 서브클래스가 구현해야 함
}

class Sub: Base {
    required init() { super.init() }
}
```

`required`는 `UIView(coder:)`처럼 시스템이 특정 이니셜라이저를 항상 기대할 때 사용됩니다.

### 🔗 참고 자료
- [The Swift Programming Language - Initialization](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/initialization/)

---

> 이 문서는 Swift 5.9+ / iOS 17+ 기준으로 작성되었습니다. 다른 주제는 [README.md](./README.md)의 목차를 참고하세요.
