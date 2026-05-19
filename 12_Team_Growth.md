---
layout: default
title: "12. 팀 협업과 개발자 성장"
nav_order: 12
---

# 12. 팀 협업과 개발자 성장

> 협업 전략, 성능 최적화 전략, 아키텍처 설계 원칙, 개발자 학습·성장.

---

## 1. 5명의 iOS 개발자가 함께 일할 때 코드 충돌을 최소화하려면? (Lv5)

### 💬 면접 답변

코드 충돌은 보통 **같은 파일을 여러 명이 동시에 수정**할 때 발생하므로, 가장 효과적인 예방은 **모듈화와 작업 분담의 명확화**입니다. 화면이나 기능 단위로 모듈을 나누고, PR은 작게 자주 올려 빠르게 main에 머지되도록 합니다. 또 코드 스타일·아키텍처·네이밍 컨벤션을 문서화하고 SwiftFormat·SwiftLint로 자동 강제하면 PR마다 불필요한 형식 차이가 줄어 충돌이 감소합니다. 코드 리뷰는 단순히 결함 검출이 아니라 **지식 공유의 자리**이고, 페어 프로그래밍이나 매주 1회 기술 공유 세션 같은 의식적인 활동이 장기적으로 충돌·재작업을 크게 줄입니다.

### 📚 보충 설명

**충돌 줄이기 실천 사항**

1. **모듈 경계**: SwiftPM로 기능별 모듈화 → 파일 단위 충돌 ↓
2. **작은 PR**: 100~300줄 권장. 큰 PR은 리뷰 누락·머지 지연·충돌 증폭
3. **자주 main 동기화**: 매일 아침 `git pull --rebase`
4. **포맷터 자동화**: pre-commit hook + CI 검증
5. **명확한 컨벤션**: 코드 스타일, 폴더 구조, 네이밍, 커밋 메시지
6. **CODEOWNERS**: 영역별 책임자 지정으로 리뷰어 자동 할당

**코드 리뷰 가이드**
- 24시간 내 1차 응답 SLA
- "이건 어떻게 생각해?" 같은 토론 권장
- 비판이 아닌 개선 제안의 톤
- 좋은 부분 칭찬도 명시
- 작성자 외 최소 2명의 승인

**효과적인 페어 프로그래밍**
- Driver/Navigator 역할 교대
- 25~45분 단위
- 어려운 부분, 신규 기능 도입 때 효과 ↑

**기술 공유 세션**
- 주 1회 30분
- 발표자 로테이션
- 주제: 새 기술, 문제 해결 사례, 리팩토링 경험

### 🔗 참고 자료
- [Google's Engineering Practices](https://google.github.io/eng-practices/)
- [SwiftLint](https://github.com/realm/SwiftLint)
- [SwiftFormat](https://github.com/nicklockwood/SwiftFormat)

---

## 2. iOS 앱의 성능 최적화 전략과 도구는? (Lv5)

### 💬 면접 답변

성능 최적화는 **추측하지 말고 측정부터**가 원칙입니다. Instruments의 Time Profiler·Allocations·Leaks·SwiftUI 템플릿으로 실제 병목을 찾아내고, 가장 영향이 큰 곳부터 개선합니다. 영역별로 보면 **앱 시작 시간**(동적 라이브러리 줄이기, lazy 초기화), **메모리**(이미지 다운샘플링, 캐시 정책, 누수 제거), **네트워크**(캐싱, 압축, 병합), **렌더링**(60·120fps 유지, 메인 스레드 차단 제거)으로 나뉩니다. 정기적인 성능 회귀 테스트(XCTest의 `measure`, Xcode의 metrics)와 프로덕션 데이터(MetricKit, Crashlytics, Firebase Performance) 모니터링이 함께 가야 지속적인 품질을 유지할 수 있습니다.

### 📚 보충 설명

**측정 도구**

| 영역 | 도구 |
|------|------|
| CPU | Instruments - Time Profiler |
| 메모리 | Allocations, Leaks, Memory Graph |
| 시작 시간 | App Launch 템플릿 |
| 렌더링 | Animation Hitches, Core Animation |
| 네트워크 | Network 템플릿, MetricKit |
| 배터리 | Energy Log |
| 프로덕션 | MetricKit, Firebase Performance, Crashlytics |

**MetricKit (iOS 13+)**

```swift
import MetricKit

class MetricsManager: NSObject, MXMetricManagerSubscriber {
    func didReceive(_ payloads: [MXMetricPayload]) {
        // 24시간마다 받는 메트릭 (시작 시간, 메모리, hang 등)
    }

    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        // 크래시, 디스크 쓰기, hang 진단
    }
}
```

**XCTest 성능 측정**

```swift
func test_search_performance() {
    measure(metrics: [XCTClockMetric(), XCTMemoryMetric()]) {
        // 측정할 작업
        sut.search(query: "test")
    }
}
```

**일반 최적화 체크리스트**
- Asset Catalog 활용, 이미지 다운샘플링
- TableView/CollectionView 셀 재사용 + Diffable Data Source
- 무거운 작업은 백그라운드, UI 업데이트만 메인
- 캐싱 정책 (NSCache, URLCache, 이미지)
- 메인 스레드 hang 제거 (Watchdog 회피)
- @ObservedObject 의존성 좁히기 (불필요한 redraw 방지)

### 🔗 참고 자료
- [Apple - MetricKit](https://developer.apple.com/documentation/metrickit)
- [WWDC 2019 - Improving Battery Life and Performance](https://developer.apple.com/videos/play/wwdc2019/417/)
- [WWDC 2022 - Eliminate animation hitches with XCTest](https://developer.apple.com/videos/play/wwdc2022/110366/)

---

## 3. 지속 가능한 iOS 앱 아키텍처와 모듈화 전략은? (Lv5)

### 💬 면접 답변

지속 가능한 아키텍처의 핵심은 **변경에 강한 경계**를 두는 것입니다. UI 프레임워크와 도메인 로직을 분리하고, 외부 시스템(서버·DB)을 인터페이스로 추상화해 구현체를 교체할 수 있게 합니다. Clean Architecture, VIPER 같은 패턴이 이를 명문화한 예시고, iOS에서는 SwiftPM로 기능 모듈을 잘게 쪼개면 빌드 시간 단축과 의존성 명확화 효과가 큽니다. 모듈 간 결합은 **인터페이스(프로토콜)** 만 공유하고 구체 구현은 숨기는 것이 원칙이며, 의존성 주입과 인터페이스 분리(ISP)로 결합도를 낮춥니다. 다만 작은 앱에 과한 모듈화는 오버엔지니어링이라, 코드베이스가 커지는 시점에 맞춰 단계적으로 적용합니다.

### 📚 보충 설명

**모듈화 단계별 접근**

| 단계 | 구조 | 적합한 시점 |
|------|------|------------|
| 1. 단일 타깃 | App 타깃 하나 | 초기 / 소규모 |
| 2. 폴더 분리 | Feature/Group 폴더 | 중규모 |
| 3. SwiftPM 로컬 패키지 | App + Local Packages | 빌드 시간 ↑ 또는 팀 ↑ |
| 4. 다중 타깃 | App + Frameworks | 큰 팀, 여러 앱 공유 |

**SwiftPM 로컬 패키지 예**

```
MyApp/
├── App/                          ← 메인 앱 타깃
├── Packages/
│   ├── Core/                     ← 공통 유틸
│   ├── Networking/               ← 네트워크
│   ├── DesignSystem/             ← UI 컴포넌트
│   ├── Feature-Login/            ← 로그인 기능
│   └── Feature-Home/             ← 홈 기능
```

**의존성 방향 원칙**

```
Feature-Login   Feature-Home
       ↓             ↓
   DesignSystem  Networking
              ↓
            Core
```

- Feature 간 직접 의존 ❌
- 공통 의존성은 아래 레이어로 이동

**Interface Package 패턴**

각 모듈을 `XxxInterface`(프로토콜만)와 `Xxx`(구현)로 분리하면, 의존하는 쪽은 인터페이스에만 의존하고 실제 구현은 컴포지션 루트(App 타깃)에서만 임포트합니다. 빌드 그래프가 평평해져 빌드가 빨라지고, 테스트도 쉬워집니다.

### 🔗 참고 자료
- [Apple - Organizing your code with local packages](https://developer.apple.com/documentation/xcode/organizing-your-code-with-local-packages)
- Robert C. Martin, *Clean Architecture*

---

## 4. iOS 개발자가 성장하려면 어떤 전략을 따라야 할까요? (Lv5)

### 💬 면접 답변

iOS 개발자의 성장은 **깊이와 폭, 그리고 협업 능력** 세 축으로 봅니다. 깊이는 Swift·UIKit/SwiftUI·Concurrency 같은 핵심 영역을 공식 문서와 WWDC로 1차 학습하는 게 가장 효율적입니다. 폭은 iOS 외부 영역 — CS 기초, 네트워크, 백엔드, ML 같은 인접 영역을 넓혀가는 것으로, 좋은 시니어와 평범한 시니어를 가르는 결정적 요인입니다. 협업은 코드 리뷰·기술 글쓰기·발표·오픈소스 기여로 키울 수 있습니다. 매년 WWDC 직후 핵심 세션 보기, 분기마다 작은 사이드 프로젝트, 매월 기술 블로그 1편 같은 **루틴**으로 만들면 자연스럽게 누적됩니다.

### 📚 보충 설명

**학습 자료 우선순위**

1. **Apple 공식**
   - [Apple Developer Documentation](https://developer.apple.com/documentation/)
   - WWDC 세션 (특히 What's New 시리즈)
   - [The Swift Programming Language](https://docs.swift.org/swift-book/)
   - [Swift Evolution](https://github.com/apple/swift-evolution)

2. **신뢰할 만한 영문 자료**
   - [Swift by Sundell](https://www.swiftbysundell.com/)
   - [Hacking with Swift](https://www.hackingwithswift.com/)
   - [Donny Wals](https://www.donnywals.com/)
   - [objc.io](https://www.objc.io/)
   - [Point-Free](https://www.pointfree.co/)

3. **한국어 자료**
   - 야곰닷넷, 한국 iOS 블로거들
   - Let'Swift, KWDC 등 컨퍼런스

**성장 트랙 예시 (2년차 → 4년차)**

| 분기 | 학습 주제 | 산출물 |
|------|----------|--------|
| 1Q | Swift Concurrency 마스터 | 토이 프로젝트 + 블로그 1편 |
| 2Q | SwiftUI + Observable | 기존 화면 마이그레이션 |
| 3Q | 모듈화 + SwiftPM | 회사 프로젝트 분리 |
| 4Q | 성능 측정 + Instruments | 성능 회귀 테스트 도입 |

**오픈소스 기여 권장 단계**
1. 사용 중인 라이브러리의 README 오타 수정
2. 이슈 재현 및 답글 작성
3. 작은 버그 수정 PR
4. 새 기능 PR
5. 자체 라이브러리 공개

**기술 글쓰기 효과**
- 학습 내용 정리 (Feynman Technique)
- 커뮤니티에 기여
- 이직 시 포트폴리오
- 글쓰기 자체가 사고 정리

### 🔗 참고 자료
- [WWDC Videos](https://developer.apple.com/videos/)
- [Swift Forums](https://forums.swift.org/)

---

## 5. UX/UI 디자인과 협업하는 방법은? (Lv5)

### 💬 면접 답변

좋은 UX는 디자이너 혼자 만드는 게 아니라 **개발 단계에서 디자이너의 의도가 정확히 구현**되어야 완성됩니다. 그래서 디자이너와는 디자인 시스템·컴포넌트 명세부터 함께 만드는 게 좋고, Figma의 inspect 기능이나 디자인 토큰 export(예: figma-tokens)로 색·간격·폰트를 코드와 동기화하면 일관성 유지가 쉬워집니다. 인터랙션과 애니메이션처럼 정적 mockup으로 표현 어려운 부분은 함께 디바이스에서 검토하고, 접근성·다국어·다양한 화면 크기까지 디자인 단계에서 고려하도록 체크리스트를 공유합니다. 좋은 협업은 **개발자가 디자인 의도를 이해**하고 **디자이너가 기술 제약을 이해**할 때 만들어집니다.

### 📚 보충 설명

**디자인 시스템 구축**

```swift
// 코드의 디자인 토큰
extension Color {
    static let primary = Color(hex: "#007AFF")
    static let secondary = Color(hex: "#5856D6")
}

extension CGFloat {
    enum Spacing {
        static let xs: CGFloat = 4
        static let s: CGFloat = 8
        static let m: CGFloat = 16
        static let l: CGFloat = 24
    }
}

// 재사용 가능한 컴포넌트
struct PrimaryButton: View {
    let title: String
    let action: () -> Void
    var body: some View {
        Button(action: action) {
            Text(title)
                .font(.headline)
                .padding()
                .frame(maxWidth: .infinity)
                .background(Color.primary)
                .foregroundStyle(.white)
                .cornerRadius(12)
        }
    }
}
```

**디자이너와의 공통 체크리스트**
- Dynamic Type 지원 (글자 크기)
- 다크 모드 색상 정의
- 접근성 (VoiceOver, 색약 대응)
- 다국어 (긴 문자열, RTL)
- 다양한 화면 크기 (Compact/Regular, iPad)
- 빈 상태 / 로딩 / 에러 상태

**디자인 시스템 라이브러리화**

SwiftPM 로컬 패키지로 DesignSystem을 분리하면 여러 모듈에서 일관되게 활용 가능.

### 🔗 참고 자료
- [Apple HIG](https://developer.apple.com/design/human-interface-guidelines/)
- [Figma Inspect](https://help.figma.com/hc/en-us/articles/360055203533-Inspect-designs)

---

## 6. 기술 부채를 어떻게 관리하나요? (보너스)

### 💬 면접 답변

기술 부채는 **부채 자체가 나쁜 게 아니라, 가시화되지 않은 채 누적되는 게 위험**합니다. 그래서 매 스프린트마다 "기술 부채 항목"을 백로그에 명시적으로 등록하고, 비즈니스 기능과 함께 우선순위 회의에 올립니다. 부채를 평가할 때는 **상환 비용**과 **방치 비용**을 같이 보는데, 자주 수정되는 영역의 부채는 빨리, 거의 안 건드리는 영역은 천천히 처리합니다. 또 "보이스카웃 룰"처럼 새 기능을 작업할 때 그 주변의 작은 부채를 함께 정리하는 문화가 누적 효과가 큽니다. 큰 리팩토링은 별도 스프린트로 잡되, 기능 개발과 100% 분리하기보단 명확한 목표(예: 빌드 시간 30% 단축)와 함께 진행해야 비즈니스와 합의가 됩니다.

### 📚 보충 설명

**기술 부채 분류 (Martin Fowler)**

| 유형 | 의도성 | 신중함 |
|------|--------|--------|
| 무모한 의도적 | "시간 없어, 지금 막 쓰자" | – |
| 신중한 의도적 | "지금 빨리 출시하고 나중에 갚자" | – |
| 무모한 우발적 | "어, 이게 부채인 줄 몰랐네" | – |
| 신중한 우발적 | "지금 보니 더 나은 방법이 있었네" | – |

**관리 패턴**
- 백로그에 부채 항목 등록 (라벨링)
- 분기별로 부채 리뷰 미팅
- 부채 해결 시간 비율 정하기 (예: 매 스프린트 20%)
- 메트릭으로 추적 (코드 커버리지, 빌드 시간, 크래시율)

### 🔗 참고 자료
- [Martin Fowler - Technical Debt Quadrant](https://martinfowler.com/bliki/TechnicalDebtQuadrant.html)

---

## 7. 코드 리뷰에서 무엇을 봐야 하나요? (보너스)

### 💬 면접 답변

좋은 코드 리뷰는 **정확성, 가독성, 유지보수성**을 모두 봅니다. 정확성은 로직이 의도대로 동작하는지·엣지 케이스가 누락되지 않았는지·동시성 안전성·메모리 누수 가능성 등을 확인하고, 가독성은 네이밍·함수의 크기·추상화 수준 일관성을 봅니다. 유지보수성은 SOLID 같은 설계 원칙, 적절한 테스트 작성, 향후 변경에 강한 구조인지를 점검합니다. 리뷰는 **결함 발견뿐만 아니라 지식 공유**의 자리이기도 해서, 다른 동료의 코드를 통해 새 패턴을 배우고 팀 표준을 정렬하는 효과도 큽니다. 톤은 비판이 아닌 개선 제안으로, 좋은 부분도 명시적으로 칭찬합니다.

### 📚 보충 설명

**iOS 코드 리뷰 체크리스트**

**정확성**
- [ ] 비즈니스 로직이 요구사항과 일치
- [ ] 옵셔널·강제 언래핑·강제 캐스팅의 안전성
- [ ] 동시성 안전성 (`@MainActor`, `Sendable`, race 가능성)
- [ ] 메모리 누수 (`[weak self]`, 옵저버 해제)
- [ ] 에러 처리 누락 없음

**가독성**
- [ ] 명확한 변수·함수명
- [ ] 함수가 단일 책임
- [ ] 마법 숫자/문자열 상수화
- [ ] 적절한 주석 (왜? 어떻게? what이 아니라 why)

**구조**
- [ ] 적절한 계층 분리 (View ↔ ViewModel ↔ Service)
- [ ] 의존성 주입
- [ ] 인터페이스에 의존
- [ ] 테스트 가능한 구조

**테스트**
- [ ] 새 로직에 대한 단위 테스트
- [ ] 엣지 케이스 커버
- [ ] 통과율 100%

**리뷰 톤 예시**
- ✅ "이 부분은 옵셔널 체이닝으로 더 간결하게 표현할 수 있을 것 같은데, 어떻게 생각하세요?"
- ❌ "왜 이렇게 짰어요?"

### 🔗 참고 자료
- [Google's Code Review Guidelines](https://google.github.io/eng-practices/review/)

---

> 모든 주제를 마쳤습니다. 인덱스는 [README.md](./README.md)에서 확인하세요.
