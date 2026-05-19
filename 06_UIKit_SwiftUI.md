---
layout: default
title: "06. UIKit & SwiftUI"
nav_order: 6
---

# 06. UIKit & SwiftUI

> Auto Layout, TableView/CollectionView, SwiftUI 상태 관리, 애니메이션, 위젯, 접근성.

---

## 1. UITableView와 UICollectionView의 차이는? (Lv1, Lv2)

### 💬 면접 답변

UITableView는 **단일 열의 수직 리스트**에 특화된 컴포넌트로, 행(row)과 섹션 단위로 구성되어 설정이 간단합니다. UICollectionView는 더 일반화된 컴포넌트로, **임의의 2차원 레이아웃**을 지원하며 격자, 가로 스크롤, 워터폴 같은 다양한 레이아웃을 `UICollectionViewLayout`(특히 `UICollectionViewCompositionalLayout`)으로 표현할 수 있습니다. 둘 다 셀 재사용 메커니즘과 DataSource/Delegate 패턴을 공유하며, 최근에는 표 형식 UI도 CollectionView로 통일해서 구현하는 경향이 있습니다. 그래서 단순 리스트면 TableView, 더 다양한 레이아웃이 필요하면 CollectionView로 시작합니다.

### 📚 보충 설명

**셀 재사용 메커니즘**

```swift
// 등록
tableView.register(MyCell.self, forCellReuseIdentifier: "cell")

// 화면에 보일 때 호출
func tableView(_ tv: UITableView,
               cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tv.dequeueReusableCell(withIdentifier: "cell", for: indexPath) as! MyCell
    cell.configure(with: items[indexPath.row])
    return cell
}
```

`dequeueReusableCell`은 풀에 재사용 가능한 셀이 있으면 꺼내 반환하고, 없으면 새로 만듭니다. 화면 밖으로 나간 셀은 풀로 돌아갑니다. 덕분에 만 개의 행이 있어도 동시에 살아있는 셀은 화면에 보이는 만큼뿐입니다.

**Diffable Data Source (iOS 13+)**

전통적인 IndexPath 기반 업데이트를 안전하게 대체:

```swift
let dataSource = UITableViewDiffableDataSource<Section, Item>(tableView: tableView) {
    tv, indexPath, item in
    let cell = tv.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    cell.textLabel?.text = item.title
    return cell
}

var snapshot = NSDiffableDataSourceSnapshot<Section, Item>()
snapshot.appendSections([.main])
snapshot.appendItems(items)
dataSource.apply(snapshot, animatingDifferences: true)
```

수동으로 `insertRows`, `deleteRows`를 호출할 때 발생하는 "rows in section after update" 크래시를 원천 차단합니다.

**Compositional Layout**

```swift
let item = NSCollectionLayoutItem(layoutSize: .init(widthDimension: .fractionalWidth(1.0),
                                                    heightDimension: .fractionalHeight(1.0)))
let group = NSCollectionLayoutGroup.horizontal(layoutSize: .init(widthDimension: .fractionalWidth(1.0),
                                                                  heightDimension: .absolute(100)),
                                               subitems: [item])
let section = NSCollectionLayoutSection(group: group)
let layout = UICollectionViewCompositionalLayout(section: section)
```

### 🔗 참고 자료
- [Apple - UITableView](https://developer.apple.com/documentation/uikit/uitableview)
- [Apple - UICollectionView](https://developer.apple.com/documentation/uikit/uicollectionview)
- [WWDC 2019 - Advances in Collection View Layout](https://developer.apple.com/videos/play/wwdc2019/215/)
- [WWDC 2019 - Advances in UI Data Sources](https://developer.apple.com/videos/play/wwdc2019/220/)

---

## 2. 동적 셀 높이(Self-Sizing Cell)는 어떻게 설정하나요? (Lv1)

### 💬 면접 답변

UITableView에서는 `rowHeight = UITableView.automaticDimension`과 `estimatedRowHeight`를 함께 설정하고, 셀 내부의 Auto Layout 제약을 위에서 아래로 끊김 없이 잇게 설계하면 시스템이 컨텐츠 기반으로 높이를 자동 계산합니다. UICollectionView도 컴포지셔널 레이아웃에서 `.estimated`를 쓰면 비슷하게 동작합니다. `estimatedRowHeight`는 스크롤 시 초기 추정용이고 실제 높이와 큰 차이가 나면 점프 현상이 생기므로 평균에 가까운 값을 줍니다.

### 📚 보충 설명

```swift
tableView.rowHeight = UITableView.automaticDimension
tableView.estimatedRowHeight = 80   // 평균에 가깝게
```

**셀 안에서**
- 최상단 제약을 contentView.top에, 최하단을 contentView.bottom에 연결해 수직 체인 완성
- 가변 텍스트 라벨은 `numberOfLines = 0`

**SwiftUI는 자연스럽게 자동**

```swift
List(items) { item in
    Text(item.text)            // 길이에 따라 자동으로 높이 결정
}
```

---

## 3. 테이블뷰 delegate가 호출되지 않을 때 디버깅 순서는? (Lv1)

### 💬 면접 답변

가장 흔한 원인은 **delegate/dataSource 연결 누락**입니다. 스토리보드에서 연결을 깜빡했거나, 코드로 만든 경우 `tableView.delegate = self`를 빠뜨린 케이스입니다. 두 번째는 **셀이 화면에 한 번도 나타나지 않은 경우**로, frame이 0이거나 부모 뷰에 추가되지 않아 layout이 안 일어난 상황입니다. 세 번째는 delegate를 채택한 객체가 **메모리에서 해제**되어 weak 참조가 nil이 된 케이스로, 코디네이터·핸들러 같은 별도 객체를 delegate로 두고 strong 보유를 안 하면 종종 발생합니다. 디버깅은 breakpoint 찍어보고, lldb에서 `po tableView.delegate`로 nil 여부를 확인하는 순서로 좁혀갑니다.

### 📚 보충 설명

**Delegate를 weak로 선언하는 이유**

```swift
protocol MyDelegate: AnyObject { ... }

class A {
    weak var delegate: MyDelegate?
}
```

delegate가 strong이면 다음 순환 참조가 흔히 발생합니다.
```
ViewController (delegate 대상)
   └─ tableView
        └─ delegate = ViewController   ← strong이면 순환
```

**Optional vs Required 메서드**

Swift 프로토콜은 기본적으로 모든 요구사항이 필수입니다. optional로 만들려면 `@objc optional`:

```swift
@objc protocol MyDelegate {
    func required()
    @objc optional func mayNotImplement()
}
```

또는 protocol extension의 기본 구현으로 optional처럼 표현하는 게 Swift다운 방식입니다.

**Delegate vs Closure vs Combine 선택**

| 통신 패턴 | 추천 |
|----------|------|
| 1:1, 메서드 여러 개 | Delegate |
| 1:1, 단일 콜백 | Closure |
| 1:N, 스트림 | Combine / NotificationCenter |
| 부모 → 자식 데이터 전달 | Property / Init |

---

## 4. 테이블뷰 빠르게 스크롤 시 이미지가 잘못된 셀에 표시되는 문제는 어떻게 해결하나요? (Lv2)

### 💬 면접 답변

원인은 **셀 재사용 + 비동기 이미지 로딩의 경쟁**입니다. A 셀에 이미지를 요청했는데 응답이 도착하기 전에 그 셀이 재사용되어 B로 바뀌고, 늦게 도착한 A의 이미지가 B에 그려지는 거죠. 해결은 두 갈래입니다 — 첫째, `prepareForReuse()`에서 진행 중인 다운로드를 취소하고 이미지를 nil로 초기화합니다. 둘째, 응답이 도착했을 때 **요청 시점의 indexPath가 여전히 유효한지** 확인하거나, 더 안전하게는 셀에 "현재 요청 토큰"을 두고 응답의 토큰과 일치할 때만 적용합니다. SDWebImage·Kingfisher 같은 라이브러리는 이미 이 처리를 내장하고 있습니다.

### 📚 보충 설명

**기본 패턴**

```swift
final class ImageCell: UITableViewCell {
    private var task: URLSessionDataTask?
    private var requestURL: URL?

    override func prepareForReuse() {
        super.prepareForReuse()
        task?.cancel()
        task = nil
        imageView?.image = nil
        requestURL = nil
    }

    func configure(with url: URL) {
        requestURL = url
        // 캐시 먼저 확인
        if let cached = ImageCache.shared.image(for: url) {
            imageView?.image = cached
            return
        }
        // 비동기 다운로드
        let t = URLSession.shared.dataTask(with: url) { [weak self] data, _, _ in
            guard let self, self.requestURL == url,        // 셀이 여전히 같은 URL을 원함
                  let data, let image = UIImage(data: data) else { return }
            ImageCache.shared.set(image, for: url)
            DispatchQueue.main.async { self.imageView?.image = image }
        }
        t.resume()
        task = t
    }
}
```

**캐싱 정책**

- `NSCache`: 메모리 압력 자동 반응, thread-safe, key는 NSObject 호환만 가능
- 라이브러리 사용: Kingfisher, SDWebImage가 디스크 + 메모리 2단 캐시와 다운샘플링 모두 지원

**LRU vs LFU**
- LRU(Least Recently Used): 최근 안 쓴 것부터 제거. 구현 단순, 일반적.
- LFU(Least Frequently Used): 빈도 낮은 것부터 제거. 구현 복잡, 특정 패턴에 유리.

### 🔗 참고 자료
- [Apple - prepareForReuse()](https://developer.apple.com/documentation/uikit/uitableviewcell/1623223-prepareforreuse)
- [Kingfisher GitHub](https://github.com/onevcat/Kingfisher)

---

## 5. Auto Layout 성능이 느려질 때 어떻게 최적화하나요? (Lv1)

### 💬 면접 답변

Auto Layout은 제약 수가 늘어날수록 계산이 비선형적으로 무거워지므로, 가장 먼저 **불필요하거나 중복된 제약을 제거**합니다. 화면이 자주 갱신되는 셀에서는 제약을 매번 만들지 말고 한 번만 만들고 활성화/비활성화만 토글하는 패턴이 효과적입니다. 큰 변화는 `UIView.performWithoutAnimation`로 batch 처리하고, 정말 성능이 critical하면 frame 기반 수동 레이아웃이나 `UICollectionViewCompositionalLayout`로 전환을 검토합니다. SwiftUI에서는 layout 시스템이 다르므로 동일 화면을 두 시스템이 혼합 호스팅할 때 충돌·중복 계산이 없는지 봐야 합니다.

### 📚 보충 설명

**제약 활성화/비활성화 패턴**

```swift
private var compactConstraints: [NSLayoutConstraint] = []
private var regularConstraints: [NSLayoutConstraint] = []

override func traitCollectionDidChange(_ previous: UITraitCollection?) {
    if traitCollection.horizontalSizeClass == .compact {
        NSLayoutConstraint.deactivate(regularConstraints)
        NSLayoutConstraint.activate(compactConstraints)
    } else {
        NSLayoutConstraint.deactivate(compactConstraints)
        NSLayoutConstraint.activate(regularConstraints)
    }
}
```

**우선순위(Priority) 활용**

```swift
let preferred = label.widthAnchor.constraint(equalToConstant: 100)
preferred.priority = .defaultLow      // 250
preferred.isActive = true
// 다른 제약과 충돌 시 양보 → 깨지지 않는 레이아웃
```

**디버깅**

- Xcode → Debug → View Debugging → Show View Frames in Rulers
- 콘솔에 출력되는 "Unable to simultaneously satisfy constraints" 로그를 무시하지 말고 즉시 해결
- 시뮬레이터 → Debug → Slow Animations로 레이아웃 단계를 시각적으로 분리

**UIViewRepresentable의 intrinsicContentSize**

UIKit 뷰를 SwiftUI에 호스팅할 때 SwiftUI는 `intrinsicContentSize`로 사이즈를 결정합니다. 명시되지 않으면 0이 되어 보이지 않으니, `invalidateIntrinsicContentSize()`나 `sizeThatFits(_:uiView:context:)`을 적절히 구현해야 합니다.

### 🔗 참고 자료
- [Apple - Auto Layout Guide](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/index.html)
- [WWDC 2018 - High Performance Auto Layout](https://developer.apple.com/videos/play/wwdc2018/220/)

---

## 6. SwiftUI에서 @State 변수를 변경했는데 화면이 업데이트되지 않는 이유는? (Lv1)

### 💬 면접 답변

가장 흔한 원인은 **`@State`를 잘못된 위치에 선언**한 경우입니다. `@State`는 같은 뷰 인스턴스 안의 값 타입 상태에만 사용해야 하는데, 자식 뷰에 단순히 값으로 넘기면 자식의 상태는 부모와 무관해집니다. 또 다른 흔한 실수는 **`@State`로 선언한 객체가 참조 타입**이라 변경을 SwiftUI가 감지 못 하는 경우입니다 — 이 경우 `@StateObject`/`@ObservedObject`를 써야 합니다. 그리고 부모-자식에서 같은 상태를 공유하려면 `@Binding`을 써야 하지, 값 복사로 넘기면 변경이 부모로 전파되지 않습니다.

### 📚 보충 설명

**Property Wrapper 정리**

| 래퍼 | 용도 | 소유 |
|------|------|------|
| `@State` | 뷰의 지역 값 상태 | View가 소유 |
| `@Binding` | 다른 곳의 상태에 양방향 바인딩 | 외부 소유 |
| `@StateObject` | View가 생성·소유하는 ObservableObject | View가 소유 |
| `@ObservedObject` | 외부에서 주입된 ObservableObject | 외부 소유 |
| `@EnvironmentObject` | 환경 주입된 ObservableObject | 환경 |
| `@Environment` | 시스템 값 (`\.colorScheme` 등) | 시스템 |
| `@Observable` (iOS 17+) | 매크로 기반 관찰 | View가 직접 관찰 |

**View body가 다시 그려지는 시점**

- 의존하는 상태가 변경되었을 때
- 부모 View가 다시 그려지고 자신을 재생성할 때
- `@Published`/`@Observable`이 발행한 변경에 의존할 때

**성능 주의**

```swift
struct ContentView: View {
    @State private var counter = 0

    var body: some View {
        VStack {
            // 비싼 작업을 body에서 하지 말 것 — 매 업데이트마다 실행됨
            ExpensiveView()       // 가능하면 자식 View로 분리
        }
    }
}
```

SwiftUI는 view를 가볍게 생성하고, 실제 렌더링 대상은 diff 결과만 갱신합니다. 그래도 body가 너무 자주 호출되면 부담이 되므로 자식 View로 쪼개 관찰 의존성을 좁히는 게 좋습니다.

### 🔗 참고 자료
- [Apple - State and Data Flow](https://developer.apple.com/documentation/swiftui/state-and-data-flow)
- [WWDC 2020 - Data Essentials in SwiftUI](https://developer.apple.com/videos/play/wwdc2020/10040/)
- [WWDC 2023 - Demystify SwiftUI performance](https://developer.apple.com/videos/play/wwdc2023/10160/)

---

## 7. SwiftUI와 UIKit을 함께 사용하는 방법은? (Lv2)

### 💬 면접 답변

두 방향 모두 가능합니다. **UIKit → SwiftUI**는 `UIHostingController(rootView:)`로 SwiftUI 뷰를 UIViewController로 감싸 기존 UIKit 화면에 끼웁니다. 반대로 **SwiftUI → UIKit**은 단일 뷰는 `UIViewRepresentable`, 뷰 컨트롤러는 `UIViewControllerRepresentable` 프로토콜을 채택해 어댑터를 만듭니다. 핵심은 두 시스템의 라이프사이클과 상태 흐름을 명확히 매핑하는 것입니다 — `makeUIView`, `updateUIView`, `Coordinator`를 통해 SwiftUI 상태 변화를 UIKit에 반영하고, delegate 콜백은 `Coordinator`에서 받아 SwiftUI Binding으로 다시 흘려보냅니다.

### 📚 보충 설명

**UIViewRepresentable 예시 (UIView를 SwiftUI에서)**

```swift
struct ActivityIndicator: UIViewRepresentable {
    @Binding var isAnimating: Bool

    func makeUIView(context: Context) -> UIActivityIndicatorView {
        UIActivityIndicatorView(style: .medium)
    }

    func updateUIView(_ view: UIActivityIndicatorView, context: Context) {
        isAnimating ? view.startAnimating() : view.stopAnimating()
    }
}
```

**Coordinator로 delegate 받기**

```swift
struct TextFieldWrapper: UIViewRepresentable {
    @Binding var text: String

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeUIView(context: Context) -> UITextField {
        let tf = UITextField()
        tf.delegate = context.coordinator
        return tf
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        uiView.text = text
    }

    class Coordinator: NSObject, UITextFieldDelegate {
        let parent: TextFieldWrapper
        init(_ parent: TextFieldWrapper) { self.parent = parent }

        func textFieldDidChangeSelection(_ tf: UITextField) {
            parent.text = tf.text ?? ""
        }
    }
}
```

**UIHostingController로 SwiftUI를 UIKit에 끼우기**

```swift
let host = UIHostingController(rootView: MySwiftUIView())
addChild(host)
host.view.frame = view.bounds
view.addSubview(host.view)
host.didMove(toParent: self)
```

**주의점**
- `updateUIView`/`updateUIViewController`는 SwiftUI 측 상태가 바뀔 때마다 호출 — 불필요한 작업을 피하기
- 사이즈 계산은 SwiftUI가 부모 뷰의 `intrinsicContentSize`를 묻는 방식 → 이를 명시적으로 제공해야 함
- `Coordinator`에서 SwiftUI 상태를 변경할 때는 `DispatchQueue.main.async`나 `Task { @MainActor in }` 안에서 (재귀 업데이트 방지)

### 🔗 참고 자료
- [Apple - UIViewRepresentable](https://developer.apple.com/documentation/swiftui/uiviewrepresentable)
- [Apple - UIHostingController](https://developer.apple.com/documentation/swiftui/uihostingcontroller)
- [WWDC 2020 - Integrate SwiftUI with UIKit](https://developer.apple.com/videos/play/wwdc2020/10119/)

---

## 8. UIView는 클래스인데 SwiftUI의 View는 왜 구조체로 정의되나요? (Lv2)

### 💬 면접 답변

SwiftUI의 View는 **선언적이고 매우 자주 재생성되는** 가벼운 기술서이지, 화면에 실제로 그려지는 객체가 아닙니다. 상태가 변경될 때마다 View 값들이 새로 만들어지고, SwiftUI 런타임이 그 값을 비교(diff)해서 실제 렌더링 백엔드의 변경만 적용합니다. 이런 사이클에 클래스를 쓰면 ARC와 힙 할당 비용이 매번 발생해 비효율적이라, 값 타입인 struct가 자연스러운 선택입니다. 상태는 `@State`, `@StateObject` 같은 프로퍼티 래퍼가 View 외부의 안정된 저장소에 두고, View struct는 그 상태를 가리키는 가벼운 핸들 역할을 합니다.

### 📚 보충 설명

**View struct의 lifecycle**

```swift
struct CounterView: View {
    @State private var count = 0     // count의 실제 저장은 SwiftUI가 별도로 관리

    var body: some View {
        Text("\(count)")
        Button("증가") { count += 1 } // body가 재호출되어도 count는 유지
    }
}
```

**왜 클래스가 아닌가**
- 매 재계산마다 메모리 할당 비용 발생 → 60·120fps 유지 어려움
- 참조 시맨틱이라 잘못된 공유로 인한 버그
- 값 타입은 컴파일러가 더 적극 최적화

**UIKit View가 클래스인 이유**
- UI 트리는 명령형으로 만들고 직접 조작 — 객체 identity와 mutable state가 필요
- 라이프사이클 메서드를 통해 외부에서 제어

### 🔗 참고 자료
- [WWDC 2019 - SwiftUI Essentials](https://developer.apple.com/videos/play/wwdc2019/216/)
- [WWDC 2023 - Demystify SwiftUI performance](https://developer.apple.com/videos/play/wwdc2023/10160/)

---

## 9. Core Animation의 핵심 개념을 설명해주세요. (Lv2)

### 💬 면접 답변

Core Animation은 iOS의 GPU 가속 그래픽 레이어 시스템으로, UIView의 시각적 표현이 실제로는 `CALayer`로 구성됩니다. 우리가 UIView 속성을 변경하면 내부적으로 CALayer가 변경되고, 이때 GPU가 컴포지팅을 담당합니다. 애니메이션은 **암시적 애니메이션**(CALayer 속성 변경 시 자동) 또는 **명시적 애니메이션**(`CABasicAnimation`, `CAKeyframeAnimation`)으로 만들 수 있습니다. UIView의 `UIView.animate(withDuration:)`도 내부적으로 이 시스템을 사용합니다. GPU에서 처리되므로 메인 스레드 부하 없이 부드러운 애니메이션이 가능합니다.

### 📚 보충 설명

**CALayer 주요 속성**
- `frame`, `bounds`, `position`
- `transform` (CATransform3D)
- `cornerRadius`, `borderWidth`, `shadowOpacity`
- `mask`, `contents`

**암시적 vs 명시적**

```swift
// 암시적 (CALayer 직접 속성 변경)
layer.opacity = 0     // 기본 0.25초 페이드 자동

// UIView 애니메이션 블록 (대부분의 경우 권장)
UIView.animate(withDuration: 0.3) {
    view.alpha = 0
    view.transform = CGAffineTransform(scaleX: 0.5, y: 0.5)
}

// 명시적 CABasicAnimation (세밀한 컨트롤)
let anim = CABasicAnimation(keyPath: "opacity")
anim.fromValue = 1
anim.toValue = 0
anim.duration = 0.3
layer.add(anim, forKey: "fade")
```

**Keyframe vs Spring**

- `CAKeyframeAnimation`: 여러 중간 값을 지정 (경로 애니메이션 등)
- `UIView.animate(withDuration:..., usingSpringWithDamping:initialSpringVelocity:...)`: 자연스러운 스프링 감속

**Animation Group**

여러 애니메이션을 묶어 동기화:

```swift
let group = CAAnimationGroup()
group.animations = [scaleAnim, fadeAnim]
group.duration = 0.5
layer.add(group, forKey: "combined")
```

**iOS 17+ SwiftUI 애니메이션 개선**
- 새로운 `phaseAnimator`, `keyframeAnimator`
- `.animation(_, value:)` 의 정밀한 트리거 제어

### 🔗 참고 자료
- [Apple - Core Animation](https://developer.apple.com/documentation/quartzcore)
- [Apple - Core Animation Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html)
- [WWDC 2023 - Wind your way through advanced animations in SwiftUI](https://developer.apple.com/videos/play/wwdc2023/10157/)

---

## 10. Adaptive Layout과 Size Classes는 무엇인가요? (Lv2)

### 💬 면접 답변

Size Class는 화면 폭과 높이를 각각 **Compact**와 **Regular**로 단순화한 추상 카테고리입니다. iPhone 세로는 폭 compact·높이 regular, iPad는 보통 둘 다 regular, 같은 식이라 디바이스 모델별로 분기하지 않고도 적응형 레이아웃을 짤 수 있습니다. Adaptive Layout은 이 size class와 다른 trait들(`UITraitCollection`)을 활용해 화면이 회전하거나 Split View가 열릴 때 자동으로 UI를 재배치하도록 하는 설계 방식입니다. SwiftUI에서는 `horizontalSizeClass`/`verticalSizeClass` 환경 값과 `if`로 같은 효과를 표현합니다.

### 📚 보충 설명

**디바이스별 일반적인 Size Class**

| 디바이스 | 세로 | 가로 |
|---------|------|------|
| iPhone (Plus 제외) | C, R | C, C |
| iPhone Plus/Max | C, R | R, C |
| iPad | R, R | R, R |
| iPad Slide Over | C, R | – |

**UIKit에서 활용**

```swift
override func traitCollectionDidChange(_ previous: UITraitCollection?) {
    super.traitCollectionDidChange(previous)
    if traitCollection.horizontalSizeClass == .compact {
        // 1열 레이아웃
    } else {
        // 2열 또는 사이드바
    }
}
```

**SwiftUI에서 활용**

```swift
struct ContentView: View {
    @Environment(\.horizontalSizeClass) var hSize

    var body: some View {
        if hSize == .compact {
            CompactLayout()
        } else {
            RegularLayout()
        }
    }
}
```

### 🔗 참고 자료
- [Apple - Adaptivity and Layout](https://developer.apple.com/design/human-interface-guidelines/layout)
- [Apple - UITraitCollection](https://developer.apple.com/documentation/uikit/uitraitcollection)

---

## 11. WidgetKit으로 홈 화면 위젯을 구현하는 방법은? (Lv3)

### 💬 면접 답변

WidgetKit은 SwiftUI 기반의 위젯 프레임워크로, 앱의 별도 익스텐션으로 추가합니다. 위젯의 UI는 100% SwiftUI로 작성하고, **TimelineProvider**가 시스템에 "언제, 어떤 데이터로" 위젯을 다시 그릴지를 알려주는 핵심입니다. 위젯은 자체 프로세스로 동작하고, 시스템이 미리 정해진 시간에 깨워서 다음 스냅샷들을 미리 그려두는 모델이라, 메인 앱과의 데이터 공유는 App Group을 통한 공유 컨테이너로 합니다. iOS 17부터는 **위젯 인터랙티브** 지원(App Intent로 버튼·토글)이 추가돼 표현력이 크게 늘었습니다.

### 📚 보충 설명

**기본 구조**

```swift
struct Provider: TimelineProvider {
    func placeholder(in context: Context) -> SimpleEntry { /* 자리표시자 */ }

    func getSnapshot(in context: Context, completion: @escaping (SimpleEntry) -> Void) {
        completion(SimpleEntry(date: Date(), data: ...))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<SimpleEntry>) -> Void) {
        let entries: [SimpleEntry] = [/* 다음 1시간 동안의 스냅샷들 */]
        let timeline = Timeline(entries: entries, policy: .atEnd)
        completion(timeline)
    }
}

struct MyWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "com.app.widget", provider: Provider()) { entry in
            WidgetView(entry: entry)
        }
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}
```

**TimelinePolicy**
- `.atEnd`: 마지막 entry의 시간에 다시 요청
- `.after(date)`: 특정 시각에 다시 요청
- `.never`: 사용자 액션 전까지 갱신 안 함

**메인 앱과 데이터 공유**

```swift
let container = FileManager.default
    .containerURL(forSecurityApplicationGroupIdentifier: "group.com.app")!
let url = container.appendingPathComponent("widget-data.json")
// 앱에서 저장 → 위젯에서 읽기
```

**위젯 갱신 트리거**
- 시스템이 정한 budget 내에서 자체 갱신
- 앱에서 `WidgetCenter.shared.reloadAllTimelines()` 호출
- 푸시(silent push)로 백그라운드 깨우기

**iOS 17+ 인터랙티브 위젯**

```swift
Button(intent: ToggleIntent()) {
    Image(systemName: "checkmark")
}
```

### 🔗 참고 자료
- [Apple - WidgetKit](https://developer.apple.com/documentation/widgetkit)
- [WWDC 2020 - Meet WidgetKit](https://developer.apple.com/videos/play/wwdc2020/10028/)
- [WWDC 2023 - Bring widgets to life](https://developer.apple.com/videos/play/wwdc2023/10028/)

---

## 12. 접근성(Accessibility)을 어떻게 구현하나요? (Lv3)

### 💬 면접 답변

접근성은 모든 사용자가 앱을 쓸 수 있게 만드는 필수 기능입니다. UIKit에서는 모든 뷰에 `isAccessibilityElement`, `accessibilityLabel`, `accessibilityHint`, `accessibilityTraits`를 적절히 설정해 **VoiceOver**가 의미 있게 읽도록 합니다. **Dynamic Type**은 사용자가 시스템 설정에서 글자 크기를 키워도 UI가 깨지지 않도록 폰트를 `UIFont.preferredFont(forTextStyle:)` 같은 시스템 폰트로 받아 자동 조정합니다. SwiftUI에서는 `.accessibilityLabel`, `.accessibilityHint`, `.dynamicTypeSize` 같은 modifier가 기본적으로 더 잘 통합되어 있습니다. WWDC와 HIG의 접근성 가이드를 따라 색약, Switch Control, 큰 글씨 모드까지 폭넓게 테스트하는 게 좋습니다.

### 📚 보충 설명

**기본 UIKit 패턴**

```swift
button.isAccessibilityElement = true
button.accessibilityLabel = "프로필 편집"
button.accessibilityHint = "두 번 탭하여 편집 화면을 엽니다."
button.accessibilityTraits = .button
```

**Dynamic Type**

```swift
label.font = UIFont.preferredFont(forTextStyle: .body)
label.adjustsFontForContentSizeCategory = true   // 사용자가 글자 크기 바꾸면 자동 갱신
```

SwiftUI:
```swift
Text("Hello").font(.body)        // 자동으로 Dynamic Type
```

**SwiftUI 접근성**

```swift
Image(systemName: "heart.fill")
    .accessibilityLabel("좋아요")
    .accessibilityValue("\(count)개")
    .accessibilityHint("두 번 탭하여 좋아요를 추가합니다.")
    .accessibilityAddTraits(.isButton)
```

**테스트 방법**

- VoiceOver 시뮬레이션: Settings → Accessibility → VoiceOver (실기기 권장)
- Xcode → Accessibility Inspector
- Accessibility Audit (Xcode 15+)
- 큰 글씨, 색약 시뮬레이터로 시각적 확인

### 🔗 참고 자료
- [Apple - Accessibility](https://developer.apple.com/accessibility/)
- [Apple - UIAccessibility](https://developer.apple.com/documentation/objectivec/nsobject/uiaccessibility)
- [WWDC 2023 - Build accessible apps with SwiftUI and UIKit](https://developer.apple.com/videos/play/wwdc2023/10036/)
- [Apple HIG - Accessibility](https://developer.apple.com/design/human-interface-guidelines/accessibility)

---

## 13. iOS 17의 @Observable 매크로의 장점은? (Lv3)

### 💬 면접 답변

iOS 17에서 도입된 `@Observable`은 기존 `ObservableObject`/`@Published`보다 **세밀한 단위로 변경을 추적**합니다. 기존에는 ObservableObject의 어떤 `@Published` 프로퍼티든 바뀌면 그 객체에 의존하는 모든 SwiftUI 뷰가 다시 그려졌는데, `@Observable`은 실제로 뷰가 **읽은 프로퍼티**가 바뀔 때만 그 뷰를 갱신합니다. 결과적으로 불필요한 body 호출이 줄어들어 성능이 좋아지고, 코드도 매크로가 보일러플레이트를 줄여줍니다. 마이그레이션은 `class`에 `@Observable` 매크로를 붙이고, 뷰에서 `@StateObject`/`@ObservedObject` 대신 그냥 변수처럼 쓰거나 `@Bindable`을 활용하는 형태입니다.

### 📚 보충 설명

**Before (ObservableObject)**

```swift
class UserVM: ObservableObject {
    @Published var name = ""
    @Published var age = 0
}

struct ContentView: View {
    @StateObject var vm = UserVM()
    var body: some View {
        Text(vm.name)        // age가 바뀌어도 이 뷰가 갱신됨
    }
}
```

**After (@Observable)**

```swift
@Observable
class UserVM {
    var name = ""
    var age = 0
}

struct ContentView: View {
    @State private var vm = UserVM()    // @Bindable로 바인딩 시
    var body: some View {
        Text(vm.name)        // age 변경은 더 이상 이 뷰를 갱신하지 않음
    }
}
```

**바인딩이 필요할 때**

```swift
struct EditView: View {
    @Bindable var vm: UserVM     // 양방향 바인딩

    var body: some View {
        TextField("이름", text: $vm.name)
    }
}
```

**iOS 17 미만을 지원해야 한다면**

```swift
@available(iOS 17.0, *)
@Observable class NewVM { ... }

class LegacyVM: ObservableObject {
    @Published var ...
}
```

또는 [swift-perception](https://github.com/pointfreeco/swift-perception) 같은 백포트 라이브러리 사용.

**Combine과의 관계**

`@Observable`은 Combine을 사용하지 않습니다. 별도의 observation 메커니즘(SE-0395)을 사용해 더 효율적입니다. 다만 기존 Combine 코드와는 직접 연결되지 않으므로 마이그레이션 시 점진적 전략 필요.

### 🔗 참고 자료
- [Swift Evolution: SE-0395 - Observation](https://github.com/apple/swift-evolution/blob/main/proposals/0395-observability.md)
- [Apple - Observable](https://developer.apple.com/documentation/observation/observable())
- [WWDC 2023 - Discover Observation in SwiftUI](https://developer.apple.com/videos/play/wwdc2023/10149/)

---

## 14. UIKit의 ViewController 라이프사이클을 설명해주세요. viewDidLoad에서 네트워크 요청하면 안 되는 이유는? (Lv2)

### 💬 면접 답변

UIViewController는 `loadView` → `viewDidLoad` → `viewWillAppear` → `viewIsAppearing`(Xcode 15+ SDK에서 노출, iOS 13.0+에서 동작) → `viewDidAppear` → `viewWillDisappear` → `viewDidDisappear` 순서로 호출됩니다. `viewDidLoad`는 뷰가 메모리에 처음 로드된 직후 한 번만 호출되어 일회성 초기 설정에 적합한데, **뷰의 사이즈가 아직 최종 확정 전**이라서 layout 의존 작업은 적절하지 않습니다. 네트워크 요청 자체를 `viewDidLoad`에서 하는 게 절대적으로 잘못은 아니지만, 뷰 컨트롤러가 캐시에서 재사용되거나 메모리 압력으로 view가 해제됐다가 다시 로드될 때 중복 호출될 수 있고, 사용자가 화면에 도달하지 않았는데 요청이 시작되어 낭비될 수도 있습니다. 그래서 **UI를 보여줄 때 시점이 명확한 `viewWillAppear`나 `viewIsAppearing`**, 또는 `Task { ... }`로 데이터 모델 단에서 트리거하는 게 더 적절합니다.

### 📚 보충 설명

**라이프사이클 메서드 역할**

| 메서드 | 호출 시점 | 적합한 작업 |
|--------|----------|------------|
| `loadView` | view 생성 | 코드로 view 계층 만들기 (대부분 손대지 않음) |
| `viewDidLoad` | view가 로드된 직후 (1회) | 데이터 모델 초기화, 일회성 UI 설정 |
| `viewWillAppear` | 화면에 나타나기 직전 (매번) | UI 상태 갱신, 옵저버 등록 |
| `viewIsAppearing` | view가 계층에 추가되고 사이즈 확정 직후 (iOS 13.0+, Xcode 15 SDK 필요) | layout 의존 작업 |
| `viewDidAppear` | 화면 표시 완료 (매번) | 애니메이션 시작, 분석 이벤트 |
| `viewWillDisappear` | 사라지기 직전 | 옵저버 해제, 변경사항 저장 |
| `viewDidDisappear` | 사라짐 완료 | 리소스 정리 |

**`viewDidLoad`가 다시 불릴 수 있는 시나리오**
- iOS 5 이전: 메모리 경고 시 view 해제 → 재로드. 현재는 자동 해제 없음.
- 캐시 정책으로 직접 `view = nil` 설정 시
- Storyboard에서 같은 VC를 다른 인스턴스로 instantiate

**비동기 요청 정리**

```swift
class VC: UIViewController {
    var task: Task<Void, Never>?

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        task = Task { [weak self] in
            guard let data = try? await api.fetch() else { return }
            await MainActor.run { self?.update(data) }
        }
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        task?.cancel()   // 사용자가 떠나면 요청 취소
    }
}
```

### 🔗 참고 자료
- [Apple - UIViewController](https://developer.apple.com/documentation/uikit/uiviewcontroller)
- [WWDC 2023 - What's new in UIKit](https://developer.apple.com/videos/play/wwdc2023/10055/) (viewIsAppearing 소개)

---

> 아키텍처 패턴(MVC, MVVM, Coordinator)은 [07_Architecture.md](./07_Architecture.md), SwiftData/Core Data는 [09_Data_Persistence.md](./09_Data_Persistence.md)를 참고하세요.
