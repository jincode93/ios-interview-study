---
layout: default
title: "11. 고급 프레임워크"
nav_order: 11
---

# 11. 고급 프레임워크

> Core ML, Vision, Siri Shortcuts, Objective-C 브리징, 금융 앱 보안.

---

## 1. Core ML로 머신러닝 모델을 앱에 통합하는 방법은? (Lv4)

### 💬 면접 답변

Core ML은 학습된 ML 모델을 iOS 앱에 통합하는 Apple 프레임워크로, `.mlmodel`이나 `.mlpackage` 파일을 Xcode에 드래그하면 자동으로 Swift 인터페이스가 생성됩니다. 모델은 디바이스에서 실행되므로 **개인정보가 외부로 나가지 않고** 오프라인에서도 동작하며, A-시리즈/M-시리즈 칩의 Neural Engine 가속도 활용됩니다. 이미지 분류·텍스트 처리 같은 일반적인 작업은 Vision이나 Natural Language 같은 상위 프레임워크가 Core ML을 감싸서 더 쓰기 쉬운 API를 제공합니다. 모델 사이즈가 클 수 있어 앱 번들이 커지는 문제는 **On-Demand Resources**나 **CloudKit**으로 모델을 별도 배포하는 식으로 해결합니다.

### 📚 보충 설명

**기본 사용**

```swift
import CoreML
import Vision

// 자동 생성된 클래스 사용
let model = try MyImageClassifier(configuration: MLModelConfiguration()).model
let vnModel = try VNCoreMLModel(for: model)

let request = VNCoreMLRequest(model: vnModel) { request, _ in
    guard let results = request.results as? [VNClassificationObservation] else { return }
    print(results.first?.identifier ?? "")
}

let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
try handler.perform([request])
```

**MLModelConfiguration**

```swift
let config = MLModelConfiguration()
config.computeUnits = .all          // .cpuOnly, .cpuAndGPU, .cpuAndNeuralEngine, .all
```

`computeUnits`로 Neural Engine 사용 여부 제어. iOS 16+ `.cpuAndNeuralEngine`이 보통 최적.

**모델 변환 도구**
- **Core ML Tools** (Python): TensorFlow, PyTorch → `.mlpackage`
- **Create ML** (macOS 앱): 데이터로 직접 모델 학습

**경량화 기법**
- **Quantization**: float32 → float16/int8로 압축, 모델 크기 ↓
- **Pruning**: 중요도 낮은 가중치 제거
- Core ML Tools의 `coremltools.optimize` 모듈로 적용

### 🔗 참고 자료
- [Apple - Core ML](https://developer.apple.com/documentation/coreml)
- [Apple - Core ML Tools](https://apple.github.io/coremltools/)
- [WWDC 2023 - Use Core ML Tools for machine learning model compression](https://developer.apple.com/videos/play/wwdc2023/10047/)
- [WWDC 2022 - Optimize your Core ML usage](https://developer.apple.com/videos/play/wwdc2022/10027/)

---

## 2. Vision 프레임워크로 어떤 작업을 할 수 있나요? (Lv4)

### 💬 면접 답변

Vision은 컴퓨터 비전 작업을 추상화한 프레임워크로, **얼굴 인식·텍스트 인식(OCR)·바코드/QR·객체 감지·이미지 유사도** 같은 작업을 별도 모델 없이 사용할 수 있습니다. 또 Core ML 모델을 `VNCoreMLRequest`로 래핑해 함께 활용할 수도 있어, 학습된 모델을 손쉽게 카메라 입력에 적용할 수 있습니다. ARKit, AVFoundation과 결합하면 실시간 비디오에서 얼굴 추적이나 객체 감지가 가능하고, iOS 17부터는 **VisionKit**의 `DataScannerViewController`로 영수증·문서 스캔도 매우 쉽게 구현할 수 있습니다.

### 📚 보충 설명

**텍스트 인식(OCR) 예시**

```swift
let request = VNRecognizeTextRequest { request, _ in
    let observations = request.results as? [VNRecognizedTextObservation] ?? []
    for obs in observations {
        if let topCandidate = obs.topCandidates(1).first {
            print(topCandidate.string)
        }
    }
}
request.recognitionLevel = .accurate    // .fast / .accurate
request.recognitionLanguages = ["ko-KR", "en-US"]

let handler = VNImageRequestHandler(cgImage: cg, options: [:])
try handler.perform([request])
```

**주요 Request 타입**

| Request | 용도 |
|---------|------|
| `VNDetectFaceRectanglesRequest` | 얼굴 영역 감지 |
| `VNDetectFaceLandmarksRequest` | 얼굴 특징점 |
| `VNRecognizeTextRequest` | OCR |
| `VNDetectBarcodesRequest` | 바코드/QR |
| `VNDetectRectanglesRequest` | 사각형 감지 (문서) |
| `VNCoreMLRequest` | Core ML 모델 적용 |
| `VNGenerateImageFeaturePrintRequest` | 이미지 유사도용 임베딩 |

**실시간 카메라와 결합**

```swift
final class CameraDelegate: NSObject, AVCaptureVideoDataOutputSampleBufferDelegate {
    func captureOutput(_ output: AVCaptureOutput,
                       didOutput sampleBuffer: CMSampleBuffer,
                       from connection: AVCaptureConnection) {
        guard let buffer = CMSampleBufferGetImageBuffer(sampleBuffer) else { return }
        let handler = VNImageRequestHandler(cvPixelBuffer: buffer, options: [:])
        try? handler.perform([request])
    }
}
```

### 🔗 참고 자료
- [Apple - Vision](https://developer.apple.com/documentation/vision)
- [Apple - VisionKit](https://developer.apple.com/documentation/visionkit)
- [WWDC 2022 - What's new in Vision](https://developer.apple.com/videos/play/wwdc2022/10024/)

---

## 3. Siri Shortcuts(App Intents)는 어떻게 구현하나요? (Lv3)

### 💬 면접 답변

Siri Shortcuts는 사용자가 자주 하는 앱 내 작업을 Siri 음성, 단축어 앱, 위젯에서 빠르게 실행할 수 있게 해주는 시스템입니다. iOS 16+ 부터는 새로운 **App Intents** 프레임워크가 표준이 되었고, 이전 방식인 NSUserActivity·Intents Framework(`.intentdefinition`)를 단순화했습니다. App Intents는 Swift 코드만으로 정의할 수 있어 보일러플레이트가 적고, 매개변수·결과·후속 단계까지 타입 안전하게 표현할 수 있습니다. 위젯의 인터랙티브 버튼도 이 App Intents 기반으로 동작해서, 한 번 잘 정의해두면 Siri·단축어·위젯·Spotlight 등 여러 곳에 자동 노출됩니다.

### 📚 보충 설명

**App Intent 정의 (iOS 16+)**

```swift
import AppIntents

struct AddTaskIntent: AppIntent {
    static var title: LocalizedStringResource = "할 일 추가"

    @Parameter(title: "내용")
    var content: String

    func perform() async throws -> some IntentResult {
        TaskStore.shared.add(content)
        return .result()
    }
}
```

**Shortcut 노출**

```swift
struct MyAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: AddTaskIntent(),
            phrases: ["\(.applicationName)에 할 일 추가"],
            shortTitle: "할 일 추가",
            systemImageName: "plus"
        )
    }
}
```

**위젯에서 활용 (iOS 17+)**

```swift
Button(intent: AddTaskIntent()) {
    Image(systemName: "plus")
}
```

**Legacy Intents Framework**
- `.intentdefinition` 파일과 INIntent 서브클래스
- iOS 12~15 지원 필요할 때만 사용
- iOS 16+ 새 코드는 App Intents 권장

### 🔗 참고 자료
- [Apple - App Intents](https://developer.apple.com/documentation/appintents)
- [WWDC 2022 - Meet App Intents](https://developer.apple.com/videos/play/wwdc2022/10032/)
- [WWDC 2023 - Spotlight your app with App Shortcuts](https://developer.apple.com/videos/play/wwdc2023/10102/)

---

## 4. Swift와 Objective-C는 어떻게 함께 사용하나요? (Lv3)

### 💬 면접 답변

Swift와 Objective-C는 같은 프로젝트에서 자유롭게 혼용할 수 있습니다. **Swift에서 ObjC 사용**은 Bridging Header에 ObjC 헤더를 import하면 컴파일러가 자동으로 Swift 인터페이스를 생성합니다. **ObjC에서 Swift 사용**은 `#import "MyApp-Swift.h"`로 자동 생성되는 헤더를 import하면 되는데, Swift 측에 `@objc` 또는 `NSObject` 상속이 있어야 노출됩니다. Swift 5+ 부터는 `@objcMembers`로 클래스 전체를 한 번에 노출할 수도 있습니다. 다만 ObjC는 Swift의 모든 타입(특히 generics, optional 외의 enum, struct)을 표현하지 못하므로, 노출되는 인터페이스는 ObjC 호환 타입으로 제한됩니다.

### 📚 보충 설명

**Swift → ObjC 노출**

```swift
@objc class MyClass: NSObject {
    @objc func greet(_ name: String) -> String { "Hello, \(name)" }
    @objc var count = 0
}

@objcMembers class All: NSObject {
    var a = 0      // 자동 @objc
    func foo() {}  // 자동 @objc
}
```

**Bridging Header (ObjC → Swift)**

```objc
// MyApp-Bridging-Header.h
#import "LegacyClass.h"
```

Swift 코드에서 바로 `LegacyClass()` 사용 가능.

**ObjC에서 표현 불가능한 Swift 타입**
- Generic (`Array<T>`은 `NSArray`로 변환)
- Enum with associated values
- Struct (`Codable`이라도 노출 안 됨)
- Tuple
- Protocol with associated type

→ ObjC에 노출할 API는 `NSObject` 상속 클래스로 제한.

**`dynamic` 키워드**

```swift
class MyClass: NSObject {
    @objc dynamic var value: Int = 0   // KVO 가능, ObjC 메시지 디스패치
}
```

KVO, NSPredicate, NSOperation 의존성, 일부 ObjC 런타임 기능(`Method swizzling`)에 필요.

### 🔗 참고 자료
- [Apple - Importing Objective-C into Swift](https://developer.apple.com/documentation/swift/importing-objective-c-into-swift)
- [Apple - Importing Swift into Objective-C](https://developer.apple.com/documentation/swift/importing-swift-into-objective-c)

---

## 5. 금융 앱처럼 보안이 중요한 앱에서 보안과 UX의 균형을 어떻게 잡나요? (Lv4)

### 💬 면접 답변

보안과 UX는 **단순 trade-off가 아니라 사용자 신뢰를 만드는 두 축**으로 봐야 합니다. 강력한 인증을 매번 요구하면 사용자가 떠나고, 너무 느슨하면 사고가 납니다. 균형의 핵심은 **위험에 비례한 보안**입니다. 잔액 조회는 생체 인증으로 빠르게, 송금이나 한도 변경처럼 위험한 작업은 추가 OTP나 PIN을 요구합니다. 또 생체 인증 실패 시 즉시 차단보다 횟수 제한과 PIN 폴백을 두고, 세션 타임아웃은 사용자가 다시 입력하기 부담스럽지 않은 간격(예: 5분)으로 잡되 민감 작업 직전에는 재인증을 요구합니다. 보안 사고 예방에는 시큐어 코딩 가이드와 코드 리뷰 체크리스트, 외부 보안 감사 결과의 정기적 반영이 효과적입니다.

### 📚 보충 설명

**위험 기반 인증(Risk-based Authentication)**

| 작업 | 인증 강도 |
|------|---------|
| 잔액 조회 | 생체 인증 |
| 거래 내역 | 생체 인증 |
| 소액 송금 | 생체 + 거래 비밀번호 |
| 대액 송금 | 생체 + OTP + 추가 검증 |
| 한도 변경 | 강력한 다중 인증 |

**생체 인증 (LocalAuthentication)**

```swift
import LocalAuthentication

let context = LAContext()
context.localizedFallbackTitle = "비밀번호로 인증"
var error: NSError?

if context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) {
    let reason = "송금을 진행하려면 인증이 필요합니다"
    do {
        let success = try await context.evaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            localizedReason: reason
        )
        // success == true → 진행
    } catch {
        // 실패: error 분기 (.userCancel, .biometryLockout 등)
    }
}
```

**보안 코딩 체크리스트 예시**
- 민감 데이터는 항상 Keychain (UserDefaults 금지)
- 네트워크는 HTTPS + Certificate Pinning
- 디버그 로그에 PII·토큰 출력 금지
- 스크린샷·앱 스위처 미리보기에서 민감 정보 가리기
- 탈옥 감지 (필요 시): SecureKit 등
- Reverse engineering 대응: 코드 난독화 (선택)
- 디버거 부착 감지

**스크린샷 가리기**

```swift
// 앱이 백그라운드로 갈 때 민감 화면 가리기
func sceneWillResignActive(_ scene: UIScene) {
    let blur = UIBlurEffect(style: .regular)
    blurView = UIVisualEffectView(effect: blur)
    blurView?.frame = window?.bounds ?? .zero
    window?.addSubview(blurView!)
}

func sceneDidBecomeActive(_ scene: UIScene) {
    blurView?.removeFromSuperview()
    blurView = nil
}
```

### 🔗 참고 자료
- [Apple - LocalAuthentication](https://developer.apple.com/documentation/localauthentication)
- [Apple - Security Framework](https://developer.apple.com/documentation/security)
- [OWASP Mobile Application Security](https://owasp.org/www-project-mobile-app-security/)

---

## 6. Combine과 MVVM을 결합한 데이터 바인딩 패턴은? (Lv3)

### 💬 면접 답변

MVVM에서 ViewModel은 보통 입력(사용자 액션)과 출력(상태)을 가지는데, Combine을 결합하면 이 둘을 **Publisher로 표현**해 흐름을 일관되게 만들 수 있습니다. 입력은 `PassthroughSubject`로 받고, 그 위에 `map`·`flatMap`·`debounce` 같은 연산자로 변환·검증·네트워크 호출을 체이닝한 뒤 `@Published` 프로퍼티에 결과를 넣어 View가 자동으로 갱신되게 합니다. 이렇게 하면 ViewModel의 로직이 declarative하고 테스트하기 쉬워지며, 단방향 데이터 흐름(unidirectional data flow)도 자연스럽게 만들어집니다.

### 📚 보충 설명

**예시: 검색 ViewModel**

```swift
@MainActor
final class SearchViewModel: ObservableObject {
    @Published var query: String = ""
    @Published private(set) var results: [Item] = []
    @Published private(set) var isLoading: Bool = false

    private var cancellables = Set<AnyCancellable>()
    private let api: APIClient

    init(api: APIClient) {
        self.api = api

        $query
            .removeDuplicates()
            .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
            .filter { $0.count >= 2 }
            .handleEvents(receiveOutput: { [weak self] _ in self?.isLoading = true })
            .flatMap { [api] q in
                api.searchPublisher(query: q)
                    .catch { _ in Just([]) }
            }
            .receive(on: DispatchQueue.main)
            .sink { [weak self] items in
                self?.results = items
                self?.isLoading = false
            }
            .store(in: &cancellables)
    }
}
```

**SwiftUI에서 사용**

```swift
struct SearchView: View {
    @StateObject var vm = SearchViewModel(api: AppEnvironment.shared.api)

    var body: some View {
        VStack {
            TextField("검색", text: $vm.query)
            if vm.isLoading { ProgressView() }
            List(vm.results) { ... }
        }
    }
}
```

**iOS 17+ Observable 매크로로 전환**

```swift
@Observable
final class SearchViewModel {
    var query: String = ""
    private(set) var results: [Item] = []
    // 직접 디바운스 등은 Task와 AsyncAlgorithms로 다시 작성
}
```

### 🔗 참고 자료
- [Apple - Combine](https://developer.apple.com/documentation/combine)
- [WWDC 2019 - Combine in Practice](https://developer.apple.com/videos/play/wwdc2019/721/)

---

## 7. Sequence/Collection 외에 사용자 정의 컬렉션을 만드는 다른 패턴은? (Lv2)

### 💬 면접 답변

표준 Sequence·Collection 채택 외에도, 특수 목적의 데이터 구조를 만들 때 자주 쓰는 패턴이 몇 가지 있습니다. **AsyncSequence**는 비동기 시퀀스로 `for await`로 순회 가능한 스트림을 정의하고, **OptionSet**은 비트 플래그를 컬렉션처럼 다루는 프로토콜이며, **첨자(subscript)**를 활용하면 도메인 특화 접근 인터페이스를 만들 수 있습니다. 이런 패턴들은 Swift의 표준 알고리즘과 자연스럽게 결합되어 코드 가독성과 표현력을 높입니다.

### 📚 보충 설명

**AsyncSequence 예시 (자체 구현)**

```swift
struct Counter: AsyncSequence {
    typealias Element = Int
    let howMany: Int

    struct AsyncIterator: AsyncIteratorProtocol {
        var current = 0
        let howMany: Int
        mutating func next() async -> Int? {
            guard current < howMany else { return nil }
            try? await Task.sleep(nanoseconds: 100_000_000)
            defer { current += 1 }
            return current
        }
    }

    func makeAsyncIterator() -> AsyncIterator {
        AsyncIterator(howMany: howMany)
    }
}

for await n in Counter(howMany: 5) { print(n) }
```

**OptionSet (비트 플래그)**

```swift
struct Permissions: OptionSet {
    let rawValue: Int
    static let read    = Permissions(rawValue: 1 << 0)
    static let write   = Permissions(rawValue: 1 << 1)
    static let execute = Permissions(rawValue: 1 << 2)
    static let all: Permissions = [.read, .write, .execute]
}

let p: Permissions = [.read, .write]
p.contains(.read)   // true
```

**사용자 정의 첨자**

```swift
struct Matrix {
    private var data: [[Double]]
    let rows: Int, cols: Int

    subscript(row: Int, col: Int) -> Double {
        get { data[row][col] }
        set { data[row][col] = newValue }
    }
}

var m = Matrix(...)
m[1, 2] = 3.14
```

### 🔗 참고 자료
- [Apple - AsyncSequence](https://developer.apple.com/documentation/swift/asyncsequence)
- [Apple - OptionSet](https://developer.apple.com/documentation/swift/optionset)
- [swift-async-algorithms](https://github.com/apple/swift-async-algorithms)

---

## 8. Apple의 머신러닝 외 다른 ML 옵션은? (Lv4)

### 💬 면접 답변

iOS에서 머신러닝을 활용하는 방법은 Core ML 외에도 여러 가지가 있습니다. **TensorFlow Lite**는 Google의 경량 모바일 ML 런타임으로, 안드로이드와 코드를 공유하기 좋고 풍부한 모델 zoo를 가집니다. **ONNX Runtime**은 다양한 프레임워크의 모델을 표준 포맷으로 다루며 크로스 플랫폼 지원이 강합니다. **MLX**는 Apple Silicon에 최적화된 새로운 ML 프레임워크로 디바이스에서 직접 학습까지 지원합니다. 일반적으로 iOS 단독이고 Neural Engine 가속이 중요하면 Core ML, 안드로이드와 모델을 공유한다면 TensorFlow Lite, 연구·실험적 워크로드는 MLX — 이런 식으로 선택합니다. 모바일에서는 보통 모델 크기와 추론 속도가 큰 제약이라, **Quantization·Pruning·Distillation**으로 모델을 경량화하는 게 거의 필수입니다.

### 📚 보충 설명

**모바일 모델 경량화 기법**

- **Quantization (양자화)**: float32 → int8/float16. 모델 크기 ↓, 속도 ↑
- **Pruning (가지치기)**: 중요도 낮은 가중치를 0으로
- **Knowledge Distillation (증류)**: 큰 모델의 지식을 작은 모델에 전달
- **Architecture Search**: MobileNet, EfficientNet 같은 모바일 친화 아키텍처

**라이브러리 비교**

| 옵션 | 장점 | 단점 |
|------|------|------|
| Core ML | Neural Engine 최적, iOS 통합 | iOS 전용, 일부 연산자 제한 |
| TensorFlow Lite | 크로스 플랫폼, 활발한 커뮤니티 | NE 가속 제한적 |
| ONNX Runtime | 표준 포맷, 다양한 백엔드 | 크기 큼 |
| MLX | Apple Silicon 최적, 학습 지원 | 신생, iOS 17+ |
| PyTorch Mobile | PyTorch 직접 호환 | 크고 무거움 |

### 🔗 참고 자료
- [Apple - Core ML Tools](https://apple.github.io/coremltools/)
- [Apple - MLX](https://github.com/ml-explore/mlx)
- [TensorFlow Lite for iOS](https://www.tensorflow.org/lite/guide/ios)

---

## 9. 앱 내 결제(In-App Purchase)와 StoreKit 2는? (보너스, 자주 빈출)

### 💬 면접 답변

iOS의 결제는 모두 **StoreKit**으로 통합되어 있고, iOS 15+ 부터는 **StoreKit 2**가 권장됩니다. StoreKit 2는 async/await 기반의 현대적 API와 자동 영수증 검증, 트랜잭션 옵저버 패턴을 제공해 보일러플레이트가 크게 줄었습니다. 구매는 `Product.purchase()`로 진행하고, 결과로 받은 `Transaction`은 서명이 검증된 상태이므로 클라이언트 단에서도 안전하게 권한 부여를 할 수 있습니다. 서버 측 영수증 검증이 여전히 권장되는데, Apple이 제공하는 **App Store Server API**와 **App Store Server Notifications V2**로 구독 상태 변경을 실시간 처리합니다.

### 📚 보충 설명

```swift
import StoreKit

// 상품 조회
let products = try await Product.products(for: ["com.app.premium"])
guard let premium = products.first else { return }

// 구매
let result = try await premium.purchase()
switch result {
case .success(let verification):
    switch verification {
    case .verified(let transaction):
        // 권한 부여
        await transaction.finish()
    case .unverified(let transaction, let error):
        // 검증 실패 처리
    }
case .userCancelled, .pending:
    break
@unknown default:
    break
}

// 트랜잭션 옵저버 (앱 시작 시 등록)
Task {
    for await update in Transaction.updates {
        if case .verified(let transaction) = update {
            // 외부에서 발생한 갱신, 환불 등 처리
            await transaction.finish()
        }
    }
}
```

**구독 상태 확인**

```swift
for await result in Transaction.currentEntitlements {
    if case .verified(let t) = result, t.productID == "premium" {
        // 사용자가 premium 권한 보유
    }
}
```

**서버 검증 권장**
- 클라이언트의 verified만으로도 신뢰 가능하지만 사기·서로 다른 기기 동기화 등을 위해
- 서버에서 App Store Server API로 추가 확인 권장

### 🔗 참고 자료
- [Apple - StoreKit](https://developer.apple.com/documentation/storekit)
- [WWDC 2021 - Meet StoreKit 2](https://developer.apple.com/videos/play/wwdc2021/10114/)
- [WWDC 2023 - What's new in StoreKit and Subscriptions](https://developer.apple.com/videos/play/wwdc2023/10142/)

---

> SwiftUI와 Combine의 연결은 [06_UIKit_SwiftUI.md](./06_UIKit_SwiftUI.md), 동시성 모델은 [05_Concurrency.md](./05_Concurrency.md)를 참고하세요.
