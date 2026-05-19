---
layout: default
title: "07. 아키텍처와 디자인 패턴"
nav_order: 7
---

# 07. 아키텍처와 디자인 패턴

> MVC/MVVM/MVVM-C/VIPER/Clean Architecture, 싱글톤, 의존성 주입, Delegation, Observer.

---

## 1. 싱글톤 패턴(Singleton Pattern)이란? 언제 사용하나요? (Lv0)

### 💬 면접 답변

싱글톤은 어떤 클래스의 인스턴스가 앱 전체에서 **단 하나만 존재**하도록 보장하는 패턴입니다. iOS에서는 `URLSession.shared`, `UserDefaults.standard`, `FileManager.default` 같은 시스템 API가 모두 싱글톤이고, 우리 코드에서는 로깅·분석·세션 관리처럼 공유 자원이 필요한 곳에 종종 씁니다. 다만 남용하면 **전역 상태가 늘어나 테스트가 어렵고**, 의존성이 숨겨져 결합도가 높아지므로, 정말로 인스턴스가 하나여야 하는 경우에만 신중히 사용하고 그 외에는 의존성 주입을 우선 고려하는 게 좋습니다.

### 📚 보충 설명

**Swift에서 안전한 싱글톤**

```swift
final class AnalyticsManager {
    static let shared = AnalyticsManager()
    private init() {}

    func log(_ event: String) { ... }
}
```

`let`으로 선언된 전역/static은 Swift 런타임이 `dispatch_once`와 같은 의미로 **lazy + thread-safe**하게 한 번만 초기화하므로 자체적으로 스레드 안전입니다. `private init`으로 외부에서 새 인스턴스 생성을 막습니다.

**싱글톤의 문제점**

1. **숨겨진 의존성**: 함수 시그니처만 보면 어떤 자원을 쓰는지 모름.
2. **테스트 어려움**: Mock으로 대체하기 까다로움.
3. **전역 상태**: 동시성 문제, 예측 어려운 부작용.

**대안**

```swift
// 1. 프로토콜 + DI
protocol Analytics { func log(_ event: String) }

final class HomeViewModel {
    let analytics: Analytics
    init(analytics: Analytics = AnalyticsManager.shared) {
        self.analytics = analytics
    }
}

// 테스트에서:
class MockAnalytics: Analytics { ... }
let vm = HomeViewModel(analytics: MockAnalytics())
```

싱글톤을 직접 호출하는 대신 **프로토콜에 의존**하고, 기본값으로 싱글톤을 주입받는 방식이면 일상 코드 작성 편의와 테스트 용이성을 모두 잡습니다.

### 🔗 참고 자료
- [Apple - Managing a Shared Resource Using a Singleton](https://developer.apple.com/documentation/swift/managing-a-shared-resource-using-a-singleton)

---

## 2. 의존성 주입(Dependency Injection)이란? (Lv2)

### 💬 면접 답변

의존성 주입은 어떤 객체가 필요로 하는 다른 객체(의존성)를 **자체적으로 생성하지 않고 외부에서 주입받는** 설계 패턴입니다. 직접 `URLSession.shared`나 싱글톤을 호출하는 대신, 생성자나 프로퍼티로 받아 사용합니다. 이렇게 하면 테스트 시 Mock으로 쉽게 대체할 수 있고, 같은 클래스를 다른 환경에서 다르게 구성할 수 있어 유연성·테스트 용이성이 크게 올라갑니다. 주입 방식은 **생성자 주입**(가장 권장), **프로퍼티 주입**, **메서드 주입** 세 가지가 있습니다.

### 📚 보충 설명

**세 가지 주입 방식**

```swift
// 1. Initializer Injection (가장 권장)
class UserService {
    let api: APIClient
    init(api: APIClient) { self.api = api }
}

// 2. Property Injection
class UserService {
    var api: APIClient!
}
let s = UserService()
s.api = realAPI

// 3. Method Injection
class UserService {
    func fetch(api: APIClient) { ... }
}
```

**생성자 주입의 장점**
- 의존성이 명시적 (시그니처만 봐도 파악)
- 불변(let) 가능 → 안전
- 컴파일 타임에 누락 검출

**DI Container**

복잡한 그래프에서는 컨테이너 사용:
- [Swinject](https://github.com/Swinject/Swinject)
- [Factory](https://github.com/hmlongco/Factory)
- iOS 17+ SwiftUI의 `@Environment` 시스템

**Service Locator 안티 패턴**

```swift
// ❌ Service Locator: 의존성을 숨기는 안티 패턴
class UserService {
    func fetch() {
        let api = Container.resolve(APIClient.self)   // 의존성이 숨겨짐
        ...
    }
}
```

겉보기엔 DI 같지만, 호출부에서 어떤 의존성이 필요한지 알 수 없어 의존성 추적이 어렵습니다.

### 🔗 참고 자료
- [Apple - Dependency Injection](https://developer.apple.com/wwdc23/10257) (참고: 일반적인 DI 개념은 다양한 자료 참고)

---

## 3. Delegation 패턴과 클로저(Callback)의 차이는? (Lv2)

### 💬 면접 답변

Delegation은 프로토콜을 채택한 객체에게 특정 시점에 콜백 메서드들을 호출하는 패턴으로, 보통 **1:1 관계에 여러 콜백이 필요할 때** 적합합니다. UITableView의 dataSource·delegate가 대표적입니다. 클로저는 **단발성·단일 콜백**에 가볍게 쓰기 좋고, 인라인으로 작성할 수 있어 가독성이 좋습니다. 다만 클로저는 self를 강하게 캡처할 수 있어 순환 참조에 주의해야 합니다. 콜백이 여러 종류면 Delegate, 단일 결과 알림이면 Closure, 시간에 따른 스트림이면 Combine/Publisher를 고려합니다.

### 📚 보충 설명

**Delegate에서의 메모리 누수 케이스**

```swift
protocol PickerDelegate: AnyObject {
    func didSelect(_ value: String)
}

class Picker {
    weak var delegate: PickerDelegate?   // 반드시 weak
}
```

`AnyObject` 제약 + `weak`이 표준 조합. 클래스에만 채택 가능해지므로 weak 참조가 가능해집니다.

**클로저의 캡처 리스트**

```swift
class DataSource {
    var onUpdate: ((Data) -> Void)?

    func load() {
        api.fetch { [weak self] data in
            self?.onUpdate?(data)
        }
    }
}
```

**어떻게 선택할까**

| 통신 모양 | 추천 |
|----------|------|
| 1:1, 콜백 여러 개 (events) | Delegate |
| 1:1, 단일 결과 | Closure |
| 1:N, 같은 이벤트를 여러 구독자에게 | NotificationCenter / Combine |
| 시간에 따른 값 스트림 | Combine / AsyncSequence |

---

## 4. NotificationCenter는 어떤 경우에 사용하나요? Observer 패턴과의 관계는? (Lv1)

### 💬 면접 답변

NotificationCenter는 발신자와 수신자가 서로 직접 참조하지 않고도 이름 기반으로 메시지를 주고받게 하는 **publish-subscribe**(Observer 패턴의 한 형태) 메커니즘입니다. 서로 모르는 모듈 사이에 느슨한 결합으로 이벤트를 알릴 때 유용한데, 예를 들어 로그아웃이 발생했음을 여러 화면에 동시에 알리거나, 키보드가 올라옴을 알리는 시스템 노티(`UIResponder.keyboardWillShowNotification`)가 대표적입니다. 다만 **타입 안전성이 낮고**, 흐름이 코드만 봐서는 추적이 어려워서, 강한 결합을 만들지 않으려고 무분별하게 쓰면 디버깅이 힘들어집니다. 명확한 1:1 관계는 Delegate, 1:N의 데이터 흐름은 Combine을 우선 고려합니다.

### 📚 보충 설명

**기본 사용**

```swift
// 발행
NotificationCenter.default.post(name: .userDidLogin, object: nil, userInfo: ["id": 1])

// 구독
let token = NotificationCenter.default.addObserver(
    forName: .userDidLogin, object: nil, queue: .main
) { notification in
    let id = notification.userInfo?["id"] as? Int
    print("logged in: \(id ?? -1)")
}

// 해제
NotificationCenter.default.removeObserver(token)
```

**iOS 9+ Auto remove**

`addObserver(_:selector:name:object:)`로 등록한 옵저버는 등록 객체가 deinit되면 시스템이 자동 정리합니다. 하지만 `forName:queue:using:` 형태(블록 기반)는 토큰을 수동으로 해제해야 합니다.

**Combine과의 결합**

```swift
NotificationCenter.default.publisher(for: .userDidLogin)
    .compactMap { $0.userInfo?["id"] as? Int }
    .sink { print("id: \($0)") }
    .store(in: &cancellables)
```

### 🔗 참고 자료
- [Apple - NotificationCenter](https://developer.apple.com/documentation/foundation/notificationcenter)

---

## 5. MVC, MVVM, MVVM-C의 차이는? (Lv3)

### 💬 면접 답변

MVC는 Apple이 전통적으로 권장한 패턴으로 **Model-View-Controller** 3개의 역할을 분리합니다. 다만 iOS의 ViewController가 View 관리와 비즈니스 로직을 모두 떠안게 되면서 흔히 "**Massive View Controller**"라는 자조가 나옵니다. MVVM은 ViewController(또는 View)와 도메인 로직 사이에 **ViewModel** 레이어를 두고, ViewModel이 데이터 가공·상태 변환을 담당해 ViewController는 표시에만 집중하게 합니다. MVVM-C는 여기에 화면 전환을 책임지는 **Coordinator**를 추가해, ViewController가 다음 화면을 모르도록 분리합니다. 화면 흐름이 복잡하거나 딥링크 처리가 많은 앱에서 효과적입니다.

### 📚 보충 설명

**MVC의 한계**

```swift
class UserViewController: UIViewController {
    var users: [User] = []                  // Model 보유
    @IBOutlet var tableView: UITableView!   // View
    func fetchUsers() {                     // 비즈니스 로직
        API.shared.get { ... }
    }
    func tableView(_ tv: UITableView, cellForRowAt: IndexPath) -> ...  // DataSource
}
```

→ View, Controller, 비즈니스 로직, 데이터 소스가 한 클래스에 다 들어감.

**MVVM 구조**

```swift
final class UserListViewModel {
    @Published var users: [User] = []

    private let api: APIClient
    init(api: APIClient) { self.api = api }

    func load() async {
        users = (try? await api.users()) ?? []
    }
}

final class UserListViewController: UIViewController {
    let vm: UserListViewModel
    var bag = Set<AnyCancellable>()

    init(vm: UserListViewModel) { self.vm = vm; super.init(nibName: nil, bundle: nil) }

    override func viewDidLoad() {
        vm.$users
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in self?.tableView.reloadData() }
            .store(in: &bag)

        Task { await vm.load() }
    }
}
```

**MVVM-C (+ Coordinator)**

```swift
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    func start()
}

final class AppCoordinator: Coordinator {
    var childCoordinators: [Coordinator] = []
    let navigationController: UINavigationController

    init(_ nav: UINavigationController) { self.navigationController = nav }

    func start() {
        let vm = UserListViewModel(api: AppEnvironment.shared.api)
        let vc = UserListViewController(vm: vm)
        vc.didSelectUser = { [weak self] user in
            self?.showDetail(user)        // 화면 전환은 Coordinator가 결정
        }
        navigationController.pushViewController(vc, animated: false)
    }

    func showDetail(_ user: User) { ... }
}
```

**Coordinator의 장점**
- ViewController가 다음 화면을 모름 → 재사용성 ↑
- 화면 전환 로직 한 곳에 집중 → 딥링크·인증 흐름 처리 쉬움
- 의존성 주입을 Coordinator가 일괄 담당

**단점**
- 작은 앱엔 오버엔지니어링
- 보일러플레이트 증가
- 메모리 관리 (parent-child Coordinator 관계) 신경 써야 함

### 🔗 참고 자료
- [Apple - Model-View-Controller (Archive)](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html)
- 화면 흐름과 DI에 관한 발표: WWDC 세션은 Coordinator 패턴을 명시적으로 다루진 않지만, 커뮤니티에 풍부한 자료가 있음.

---

## 6. 레거시 MVC 프로젝트를 MVVM으로 점진적으로 마이그레이션하려면? (Lv2)

### 💬 면접 답변

빅뱅 마이그레이션은 위험하므로 **모듈 단위, 화면 단위로 점진적**으로 진행합니다. 우선순위는 두 축으로 정합니다 — 첫째, **테스트 커버리지가 높은 부분**(회귀 위험이 낮음), 둘째, **비즈니스 로직이 많아 효과가 큰 부분**(개선 효과가 큼). 새로 만드는 화면은 MVVM으로 시작하고, 기존 화면은 ViewController에서 비즈니스 로직을 ViewModel로 추출하는 리팩토링부터 합니다. 과도기에는 MVC와 MVVM이 공존하므로, 팀 가이드로 **새 코드는 MVVM**, **기존 코드는 손볼 때 MVVM으로 전환**이라는 규칙을 명확히 합니다. 성공 지표는 ViewController 평균 LOC 감소, 단위 테스트 커버리지 증가, 새 기능 개발 속도로 측정합니다.

### 📚 보충 설명

**Step-by-step**

1. **ViewModel 추출**: VC에서 비즈니스 로직 → ViewModel로 이동
2. **바인딩**: `@Published` + Combine, 또는 `@Observable` (iOS 17+) 로 ViewModel ↔ View
3. **테스트 추가**: ViewModel은 View와 무관하므로 단위 테스트 작성하기 좋음
4. **Coordinator 도입 (선택)**: 화면 전환이 복잡한 경우 후속 도입

**과도기 충돌 줄이기**

- 새 코드가 어디 들어가는지 헷갈리지 않게 폴더/모듈 분리 (예: `Legacy/`, `New/`)
- 같은 화면을 두 패턴으로 동시 작업하지 않기
- 코드 리뷰에 명확한 규칙 추가

---

## 7. Clean Architecture를 iOS에 적용한다면? (Lv5)

### 💬 면접 답변

Clean Architecture는 **의존성 방향이 항상 안쪽(도메인) 으로** 향하도록 레이어를 나눈 구조입니다. iOS에서는 보통 **Presentation(UI/ViewModel) → Domain(UseCase, Entity) → Data(Repository 구현)** 세 레이어로 나누고, Data 레이어의 구체 구현은 Domain의 프로토콜(Repository interface)에 맞춥니다. 이렇게 하면 UseCase와 도메인 모델은 UIKit·CoreData·URLSession 같은 외부 프레임워크를 알지 못하므로 **순수하게 단위 테스트** 가능하고, 데이터 소스를 바꿔도(서버 → 로컬 캐시) Domain은 영향받지 않습니다. 다만 작은 앱에서는 레이어와 보일러플레이트의 비용이 효익을 넘을 수 있어, 도메인이 충분히 복잡하거나 장수 프로젝트일 때 도입하는 게 합리적입니다.

### 📚 보충 설명

**의존성 방향**

```
Presentation  →  Domain  ←  Data
 (UI, VM)        (UseCase    (Repository 구현,
                  Entity,     API/DB)
                  Repo
                  Interface)
```

**예시 구조**

```swift
// Domain
struct User { let id: Int; let name: String }
protocol UserRepository { func fetch() async throws -> [User] }

final class FetchUsersUseCase {
    private let repo: UserRepository
    init(repo: UserRepository) { self.repo = repo }
    func execute() async throws -> [User] { try await repo.fetch() }
}

// Data
final class UserAPIRepository: UserRepository {
    func fetch() async throws -> [User] { /* URLSession ... */ }
}

// Presentation
@MainActor
final class UserListViewModel: ObservableObject {
    @Published var users: [User] = []
    private let usecase: FetchUsersUseCase

    init(usecase: FetchUsersUseCase) { self.usecase = usecase }

    func load() async {
        users = (try? await usecase.execute()) ?? []
    }
}
```

**장점**
- Domain 100% 단위 테스트 가능
- 데이터 소스 교체 자유
- 비즈니스 로직이 UI 프레임워크와 분리

**단점**
- 화면이 단순해도 4~5개 파일 필요
- 매핑 로직(DTO ↔ Entity) 추가
- 학습 비용

### 🔗 참고 자료
- Robert C. Martin의 [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) 원문

---

## 8. VIPER 아키텍처는 무엇인가요? (Lv5)

### 💬 면접 답변

VIPER는 **View, Interactor, Presenter, Entity, Router** 다섯 컴포넌트로 책임을 더 세밀하게 나눈 아키텍처입니다. View는 표시만, Presenter는 View ↔ Interactor 사이의 통신과 표시 데이터 가공, Interactor는 비즈니스 로직, Entity는 모델, Router는 화면 전환을 담당합니다. 각 컴포넌트가 프로토콜로 정의되어 단위 테스트와 모듈화가 매우 쉽지만, **간단한 화면도 5개의 파일과 다수의 프로토콜이 필요**해서 작은 앱에는 과한 면이 있습니다. 큰 팀, 큰 코드베이스, 명확한 책임 분리가 절실한 경우에 효과를 봅니다.

### 📚 보충 설명

**VIPER 컴포넌트 관계**

```
View ↔ Presenter ↔ Interactor → (Entity)
                 ↓
              Router
```

**장단점 요약**

| 항목 | 평가 |
|------|------|
| 테스트 용이성 | ⭐⭐⭐⭐⭐ |
| 모듈화 | ⭐⭐⭐⭐⭐ |
| 학습 비용 | ⭐⭐ |
| 보일러플레이트 | ⭐ |
| 작은 앱 적합도 | ⭐ |

실무에서는 VIPER의 정신을 따르되 약간 단순화한 **MVP + Coordinator**나 **TCA(The Composable Architecture)** 같은 변형이 자주 선택됩니다.

### 🔗 참고 자료
- [Architecting iOS Apps with VIPER](https://www.objc.io/issues/13-architecture/viper/)

---

## 9. 여러 화면에서 동일한 알림 기능을 사용해야 할 때 싱글톤 대신 어떤 패턴을 쓸까요? (Lv1)

### 💬 면접 답변

가장 좋은 선택은 **프로토콜 + DI**입니다. `Notifier` 같은 프로토콜을 정의하고 구체 구현은 한 곳에 두되, 각 화면에서는 그 프로토콜에 의존합니다. 이렇게 하면 테스트 시 Mock으로 쉽게 대체할 수 있고, 같은 화면을 다른 컨텍스트(가짜 환경, 디버그 모드)에서 다르게 구성할 수 있습니다. SwiftUI라면 `@EnvironmentObject`나 `@Environment` 시스템으로 환경에 한 번 주입해두면 자식 뷰들이 자동으로 받을 수 있습니다. UIKit이라면 생성자 주입을 기본으로 하고, 화면 그래프가 복잡하면 Coordinator가 의존성 전달을 책임집니다.

### 📚 보충 설명

```swift
protocol Notifier {
    func notify(_ message: String)
}

final class PushNotifier: Notifier {
    func notify(_ message: String) { /* UNNotificationCenter ... */ }
}

@MainActor
final class HomeViewModel {
    let notifier: Notifier
    init(notifier: Notifier) { self.notifier = notifier }
}

// SwiftUI
@MainActor
final class AppEnvironment: ObservableObject {
    let notifier: Notifier = PushNotifier()
}

@main
struct MyApp: App {
    @StateObject var env = AppEnvironment()
    var body: some Scene {
        WindowGroup {
            ContentView().environmentObject(env)
        }
    }
}
```

### 🔗 참고 자료
- [WWDC 2023 - Discover Observation in SwiftUI](https://developer.apple.com/videos/play/wwdc2023/10149/)

---

## 10. 비슷한 네트워크 클라이언트들을 프로토콜로 추상화한다면 어떻게 설계하나요? (Lv2)

### 💬 면접 답변

핵심은 **요청을 값 타입으로 표현하고, 실행자(클라이언트)를 프로토콜로 분리**하는 것입니다. `Endpoint`나 `Request` 타입에 path·method·params·body·decoding 타입까지 담고, `Client`는 그 요청을 받아 결과를 돌려주는 인터페이스만 정의합니다. 그러면 실 서버에 붙는 `URLSessionClient`와 테스트용 `MockClient`를 같은 프로토콜로 갈아끼울 수 있고, 새 API가 추가될 때마다 Endpoint만 정의하면 됩니다. associated type을 활용하면 응답 타입까지 정적으로 보장할 수 있어 안전합니다.

### 📚 보충 설명

```swift
// Request as a value
protocol APIRequest {
    associatedtype Response: Decodable
    var path: String { get }
    var method: HTTPMethod { get }
    var query: [String: String] { get }
}

extension APIRequest {
    var method: HTTPMethod { .get }
    var query: [String: String] { [:] }
}

// Client
protocol APIClient {
    func send<R: APIRequest>(_ request: R) async throws -> R.Response
}

// Concrete
struct GetUser: APIRequest {
    typealias Response = User
    let id: Int
    var path: String { "/users/\(id)" }
}

struct URLSessionClient: APIClient {
    let baseURL: URL
    func send<R: APIRequest>(_ request: R) async throws -> R.Response {
        var components = URLComponents(url: baseURL.appendingPathComponent(request.path),
                                       resolvingAgainstBaseURL: false)!
        components.queryItems = request.query.map { URLQueryItem(name: $0.key, value: $0.value) }
        var urlReq = URLRequest(url: components.url!)
        urlReq.httpMethod = request.method.rawValue
        let (data, _) = try await URLSession.shared.data(for: urlReq)
        return try JSONDecoder().decode(R.Response.self, from: data)
    }
}

// 사용
let user: User = try await client.send(GetUser(id: 1))
```

**Mock for 테스트**

```swift
struct MockClient: APIClient {
    var responses: [String: Any] = [:]
    func send<R: APIRequest>(_ request: R) async throws -> R.Response {
        guard let r = responses[request.path] as? R.Response else { fatalError("missing mock") }
        return r
    }
}
```

**Protocol Extension의 메서드 디스패치 주의**

```swift
protocol Logger { func log(_ msg: String) }
extension Logger {
    func log(_ msg: String) { print("[Default] \(msg)") }     // 요구사항 O
    func info(_ msg: String) { log("INFO: \(msg)") }          // 요구사항 X
}
```

`log`는 채택 타입에서 오버라이드 가능 (witness table 동적 디스패치). `info`는 정적 디스패치되어 채택 타입이 새로 정의해도 프로토콜 타입으로 호출하면 extension의 구현이 불립니다.

### 🔗 참고 자료
- [Apple - URLSession](https://developer.apple.com/documentation/foundation/urlsession)

---

> 동시성 관련 패턴은 [05_Concurrency.md](./05_Concurrency.md), Repository 패턴의 데이터 소스는 [09_Data_Persistence.md](./09_Data_Persistence.md)를 참고하세요.
