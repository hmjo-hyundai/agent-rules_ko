# 모던 Swift 개발

Apple의 최신 아키텍처 권장사항과 모범 사례를 따르는 관용적인 SwiftUI 코드를 작성하세요.

## 핵심 철학

- SwiftUI는 Apple 플랫폼의 기본 UI 패러다임입니다 - 선언적 특성을 받아들이세요
- 레거시 UIKit 패턴과 불필요한 추상화를 피하세요
- 단순함, 명확성, 네이티브 데이터 플로우에 집중하세요
- SwiftUI가 복잡성을 처리하도록 하세요 - 프레임워크와 싸우지 마세요

## 아키텍처 가이드라인

### 1. 네이티브 상태 관리 받아들이기

SwiftUI의 내장 프로퍼티 래퍼들을 적절히 사용하세요:
- `@State` - 로컬, 임시 뷰 상태
- `@Binding` - 뷰 간 양방향 데이터 플로우
- `@Observable` - 공유 상태 (iOS 17+)
- `@ObservableObject` - 레거시 공유 상태 (iOS 17 이전)
- `@Environment` - 앱 전체 관심사를 위한 의존성 주입

### 2. 상태 소유권 원칙

- 뷰는 공유가 필요하지 않는 한 자신의 로컬 상태를 소유합니다
- 상태는 아래로 흐르고, 액션은 위로 흐릅니다
- 상태를 사용되는 곳에 최대한 가깝게 유지하세요
- 여러 뷰가 필요할 때만 공유 상태를 추출하세요

### 3. 모던 비동기 패턴

- 비동기 작업의 기본으로 `async/await` 사용
- 라이프사이클 인식 비동기 작업을 위해 `.task` 수정자 활용
- 절대적으로 필요하지 않는 한 Combine 피하기
- try/catch로 에러를 우아하게 처리

### 4. 뷰 구성

- 작고 집중된 뷰로 UI 구축
- 재사용 가능한 컴포넌트를 자연스럽게 추출
- 공통 스타일링을 캡슐화하기 위해 뷰 수정자 사용
- 상속보다 구성을 선호

### 5. 코드 조직

- 타입별이 아닌 기능별로 조직화 (Views/, Models/, ViewModels/ 폴더 피하기)
- 적절할 때 관련 코드를 같은 파일에 함께 보관
- 큰 파일을 조직화하기 위해 확장 사용
- Swift 네이밍 컨벤션을 일관되게 따르기

## 구현 패턴

### 간단한 상태 예제
```swift
struct CounterView: View {
    @State private var count = 0
    
    var body: some View {
        VStack {
            Text("Count: \(count)")
            Button("Increment") { 
                count += 1 
            }
        }
    }
}
```

### @Observable을 사용한 공유 상태
```swift
@Observable
class UserSession {
    var isAuthenticated = false
    var currentUser: User?
    
    func signIn(user: User) {
        currentUser = user
        isAuthenticated = true
    }
}

struct MyApp: App {
    @State private var session = UserSession()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(session)
        }
    }
}
```

### 비동기 데이터 로딩
```swift
struct ProfileView: View {
    @State private var profile: Profile?
    @State private var isLoading = false
    @State private var error: Error?
    
    var body: some View {
        Group {
            if isLoading {
                ProgressView()
            } else if let profile {
                ProfileContent(profile: profile)
            } else if let error {
                ErrorView(error: error)
            }
        }
        .task {
            await loadProfile()
        }
    }
    
    private func loadProfile() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            profile = try await ProfileService.fetch()
        } catch {
            self.error = error
        }
    }
}
```

## 모범 사례

### 해야 할 것:
- 가능할 때 자체 포함된 뷰 작성
- Apple이 의도한 대로 프로퍼티 래퍼 사용
- 로직을 독립적으로 테스트하고, UI를 시각적으로 미리보기
- 로딩 및 에러 상태를 명시적으로 처리
- 뷰가 프레젠테이션에 집중하도록 유지
- 안전성을 위해 Swift의 타입 시스템 사용

### 하지 말아야 할 것:
- 모든 뷰에 대해 ViewModel 생성
- 불필요하게 뷰에서 상태를 밖으로 이동
- 명확한 이점 없이 추상화 레이어 추가
- 간단한 비동기 작업에 Combine 사용
- SwiftUI의 업데이트 메커니즘과 싸우기
- 간단한 기능을 과도하게 복잡화

## 테스팅 전략

- 비즈니스 로직과 데이터 변환을 단위 테스트
- 시각적 테스팅을 위해 SwiftUI Previews 사용
- @Observable 클래스를 독립적으로 테스트
- 테스트를 간단하고 집중되게 유지
- 테스트 가능성을 위해 코드 명확성을 희생하지 마세요

## 모던 Swift 기능

- Swift Concurrency (async/await, actors) 사용
- 사용 가능할 때 Swift 6 데이터 경합 안전성 활용
- 프로퍼티 래퍼를 효과적으로 활용
- 적절한 곳에서 값 타입 받아들이기
- 테스트를 위해서만이 아닌 추상화를 위해 프로토콜 사용

## 요약

SwiftUI처럼 보이고 느껴지는 SwiftUI 코드를 작성하세요. 프레임워크는 상당히 성숙해졌습니다 - 그 패턴과 도구를 신뢰하세요. 다른 플랫폼의 아키텍처 패턴을 구현하는 것보다 사용자 문제 해결에 집중하세요.