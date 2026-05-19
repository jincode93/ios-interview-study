---
layout: default
title: "08. 네트워킹"
nav_order: 8
---

# 08. 네트워킹

> HTTP/HTTPS/TLS, REST API, URLSession, BLE/NFC, 푸시 알림, Deep Link/Universal Link.

---

## 1. HTTP와 HTTPS의 차이는? iOS의 ATS는 무엇인가요? (Lv0)

### 💬 면접 답변

HTTP는 평문 통신, HTTPS는 HTTP를 **TLS**로 암호화한 통신입니다. TLS가 보장하는 건 세 가지로, 통신 내용을 가로채도 볼 수 없게 하는 **기밀성**, 도중에 변조되었는지 검증할 수 있는 **무결성**, 그리고 서버가 진짜 그 서버임을 확인하는 **인증**입니다. iOS에는 **ATS(App Transport Security)** 가 있어서 기본적으로 모든 외부 통신을 HTTPS로 강제하고, HTTP를 쓰려면 Info.plist에 명시적인 예외를 등록해야 합니다. 그것도 새 앱 심사에서는 정당한 사유 없이는 거의 통과되지 않습니다.

### 📚 보충 설명

**TLS Handshake 흐름 (간략)**

1. ClientHello: 클라이언트가 지원하는 cipher suite, 랜덤 값 전송
2. ServerHello + Certificate: 서버가 선택한 cipher, 자신의 인증서(공개키 포함) 전송
3. 키 교환: ECDHE 등으로 세션 키 교환 (전방 비밀성)
4. Finished: 양쪽이 같은 세션 키를 보유. 이후 대칭키로 빠르게 암호화 통신

**인증서 검증**

- 서버 인증서가 신뢰할 수 있는 CA에 의해 서명되었는지
- 인증서 도메인이 요청 호스트와 일치하는지
- 만료되지 않았는지
- 폐기되지 않았는지(OCSP, CRL)

**ATS 예외 (Info.plist)**

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
        </dict>
    </dict>
</dict>
```

App Store 심사 시 합당한 사유 필요. 개인 정보 송수신이 있다면 거의 불가능.

**Certificate Pinning**

신뢰하는 CA를 거치는 것 외에 특정 인증서/공개키만 허용 → MITM 방어 강화.

```swift
final class PinningDelegate: NSObject, URLSessionDelegate {
    let expectedHash: String   // SPKI(공개키)의 SHA-256

    func urlSession(_ s: URLSession,
                    didReceive challenge: URLAuthenticationChallenge,
                    completionHandler: @escaping (URLSession.AuthChallengeDisposition,
                                                  URLCredential?) -> Void) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let trust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil); return
        }
        // trust 안의 인증서를 꺼내 SHA-256 비교
        // 일치 시 useCredential, 아니면 cancel
    }
}
```

### 🔗 참고 자료
- [Apple - App Transport Security](https://developer.apple.com/documentation/security/preventing_insecure_network_connections)
- [Apple - Performing Manual Server Trust Authentication](https://developer.apple.com/documentation/foundation/url_loading_system/handling_an_authentication_challenge/performing_manual_server_trust_authentication)

---

## 2. OSI 7계층 중 iOS의 URLSession 호출에 관여하는 계층은? (Lv0)

### 💬 면접 답변

URLSession이 HTTP/HTTPS 요청을 보낼 때 OSI 모델 기준으로는 **Application(7)** 에서 HTTP 요청을 만들고, **Presentation(6)** 에서 TLS가 암복호화를, **Transport(4)** 에서 TCP가 신뢰성 있는 연결을, **Network(3)** 에서 IP가 패킷 라우팅을 담당합니다. Data Link(2)·Physical(1)은 OS와 하드웨어가 처리하므로 앱 개발자가 직접 다룰 일은 거의 없습니다. 실제 코드에서 우리가 다루는 건 7계층 메시지(URL, 헤더, body)와 그 결과뿐이고, 나머지는 시스템이 추상화해줍니다.

### 📚 보충 설명

**iOS의 네트워크 스택**

```
URLSession / Network.framework
    ↓
HTTP/2 or HTTP/3 (h2, h3)
    ↓
TLS 1.2 / 1.3
    ↓
TCP / QUIC(UDP)
    ↓
IP
```

iOS 12+의 **Network.framework**(`NWConnection`)는 더 저수준에서 직접 TCP/UDP/TLS를 다룰 수 있게 해줍니다. URLSession은 그 위에서 HTTP를 처리합니다.

**Keep-Alive와 연결 재사용**

같은 호스트로 가는 요청들은 URLSession이 내부적으로 연결을 재사용해서 매번 TCP/TLS handshake 비용을 치르지 않습니다. `URLSessionConfiguration.httpMaximumConnectionsPerHost`로 호스트당 최대 동시 연결 수 조정 가능.

### 🔗 참고 자료
- [Apple - URL Loading System](https://developer.apple.com/documentation/foundation/url_loading_system)
- [Apple - Network.framework](https://developer.apple.com/documentation/network)

---

## 3. HTTP/1.1, HTTP/2, HTTP/3의 차이는? (Lv0)

### 💬 면접 답변

HTTP/1.1은 **요청-응답을 시리얼**로 처리해 같은 연결에서 다음 요청은 이전이 끝나야 보낼 수 있고, 이로 인한 head-of-line blocking을 피하려고 여러 TCP 연결을 동시에 열었습니다. HTTP/2는 **멀티플렉싱**을 도입해 하나의 TCP 연결에서 여러 요청·응답을 스트림으로 동시에 주고받고, 헤더 압축(HPACK)도 더해 효율을 크게 높였습니다. 다만 HTTP/2도 TCP 위에서 동작하므로 패킷 손실 시 TCP 레벨의 HOL blocking이 남아있습니다. HTTP/3는 TCP 대신 **UDP 기반의 QUIC**을 사용해 이 문제까지 해결하고 연결 재개·이동 시 핸드셰이크 비용도 줄였습니다. iOS는 HTTP/2를 오랫동안 지원했고 iOS 15+ 부터 HTTP/3를 URLSession에서 자동 사용합니다.

### 📚 보충 설명

**HTTP/2 주요 특징**
- **Multiplexing**: 하나의 연결에서 여러 스트림
- **Header Compression (HPACK)**: 헤더 중복 제거
- **Server Push**: 서버가 클라이언트 요청 전에 리소스 미리 전송 (이후 deprecated)
- **Binary Framing**: 텍스트 → 바이너리

**HTTP/3 (QUIC)**
- TCP 대신 UDP + QUIC
- TLS 1.3 통합 핸드셰이크 → 0-RTT 가능
- 연결 식별이 IP가 아닌 Connection ID → Wi-Fi ↔ 셀룰러 전환 시 연결 유지
- 스트림별 독립 → 한 스트림의 패킷 손실이 다른 스트림에 영향 ❌

**URLSession에서 HTTP/3 활용 확인**

iOS 15+에서는 서버가 HTTP/3를 지원하면 자동으로 사용합니다(QUIC + Alt-Svc 헤더). 클라이언트가 강제로 끄거나 켜는 옵션은 일반적으로 노출되지 않습니다.

### 🔗 참고 자료
- [Apple - Accelerating HTTP/3 deployment](https://developer.apple.com/news/?id=4eo38vqi)
- [RFC 9000 - QUIC](https://www.rfc-editor.org/rfc/rfc9000)

---

## 4. TCP와 UDP의 차이는? 화상통화 앱이라면 어느 쪽을 쓰나요? (Lv0)

### 💬 면접 답변

TCP는 **연결 지향 + 신뢰성**을 제공하는 프로토콜로, 핸드셰이크로 연결을 만들고 손실·순서 뒤바뀜·중복을 OS가 책임지고 보장합니다. 대신 재전송 등으로 지연이 발생할 수 있습니다. UDP는 **비연결 + 빠른 전송**이 특징으로, 손실 가능성이 있지만 즉시 보낼 수 있어 지연이 거의 없습니다. 화상통화는 **약간의 패킷 손실보다 지연이 훨씬 치명적**입니다. 화면이 잠깐 깨지는 건 사람이 잘 인지 못하지만, 음성이 1초 늦으면 대화가 안 됩니다. 그래서 WebRTC, RTP 같은 실시간 미디어 프로토콜이 UDP 기반이고, iOS의 CallKit·VoIP 푸시도 이와 함께 동작합니다. 신뢰성이 필요한 채팅·시그널링은 별도 TCP 연결로 처리하는 식으로 혼용합니다.

### 📚 보충 설명

**TCP vs UDP 비교**

| 항목 | TCP | UDP |
|------|-----|-----|
| 연결 | 핸드셰이크 필요 | 없음 |
| 신뢰성 | ⭕ 보장 | ❌ best effort |
| 순서 | ⭕ 보장 | ❌ |
| 헤더 | 20+ bytes | 8 bytes |
| 흐름 제어 | ⭕ | ❌ |
| 사용 예 | HTTP, FTP, SSH | DNS, 음성/영상, 게임 |

**iOS의 백그라운드 VoIP**

- **PushKit + CallKit**: VoIP 전용 푸시(`PKPushType.voIP`)는 일반 푸시보다 우선되며, 앱을 깨워서 즉시 CallKit으로 인입 전화를 표시할 수 있음.
- iOS 13+: VoIP 푸시를 받았으면 **반드시 CallKit으로 통화 표시**해야 함 (그렇지 않으면 푸시 권한 박탈).

**Network.framework로 커스텀**

```swift
let connection = NWConnection(
    host: "example.com",
    port: 1234,
    using: .udp           // 또는 .tcp, .tls 등
)
connection.start(queue: .global())
```

### 🔗 참고 자료
- [Apple - Network.framework](https://developer.apple.com/documentation/network)
- [Apple - PushKit](https://developer.apple.com/documentation/pushkit)
- [Apple - CallKit](https://developer.apple.com/documentation/callkit)

---

## 5. REST API의 특징과 URLSession 기본 사용법은? (Lv0)

### 💬 면접 답변

REST는 자원을 URI로 표현하고, HTTP 메서드(GET, POST, PUT, PATCH, DELETE)로 자원에 대한 행위를 표현하는 아키텍처 스타일입니다. 무상태(stateless), 클라이언트-서버 분리, 캐시 가능, 계층 구조 같은 제약을 따릅니다. iOS에서는 URLSession이 기본 도구이고, async/await가 도입된 iOS 15+ 이후로는 `try await URLSession.shared.data(from:)`이 가장 깔끔한 표준 패턴입니다. 응답 JSON은 Codable과 JSONDecoder로 모델로 변환하고, 4xx/5xx 같은 상태 코드는 명시적으로 분기 처리합니다.

### 📚 보충 설명

**기본 GET (async/await)**

```swift
struct User: Decodable { let id: Int; let name: String }

func fetchUser(id: Int) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, response) = try await URLSession.shared.data(from: url)
    guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
        throw URLError(.badServerResponse)
    }
    return try JSONDecoder().decode(User.self, from: data)
}
```

**POST with body**

```swift
var req = URLRequest(url: URL(string: ".../users")!)
req.httpMethod = "POST"
req.setValue("application/json", forHTTPHeaderField: "Content-Type")
req.httpBody = try JSONEncoder().encode(newUser)

let (data, _) = try await URLSession.shared.data(for: req)
```

**HTTP 메서드 의미**

| 메서드 | 멱등성 | 안전성 | 용도 |
|--------|--------|--------|------|
| GET | ⭕ | ⭕ | 조회 |
| POST | ❌ | ❌ | 생성, 비-멱등 작업 |
| PUT | ⭕ | ❌ | 전체 교체/생성 |
| PATCH | ❌ | ❌ | 부분 수정 |
| DELETE | ⭕ | ❌ | 삭제 |

> 멱등(Idempotent): 같은 요청을 N번 보내도 결과가 같음.

**상태 코드 분류**

- 1xx Informational
- 2xx Success
- 3xx Redirection
- 4xx Client error (잘못된 요청, 인증 실패)
- 5xx Server error

**4xx vs 5xx 대응**
- 4xx: 클라이언트가 고쳐야 할 문제 → 사용자에게 알림, 재시도 의미 없음
- 5xx: 서버 문제 → 일정 시간 후 재시도, 지수 백오프

### 🔗 참고 자료
- [Apple - URLSession](https://developer.apple.com/documentation/foundation/urlsession)
- [Apple - URLSession async](https://developer.apple.com/documentation/foundation/urlsession/3767349-data)

---

## 6. URLSession에서 재시도 로직과 백오프는 어떻게 구현하나요? (Lv1)

### 💬 면접 답변

재시도는 **에러 종류**에 따라 다르게 대응해야 합니다. 타임아웃·일시적 5xx는 재시도하고, 인증 실패(401)·잘못된 요청(400)은 재시도해도 같은 결과라 의미가 없습니다. 재시도 간격은 **지수 백오프 + 지터**(랜덤 변동)로 두는 게 표준인데, 모든 클라이언트가 같은 간격으로 재시도하면 서버가 회복할 때 한꺼번에 몰리는 thundering herd가 생기기 때문입니다. 또 무한 재시도는 안 되고 최대 횟수(보통 3~5회)와 누적 시간 한계를 두어야 하며, 사용자 입장에서 너무 오래 기다리지 않도록 UX와 균형을 맞춥니다.

### 📚 보충 설명

```swift
func fetchWithRetry<T: Decodable>(_ url: URL,
                                  type: T.Type,
                                  maxAttempts: Int = 3) async throws -> T {
    for attempt in 0..<maxAttempts {
        do {
            let (data, response) = try await URLSession.shared.data(from: url)
            guard let http = response as? HTTPURLResponse else {
                throw URLError(.badServerResponse)
            }
            switch http.statusCode {
            case 200..<300:
                return try JSONDecoder().decode(T.self, from: data)
            case 400..<500:
                // 4xx는 재시도 무의미
                throw URLError(.cannotConnectToHost)
            case 500..<600:
                // 5xx만 재시도
                break
            default:
                throw URLError(.badServerResponse)
            }
        } catch {
            // 마지막 시도면 throw
            if attempt == maxAttempts - 1 { throw error }
        }
        // 지수 백오프 + 지터
        let base = pow(2.0, Double(attempt))         // 1, 2, 4, 8 ...
        let jitter = Double.random(in: 0...0.5)
        try await Task.sleep(nanoseconds: UInt64((base + jitter) * 1_000_000_000))
    }
    throw URLError(.timedOut)
}
```

**Background URLSession**

큰 파일을 백그라운드에서 다운로드/업로드해야 하면 background config를 씁니다.

```swift
let config = URLSessionConfiguration.background(withIdentifier: "com.app.bg")
config.isDiscretionary = true        // 시스템이 적절한 시점 결정
let session = URLSession(configuration: config, delegate: delegate, delegateQueue: nil)
```

제약:
- delegate 기반만 가능 (completion handler ❌)
- 시뮬레이터에서 일부 동작 다름
- 앱이 종료되어도 진행 → 완료 시 시스템이 앱 깨우고 `handleEventsForBackgroundURLSession` 호출

### 🔗 참고 자료
- [Apple - Downloading Files in the Background](https://developer.apple.com/documentation/foundation/url_loading_system/downloading_files_in_the_background)
- [AWS Architecture Blog - Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

---

## 7. URLSession의 응답 캐싱은 어떻게 동작하나요? (Lv2)

### 💬 면접 답변

URLSession은 `URLCache`라는 메모리·디스크 2단 캐시를 가지고 있어 HTTP 응답을 자동으로 캐싱합니다. 서버가 `Cache-Control`, `Expires`, `ETag` 헤더를 보내면 그에 맞춰 캐시 유효성을 판단하고, 같은 요청이 오면 네트워크를 거치지 않고 캐시에서 응답하거나, 조건부 요청으로 304 Not Modified를 받아 트래픽을 절약합니다. 클라이언트는 `URLRequest.cachePolicy`로 캐시 정책을 세부 조정할 수 있고, 사용자 정의 캐시 사이즈도 설정 가능합니다. 다만 캐시가 모든 응답을 무조건 저장하지 않으며, 응답 크기·헤더·메모리 상황 등의 조건이 맞아야 캐싱됩니다.

### 📚 보충 설명

**캐시 정책**

```swift
let cache = URLCache(memoryCapacity: 10 * 1024 * 1024,   // 10 MB
                    diskCapacity: 100 * 1024 * 1024,     // 100 MB
                    diskPath: "myAppCache")
URLCache.shared = cache

var req = URLRequest(url: url)
req.cachePolicy = .returnCacheDataElseLoad   // 캐시 우선
```

**`URLRequest.CachePolicy` 종류**
- `.useProtocolCachePolicy`: HTTP 표준 (Cache-Control 헤더 따름)
- `.reloadIgnoringLocalCacheData`: 캐시 무시하고 항상 새로 받음
- `.returnCacheDataElseLoad`: 캐시 있으면 사용, 없으면 네트워크
- `.returnCacheDataDontLoad`: 캐시만 사용, 없으면 실패 (오프라인 모드)

**커스텀 응답 캐싱**

```swift
class MySessionDelegate: NSObject, URLSessionDataDelegate {
    func urlSession(_ session: URLSession, dataTask: URLSessionDataTask,
                    willCacheResponse proposedResponse: CachedURLResponse,
                    completionHandler: @escaping (CachedURLResponse?) -> Void) {
        // 응답을 검사해 캐싱 거부/수정 가능
        completionHandler(proposedResponse)
    }
}
```

**캐시되는 조건 (간략)**
- HTTPS면 적절한 헤더 필요
- 응답 크기가 캐시 용량의 일부분 이하
- 메모리/디스크 capacity 충분
- request method가 GET (일반적으로)

### 🔗 참고 자료
- [Apple - URLCache](https://developer.apple.com/documentation/foundation/urlcache)
- [Apple - Accessing Cached Data](https://developer.apple.com/documentation/foundation/url_loading_system/accessing_cached_data)

---

## 8. iOS의 푸시 알림(Local vs Remote)은 어떻게 구현하나요? (Lv2)

### 💬 면접 답변

iOS 푸시는 두 종류입니다. **로컬 푸시**는 앱이 직접 만들어 OS의 알림 시스템에 예약하는 것으로, 인터넷 연결 없이도 동작하며 알람·리마인더 같은 시점 기반 알림에 적합합니다. **원격 푸시**는 서버가 Apple의 APNs로 페이로드를 보내고 APNs가 디바이스에 전달하는 방식으로, 디바이스 토큰을 받아 서버에 등록하는 절차가 필요합니다. 둘 다 `UNUserNotificationCenter`로 권한 요청·알림 발행·탭 핸들링을 통일된 방식으로 다루고, 콘텐츠(`UNMutableNotificationContent`)와 트리거(`UNTimeIntervalNotificationTrigger`, `UNCalendarNotificationTrigger` 등)를 조합해 만듭니다.

### 📚 보충 설명

**권한 요청과 로컬 푸시**

```swift
import UserNotifications

UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { ok, err in }

let content = UNMutableNotificationContent()
content.title = "안녕"
content.body = "10초 후에 알림"

let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 10, repeats: false)
let request = UNNotificationRequest(identifier: UUID().uuidString, content: content, trigger: trigger)

UNUserNotificationCenter.current().add(request)
```

**원격 푸시 등록**

```swift
// AppDelegate
func application(_ app: UIApplication, didFinishLaunchingWithOptions: ...) -> Bool {
    UIApplication.shared.registerForRemoteNotifications()
    return true
}

func application(_ app: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken token: Data) {
    let hex = token.map { String(format: "%02x", $0) }.joined()
    // 서버에 hex 토큰 등록
}
```

**알림 탭 처리**

```swift
class NotificationDelegate: NSObject, UNUserNotificationCenterDelegate {
    // 앱 실행 중에도 알림 표시
    func userNotificationCenter(_ center: UNUserNotificationCenter,
        willPresent notification: UNNotification,
        withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void) {
        completionHandler([.banner, .sound])
    }

    // 사용자가 알림을 탭했을 때
    func userNotificationCenter(_ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler completionHandler: @escaping () -> Void) {
        let info = response.notification.request.content.userInfo
        // 딥링크 처리
        completionHandler()
    }
}
```

**Rich Notification, Notification Extension**

- **Service Extension**: 페이로드를 받은 직후 미디어 다운로드 등 가공
- **Content Extension**: 알림을 길게 눌렀을 때 커스텀 UI 표시

### 🔗 참고 자료
- [Apple - User Notifications](https://developer.apple.com/documentation/usernotifications)
- [Apple - Registering Your App with APNs](https://developer.apple.com/documentation/usernotifications/registering_your_app_with_apns)
- [Apple - Setting Up a Remote Notification Server](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server)

---

## 9. Deep Link와 Universal Link의 차이는? (Lv2)

### 💬 면접 답변

Deep Link는 보통 **URL Scheme**(예: `myapp://product/123`)을 등록해 외부에서 해당 스킴 URL을 열면 우리 앱이 실행되는 방식입니다. 구현은 간단하지만, 같은 스킴을 가진 다른 앱이 가로챌 수 있고, 앱이 설치되지 않았을 때 웹으로 폴백할 방법이 없다는 한계가 있습니다. Universal Link는 **HTTPS URL**(예: `https://example.com/product/123`)을 사용하고, Apple의 AASA(`apple-app-site-association`) 파일을 통해 도메인 소유 검증을 합니다. 앱이 설치되어 있으면 앱이 열리고, 없으면 사파리에서 웹페이지가 열려 자연스러운 폴백이 됩니다. 보안과 UX 모두 우수해서 새 앱은 Universal Link를 기본으로 채택하는 게 정석입니다.

### 📚 보충 설명

**URL Scheme 설정**

Info.plist에 `CFBundleURLTypes` 추가:
```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
    </dict>
</array>
```

```swift
// AppDelegate
func application(_ app: UIApplication, open url: URL, options: [...]) -> Bool {
    // url 파싱해서 화면 이동
    return true
}

// SwiftUI
.onOpenURL { url in /* ... */ }
```

**Universal Link 설정 (서버)**

`https://example.com/.well-known/apple-app-site-association`:

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["TEAMID.com.example.app"],
        "components": [{ "/": "/product/*" }]
      }
    ]
  }
}
```

**Universal Link 설정 (앱)**

- Capabilities → Associated Domains → `applinks:example.com`

```swift
func application(_ app: UIApplication,
                 continue userActivity: NSUserActivity,
                 restorationHandler: ...) -> Bool {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else { return false }
    // url.path 등으로 화면 라우팅
    return true
}
```

**둘 다 지원하기**

대부분의 앱은 마케팅 채널별로 둘 다 지원합니다. 내부에서 같은 라우터로 모이게 추상화하면 변경에 강합니다.

### 🔗 참고 자료
- [Apple - Supporting Universal Links](https://developer.apple.com/documentation/xcode/supporting-universal-links-in-your-app)
- [Apple - Defining a Custom URL Scheme](https://developer.apple.com/documentation/xcode/defining-a-custom-url-scheme-for-your-app)

---

## 10. Core Bluetooth로 BLE 통신을 구현한다면? (Lv4)

### 💬 면접 답변

BLE는 **Central**과 **Peripheral** 두 역할이 있고, iOS에서는 보통 앱이 Central, 외부 기기가 Peripheral이 됩니다. Central은 `CBCentralManager`로 기기를 스캔하고 연결한 뒤 Peripheral의 **Service**와 그 안의 **Characteristic**을 발견해 읽기·쓰기·noti를 주고받습니다. iOS는 권한이 까다로워서 Info.plist에 `NSBluetoothAlwaysUsageDescription`을 반드시 추가해야 하고, 백그라운드에서도 동작하려면 Background Mode의 "Uses Bluetooth LE accessories"를 활성화해야 합니다. 또 iOS 13+ 부터는 Bluetooth 권한이 별도 권한이 되어 사용자가 명시적으로 허용해야 합니다.

### 📚 보충 설명

**핵심 객체**

- `CBCentralManager`: 스캔, 연결 관리
- `CBPeripheral`: 연결된 기기
- `CBService`: 기기가 노출하는 서비스 (UUID로 식별)
- `CBCharacteristic`: 서비스 안의 특성 (실제 데이터 단위)
- `CBDescriptor`: characteristic의 메타데이터

**기본 흐름**

```swift
final class BLEManager: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate {
    var central: CBCentralManager!
    var peripheral: CBPeripheral?

    override init() {
        super.init()
        central = CBCentralManager(delegate: self, queue: nil)
    }

    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        if central.state == .poweredOn {
            central.scanForPeripherals(withServices: [serviceUUID])
        }
    }

    func centralManager(_ central: CBCentralManager,
                        didDiscover peripheral: CBPeripheral,
                        advertisementData: [String: Any], rssi: NSNumber) {
        self.peripheral = peripheral
        peripheral.delegate = self
        central.connect(peripheral)
    }

    func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
        peripheral.discoverServices([serviceUUID])
    }

    func peripheral(_ peripheral: CBPeripheral, didDiscoverServices error: Error?) {
        for service in peripheral.services ?? [] {
            peripheral.discoverCharacteristics(nil, for: service)
        }
    }
}
```

**권한과 백그라운드**

- Info.plist: `NSBluetoothAlwaysUsageDescription` (iOS 13+)
- Background Modes: Uses Bluetooth LE accessories (Central) / Acts as Bluetooth LE accessory (Peripheral)
- 백그라운드에서는 스캔 동작이 제한됨 (특정 서비스 UUID로만 필터된 결과만 허용)

### 🔗 참고 자료
- [Apple - Core Bluetooth](https://developer.apple.com/documentation/corebluetooth)
- [Apple - Core Bluetooth Programming Guide](https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/AboutCoreBluetooth/Introduction.html)

---

## 11. Core NFC를 활용한 태그 통신은? (Lv4)

### 💬 면접 답변

Core NFC는 iPhone 7 이상에서 NFC 태그를 읽고(iOS 11+) 쓸 수 있게(iOS 13+) 해주는 프레임워크입니다. 두 가지 세션이 있는데, **NFCNDEFReaderSession**은 표준 NDEF 메시지 포맷의 태그를, **NFCTagReaderSession**은 ISO 7816·15693·FeliCa 같은 더 다양한 태그 타입에 접근할 수 있습니다. 사용하려면 Capabilities에 Near Field Communication Tag Reading을 추가하고, Info.plist에 사용 목적과 지원 태그 타입(`com.apple.developer.nfc.readersession.formats`)을 명시해야 합니다. 동작은 iPhone을 태그에 가까이 댔을 때만 작동하고, 백그라운드 동작은 제한적입니다(설정 메뉴의 Background Tag Reading은 NDEF만 자동 감지).

### 📚 보충 설명

**기본 사용 (NDEF 읽기)**

```swift
import CoreNFC

class NFCReader: NSObject, NFCNDEFReaderSessionDelegate {
    var session: NFCNDEFReaderSession?

    func begin() {
        session = NFCNDEFReaderSession(delegate: self,
                                        queue: nil,
                                        invalidateAfterFirstRead: true)
        session?.alertMessage = "태그에 폰을 가까이 대주세요"
        session?.begin()
    }

    func readerSession(_ session: NFCNDEFReaderSession, didDetectNDEFs messages: [NFCNDEFMessage]) {
        for msg in messages {
            for record in msg.records {
                print(record.payload)
            }
        }
    }

    func readerSession(_ session: NFCNDEFReaderSession, didInvalidateWithError error: Error) {
        // 처리
    }
}
```

**보안·UX 주의**
- 태그 데이터가 위변조될 수 있으므로 토큰 자체보다 서버 측 검증을 우선
- 사용자가 의도적으로 가까이 대야만 동작 (passive 감지 안 됨)
- 시뮬레이터에서 동작 안 함 (실기기 필수)

### 🔗 참고 자료
- [Apple - Core NFC](https://developer.apple.com/documentation/corenfc)
- [Apple - Building an NFC Tag-Reader App](https://developer.apple.com/documentation/corenfc/building_an_nfc_tag-reader_app)

---

## 12. 이미지 포맷(PNG/JPEG/HEIC/WebP)의 차이는? (Lv0)

### 💬 면접 답변

PNG는 **무손실 압축**이라 화질을 그대로 보존하고 투명도(alpha)를 지원해 UI 아이콘·버튼에 적합합니다. JPEG는 **손실 압축**으로 색이 풍부한 사진 같은 자연 이미지에서 PNG보다 훨씬 작은 파일 크기를 만들 수 있지만 투명도가 없습니다. HEIC는 Apple이 iOS 11+ 부터 도입한 포맷으로, JPEG 대비 같은 화질에서 약 절반 크기라 사진 라이브러리의 기본 포맷이고, 단일 파일에 여러 프레임(Live Photo 등)을 담을 수 있습니다. WebP는 Google이 만든 포맷으로 손실·무손실·애니메이션을 모두 지원하지만 iOS의 기본 디코더는 14+ 부터 지원합니다.

### 📚 보충 설명

**선택 가이드**

| 용도 | 포맷 |
|------|------|
| UI 아이콘, 로고 (투명도 필요) | PNG, PDF (벡터), SF Symbols |
| 사진, 자연 이미지 | JPEG, HEIC |
| 작은 파일 + 알파 + 애니메이션 | WebP |

**Asset Catalog 최적화**
- 시스템이 디바이스 픽셀 밀도에 맞춰 자동 선택 (@1x/@2x/@3x)
- 빌드 시점에 LZFSE 등으로 압축
- App Thinning으로 사용자 기기에 필요한 리소스만 다운로드

**메모리 사용량 계산**

```
메모리 = width × height × bytesPerPixel
예: 1024 × 1024 RGBA → 1024 * 1024 * 4 = 4 MB
```

→ 큰 이미지는 항상 다운샘플링 (자세한 내용은 [04_Memory_ARC.md](./04_Memory_ARC.md) 참고)

### 🔗 참고 자료
- [Apple - ImageIO](https://developer.apple.com/documentation/imageio)
- [WWDC 2018 - Image and Graphics Best Practices](https://developer.apple.com/videos/play/wwdc2018/219/)

---

> 보안 관련 전반(키체인, 인증서)은 [01_CS_Fundamentals.md](./01_CS_Fundamentals.md)와 [09_Data_Persistence.md](./09_Data_Persistence.md)를 참고하세요.
