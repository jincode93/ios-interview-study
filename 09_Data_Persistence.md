---
layout: default
title: "09. 데이터 영속성"
nav_order: 9
---

# 09. 데이터 영속성 (Data Persistence)

> UserDefaults, Keychain, Core Data, SwiftData, 마이그레이션, 데이터 저장 옵션 비교.

---

## 1. iOS에서 데이터를 영구 저장하는 방법들에는 어떤 것이 있나요? (Lv0, Lv1)

### 💬 면접 답변

iOS는 데이터 성격에 따라 여러 저장소를 제공합니다. **UserDefaults**는 키-값 형태의 단순한 설정 저장용으로, 다크 모드 여부 같은 작은 환경 값에 적합합니다. **Keychain**은 비밀번호·토큰 같은 민감한 정보를 암호화해 안전하게 보관합니다. **파일 시스템**(Documents, Library, Caches)은 일반 파일을, **Core Data / SwiftData**는 객체 그래프와 관계를, **SQLite**는 직접 SQL을 다룰 때 씁니다. 선택은 데이터의 크기·구조·민감도에 따라 결정되며, 잘못된 곳에 잘못된 데이터를 넣으면 보안 문제(UserDefaults에 토큰 저장)나 성능 문제(Documents에 캐시 저장)를 일으킵니다.

### 📚 보충 설명

**선택 가이드**

| 데이터 종류 | 추천 저장소 |
|-----------|-----------|
| 사용자 설정, 환경 값 (key-value) | UserDefaults |
| 비밀번호, 인증 토큰, API 키 | Keychain |
| 사용자가 만든 파일 (백업 ⭕) | ~/Documents |
| 앱이 만든 데이터 (백업 ⭕) | ~/Library/Application Support |
| 캐시 (백업 ❌, 시스템이 정리 가능) | ~/Library/Caches |
| 임시 파일 | NSTemporaryDirectory() / tmp |
| 객체 그래프, 관계, 쿼리 | Core Data / SwiftData |
| 복잡한 SQL이 필요한 경우 | SQLite, GRDB |

**파일 위치 가져오기**

```swift
let docs = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask).first!
let support = FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask).first!
let caches = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask).first!
```

**iCloud 백업 제외**

```swift
var url = caches.appendingPathComponent("big_cache.dat")
var values = URLResourceValues()
values.isExcludedFromBackup = true
try url.setResourceValues(values)
```

### 🔗 참고 자료
- [Apple - File System Programming Guide](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/Introduction/Introduction.html)

---

## 2. UserDefaults 사용 시 주의할 점은? (Lv1)

### 💬 면접 답변

UserDefaults는 단순하지만 **모든 데이터가 plist로 평문 저장**되기 때문에 비밀번호·토큰처럼 민감한 정보는 절대 넣으면 안 됩니다. 그건 Keychain의 영역입니다. 또 큰 데이터를 자주 쓰면 성능 문제가 생기는데, UserDefaults는 메모리에 캐시하고 OS가 적절한 시점에 디스크로 동기화하는 모델이라 작은 데이터에 최적화되어 있습니다. 이미지나 큰 JSON을 넣지 말고, 파일이나 데이터베이스를 쓰세요. 마지막으로 `synchronize()`는 iOS 12 이후 사실상 no-op이라 직접 호출할 필요가 없습니다.

### 📚 보충 설명

**기본 사용**

```swift
UserDefaults.standard.set(true, forKey: "isDarkMode")
let dark = UserDefaults.standard.bool(forKey: "isDarkMode")

// Codable도 가능 (encoding 거쳐서)
struct Settings: Codable { let theme: String }
let data = try JSONEncoder().encode(Settings(theme: "dark"))
UserDefaults.standard.set(data, forKey: "settings")
```

**Property Wrapper로 깔끔하게**

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T

    var wrappedValue: T {
        get { (UserDefaults.standard.object(forKey: key) as? T) ?? defaultValue }
        set { UserDefaults.standard.set(newValue, forKey: key) }
    }
}

struct Preferences {
    @UserDefault(key: "isDarkMode", defaultValue: false) static var isDarkMode: Bool
}
```

SwiftUI에는 이미 `@AppStorage`가 같은 역할을 합니다.

```swift
@AppStorage("isDarkMode") var isDarkMode: Bool = false
```

**저장 가능한 타입**
- Property List 타입(String, Number, Date, Data, Array, Dictionary, Bool)만 직접 저장 가능
- 다른 타입은 Data로 인코딩해서 저장

### 🔗 참고 자료
- [Apple - UserDefaults](https://developer.apple.com/documentation/foundation/userdefaults)
- [Apple - AppStorage](https://developer.apple.com/documentation/swiftui/appstorage)

---

## 3. Keychain은 어떤 데이터를 저장하기에 적합한가요? (Lv1, Lv3)

### 💬 면접 답변

Keychain은 **민감한 데이터를 시스템이 암호화해서 보관**해주는 보안 저장소입니다. 비밀번호, 인증 토큰, API 키, 개인 식별 정보처럼 유출되면 안 되는 작은 데이터에 적합합니다. iCloud Keychain으로 다른 기기와 동기화도 가능하고, 앱이 삭제되어도 데이터가 유지되도록 설정할 수 있어 토큰 보관에 알맞습니다. 또 접근 제어(`SecAccessControl`)로 Face ID·Touch ID·passcode 인증을 통과해야만 데이터를 꺼낼 수 있게 할 수도 있습니다. Keychain은 C 기반 API라 사용이 다소 번거롭기 때문에 KeychainAccess 같은 래퍼 라이브러리를 많이 씁니다.

### 📚 보충 설명

**기본 사용 (Security 프레임워크)**

```swift
import Security

// 저장
func save(_ data: Data, account: String) {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecValueData as String: data
    ]
    SecItemDelete(query as CFDictionary)         // 기존 삭제 후
    SecItemAdd(query as CFDictionary, nil)
}

// 읽기
func load(account: String) -> Data? {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne
    ]
    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    return status == errSecSuccess ? result as? Data : nil
}
```

**Access Control (생체 인증 게이트)**

```swift
let access = SecAccessControlCreateWithFlags(
    nil,
    kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
    .biometryCurrentSet,           // 현재 등록된 생체만 허용
    nil
)
// query에 kSecAttrAccessControl 추가
```

**`kSecAttrAccessible` 옵션**

| 옵션 | 의미 |
|------|------|
| `WhenUnlocked` | 기기가 잠금 해제된 상태에서만 |
| `WhenUnlockedThisDeviceOnly` | 위 + iCloud 백업 제외 |
| `AfterFirstUnlock` | 부팅 후 한 번 잠금 해제 이후 |
| `WhenPasscodeSetThisDeviceOnly` | passcode 설정된 기기에서만 |

**Keychain Access Groups (앱 간 공유)**

같은 팀의 앱끼리 Keychain을 공유하려면 Entitlements에 `keychain-access-groups` 추가:

```xml
<key>keychain-access-groups</key>
<array>
    <string>$(AppIdentifierPrefix)com.company.shared</string>
</array>
```

### 🔗 참고 자료
- [Apple - Keychain Services](https://developer.apple.com/documentation/security/keychain_services)
- [Apple - Storing Keys in the Secure Enclave](https://developer.apple.com/documentation/security/certificate_key_and_trust_services/keys/storing_keys_in_the_secure_enclave)

---

## 4. Core Data와 SQLite의 차이는? 언제 어느 것을 쓰나요? (Lv1)

### 💬 면접 답변

SQLite는 **C 기반의 관계형 데이터베이스 엔진**으로, SQL을 직접 다루며 가볍고 유연하지만 모든 객체 매핑·관계 관리·동시성 처리를 직접 구현해야 합니다. Core Data는 그 위에서 동작하는 **객체-그래프 관리 프레임워크**로, 엔티티(클래스)와 관계(릴레이션)를 모델 에디터로 정의하면 자동으로 객체 생성·관계 관리·undo·변경 추적·CloudKit 동기화까지 처리해줍니다. 객체 그래프와 관계가 중요한 앱에는 Core Data, 단순한 테이블 위주거나 SQL 자체가 필요한 경우엔 SQLite(또는 GRDB) — 이런 식으로 선택합니다.

### 📚 보충 설명

**비교 표**

| 항목 | Core Data | SQLite (직접) |
|------|-----------|-------------|
| 추상화 수준 | 높음 (객체) | 낮음 (테이블/행) |
| 관계 관리 | 자동 | 수동 (FK, JOIN) |
| 변경 추적 / undo | 내장 | 직접 구현 |
| iCloud 동기화 | CloudKit 통합 | 직접 구현 |
| 학습 곡선 | 가파름 | 완만 (SQL 알면) |
| 마이그레이션 | 모델 버저닝 + 매핑 모델 | 수동 SQL |

**Core Data 기본 구조**

```
NSPersistentContainer
  └─ NSManagedObjectModel  (스키마)
  └─ NSPersistentStoreCoordinator
       └─ NSPersistentStore  (SQLite 파일)
  └─ NSManagedObjectContext  (작업용 컨텍스트)
       └─ NSManagedObject  (엔티티 인스턴스)
```

**GRDB (요즘 SQLite 래퍼 권장)**

[GRDB.swift](https://github.com/groue/GRDB.swift)는 Swift다운 인터페이스와 동시성·Codable 통합을 제공하는 인기 라이브러리. Core Data가 부담스럽고 SQL의 표현력이 필요할 때 좋은 선택.

### 🔗 참고 자료
- [Apple - Core Data](https://developer.apple.com/documentation/coredata)
- [GRDB GitHub](https://github.com/groue/GRDB.swift)

---

## 5. Core Data의 NSManagedObjectContext의 종류와 동시성 모델은? (Lv2)

### 💬 면접 답변

Core Data는 컨텍스트별로 동시성 타입을 가지는데, **mainQueueConcurrencyType**은 메인 스레드에서만 접근하는 컨텍스트로 UI 갱신과 직접 연결되는 객체에 쓰고, **privateQueueConcurrencyType**은 자체 백그라운드 큐에서 동작해 무거운 import·동기화에 씁니다. 컨텍스트 안의 객체는 그 컨텍스트의 큐에서만 접근해야 하고, 다른 컨텍스트로 객체를 넘기려면 `objectID`로 다시 조회하거나 `perform`/`performAndWait`로 안전하게 실행해야 합니다. 보통 메인 컨텍스트는 UI용, 백그라운드 컨텍스트는 쓰기용으로 분리하고, 백그라운드에서 저장하면 메인 컨텍스트가 변경을 자동 머지하도록 설정합니다.

### 📚 보충 설명

**컨테이너 설정**

```swift
let container = NSPersistentContainer(name: "Model")
container.loadPersistentStores { _, error in /* ... */ }

// 메인 컨텍스트가 백그라운드 변경을 자동으로 머지하도록
container.viewContext.automaticallyMergesChangesFromParent = true

// 충돌 정책
container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
```

**백그라운드에서 쓰기**

```swift
container.performBackgroundTask { ctx in
    let user = User(context: ctx)
    user.name = "철수"
    try? ctx.save()    // 자동으로 viewContext에 머지
}
```

**Child Context**

UI에서 편집 중인 임시 변경을 분리하고 싶을 때:

```swift
let child = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
child.parent = container.viewContext
// child에서 변경 → child.save() → parent로 push, 아직 디스크엔 미저장
// parent(.viewContext).save() 해야 실제 디스크에 저장
```

**Concurrency 함정**
- ManagedObject를 다른 큐로 가져가면 즉시 크래시 또는 corruption
- `perform`/`performAndWait`로 항상 컨텍스트의 큐에서 실행
- 객체 식별은 `objectID`로

### 🔗 참고 자료
- [Apple - Using Core Data in the Background](https://developer.apple.com/documentation/coredata/using_core_data_in_the_background)
- [Apple - NSManagedObjectContext](https://developer.apple.com/documentation/coredata/nsmanagedobjectcontext)

---

## 6. Core Data 스키마 변경으로 인한 사용자 데이터 손실을 어떻게 방지하나요? (Lv3)

### 💬 면접 답변

스키마 변경 시에는 항상 **모델 버저닝**을 만들고, 가능한 한 **경량 마이그레이션(Lightweight Migration)** 이 적용되도록 변경을 단순하게 유지합니다. 컬럼 추가, 옵셔널로 변경, 단순 rename(renamingIdentifier 활용) 같은 변경은 자동으로 마이그레이션됩니다. 변경이 더 복잡하면 **매핑 모델(.xcmappingmodel)** 을 직접 만들어 엔티티/속성 간 변환 규칙을 명시하고, 그래도 부족하면 NSEntityMigrationPolicy를 서브클래싱해 코드로 변환합니다. 마이그레이션은 잘못되면 사용자 데이터 손실로 직결되므로 **실사용 데이터 시드를 가지고 시뮬레이션** 테스트를 거치고, 큰 변경은 여러 버전을 거치는 **단계적 마이그레이션**으로 안전을 확보합니다.

### 📚 보충 설명

**경량 마이그레이션 활성화**

```swift
let container = NSPersistentContainer(name: "Model")
let description = container.persistentStoreDescriptions.first!
description.shouldInferMappingModelAutomatically = true
description.shouldMigrateStoreAutomatically = true
container.loadPersistentStores { ... }
```

**언제 경량으로 충분한가**
- 새 attribute/relationship 추가 (옵셔널 또는 기본값 있음)
- 단순 이름 변경 (Model Editor에서 renamingIdentifier 명시)
- 일대일에서 다대다로 변경 같은 간단한 카디널리티 변경

**언제 매핑 모델이 필요한가**
- 엔티티 분할/병합
- 속성 타입 변경 (Int → String 등)
- 복잡한 데이터 변환

**Heavy Migration 예**

```swift
// EntityMigrationPolicy 서브클래스
class UserMigrationPolicy: NSEntityMigrationPolicy {
    override func createDestinationInstances(forSource source: NSManagedObject,
                                              in mapping: NSEntityMapping,
                                              manager: NSMigrationManager) throws {
        try super.createDestinationInstances(forSource: source, in: mapping, manager: manager)
        // 추가 변환 로직
    }
}
```

**단계적 마이그레이션**

iOS 17+ 에서 NSStagedMigrationManager로 더 안전한 단계별 마이그레이션 지원.

```swift
let stages: [NSCustomMigrationStage] = [
    NSCustomMigrationStage(migratingFrom: .v1, to: .v2) { /* ... */ },
    NSCustomMigrationStage(migratingFrom: .v2, to: .v3) { /* ... */ }
]
let manager = NSStagedMigrationManager(stages)
description.setOption(manager, forKey: NSPersistentStoreStagedMigrationManagerOptionKey)
```

### 🔗 참고 자료
- [Apple - Core Data Model Versioning and Data Migration](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreDataVersioning/Articles/Introduction.html)
- [WWDC 2022 - Evolve your Core Data schema](https://developer.apple.com/videos/play/wwdc2022/10120/)

---

## 7. SwiftData를 Core Data 대신 쓰기로 결정한 기준은? (Lv3)

### 💬 면접 답변

SwiftData는 iOS 17에서 도입된 Apple의 새 영속성 프레임워크로, Core Data의 후속에 가깝습니다. 가장 큰 장점은 **모델을 Swift 매크로(`@Model`)로 정의**해서 별도 모델 에디터(.xcdatamodeld)와 코드 생성 없이 순수 Swift 코드로 스키마를 표현한다는 점입니다. SwiftUI와 깊게 통합되어 `@Query`로 선언적 쿼리, `@Environment(\.modelContext)`로 컨텍스트에 접근하는 등 보일러플레이트가 크게 줄어듭니다. 다만 아직 iOS 17+ 전용이고, 복잡한 마이그레이션·세밀한 페치 최적화·기존 Core Data 자산이 많은 프로젝트에서는 Core Data가 여전히 안정적인 선택입니다. 새 프로젝트이거나 SwiftUI 중심이라면 SwiftData를, 레거시·복잡 모델이면 Core Data를 권합니다.

### 📚 보충 설명

**SwiftData 기본**

```swift
import SwiftData

@Model
final class Book {
    var title: String
    var author: String
    var publishedDate: Date

    init(title: String, author: String, publishedDate: Date) {
        self.title = title
        self.author = author
        self.publishedDate = publishedDate
    }
}

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup { ContentView() }
        .modelContainer(for: Book.self)
    }
}

struct ContentView: View {
    @Environment(\.modelContext) var ctx
    @Query(sort: \Book.publishedDate, order: .reverse) var books: [Book]

    var body: some View {
        List(books) { Text($0.title) }
            .toolbar {
                Button("추가") {
                    ctx.insert(Book(title: "...", author: "...", publishedDate: .now))
                }
            }
    }
}
```

**SwiftData vs Core Data 빠른 비교**

| 항목 | SwiftData | Core Data |
|------|-----------|-----------|
| 모델 정의 | Swift 매크로 | xcdatamodeld |
| SwiftUI 통합 | 매우 깊음 | 어댑터 필요 |
| 최소 버전 | iOS 17+ | iOS 3+ |
| 마이그레이션 | VersionedSchema | xcmappingmodel |
| CloudKit | 자동 | 명시적 설정 |
| 성숙도 | 빠르게 발전 중 | 매우 안정 |

**CloudKit 통합**

```swift
.modelContainer(for: Book.self, isAutosaveEnabled: true)
```

iCloud 자동 동기화는 `@Model`에서 모든 프로퍼티가 옵셔널이거나 기본값을 가져야 하는 등 제약이 있습니다.

### 🔗 참고 자료
- [Apple - SwiftData](https://developer.apple.com/documentation/swiftdata)
- [WWDC 2023 - Meet SwiftData](https://developer.apple.com/videos/play/wwdc2023/10187/)
- [WWDC 2023 - Migrate to SwiftData](https://developer.apple.com/videos/play/wwdc2023/10189/)

---

## 8. 관계형 DB와 NoSQL의 차이는? iOS에서 NoSQL은 어떻게 쓰나요? (Lv0)

### 💬 면접 답변

관계형 DB는 **고정된 스키마**와 **정규화된 테이블 간 관계**를 가지고 SQL로 질의합니다. 강한 일관성과 트랜잭션 보장이 장점입니다. NoSQL은 스키마가 유연하고 키-값, 문서, 컬럼, 그래프 같은 다양한 모델을 제공해 수평 확장과 비정형 데이터에 강합니다. iOS에서는 **Realm**이 객체 그래프형 모바일 DB로 인기 있고, **Firebase Firestore**나 **CoreData with CloudKit**으로 클라우드 NoSQL을 백엔드로 쓰는 경우도 많습니다. 클라이언트 단에서는 사실 SQLite 기반 Core Data가 대부분의 용도를 커버하므로, NoSQL을 별도로 도입할지는 동기화·오프라인·서버 모델과의 정합성을 기준으로 판단합니다.

### 📚 보충 설명

**Realm 예시**

```swift
import RealmSwift

class Book: Object {
    @Persisted var title: String
    @Persisted var author: String
}

let realm = try Realm()
try realm.write {
    realm.add(Book(value: ["title": "...", "author": "..."]))
}
let books = realm.objects(Book.self)
```

특징:
- 객체 기반 (NoSQL 객체 DB)
- 라이브 결과 + 자동 UI 업데이트
- 멀티 스레드 동시 읽기 가능
- 큰 데이터셋·복잡 쿼리에서도 성능 우수

### 🔗 참고 자료
- [Realm Swift](https://www.mongodb.com/docs/realm/sdk/swift/)
- [Firebase Firestore for iOS](https://firebase.google.com/docs/firestore/quickstart)

---

## 9. 직렬화·역직렬화에서 Codable의 한계는? 언제 다른 형식을 쓰나요? (Lv2)

### 💬 면접 답변

Codable은 JSON·Plist 같은 표준 포맷에서 매우 편리하지만, 다음 경우엔 한계가 있습니다. 첫째, **다형성 처리**가 까다롭습니다 — 같은 키지만 타입이 다른 응답을 표현하려면 커스텀 init(from:)을 직접 작성해야 합니다. 둘째, **성능이 critical한 곳**에서는 JSON 자체가 텍스트라 파싱 비용이 큰데, Protocol Buffers·FlatBuffers·MessagePack 같은 바이너리 포맷이 훨씬 빠르고 작습니다. 셋째, **스키마 진화**가 잦은 시스템에서는 .proto 파일 같은 명시적 스키마와 코드 생성이 더 안전합니다. 보통의 모바일 앱은 JSON + Codable로 충분하지만, 실시간 게임·대용량 메시지·서버 사이드 Swift에서는 다른 형식을 검토합니다.

### 📚 보충 설명

**다형성 처리 예**

```swift
enum Notification: Decodable {
    case message(String)
    case image(URL)

    enum CodingKeys: String, CodingKey { case type, value }

    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        let type = try c.decode(String.self, forKey: .type)
        switch type {
        case "message":
            self = .message(try c.decode(String.self, forKey: .value))
        case "image":
            self = .image(try c.decode(URL.self, forKey: .value))
        default:
            throw DecodingError.dataCorrupted(.init(codingPath: [], debugDescription: "unknown"))
        }
    }
}
```

**대안 포맷**

| 포맷 | 특징 | 용도 |
|------|------|------|
| JSON | 텍스트, 가독성 ⭕ | 일반 API |
| Plist | XML 또는 binary, Apple 전용 | 설정, 시스템 |
| Protocol Buffers | 바이너리, 스키마 명시, 작고 빠름 | gRPC, 실시간 |
| MessagePack | JSON과 유사하지만 바이너리 | JSON 대체 |
| FlatBuffers | 메모리 매핑 가능, 파싱 0 비용 | 게임, IoT |

### 🔗 참고 자료
- [Apple - Encoding and Decoding Custom Types](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types)
- [swift-protobuf](https://github.com/apple/swift-protobuf)

---

## 10. URLCache, NSCache의 차이와 사용 기준은? (Lv2)

### 💬 면접 답변

`URLCache`는 URLSession의 HTTP 응답 전용 캐시로 HTTP 헤더의 Cache-Control·ETag 같은 표준을 따라 자동 동작합니다. 메모리와 디스크 양쪽을 사용하고, 같은 URL의 응답을 재사용해 트래픽을 줄입니다. `NSCache`는 임의의 객체에 대한 **메모리 캐시**로, 키-값 형태이며 thread-safe하고 메모리 압력 시 자동 정리됩니다. URL 응답 캐싱은 URLCache, 디코딩된 이미지나 파싱된 객체를 보관하는 건 NSCache, 디스크에도 유지하고 싶다면 직접 LRU + 파일 시스템 구현 또는 라이브러리 활용 — 이 정도 구분이 표준입니다.

### 📚 보충 설명

**NSCache 특징**
- key는 NSObject 호환만 가능 (`NSString`, `NSNumber` 등). 커스텀 키는 NSObject 서브클래스로
- `setObject(_:forKey:cost:)`로 비용 가중치 부여 → totalCostLimit과 함께 사용
- 메모리 압력 시 시스템이 자동 정리

```swift
let cache = NSCache<NSURL, UIImage>()
cache.countLimit = 100
cache.totalCostLimit = 50 * 1024 * 1024   // 50MB

cache.setObject(image, forKey: url as NSURL, cost: imageBytes)
let cached = cache.object(forKey: url as NSURL)
```

**Dictionary 대신 NSCache를 쓰는 이유**
- 메모리 압력 대응
- thread-safe (Dictionary는 동시 변경 시 crash 가능)

### 🔗 참고 자료
- [Apple - NSCache](https://developer.apple.com/documentation/foundation/nscache)
- [Apple - URLCache](https://developer.apple.com/documentation/foundation/urlcache)

---

> 보안 관련 저장은 [01_CS_Fundamentals.md](./01_CS_Fundamentals.md)와 [08_Networking.md](./08_Networking.md)에서도 함께 다룹니다.
