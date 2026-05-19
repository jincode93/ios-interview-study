---
layout: home
title: 홈
nav_order: 0
---

# iOS 면접 준비 - 질문 & 답변 모음

> 출처: [JeaSungLEE/iOSInterviewquestions](https://github.com/JeaSungLEE/iOSInterviewquestions)
> 답변은 실제 면접에서 말하듯 작성된 핵심 답변과, 그 아래의 보충 설명으로 구성되어 있습니다.

---

## 학습 가이드

- **현재 레벨**: 2년차 iOS 개발자 (저장소 기준 레벨 2)
- **목표 레벨**: 3년차 이상 이직 (레벨 3)
- **학습 우선순위**: 레벨 1~3 → 레벨 0 (기초 다지기) → 레벨 4~5 (참고)

### 각 문항 구성

| 구성 요소 | 설명 |
|----------|------|
| 💬 면접 답변 | 실제 면접에서 답변하듯 핵심을 3~6문장으로 |
| 📚 보충 설명 | 원리, 내부 동작, 예제 코드, 흔한 함정 등 깊이 있는 내용 |
| 🔗 참고 자료 | Apple 공식 문서 / Swift.org / WWDC 세션 링크 |

---

## 2년차 면접 핵심 체크리스트

### 반드시 막힘 없이 설명해야 하는 것
- ARC와 순환 참조 (`weak` vs `unowned` 선택 기준)
- 값 타입 vs 참조 타입 (Copy-on-Write 포함)
- GCD vs OperationQueue vs async/await
- `escaping` 클로저와 캡처 리스트
- UITableView 셀 재사용 메커니즘
- SwiftUI 상태 관리 (`@State`, `@Binding`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`)
- MVC vs MVVM 차이와 SwiftUI에서의 적용
- HTTP/HTTPS, REST, `URLSession` 기본 흐름
- 옵셔널 처리 패턴 (`if let`, `guard let`, `??`, `Optional Chaining`)
- 제네릭과 프로토콜의 차이

### 차별화를 위해 깊이 있게 알아두면 좋은 것
- Swift Concurrency (`actor`, `Sendable`, `MainActor`)
- Combine 핵심 (`Publisher`/`Subscriber`, `@Published`, 백프레셔)
- iOS 17+ `@Observable` 매크로
- SwiftData vs Core Data
- 메모리 그래프 디버거와 Instruments 활용
- Clean Architecture / Coordinator 패턴
