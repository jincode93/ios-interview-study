---
layout: default
title: "10. 앱 생명주기와 개발 도구"
nav_order: 10
---

# 10. 앱 생명주기와 개발 도구

> 앱/씬 라이프사이클, Xcode 디버깅, Instruments, Git, 테스트(Unit/UI), TDD, 의존성 관리.

---

## 1. iOS 앱의 생명주기(App Lifecycle)에 대해 설명해주세요. (Lv1)

### 💬 면접 답변

앱 상태는 **Not Running → Inactive → Active → Background → Suspended**로 전환됩니다. Active는 포그라운드에서 이벤트를 받는 상태, Inactive는 전화 수신처럼 일시 정지된 상태, Background는 화면이 안 보이지만 코드가 짧게 실행될 수 있는 상태, Suspended는 메모리에는 있지만 코드 실행이 멈춘 상태입니다. iOS 13+ 부터는 멀티 윈도우를 위해 **Scene** 개념이 추가되어, `UISceneDelegate`가 각 씬의 라이프사이클을 담당하고 `UIApplicationDelegate`는 프로세스 전체 이벤트만 처리하도록 분리되었습니다. SwiftUI에서는 이 모든 걸 `@Environment(\.scenePhase)`나 `App`/`Scene` 프로토콜로 더 단순하게 다룹니다.

### 📚 보충 설명

**상태 전이 다이어그램**

```
                ┌──── Active ────┐
                │                ▼
Not Running → Inactive       Inactive
                ▲                │
                │                ▼
                └── Background ─→ Suspended
                          │
                          ▼
                  System Terminate
```

**AppDelegate vs SceneDelegate (iOS 13+)**

| 책임 | AppDelegate | SceneDelegate |
|------|-------------|---------------|
| 앱 실행/종료 | ⭕ | ❌ |
| UI 상태 전이 (foreground/background) | (iOS 12 이전만) | ⭕ |
| 다중 윈도우 처리 | ❌ | ⭕ |
| Push 토큰 등록 | ⭕ | ❌ |
| URL 열기 | (Scene이 우선) | ⭕ |

**SwiftUI App의 라이프사이클**

```swift
@main
struct MyApp: App {
    @Environment(\.scenePhase) var scenePhase

    var body: some Scene {
        WindowGroup { ContentView() }
            .onChange(of: scenePhase) { _, phase in
                switch phase {
                case .active:     print("active")
                case .inactive:   print("inactive")
                case .background: print("background")
                @unknown default: break
                }
            }
    }
}
```

**백그라운드에서 작업 마무리하기**

```swift
var task = UIBackgroundTaskIdentifier.invalid

func saveBeforeBackground() {
    task = UIApplication.shared.beginBackgroundTask {
        // 만료되기 직전 호출 (보통 30초)
        UIApplication.shared.endBackgroundTask(self.task)
        self.task = .invalid
    }

    DispatchQueue.global().async {
        // 저장 작업
        UIApplication.shared.endBackgroundTask(self.task)
        self.task = .invalid
    }
}
```

장기 백그라운드 작업은 `BGTaskScheduler`로 시스템이 적절한 시점에 깨워주는 방식을 써야 합니다.

### 🔗 참고 자료
- [Apple - Managing Your App's Life Cycle](https://developer.apple.com/documentation/uikit/app_and_environment/managing_your_app_s_life_cycle)
- [Apple - Specifying the scenes your app supports](https://developer.apple.com/documentation/uikit/app_and_environment/scenes/specifying_the_scenes_your_app_supports)
- [Apple - BGTaskScheduler](https://developer.apple.com/documentation/backgroundtasks)

---

## 2. Xcode에서 디버깅 시 자주 사용하는 기능은? (Lv1)

### 💬 면접 답변

가장 기본은 **Breakpoint**(중단점)입니다. 일반 breakpoint 외에 조건부 breakpoint, exception breakpoint, symbolic breakpoint, log message breakpoint 등 다양한 종류가 있어 코드 수정 없이 원하는 동작을 지정할 수 있습니다. **LLDB** 콘솔에서는 `po`로 객체 출력, `expr`로 즉석 표현식 실행, `bt`로 스택 추적 같은 명령어를 자주 씁니다. UI 문제는 **View Debugger**로 뷰 계층과 제약을 시각적으로 검사하고, 메모리 누수와 순환 참조는 **Memory Graph Debugger**로 잡습니다. 동시성 이슈는 Thread Sanitizer, 메모리 손상은 Address Sanitizer 같은 sanitizer를 켜서 잡아낼 수 있습니다.

### 📚 보충 설명

**Breakpoint 종류**

| 종류 | 용도 |
|------|------|
| 일반 | 특정 줄에서 멈춤 |
| 조건부 | 조건 만족 시에만 멈춤 (`x > 5`) |
| Exception | NSException, Swift error 발생 시 |
| Symbolic | 함수명으로 (`-[UIViewController viewDidLoad]`) |
| Log message | 멈추지 않고 로그만 출력 — print 디버깅 대체 |
| Test failure | XCTAssert 실패 시 |

**LLDB 자주 쓰는 명령**

```
(lldb) po user.name              # 객체 출력
(lldb) p user.name               # 더 자세한 표현
(lldb) expr user.name = "철수"   # 값 변경 후 계속
(lldb) bt                        # 스택 추적
(lldb) frame variable            # 현재 프레임의 모든 변수
(lldb) thread list               # 모든 스레드
(lldb) c                         # continue
(lldb) n                         # next (step over)
(lldb) s                         # step into
(lldb) fin                       # finish (현재 함수 끝까지)
```

**SwiftUI 디버깅 팁**

```swift
extension View {
    func debugPrint(_ msg: String) -> some View {
        Swift.print(msg)
        return self
    }
}

var body: some View {
    Text(name).debugPrint("body re-evaluated")
}
```

iOS 17+ `_printChanges()`:
```swift
var body: some View {
    let _ = Self._printChanges()       // 어떤 의존성이 바뀌어서 재계산되는지
    Text(name)
}
```

**Sanitizer 활성화**
- Scheme → Run → Diagnostics
  - Address Sanitizer (메모리 손상)
  - Thread Sanitizer (data race)
  - Undefined Behavior Sanitizer
  - Main Thread Checker (백그라운드에서의 UI 호출)

### 🔗 참고 자료
- [Apple - Debugging](https://developer.apple.com/documentation/xcode/debugging)
- [LLDB Tutorial](https://lldb.llvm.org/use/tutorial.html)

---

## 3. Instruments로 앱 성능을 분석하는 방법은? (Lv1)

### 💬 면접 답변

Instruments는 Xcode에 포함된 성능 분석 도구로, 여러 템플릿(Time Profiler, Allocations, Leaks, Network 등)을 제공합니다. **Time Profiler**는 CPU 시간이 어디서 소요되는지 함수 호출 트리로 보여줘 메인 스레드 병목 찾는 데 가장 유용합니다. **Allocations**는 객체 할당 추이를 보여줘 메모리 증가 패턴과 비정상 누적을 찾고, **Leaks**는 명백한 누수 객체를 자동으로 표시합니다. 메모리 누수는 Leaks + Memory Graph Debugger를 같이 쓰는 게 좋고, 측정은 반드시 **실기기 + Release 또는 적어도 Debug 최적화 활성화** 상태에서 합니다. 시뮬레이터에서 측정한 성능은 신뢰할 수 없습니다.

### 📚 보충 설명

**주요 템플릿**

| 템플릿 | 용도 |
|--------|------|
| Time Profiler | CPU 사용 분석 |
| Allocations | 메모리 할당 추적 |
| Leaks | 메모리 누수 탐지 |
| Network | URLSession 요청 분석 |
| Energy Log | 배터리 사용 |
| System Trace | OS 레벨 이벤트 |
| Metal System Trace | GPU 워크로드 |
| SwiftUI | SwiftUI 뷰 업데이트 분석 (Xcode 14+) |

**Time Profiler 사용 팁**
- "Hide System Libraries" 켜면 우리 코드만 보임
- "Invert Call Tree"로 가장 자주 불린 함수부터 정렬
- Heaviest Stack Trace로 가장 무거운 경로 확인

**Allocations 워크플로우**
1. 앱 실행 후 의심되는 시나리오(목록 스크롤, 모달 진입/이탈) 반복
2. Mark Generation(⌘+Shift+M)으로 구간 표시
3. 구간 간 Persistent Bytes 비교 → 비정상 증가 분석

**Memory Graph Debugger**
- Xcode → Debug → Capture View Hierarchy 옆 버튼
- 또는 디버깅 중 Memory Graph 아이콘
- 보라색 느낌표 = 누수 의심
- 객체 클릭 → "Show only content from workspace"로 우리 코드만 추적

### 🔗 참고 자료
- [Apple - Instruments](https://help.apple.com/instruments/mac/current/)
- [WWDC 2019 - Optimizing App Launch](https://developer.apple.com/videos/play/wwdc2019/423/)
- [WWDC 2023 - Analyze hangs with Instruments](https://developer.apple.com/videos/play/wwdc2023/10248/)

---

## 4. CocoaPods, Carthage, Swift Package Manager의 차이는? (Lv1)

### 💬 면접 답변

세 가지 모두 외부 라이브러리 관리 도구지만 동작 방식이 다릅니다. **CocoaPods**는 중앙 저장소에서 소스를 받아 통합된 워크스페이스를 자동 생성하는 방식으로, 가장 오래되고 라이브러리 수가 많지만 빌드 시간이 길고 Xcode 워크스페이스 구조를 바꿉니다. **Carthage**는 라이브러리를 빌드한 바이너리만 제공하고 통합은 개발자에게 맡기는 분산형 접근으로, 빌드 시간이 짧지만 설정이 번거롭습니다. **Swift Package Manager(SPM)** 는 Apple이 만들고 Xcode에 기본 통합된 도구로, Package.swift 하나로 의존성을 선언하고 Xcode가 자동 처리해 줍니다. 새 프로젝트라면 SPM이 사실상 표준입니다.

### 📚 보충 설명

**비교 표**

| 항목 | CocoaPods | Carthage | SPM |
|------|-----------|----------|-----|
| Apple 공식 | ❌ | ❌ | ⭕ |
| 통합 방식 | Workspace 자동 생성 | 바이너리 + 수동 통합 | Xcode 내장 |
| 빌드 영향 | 매번 빌드 | 사전 빌드된 바이너리 | Xcode가 처리 |
| 호환성 | Obj-C, Swift, 매우 광범위 | Obj-C, Swift | Swift 위주 |
| 설정 | Podfile + Podfile.lock | Cartfile + 수동 임포트 | Package.swift |
| 학습 곡선 | 중간 | 낮음 | 매우 낮음 |

**SPM이 표준이 된 이유**
- Xcode 기본 내장 (별도 설치 불필요)
- 의존성 그래프 자동 해결
- 같은 도구로 라이브러리 제작·게시 가능
- Xcode 워크스페이스 구조 안 깸
- Apple이 적극 개발

**SPM 추가 방법**
- Xcode → File → Add Package Dependencies → URL 입력
- 또는 Package.swift에 직접 추가

```swift
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire", from: "5.0.0")
]
```

### 🔗 참고 자료
- [Apple - Swift Package Manager](https://www.swift.org/package-manager/)
- [Apple - Adding Package Dependencies to Your App](https://developer.apple.com/documentation/xcode/adding-package-dependencies-to-your-app)

---

## 5. Git에서 브랜치 전략과 충돌 해결 방법은? (Lv1)

### 💬 면접 답변

대표적인 브랜치 전략은 **Git Flow**와 **GitHub Flow** 두 가지입니다. Git Flow는 main·develop·feature·release·hotfix 같은 여러 브랜치를 두고 명확한 릴리스 주기를 운영하는 모델로, 모바일 앱처럼 정기 릴리스가 있는 환경에 적합합니다. GitHub Flow는 main 한 줄을 중심으로 feature 브랜치만 추가하는 단순한 모델로, 지속적 배포가 가능한 웹 서비스에 어울립니다. 충돌은 같은 라인을 양쪽에서 수정했을 때 발생하는데, `git status`로 충돌 파일을 확인하고 양쪽 변경을 직접 머지한 뒤 `git add` → `git commit`(또는 `git rebase --continue`)로 마무리합니다. 큰 충돌이 자주 생긴다면 더 작은 단위의 PR과 잦은 동기화가 해결책입니다.

### 📚 보충 설명

**Merge vs Rebase**

- `git merge`: 두 브랜치의 히스토리를 머지 커밋으로 합침. 히스토리 보존, 안전.
- `git rebase`: 한 브랜치의 커밋들을 다른 브랜치 위로 재배치. 깔끔한 선형 히스토리, 단 공유 브랜치 rebase는 위험.

**일반적 워크플로**

```bash
git checkout -b feature/login
# 작업...
git add . && git commit -m "..."
git fetch origin
git rebase origin/main         # 또는 git merge origin/main
git push -u origin feature/login
# PR → 리뷰 → main에 머지
```

**충돌 해결**

```bash
git status                     # 충돌 파일 확인
# 파일 열어서 <<<<<<<<, =======, >>>>>>> 마커 보고 수정
git add <file>
git commit                     # merge의 경우
# 또는
git rebase --continue          # rebase의 경우
```

**커밋 메시지 컨벤션**

Conventional Commits:
```
feat: 사용자 로그인 추가
fix: 메모리 누수 수정
refactor: ViewModel 분리
docs: README 갱신
test: 로그인 ViewModel 단위 테스트
chore: 의존성 업데이트
```

### 🔗 참고 자료
- [Pro Git Book](https://git-scm.com/book/en/v2)
- [Conventional Commits](https://www.conventionalcommits.org/)

---

## 6. 단위 테스트와 UI 테스트의 차이는? XCTest는 어떻게 쓰나요? (Lv1)

### 💬 면접 답변

**단위 테스트**는 함수·클래스 같은 작은 단위의 로직을 외부 의존성 없이 검증하는 테스트로, 빠르고 결정적이라 자주 돌릴 수 있습니다. **UI 테스트**는 실제 앱을 띄우고 사용자의 액션을 시뮬레이션해 화면 흐름을 검증하는 통합 테스트로, 신뢰도가 높지만 느리고 깨지기 쉽습니다. iOS는 둘 다 **XCTest** 프레임워크로 작성하고, `XCTAssertEqual`, `XCTAssertTrue` 같은 단언으로 검증합니다. UI 테스트는 별도 타깃에서 `XCUIApplication`을 사용해 앱을 제어합니다. 단위 테스트는 빠르게 많이, UI 테스트는 핵심 사용자 흐름 위주로 — **테스트 피라미드**가 일반적인 균형입니다.

### 📚 보충 설명

**단위 테스트 예시**

```swift
import XCTest
@testable import MyApp

final class CalculatorTests: XCTestCase {
    var sut: Calculator!     // System Under Test

    override func setUp() {
        super.setUp()
        sut = Calculator()
    }

    override func tearDown() {
        sut = nil
        super.tearDown()
    }

    func test_add_returnsSum() {
        XCTAssertEqual(sut.add(2, 3), 5)
    }

    func test_divideByZero_throws() {
        XCTAssertThrowsError(try sut.divide(1, by: 0))
    }
}
```

**비동기 테스트 (async/await)**

```swift
func test_fetchUser_returnsUser() async throws {
    let user = try await sut.fetchUser(id: 1)
    XCTAssertEqual(user.id, 1)
}
```

**XCTestExpectation (콜백 기반 코드)**

```swift
func test_callback() {
    let exp = expectation(description: "callback")
    sut.load { result in
        XCTAssertNotNil(result)
        exp.fulfill()
    }
    wait(for: [exp], timeout: 2)
}
```

**UI 테스트 예시**

```swift
final class LoginUITests: XCTestCase {
    func test_login_navigatesToHome() {
        let app = XCUIApplication()
        app.launch()

        app.textFields["email"].tap()
        app.textFields["email"].typeText("test@example.com")
        app.secureTextFields["password"].tap()
        app.secureTextFields["password"].typeText("password")
        app.buttons["login"].tap()

        XCTAssertTrue(app.staticTexts["welcome"].waitForExistence(timeout: 2))
    }
}
```

**Accessibility Identifier로 안정적 선택**

```swift
// 앱 코드
button.accessibilityIdentifier = "loginButton"

// 테스트
app.buttons["loginButton"].tap()
```

문자열에 의존하지 않으므로 다국어/디자인 변경에 안정적입니다.

### 🔗 참고 자료
- [Apple - XCTest](https://developer.apple.com/documentation/xctest)
- [Apple - Testing in Xcode](https://developer.apple.com/documentation/xcode/testing)

---

## 7. TDD(Test-Driven Development)의 장점은? (Lv1)

### 💬 면접 답변

TDD는 **테스트를 먼저 쓰고 그 테스트를 통과시키는 최소한의 코드를 작성한 뒤 리팩토링**하는 사이클을 반복하는 개발 방식입니다. 장점은 첫째, **테스트 가능한 설계**가 자연스럽게 나옵니다 — 테스트하기 어렵게 짠 코드는 사용성이 나쁜 경우가 많습니다. 둘째, **리팩토링 안전망**이 생겨 코드 개선이 두렵지 않습니다. 셋째, **요구사항 명세 역할**을 해서 테스트가 곧 살아있는 문서가 됩니다. 단점은 학습 비용과 초기 작성 시간으로, UI나 외부 시스템과 연결된 부분은 TDD가 어려운 경우가 많습니다. 그래서 핵심 로직(ViewModel, UseCase, 도메인 모델)부터 TDD를 적용하고, 점진적으로 영역을 넓히는 게 현실적입니다.

### 📚 보충 설명

**Red-Green-Refactor 사이클**
1. **Red**: 실패하는 테스트 작성
2. **Green**: 최소한의 코드로 통과
3. **Refactor**: 중복 제거, 가독성 향상 (테스트는 계속 통과)

**의존성 주입과 TDD**

테스트 가능성을 위해 외부 의존성은 프로토콜로 추상화하고 주입받습니다.

```swift
protocol UserRepository {
    func fetchUser(id: Int) async throws -> User
}

@MainActor
final class UserViewModel {
    let repo: UserRepository
    @Published var user: User?

    init(repo: UserRepository) { self.repo = repo }

    func load(id: Int) async {
        user = try? await repo.fetchUser(id: id)
    }
}

// 테스트
final class MockUserRepo: UserRepository {
    var stub: User?
    func fetchUser(id: Int) async throws -> User { stub! }
}

func test_load_setsUser() async {
    let mock = MockUserRepo()
    mock.stub = User(id: 1, name: "철수")
    let vm = await UserViewModel(repo: mock)
    await vm.load(id: 1)
    let user = await vm.user
    XCTAssertEqual(user?.id, 1)
}
```

### 🔗 참고 자료
- Kent Beck, *Test-Driven Development: By Example*
- [Apple - Testing Asynchronous Operations](https://developer.apple.com/documentation/xctest/asynchronous_tests_and_expectations)

---

## 8. 의존성 주입(DI)을 통해 테스트 가능한 코드를 어떻게 작성하나요? (Lv1, Lv2)

### 💬 면접 답변

테스트 가능한 코드의 핵심은 **외부 의존성을 직접 호출하지 않고 추상화에 의존**하는 것입니다. URLSession이나 Date(), UserDefaults 같은 글로벌 자원을 직접 쓰면 테스트에서 격리가 어려워지므로, 이들을 프로토콜로 감싸 주입받으면 Mock으로 쉽게 대체할 수 있습니다. 의존성은 보통 **생성자 주입**으로 받고, 기본값으로 실제 구현을 두면 일반 사용에도 편리합니다. 시간이나 무작위성처럼 비결정적인 요소도 추상화하면 테스트가 결정적으로 만들어집니다.

### 📚 보충 설명

**시간을 추상화하기**

```swift
protocol Clock {
    func now() -> Date
}

struct SystemClock: Clock {
    func now() -> Date { Date() }
}

struct FixedClock: Clock {
    let date: Date
    func now() -> Date { date }
}

// 테스트에서
let clock = FixedClock(date: Date(timeIntervalSince1970: 0))
let session = Session(clock: clock)
XCTAssertEqual(session.expiresAt, ...)
```

**URLSession 추상화**

```swift
protocol HTTPClient {
    func data(for: URLRequest) async throws -> (Data, URLResponse)
}

extension URLSession: HTTPClient {}     // 시스템도 채택
```

테스트에서는 `MockHTTPClient`로 임의 응답을 주입.

### 🔗 참고 자료
- [Apple - WWDC 2023 - Discover Observation in SwiftUI](https://developer.apple.com/videos/play/wwdc2023/10149/)

---

## 9. 바이너리 프레임워크는 무엇이며 언제 사용하나요? (Lv3)

### 💬 면접 답변

바이너리 프레임워크는 컴파일된 형태로 배포되는 라이브러리로, 소스 코드를 노출하지 않으면서도 임포트해 사용할 수 있습니다. 폐쇄형 제품 SDK, 빌드 시간 단축이 중요한 대규모 프로젝트, 소스 라이선스 보호가 필요한 경우에 사용합니다. iOS에서는 **XCFramework**가 표준 포맷으로, 같은 프레임워크의 여러 아키텍처(arm64 디바이스, arm64 시뮬레이터, x86_64 시뮬레이터)와 플랫폼(iOS, macOS, watchOS)을 한 번에 담을 수 있습니다. 단점은 라이브러리 측의 Swift 버전·ABI 호환성을 신경 써야 하고(클라이언트 환경과 일치하거나 library evolution 활성화 필요), 디버깅이 소스 프레임워크보다 불편하다는 점입니다.

### 📚 보충 설명

**XCFramework 만들기**

```bash
# 1) 각 슬라이스 빌드
xcodebuild archive \
    -scheme MyKit \
    -destination "generic/platform=iOS" \
    -archivePath build/MyKit-iOS \
    SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES

xcodebuild archive \
    -scheme MyKit \
    -destination "generic/platform=iOS Simulator" \
    -archivePath build/MyKit-Sim \
    SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES

# 2) XCFramework 결합
xcodebuild -create-xcframework \
    -framework build/MyKit-iOS.xcarchive/Products/Library/Frameworks/MyKit.framework \
    -framework build/MyKit-Sim.xcarchive/Products/Library/Frameworks/MyKit.framework \
    -output build/MyKit.xcframework
```

**`BUILD_LIBRARY_FOR_DISTRIBUTION = YES` 가 필요한 이유**

Swift 5.1+ Library Evolution을 활성화해서 라이브러리와 클라이언트의 Swift 버전이 달라도 호환되도록 합니다. 이걸 안 켜면 같은 Swift 버전에서만 사용 가능합니다.

**SPM으로 배포**

```swift
.binaryTarget(name: "MyKit", url: "https://example.com/MyKit.xcframework.zip",
              checksum: "abcd...")
```

### 🔗 참고 자료
- [Apple - Distributing Binary Frameworks as Swift Packages](https://developer.apple.com/documentation/xcode/distributing-binary-frameworks-as-swift-packages)
- [Apple - Creating a multiplatform binary framework bundle](https://developer.apple.com/documentation/xcode/creating-a-multi-platform-binary-framework-bundle)

---

## 10. 앱 시작 시간(Launch Time)을 단축하는 방법은? (Lv5 / 빈출)

### 💬 면접 답변

앱 시작은 크게 **pre-main**(이미지 로딩, dynamic linking, Swift runtime, +load·static initializer 실행)과 **post-main**(`main()` 이후, `application:didFinishLaunchingWithOptions:`, 첫 화면 렌더링)으로 나뉩니다. 보통 시스템이 처리하는 pre-main을 줄이려면 동적 라이브러리 수를 줄이고 dyld가 빠르게 처리할 수 있는 형태로 만듭니다. post-main 단축은 개발자 영역으로, AppDelegate에서 무거운 초기화를 피하고 lazy 로딩이나 사용자가 첫 화면을 본 후 비동기로 미루는 게 핵심입니다. **Instruments의 App Launch 템플릿**이 phase별 시간을 보여주므로, 거기서 병목을 찾아 단계적으로 개선합니다.

### 📚 보충 설명

**pre-main 단계 (대략)**
1. **dylib loading**: 시스템·앱 동적 라이브러리 로딩
2. **rebase/binding**: 주소 fixup
3. **ObjC setup**: 클래스 등록
4. **Initializers**: `+load`, C++ static, Swift global
5. **main**: 우리 코드 진입

**개선 방법**
- 동적 라이브러리(framework) 수 줄이기 (static link 활용)
- `+load` 메서드 제거 (Objective-C)
- Swift global initializer 줄이기 (lazy var 활용)
- AppDelegate에서 동기 작업 최소화 → 첫 화면 보인 후 Task로
- Storyboard보다 코드/SwiftUI가 보통 가벼움
- 리소스 prefetch 지연

**Pre-warming (iOS 15+)**

시스템이 사용자가 앱을 열기 전에 미리 앱을 시작해두는 기능. AppDelegate 메서드가 호출되지만 UI는 아직 표시되지 않는 상태. 이 단계에서 무거운 작업을 하면 사용자 체감 시간은 빨라지지만 백그라운드 자원 사용이 늘어남.

### 🔗 참고 자료
- [WWDC 2019 - Optimizing App Launch](https://developer.apple.com/videos/play/wwdc2019/423/)
- [WWDC 2022 - Eliminate animation hitches with XCTest](https://developer.apple.com/videos/play/wwdc2022/110366/)

---

> 단위 테스트 + DI 패턴은 [07_Architecture.md](./07_Architecture.md)와도 연결됩니다.
