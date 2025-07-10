# Swift 6 동시성: 데이터 레이스 안전성과 엄격한 동시성 완벽 가이드

## 목차

1. [서론](#서론)
2. [핵심 개념](#핵심-개념)
3. [Swift Evolution 제안](#swift-evolution-제안)
4. [데이터 레이스 안전성](#데이터-레이스-안전성)
5. [액터 시스템](#액터-시스템)
6. [Sendable 프로토콜](#sendable-프로토콜)
7. [마이그레이션 가이드](#마이그레이션-가이드)
8. [코드 예제](#코드-예제)
9. [모범 사례](#모범-사례)
10. [테스트 및 디버깅](#테스트-및-디버깅)
11. [일반적인 컴파일러 오류](#일반적인-컴파일러-오류)
12. [성능 튜닝](#성능-튜닝)
13. [Swift 6.1+ 로드맵](#swift-61-로드맵)
14. [리소스](#리소스)

## 서론

Swift 6는 동시 프로그래밍의 혁명적인 도약을 나타내며, 전체 클래스의 버그를 제거하는 컴파일 타임 데이터 레이스 안전성을 도입합니다. 이 포괄적인 가이드는 모든 Swift Evolution 제안, 실용적인 예제, 그리고 Swift 6의 엄격한 동시성 모델 채택을 위한 마이그레이션 전략을 다룹니다.

### 빠른 시작: 엄격한 동시성 활성화

**Xcode:**
```
Build Settings → Swift Compiler - Language → Strict Concurrency Checking → Complete
```

**명령줄:**
```bash
swiftc -strict-concurrency=complete -swift-version 6 MyFile.swift
```

**Package.swift:**
```swift
.target(
    name: "MyTarget",
    swiftSettings: [
        .enableUpcomingFeature("StrictConcurrency"),
        .swiftLanguageMode(.v6)  // 또는 경고와 함께 .v5 유지
    ]
)
```

### Swift 6 동시성이란?

Swift 6의 동시성 모델은 Swift 5.5의 async/await 기반 위에 구축되었습니다:
- **완전한 데이터 레이스 안전성** 컴파일 타임에
- **액터 격리** 강제
- **Sendable 프로토콜** 요구사항
- **지역 기반 격리** 더 스마트한 타입 검사를 위한
- **개선된 인체공학** 가양성 감소

### 주요 장점

1. **컴파일 타임 안전성**: 런타임 전에 데이터 레이스 포착
2. **점진적 마이그레이션**: 먼저 경고와 함께 점진적 채택
3. **더 나은 성능**: 격리 보장에 의해 활성화된 컴파일러 최적화
4. **명확한 의도**: 코드의 명시적인 동시성 경계

## 핵심 개념

### 1. 격리 도메인

Swift 6는 데이터가 안전하게 접근될 수 있는 명확한 격리 도메인을 정의합니다:

```swift
// MainActor 격리 도메인 - UI 코드
@MainActor
class ViewController: UIViewController {
    var label = UILabel() // MainActor 내에서 안전
    
    func updateUI() {
        label.text = "Updated" // await 불필요 - 동일한 격리
    }
}

// 사용자 정의 액터 격리 도메인
actor DataManager {
    private var cache: [String: Data] = [:] // 액터에 의해 보호됨
    
    func store(key: String, data: Data) {
        cache[key] = data // 액터 내에서 안전
    }
}

// 격리 없음 - Sendable이어야 함
struct Point: Sendable {
    let x: Double
    let y: Double
}
```

### 2. 동시성 경계

동시성 경계를 넘나드는 데이터는 Sendable이어야 합니다:

```swift
// ❌ Swift 5: 잠재적 데이터 레이스
class Model {
    var items: [Item] = []
}

func process(model: Model) async {
    await Task.detached {
        model.items.append(Item()) // 데이터 레이스!
    }.value
}

// ✅ Swift 6: 컴파일 타임 오류가 데이터 레이스 방지
@MainActor
final class Model: Sendable {
    private(set) var items: [Item] = []
    
    func addItem(_ item: Item) {
        items.append(item) // 안전 - MainActor 동기화됨
    }
}
```

### 3. 지역 기반 격리

Swift 6는 데이터 플로우를 추적하는 "격리 지역"을 도입합니다:

```swift
// 지역 기반 격리는 Sendable 없이 안전한 전송 허용
func processImage(_ image: UIImage) async -> ProcessedImage {
    // Swift 6는 전송 후 이미지에 접근하지 않음을 증명
    let processed = await withTaskGroup(of: ProcessedTile.self) { group in
        for tile in image.tiles {
            group.addTask {
                // 안전한 전송 - 컴파일러가 지역을 추적
                await processTile(tile)
            }
        }
        // 결과 수집...
    }
    return processed
}
```

## Swift Evolution 제안

### 기반 제안 (Swift 5.5-5.10)

#### SE-0302: Sendable 및 @Sendable
기본 `Sendable` 프로토콜을 도입합니다:

```swift
// 동시성 도메인 간 공유하기 안전한 타입
protocol Sendable {}

// Sendable 클로저
let operation: @Sendable () -> Void = {
    print("이 클로저는 Sendable 값만 캡처합니다")
}

// 조건부 Sendable
struct Container<T>: Sendable where T: Sendable {
    let value: T
}
```

#### SE-0306: 액터
가변 상태를 보호하는 액터 모델:

```swift
actor BankAccount {
    private var balance: Decimal = 0
    
    func deposit(amount: Decimal) {
        balance += amount
    }
    
    func withdraw(amount: Decimal) -> Bool {
        guard balance >= amount else { return false }
        balance -= amount
        return true
    }
    
    // await 없이 접근 가능한 계산 속성
    nonisolated var accountDescription: String {
        "Bank Account" // 상태 접근 없음
    }
}
```

#### SE-0316: 전역 액터
시스템 전체 격리 도메인:

```swift
@globalActor
actor DataActor {
    static let shared = DataActor()
}

// 전체 타입에 적용
@DataActor
class DataStore {
    var items: [Item] = []
    
    func add(_ item: Item) {
        items.append(item)
    }
}

// 특정 멤버에 적용
class MixedClass {
    @DataActor var data: [String] = []
    @MainActor var uiState = UIState()
    
    @DataActor
    func processData() async {
        // DataActor에서 실행
    }
    
    @MainActor
    func updateUI() {
        // MainActor에서 실행
    }
}
```

### Swift 6 핵심 제안

#### SE-0337: 동시성 검사에 대한 점진적 마이그레이션
점진적 채택을 가능하게 합니다:

```swift
// Package.swift
.target(
    name: "MyTarget",
    swiftSettings: [
        .enableUpcomingFeature("StrictConcurrency"),
        .enableUpcomingFeature("CompleteAsync"),
        .enableExperimentalFeature("StrictConcurrency=minimal")
    ]
)

// 또는 컴파일러 플래그를 통해
// swiftc -strict-concurrency=complete
```

#### SE-0401: 프로퍼티 래퍼에서 액터 격리 추론 제거
예상치 못한 격리를 제거합니다:

```swift
// SE-0401 이전
struct ContentView: View {
    @StateObject private var model = Model() // View를 MainActor-isolated로 만듦!
    
    func doWork() { // 암시적으로 @MainActor
        // ...
    }
}

// SE-0401 이후
struct ContentView: View {
    @StateObject private var model = Model() // 격리 추론 없음
    
    nonisolated func doWork() { // 명시적으로 비격리
        // ...
    }
}
```

#### SE-0412: 전역 변수에 대한 엄격한 동시성
`nonisolated(unsafe)`를 사용한 전역 변수 안전성:

```swift
// ❌ Swift 6 오류: 전역 변수가 동시성 안전하지 않음
var sharedCache: [String: Data] = [:]

// ✅ 옵션 1: let 상수로 만들기
let sharedConstants = Constants()

// ✅ 옵션 2: 전역 액터 사용
@MainActor
var sharedUICache: [String: UIImage] = [:]

// ✅ 옵션 3: 액터 격리
actor CacheActor {
    static let shared = CacheActor()
    private var cache: [String: Data] = [:]
}

// ✅ 옵션 4: 명시적 unsafe 옵트아웃
struct LegacyAPI {
    nonisolated(unsafe) static var shared: LegacyAPI?
}
```

#### SE-0414: 지역 기반 격리
격리 검사의 혁신적인 개선:

```swift
// 비Sendable 타입이 안전하게 전송될 수 있음
class MutableData {
    var value: Int = 0
}

func process() async {
    let data = MutableData() // 비Sendable
    
    // ✅ 안전: 전송 후 데이터 사용 안 함
    await withTaskGroup(of: Void.self) { group in
        group.addTask {
            data.value = 42 // 지역 분석이 안전성 증명
        }
    }
    
    // ❌ 오류: 전송 후 데이터 사용
    // print(data.value)
}
```

#### SE-0420: 액터 격리 상속
동적 격리 상속:

```swift
// 함수가 호출자의 격리를 상속
func log(
    _ message: String,
    isolation: isolated (any Actor)? = #isolation
) async {
    print("[\(isolation)] \(message)")
}

@MainActor
func updateUI() async {
    await log("UI 업데이트 중") // MainActor 격리 상속
}

actor DataProcessor {
    func process() async {
        await log("처리 중") // DataProcessor 격리 상속
    }
}
```

#### SE-0430: 매개변수 및 결과 값 전송
완전한 Sendable 없이 안전한 전송:

```swift
// 'sending'은 소유권 전송 허용
func processData(_ data: sending MutableData) async -> sending ProcessedData {
    // data가 소비됨 - 원본 참조 무효화
    return ProcessedData(from: data)
}

// 업데이트된 Task API
extension Task where Failure == Never {
    init(
        priority: TaskPriority? = nil,
        operation: sending @escaping () async -> Success
    )
}
```

#### SE-0431: @isolated(any) 함수 타입
격리 무관 함수 타입:

```swift
// 모든 격리를 보존하는 함수 타입
typealias IsolatedOperation = @isolated(any) () async -> Void

struct Executor {
    func run(_ operation: IsolatedOperation) async {
        await operation() // 호출자의 격리 유지
    }
}
```

#### SE-0434: 전역 액터 격리 타입의 사용성
전역 액터 사용을 위한 개선사항:

```swift
// Sendable 프로퍼티는 nonisolated 가능
@MainActor
final class ViewModel: Sendable {
    // ✅ 암시적으로 nonisolated (Sendable 저장 프로퍼티)
    let id = UUID()
    
    // ❌ 격리되어야 함 (비Sendable)
    var items: [Item] = []
    
    // ✅ 명시적으로 nonisolated 가능
    nonisolated let configuration: Configuration
}

// 클로저에 대한 개선된 추론
@MainActor
class Controller {
    func setup() {
        // ✅ 클로저가 @MainActor @Sendable로 추론됨
        Task {
            await updateData()
        }
    }
}
```

## 데이터 레이스 안전성

### 완전한 동시성 검사

Swift 6는 기본적으로 완전한 검사를 강제합니다:

```swift
// 마이그레이션을 위해 Swift 5 모드에서 활성화
// swift -strict-concurrency=complete

// 검사 수준:
// 1. minimal - 명시적 Sendable 준수만
// 2. targeted - 일부 타입에 대해 Sendable 추론
// 3. complete - 완전한 데이터 레이스 검사
```

### 일반적인 데이터 레이스 패턴과 수정

#### 패턴 1: 공유 가변 상태

```swift
// ❌ 데이터 레이스
class Counter {
    var value = 0
    
    func increment() {
        value += 1 // 경쟁 상태!
    }
}

// ✅ 수정 1: 액터 사용
actor Counter {
    private var value = 0
    
    func increment() {
        value += 1 // 액터 격리됨
    }
    
    var currentValue: Int {
        value
    }
}

// ✅ 수정 2: 원자적 연산 사용
import Atomics

final class Counter: Sendable {
    private let value = ManagedAtomic<Int>(0)
    
    func increment() {
        value.wrappingIncrement(ordering: .relaxed)
    }
    
    var currentValue: Int {
        value.load(ordering: .relaxed)
    }
}
```

#### 패턴 2: 콜백 격리

```swift
// ❌ 불분명한 격리
class NetworkManager {
    func fetch(completion: @escaping (Data) -> Void) {
        URLSession.shared.dataTask(with: url) { data, _, _ in
            completion(data!) // 어떤 스레드?
        }
    }
}

// ✅ async/await로 명확한 격리
class NetworkManager {
    func fetch() async throws -> Data {
        let (data, _) = try await URLSession.shared.data(from: url)
        return data
    }
}

// ✅ 또는 명시적 MainActor 격리
class NetworkManager {
    func fetch(completion: @MainActor @escaping (Data) -> Void) {
        Task {
            let data = try await URLSession.shared.data(from: url).0
            await completion(data)
        }
    }
}
```

## 액터 시스템

### 기본 액터 사용

```swift
actor DatabaseConnection {
    private var isConnected = false
    private var activeQueries = 0
    
    func connect() async throws {
        guard !isConnected else { return }
        // 연결 로직...
        isConnected = true
    }
    
    func query(_ sql: String) async throws -> [Row] {
        activeQueries += 1
        defer { activeQueries -= 1 }
        
        // 쿼리 실행...
        return rows
    }
    
    // 불변 데이터에 대한 동기 접근
    nonisolated let connectionString: String
    
    // 상태 접근 없는 계산 속성
    nonisolated var description: String {
        "Database connection to \(connectionString)"
    }
}
```

### UI 코드용 MainActor

```swift
// 전체 클래스가 MainActor에서
@MainActor
final class LoginViewModel: ObservableObject {
    @Published private(set) var isLoading = false
    @Published private(set) var error: Error?
    
    func login(username: String, password: String) async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            // 네트워크 호출을 위해 백그라운드로 전환
            let user = try await AuthService.shared.login(
                username: username,
                password: password
            )
            // 자동으로 MainActor로 돌아옴
            navigateToHome(user: user)
        } catch {
            self.error = error
        }
    }
    
    // 모든 스레드에서 실행 가능
    nonisolated func validateEmail(_ email: String) -> Bool {
        // 이메일 검증 로직...
    }
}
```

### 사용자 정의 전역 액터

```swift
// 데이터베이스 연산을 위한 전역 액터 정의
@globalActor
actor DatabaseActor {
    static let shared = DatabaseActor()
    
    // 통합을 위한 사용자 정의 실행자
    nonisolated var unownedExecutor: UnownedSerialExecutor {
        DatabaseQueue.shared.unownedExecutor
    }
}

// 데이터베이스 연산을 처리하는 타입에 적용
@DatabaseActor
class UserRepository {
    private var cache: [UUID: User] = [:]
    
    func findUser(id: UUID) async throws -> User {
        if let cached = cache[id] {
            return cached
        }
        
        let user = try await database.fetch(User.self, id: id)
        cache[id] = user
        return user
    }
    
    func saveUser(_ user: User) async throws {
        try await database.save(user)
        cache[user.id] = user
    }
}

// 하나의 타입에서 다른 액터들 혼합
class DataCoordinator {
    @DatabaseActor
    private var userRepo = UserRepository()
    
    @MainActor
    private var viewModel = UserListViewModel()
    
    func refreshUsers() async {
        // DatabaseActor에서 가져오기
        let users = await userRepo.fetchAllUsers()
        
        // MainActor에서 업데이트
        await viewModel.update(users: users)
    }
}
```

## Sendable 프로토콜

### Sendable 이해하기

```swift
// Sendable은 스레드 안전한 타입을 나타냅니다
public protocol Sendable {}

// 다음에 대한 자동 준수:
// 1. 액터 (동기화 처리)
// 2. 불변 구조체/열거형
// 3. 불변 저장소를 가진 final 클래스
// 4. 수동 안전성을 위한 @unchecked Sendable

// 자동 Sendable 예제
struct Point: Sendable { // 암시적
    let x: Double
    let y: Double
}

enum Status: Sendable { // 암시적
    case pending
    case completed(at: Date)
}

actor DataManager {} // 암시적으로 Sendable

final class User: Sendable {
    let id: UUID
    let name: String
    // 모든 저장 프로퍼티는 불변
}
```

### 조건부 Sendable

```swift
// 제네릭 타입은 조건부로 Sendable 가능
struct Container<T> {
    let value: T
}

// 자동 조건부 준수
extension Container: Sendable where T: Sendable {}

// 사용자 정의 조건부 준수
struct Cache<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    private let lock = NSLock()
}

extension Cache: @unchecked Sendable where Key: Sendable, Value: Sendable {
    // 락으로 스레드 안전성 보장
}
```

### @unchecked Sendable

```swift
// 스레드 안전하지만 컴파일러가 증명할 수 없는 타입용
final class ThreadSafeCache: @unchecked Sendable {
    private var cache: [String: Data] = [:]
    private let queue = DispatchQueue(label: "cache.queue")
    
    func get(_ key: String) -> Data? {
        queue.sync { cache[key] }
    }
    
    func set(_ key: String, value: Data) {
        queue.async { self.cache[key] = value }
    }
}

// 불변 데이터를 가진 참조 타입
final class ImageWrapper: @unchecked Sendable {
    let cgImage: CGImage
    
    init(cgImage: CGImage) {
        self.cgImage = cgImage
    }
}
```

### Sendable 함수와 클로저

```swift
// Sendable 함수 타입
typealias AsyncOperation = @Sendable () async -> Void
typealias CompletionHandler = @Sendable (Result<Data, Error>) -> Void

// Sendable 클로저 사용
func performAsync(operation: @Sendable @escaping () async -> Void) {
    Task {
        await operation()
    }
}

// Sendable 캡처
func createTimer(interval: TimeInterval) -> AsyncStream<Date> {
    AsyncStream { continuation in
        let timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { _ in
            continuation.yield(Date())
        }
        
        continuation.onTermination = { @Sendable _ in
            timer.invalidate() // Sendable이어야 함
        }
    }
}
```

## 마이그레이션 가이드

### 1단계: Swift 5 모드에서 경고 활성화

```swift
// Package.swift에서
.target(
    name: "MyApp",
    swiftSettings: [
        .enableUpcomingFeature("StrictConcurrency"),
        .enableUpcomingFeature("ExistentialAny"),
        .enableUpcomingFeature("ConciseMagicFile")
    ]
)

// 또는 Xcode Build Settings에서
// Strict Concurrency Checking: Complete
// SWIFT_STRICT_CONCURRENCY = complete
```

### 2단계: 전역 변수 수정

```swift
// 이전
var sharedFormatter = DateFormatter()

// 이후 - 옵션 1: 불변으로 만들기
let sharedFormatter: DateFormatter = {
    let formatter = DateFormatter()
    formatter.dateStyle = .short
    return formatter
}()

// 이후 - 옵션 2: 액터 격리 추가
extension DateFormatter {
    @MainActor
    static let shared: DateFormatter = {
        let formatter = DateFormatter()
        formatter.dateStyle = .short
        return formatter
    }()
}

// 이후 - 옵션 3: 레거시 코드에 nonisolated(unsafe) 사용
nonisolated(unsafe) var legacyGlobal: LegacyType?
```

### 3단계: Sendable 준수 추가

```swift
// 모델 타입을 Sendable로 만들기
struct User: Codable, Sendable {
    let id: UUID
    let name: String
    let email: String
}

// 참조 타입의 경우 불변성 보장
final class Configuration: Sendable {
    let apiKey: String
    let baseURL: URL
    
    init(apiKey: String, baseURL: URL) {
        self.apiKey = apiKey
        self.baseURL = baseURL
    }
}
```

### 4단계: UI 코드 격리

```swift
// 이전
class ViewModel: ObservableObject {
    @Published var items: [Item] = []
    
    func loadItems() {
        Task {
            items = await fetchItems() // 잠재적 경쟁
        }
    }
}

// 이후
@MainActor
final class ViewModel: ObservableObject {
    @Published private(set) var items: [Item] = []
    
    func loadItems() async {
        items = await fetchItems() // MainActor에서 안전
    }
}
```

### 5단계: 콜백과 델리게이트 처리

```swift
// 이전 - 불분명한 격리
protocol DataDelegate: AnyObject {
    func dataDidUpdate(_ data: Data)
}

// 이후 - 명시적 격리
@MainActor
protocol DataDelegate: AnyObject {
    func dataDidUpdate(_ data: Data)
}

// 또는 async 대안 사용
protocol DataProvider {
    func fetchData() async throws -> Data
}
```

### 점진적 마이그레이션 전략

1. **잎 모듈부터 시작**: 의존성이 적은 모듈부터 시작
2. **간단한 문제 먼저 해결**: 불변 전역 변수, 누락된 Sendable 준수
3. **UI 레이어 격리**: 뷰 컨트롤러와 뷰 모델에 @MainActor 추가
4. **공유 상태 해결**: 액터로 변환하거나 동기화 사용
5. **Swift 6 모드 활성화**: 경고가 해결되면

## 코드 예제

### 예제 1: 이미지 처리 파이프라인

```swift
// 액터와 Sendable을 사용한 이미지 프로세서
actor ImageProcessor {
    private let cache = ImageCache()
    
    func process(_ image: UIImage, filters: [Filter]) async throws -> UIImage {
        // 캐시 확인
        let cacheKey = CacheKey(image: image, filters: filters)
        if let cached = await cache.get(cacheKey) {
            return cached
        }
        
        // 이미지 처리
        var result = image
        for filter in filters {
            result = try await filter.apply(to: result)
        }
        
        // 결과 캐시
        await cache.set(cacheKey, image: result)
        return result
    }
}

// Sendable 필터 프로토콜
protocol Filter: Sendable {
    func apply(to image: UIImage) async throws -> UIImage
}

// 구체적인 필터 구현
struct BlurFilter: Filter {
    let radius: Double
    
    func apply(to image: UIImage) async throws -> UIImage {
        // Core Image를 사용한 구현
        let ciImage = CIImage(image: image)!
        let filter = CIFilter.gaussianBlur()
        filter.inputImage = ciImage
        filter.radius = Float(radius)
        
        let context = CIContext()
        let output = filter.outputImage!
        let cgImage = context.createCGImage(output, from: output.extent)!
        
        return UIImage(cgImage: cgImage)
    }
}
```

### 예제 2: 적절한 격리를 가진 네트워크 레이어

```swift
// 명확한 격리 경계를 가진 네트워크 서비스
actor NetworkService {
    private let session: URLSession
    private let decoder = JSONDecoder()
    private var activeTasks: [UUID: URLSessionTask] = [:]
    
    init(configuration: URLSessionConfiguration = .default) {
        self.session = URLSession(configuration: configuration)
    }
    
    func fetch<T: Decodable & Sendable>(
        _ type: T.Type,
        from url: URL
    ) async throws -> T {
        let taskID = UUID()
        
        let task = session.dataTask(with: url)
        activeTasks[taskID] = task
        
        defer { activeTasks.removeValue(forKey: taskID) }
        
        let (data, response) = try await withTaskCancellationHandler {
            try await session.data(from: url)
        } onCancel: {
            task.cancel()
        }
        
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.invalidResponse
        }
        
        return try decoder.decode(type, from: data)
    }
    
    func cancelAll() {
        activeTasks.values.forEach { $0.cancel() }
        activeTasks.removeAll()
    }
    
    nonisolated var activeTaskCount: Int {
        get async { await activeTasks.count }
    }
}

// 적절한 오류 처리와 함께 사용
@MainActor
class UserListViewModel: ObservableObject {
    @Published private(set) var users: [User] = []
    @Published private(set) var isLoading = false
    @Published private(set) var error: Error?
    
    private let networkService = NetworkService()
    
    func loadUsers() async {
        isLoading = true
        error = nil
        
        do {
            let url = URL(string: "https://api.example.com/users")!
            users = try await networkService.fetch([User].self, from: url)
        } catch {
            self.error = error
        }
        
        isLoading = false
    }
}
```

### 예제 3: 동시 데이터 처리

```swift
// 적절한 격리를 가진 병렬 처리
struct DataProcessor {
    func processFiles(_ urls: [URL]) async throws -> [ProcessedData] {
        try await withThrowingTaskGroup(of: ProcessedData.self) { group in
            // 각 파일에 대한 작업 추가
            for url in urls {
                group.addTask {
                    try await processFile(url)
                }
            }
            
            // 결과 수집
            var results: [ProcessedData] = []
            for try await result in group {
                results.append(result)
            }
            
            return results
        }
    }
    
    private func processFile(_ url: URL) async throws -> ProcessedData {
        let data = try await readFile(url)
        let processed = try await transform(data)
        return ProcessedData(
            originalURL: url,
            processedData: processed,
            timestamp: Date()
        )
    }
}

// Sendable인 결과 타입
struct ProcessedData: Sendable {
    let originalURL: URL
    let processedData: Data
    let timestamp: Date
}
```

## 모범 사례

### 1. Sendability를 위한 설계

```swift
// ❌ 가변 참조 타입 피하기
class Settings {
    var theme: Theme
    var notifications: Bool
}

// ✅ 값 타입 또는 불변 참조 타입 선호
struct Settings: Sendable {
    let theme: Theme
    let notifications: Bool
}

// ✅ 또는 가변 상태에 액터 사용
actor SettingsManager {
    private var settings: Settings
    
    func update(theme: Theme) {
        settings = Settings(
            theme: theme,
            notifications: settings.notifications
        )
    }
}
```

### 2. 액터 홉 최소화

```swift
// ❌ 과도한 액터 호핑
@MainActor
class ViewModel {
    func processData() async {
        let data = await dataActor.getData()
        let processed = await processorActor.process(data)
        let formatted = await formatterActor.format(processed)
        updateUI(formatted)
    }
}

// ✅ 배치 연산
@MainActor
class ViewModel {
    func processData() async {
        let result = await dataActor.getProcessedAndFormattedData()
        updateUI(result)
    }
}
```

### 3. 순수 함수에 nonisolated 사용

```swift
actor Calculator {
    private var history: [Calculation] = []
    
    // ✅ 순수 함수는 격리가 필요 없음
    nonisolated func add(_ a: Double, _ b: Double) -> Double {
        a + b
    }
    
    nonisolated func multiply(_ a: Double, _ b: Double) -> Double {
        a * b
    }
    
    // 상태 수정 함수는 격리 필요
    func recordCalculation(_ calc: Calculation) {
        history.append(calc)
    }
}
```

### 4. 구조적 동시성 활용

```swift
// ✅ 병렬 작업에 작업 그룹 사용
func downloadImages(urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: (Int, UIImage).self) { group in
        for (index, url) in urls.enumerated() {
            group.addTask {
                let image = try await downloadImage(from: url)
                return (index, image)
            }
        }
        
        var images = Array<UIImage?>(repeating: nil, count: urls.count)
        for try await (index, image) in group {
            images[index] = image
        }
        
        return images.compactMap { $0 }
    }
}
```

### 5. 취소 적절히 처리

```swift
func longRunningOperation() async throws -> Result {
    try await withTaskCancellationHandler {
        var progress = 0.0
        
        while progress < 1.0 {
            try Task.checkCancellation()
            
            // 작업 수행...
            progress += 0.1
            
            try await Task.sleep(for: .seconds(1))
        }
        
        return result
    } onCancel: {
        // 리소스 정리
        cleanupOperation()
    }
}
```

## 테스트 및 디버깅

### Thread Sanitizer (TSan)

Thread Sanitizer는 컴파일 타임 검사를 벗어나는 런타임 데이터 레이스를 포착하는 데 여전히 필수적입니다:

```bash
# Xcode에서 활성화
Product → Scheme → Edit Scheme → Diagnostics → Thread Sanitizer

# 또는 명령줄을 통해
swift test -Xswiftc -sanitize=thread
```

**TSan 사용 시기:**
- @preconcurrency 임포트가 있는 레거시 코드 테스트
- @unchecked Sendable 구현 검증
- C/Objective-C 상호 운용에서 경쟁 포착

### Swift Concurrency Debugger (Xcode 16+)

동시 코드를 위한 새로운 디버깅 도구:

1. **Task Tree View**: 부모-자식 작업 관계 시각화
2. **Actor Memory Graph**: 액터 격리 경계 보기
3. **Hop Tracking**: 격리 도메인 간 실행 추적

```swift
// 디버깅 도우미
extension Task {
    static func currentPriority() -> TaskPriority {
        Task.currentPriority
    }
    
    static func printTaskTree() {
        // 디버그 빌드에서 사용 가능
        #if DEBUG
        print("Task: \(Task<Never, Never>.currentPriority)")
        #endif
    }
}
```

### 동시 코드 단위 테스트

```swift
// 타임아웃이 있는 비동기 코드 테스트 도우미
func withTimeout<T>(
    _ duration: Duration = .seconds(5),
    operation: @escaping () async throws -> T
) async throws -> T {
    try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask {
            try await operation()
        }
        
        group.addTask {
            try await Task.sleep(for: duration)
            throw TimeoutError()
        }
        
        let result = try await group.next()!
        group.cancelAll()
        return result
    }
}

// 액터 격리 테스트
final class ActorTests: XCTestCase {
    func testActorIsolation() async throws {
        let actor = TestActor()
        
        // 여러 동시 연산으로 격리 확인
        try await withThrowingTaskGroup(of: Int.self) { group in
            for i in 0..<100 {
                group.addTask {
                    await actor.increment()
                    return await actor.value
                }
            }
            
            var results: Set<Int> = []
            for try await result in group {
                results.insert(result)
            }
            
            // 적절히 격리되었다면 모든 결과가 고유해야 함
            XCTAssertEqual(results.count, 100)
        }
    }
}
```

### 일반적인 문제 디버깅

```swift
// 1. 예상치 못한 중단 지점 디버깅
actor DataManager {
    func debugSuspension() async {
        print("중단 전: \(Thread.current)")
        await someAsyncOperation()
        print("중단 후: \(Thread.current)") // 다른 스레드일 수 있음
    }
}

// 2. 격리 컨텍스트 추적
func debugIsolation(
    isolation: isolated (any Actor)? = #isolation
) async {
    if let isolation {
        print("실행 중: \(type(of: isolation))")
    } else {
        print("비격리 컨텍스트에서 실행")
    }
}

// 3. 우선순위 역전 감지
Task(priority: .low) {
    await debugPriority() // 에스컬레이션으로 인해 더 높은 우선순위로 실행될 수 있음
}

func debugPriority() async {
    print("현재 우선순위: \(Task.currentPriority)")
}
```

## 일반적인 컴파일러 오류

### 오류 참조 표

| 진단 | 예제 | 수정 |
|------|------|------|
| **액터 경계를 넘는 비Sendable 타입** | `Capture of 'nonSendable' with non-sendable type 'MyClass'` | 1. 타입을 Sendable로 만들기<br>2. `sending` 매개변수 사용<br>3. Sendable 타입으로 복사/변환 |
| **비격리에서 참조된 액터 격리 프로퍼티** | `Actor-isolated property 'items' can not be referenced from a non-isolated context` | 1. `await` 추가<br>2. 코드를 액터로 이동<br>3. 프로퍼티를 `nonisolated`로 만들기 |
| **비격리에서 메인 액터 격리 호출** | `Call to main actor-isolated instance method 'updateUI()' in a synchronous nonisolated context` | 1. 호출자에 `@MainActor` 추가<br>2. `await MainActor.run { }` 사용<br>3. 메서드를 `nonisolated`로 만들기 |
| **캡처된 var의 변경** | `Mutation of captured var 'counter' in concurrently-executing code` | 1. 상태에 액터 사용<br>2. 불변으로 만들기<br>3. `Mutex` 사용 (Swift 6.1+) |
| **Sendable 클로저가 비Sendable 캡처** | `Capture of 'self' with non-sendable type 'ViewController?' in a `@Sendable` closure` | 1. `[weak self]` 사용<br>2. 타입을 Sendable로 만들기<br>3. 클로저 전에 필요한 값 추출 |

### 상세한 오류 해결책

#### 1. 비Sendable 타입 오류

```swift
// ❌ 오류: 비Sendable 타입 'UIImage'가 액터 경계 넘음
class ImageProcessor {
    func process(image: UIImage) async {
        Task.detached {
            // 오류: 비sendable 타입과 함께 'image' 캡처
            manipulate(image)
        }
    }
}

// ✅ 해결책 1: sending 매개변수 사용
class ImageProcessor {
    func process(image: sending UIImage) async {
        Task.detached {
            manipulate(image) // 소유권 전송됨
        }
    }
}

// ✅ 해결책 2: Sendable 표현으로 변환
class ImageProcessor {
    func process(image: UIImage) async {
        let imageData = image.pngData()! // Data는 Sendable
        Task.detached {
            let recreated = UIImage(data: imageData)!
            manipulate(recreated)
        }
    }
}
```

#### 2. 액터 격리 오류

```swift
// ❌ 오류: await 없이 액터 격리 프로퍼티 접근
actor DataStore {
    var items: [Item] = []
    
    nonisolated func getItemCount() -> Int {
        items.count // 오류: 액터 격리 프로퍼티
    }
}

// ✅ 해결책 1: 메서드를 async로 만들기
actor DataStore {
    var items: [Item] = []
    
    func getItemCount() async -> Int {
        items.count // OK: 암시적으로 액터에 격리됨
    }
}

// ✅ 해결책 2: 계산 프로퍼티 사용
actor DataStore {
    private var items: [Item] = []
    
    var itemCount: Int {
        items.count // OK: 계산 프로퍼티는 격리됨
    }
}
```

#### 3. MainActor 격리 오류

```swift
// ❌ 오류: 백그라운드에서 MainActor 격리 호출
func backgroundWork() {
    updateUI() // 오류: MainActor 격리
}

@MainActor
func updateUI() { }

// ✅ 해결책 1: 호출자를 MainActor로 만들기
@MainActor
func backgroundWork() async {
    await fetchData()
    updateUI() // OK: 둘 다 MainActor에서
}

// ✅ 해결책 2: 명시적 MainActor.run
func backgroundWork() async {
    let data = await fetchData()
    await MainActor.run {
        updateUI()
    }
}
```

## 성능 튜닝

### 작업 생성 오버헤드

```swift
// ❌ 과도한 작업 생성
for item in items {
    Task {
        await process(item) // N개의 비구조적 작업 생성
    }
}

// ✅ 배치 연산에 TaskGroup 사용
await withTaskGroup(of: Void.self) { group in
    for item in items {
        group.addTask {
            await process(item) // 구조적, 제한된 동시성
        }
    }
}

// ✅ 또는 동시 forEach 사용
await items.concurrentForEach { item in
    await process(item)
}
```

### 효율적인 타이밍을 위한 Clock API

```swift
// ❌ 구식 sleep
Task {
    Thread.sleep(forTimeInterval: 1.0) // 스레드 블록
}

// ❌ 나노초로 Task.sleep
Task {
    try await Task.sleep(nanoseconds: 1_000_000_000)
}

// ✅ 현대적인 Clock 기반 접근
let clock = ContinuousClock()
try await clock.sleep(for: .seconds(1))

// ✅ 경과 시간 측정
let elapsed = await clock.measure {
    await expensiveOperation()
}
print("연산 소요 시간: \(elapsed)")

// ✅ 테스트용 사용자 정의 clock
struct TestClock: Clock {
    var now: Instant { .init() }
    func sleep(until deadline: Instant) async throws {
        // 테스트에서 즉시 반환
    }
}
```

### 구조적 vs 비구조적 작업

```swift
// ❌ 비구조적 작업은 컨텍스트 잃음
class Service {
    func startBackgroundWork() {
        Task {
            await longRunningWork() // 취소 전파 없음
        }
    }
}

// ✅ 적절한 라이프사이클을 가진 구조적 작업
class Service {
    private var workTask: Task<Void, Never>?
    
    func startBackgroundWork() {
        workTask = Task {
            try await withTaskCancellationHandler {
                await longRunningWork()
            } onCancel: {
                cleanup()
            }
        }
    }
    
    func stopWork() {
        workTask?.cancel()
    }
}

// ✅ 진정한 데몬에만 분리된 작업
Task.detached(priority: .background) {
    // 장기 실행 백그라운드 모니터링
    while !Task.isCancelled {
        await checkSystemHealth()
        try await Task.sleep(for: .minutes(5))
    }
}
```

### 액터 경합 최적화

```swift
// ❌ 단일 액터에서 높은 경합
actor Counter {
    private var value = 0
    
    func increment() {
        value += 1
    }
}

// ✅ 배치로 경합 감소
actor Counter {
    private var value = 0
    
    func increment(by amount: Int = 1) {
        value += amount
    }
    
    func batchIncrement(_ operations: [Int]) {
        value += operations.reduce(0, +)
    }
}

// ✅ 또는 높은 처리량을 위한 샤딩
actor ShardedCounter {
    private var shards: [Int]
    
    init(shardCount: Int = ProcessInfo.processInfo.activeProcessorCount) {
        self.shards = Array(repeating: 0, count: shardCount)
    }
    
    func increment() {
        let shard = Int.random(in: 0..<shards.count)
        shards[shard] += 1
    }
    
    var total: Int {
        shards.reduce(0, +)
    }
}
```

## Swift 6.1+ 로드맵

### Swift 6.1 (출시됨)

#### SE-0431: @isolated(any) 함수 타입
```swift
// 격리를 보존하는 함수 타입
typealias IsolatedHandler = @isolated(any) () async -> Void

func withIsolation(_ handler: IsolatedHandler) async {
    await handler() // 호출자의 격리 유지
}

// 액터와 함께 사용
actor MyActor {
    func doWork() async {
        await withIsolation {
            // MyActor에서 실행
            print("격리됨: \(self)")
        }
    }
}
```

#### SE-0433: 동기 상호 배제 락 (Mutex)
```swift
import Synchronization

// async 없이 임계 구역 보호
final class Statistics: Sendable {
    private let mutex = Mutex<Stats>(.init())
    
    func record(value: Double) {
        mutex.withLock { stats in
            stats.count += 1
            stats.sum += value
        }
    }
    
    var average: Double {
        mutex.withLock { stats in
            stats.count > 0 ? stats.sum / Double(stats.count) : 0
        }
    }
}

private struct Stats {
    var count = 0
    var sum = 0.0
}
```

### Swift 6.2 (개발 중)

#### SE-0461: 격리된 기본 인수
```swift
// 기본값이 격리 컨텍스트 사용 가능
@MainActor
class ViewModel {
    // ✅ 기본값이 MainActor 상태 접근 가능
    func configure(
        title: String = defaultTitle // 6.2에서 제공
    ) { }
    
    @MainActor
    static var defaultTitle: String { "Default" }
}
```

### 검토 중인 향후 제안

1. **SE-0449**: 전역 액터 추론을 방지하는 nonisolated 허용
2. **SE-0450**: 액터 격리 추론 제한
3. **SE-0451**: 격리된 동기 deinit
4. **동시성의 타입 Throws**: 더 나은 오류 전파
5. **사용자 정의 실행자 v2**: 작업 실행에 대한 더 많은 제어

### 마이그레이션 타임라인

| 버전 | 주요 기능 | 마이그레이션 영향 |
|------|-----------|-------------------|
| Swift 6.0 | 기본적으로 엄격한 동시성 | 주요 - 코드 업데이트 필요 |
| Swift 6.1 | Mutex, @isolated(any) | 소규모 - 추가 기능 |
| Swift 6.2 | 격리된 기본값, deinit | 소규모 - 삶의 질 향상 |
| Swift 7.0 | 사용자 정의 실행자 v2 | TBD - 성능 중심 |

최신 정보: [Swift Evolution Dashboard](https://www.swift.org/swift-evolution/)

## 리소스

### 공식 문서
- [Swift.org - 동시성](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [Swift 6로 마이그레이션](https://www.swift.org/migration/documentation/migrationguide/)
- [데이터 레이스 안전성](https://www.swift.org/migration/documentation/swift-6-concurrency-migration-guide/dataracesafety/)

### Swift Evolution 제안
- [SE-0401: 액터 격리 추론 제거](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0401-remove-property-wrapper-isolation.md)
- [SE-0414: 지역 기반 격리](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0414-region-based-isolation.md)
- [SE-0420: 액터 격리 상속](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0420-inheritance-of-actor-isolation.md)
- [SE-0430: 매개변수 값 전송](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md)

### WWDC 세션
- **WWDC 2024**: "앱을 Swift 6로 마이그레이션" - 실용적인 마이그레이션 가이드
- **WWDC 2022**: "Swift 동시성을 사용하여 데이터 레이스 제거" - 기본 개념
- **WWDC 2021**: "Swift의 async/await 만나기" - Swift 동시성 소개

### 커뮤니티 리소스
- [Swift 포럼 - 동시성](https://forums.swift.org/c/swift-evolution/concurrency/23)
- [동시성 인덱스 스레드](https://developer.apple.com/forums/thread/768776)
- [Swift Package Index](https://swiftpackageindex.com) - "데이터 레이스로부터 안전" 배지 표시

## 결론

Swift 6의 엄격한 동시성은 동시 코드를 작성하는 방식의 패러다임 변화를 나타냅니다. 다음을 수용함으로써:
- 컴파일 타임에 완전한 데이터 레이스 안전성
- 액터와 함께 명확한 격리 경계
- 명시적 Sendable 요구사항
- 점진적 마이그레이션 전략

우리는 더 안전하고 유지 보수가 가능한 동시 코드를 작성할 수 있습니다. 컴파일러는 전체 클래스의 버그를 방지하는 데 우리의 파트너가 되어 더 안정적인 애플리케이션으로 이어집니다.

기억하세요: 목표는 단순히 컴파일러 경고를 조용히 하는 것이 아니라, 시간이 지남에 따라 추론하고 유지하기 쉬운 명확한 동시성 경계를 가진 시스템을 설계하는 것입니다.