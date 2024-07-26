# Dependency

- Dependency란 현재 도메인과 다른 기능을 가진 객체를 참조하는 것을 의미함.
    - UserDefaults, 네트워크 통신등이 있음.
    - 의존성을 처리하기 위해 의존성 주입(Dependency Injection)과 같은 방식을 사용함.

<aside>
💡 SwiftUI에서는 Preview 기능을 제공하는데 mock과 같은 의존성 처리가 되지 않으면 Preview 환경에서의 개발이 어렵다.

</aside>

## TCA Dependency

개발시 의존성을 쉽게 관리할 수 있도록 아래와 같은 부분들을 고려하여 만들어진 의존성 관리 라이브러리.

<aside>
💡 TCA Dependency 라이브러리의 고려 사항

- 전역 종속성을 갖는 것보다 안전한 방식으로 앱 내에 종속성을 전파할 수 있는 방법은 무엇일까?
- 애플리케이션의 한 부분에 대한 종속성을 어떻게 재정의 할 수 있을까?
- 테스트에서 기능이 사용하는 모든 종속성을 재정의했는지 어떻게 확인할 수 있을까?
</aside>

## ReducerProtocol 이전의 Dependency 관리 방식: Environment

### Environment의 단점

- 의존성의 확장이나 변동에 유연하지 못함

### 예시

```swift
struct SettingsEnvironment {
    public var apiClient: ApiClient
    public var fileClient: FileClient
    public var userDefaults: UserDefaultsClient
    public var userNotifications: UserNotificationClient
    /* code */
    // 추가된 additionalClient
    public var additionalClient: AdditionalClient
}
```

- SettingEnvironment라는 Environment 구조체에 additionalClient라는 의존성을 추가하려고 함.

```swift
struct SettingsEnvironment {
    /* code */
    init(additionalClient: AdditionalClient, /* code */) {
        self.additionalClient = AdditionalClient
    }
}

extension SettingsEnvironment {
    public static let failing = Self(
        additionalClient: ,
        /* code */
    )
    public static let noop = Self(
        additionalClient: ,
        /* code */
    )
}
```

- init에 파라미터 추가 해줘야 하고, 그러면 해당 함수를 사용하는 곳에서 수정이 필요함.
- 연쇄적인 컴파일 오류로 개발자가 수동으로 다 고쳐줘야 하는 문제가 있음. 🤮

## ReducerProtocol도입과 변화된 Dependency 관리 방식

- 기존 Environment로 의존성을 관리할 떄 생기는 문제점을 보완하기 위해 Dependency 라이브러리를 추가함.

### @Dependency

```swift
public struct Dependency<Value>: @unchecked Sendable, _HasInitialValues {
    // 앱내에 필요한 의존성이 저장되어있는 DependencyValues
    let initialValues: DependencyValues
    private let keyPath: KeyPath<DependencyValues, Value>
    private let file: StaticString
    private let fileID: StaticString
    private let line: UInt
}
```

- Dependency라는 Property Wrapper는 DependencyValues에 저장된 특정 의존성에 KeyPath를 통해서 접근하도록 정의되어 있음.

```swift
// 필수
import Dependencies

// (1) ObservableObject로 구현된 Model에서 사용하는 방식
struct FeatureModel: ObservableObject {
    @Dependency(\.apiclient) var apiClient
    @Dependency(\.uuid) var uuid
}

// (2) ComposableArchitecture에서 사용하는 방식
struct Feature: ReducerProtocol {
    @Dependency(\.apiClient) var apiClient
    @Dependency(\.uuid) var uuid
}
```

- 해당 Property Wrapper는 위와 같이 사용할 수 있다.
- 위와 같이 특정 Dependency에 전역적으로 접근하기 위해서는 DependencyValues에 필요한 의존성을 등록하는 과정이 필요함

### DependencyKey

- DependencyValue에 특정 의존성을 추가/등록하기 위해서는 DependencyKey를 준수하는 의존성을 만들어 줘야함.

```swift
public protocol DependencyKey: TestDependencyKey {
    /// ``` 실제 앱 동작 및 시뮬레이터 동작에 사용될 liveValue
    static var liveValue: Value { get }
    
    associatedtype Value = Self
    
    /// ``` SwiftUI의 프리뷰를 위한 previewValue
    static var previewValue: Value { get }
    
    /// ``` Test를 위해 사용될 mock testValue
    static var testValue: Value { get }
}
```

- liveValue
    - 실제 기기나 시뮬레이터에서 동작할 때 사용되는 의존성. 반드시 구현되어야 함.
- previewValue
    - SwiftUI의 Preview 환경에서 사용될 의존성.
    - extension으로 Default 값이 liveValue로 되어있기 떄문에, 항상 구현해야 할 필요는 없음.
- testValue
    - Test 환경에서 사용될 의존성
    - extension으로 Default 값이 liveValue로 되어있기 떄문에, 항상 구현해야 할 필요는 없음.
- 적용 예시
    
    ```swift
    enum SpeechRecognitionKey: DependencyKey {
        public static var liveValue: any SpeechRecognitionUseCase {
            SpeechRecognitionUseCaseImpl(service: FakeSpeechRecognizeService())
        }
        
        public static var previewValue: SpeechRecognitionUseCase {
            SpeechRecognitionUseCaseImpl(service: FakeSpeechRecognizeService())
        }
        
        public static var testValue: any SpeechRecognitionUseCase {
            SpeechRecognitionUseCaseImpl(service: FakeSpeechRecognizeService())
        }
    }
    ```
    

### DependencyValue

- DependencyKey를 통해서 의존성을 return하고 의존성을 관리하는 역할을 함.

```swift
public struct DependencyValues: Sendable {
    @TaskLocal public static var _current = Self()
#if DEBUG
    @TaskLocal static var isSetting = false
#endif
    // 현재 의존성
    @TaskLocal static var currentDependency = CurrentDependency()
    
    fileprivate var cachedValues = CachedValues()
    // DepedencyKey를 통해, 앱에서 사용될 Dependency를 관리하는 storage
    private var storage: [ObjectIdentifier: AnySendable] = [:]
    
    public init() {
#if canImport(XCTest)
        _ = setUpTestObservers
#endif
    }
    
    /* DependencyValue를 DependencyKey를 통해, 접근할 수 있도록 구현된 subscript */
    public subscript<Key: TestDependencyKey>(
        key: Key.Type,
        file: StaticString = #file,
        function: StaticString = #function,
        line: UInt = #line
    ) -> Key.Value where Key.Value: Sendable {
        /*
         (1) 커스텀하게 의존성을 등록하고 사용하기위해, 앞서 배운 DependencyKey 정의
         private struct MyDependencyKey: DependencyKey {
         static let testValue = "Default value"
         }
         (2) 정의된 DependencyKey값을 통해 아래와 같이, computed-property를 통해 의존성 등록 및 접근
         extension DependencyValues {
         var myCustomValue: String {
         get { self[MyDependencyKey.self] }
         set { self[MyDependencyKey.self] = newValue }
         }
         */
        get {
            guard let base = self.storage[ObjectIdentifier(key)]?.base,
                  let dependency = base as? Key.Value
            else {
                let context =
                self.storage[ObjectIdentifier(DependencyContextKey.self)]?.base as? DependencyContext
                ?? defaultContext
                
                switch context {
                case .live, .preview:
                    return self.cachedValues.value(
                        for: Key.self,
                        context: context,
                        file: file,
                        function: function,
                        line: line
                    )
                case .test:
                    var currentDependency = Self.currentDependency
                    currentDependency.name = function
                    return Self.$currentDependency.withValue(currentDependency) {
                        self.cachedValues.value(
                            for: Key.self,
                            context: context,
                            file: file,
                            function: function,
                            line: line
                        )
                    }
                }
            }
            return dependency
        }
        set {
            self.storage[ObjectIdentifier(key)] = AnySendable(newValue)
        }
    }
    
    public static var live: Self {
        var values = Self()
        values.context = .live
        return values
    }
    
    /// A collection of "preview" dependencies.
    public static var preview: Self {
        var values = Self()
        values.context = .preview
        return values
    }
    
    /// A collection of "test" dependencies.
    public static var test: Self {
        var values = Self()
        values.context = .test
        return values
    }
    
    func merging(_ other: Self) -> Self {
        var values = self
        values.storage.merge(other.storage, uniquingKeysWith: { $1 })
        return values
    }
}
```

- currentDependency
    - 현재 DependencyKey를 통해 사용중인 의존성
- subscript
    - DependencyKey를 통해서 storage에 저장된 의존성 탐색 및 접근
- storage
    - DependencyKey를 통해서 storage에 의존성 저장

- 글을 읽고 든 의문
    - @Dependency 프로퍼티 래퍼를 써서 의존성을 선언하면 DependencyValue 및 DependencyKey가 필요한데 그것들은 프로토콜에 적용이 안되고 구체 타입에만 적용이된다.
    그러면, 의존성을 담는 변수 자체가 구체타입으로 선언되어야 하고 그럼 해당 Reducer는 구체타입에 의존하게 되므로 결합도가 커지는거 아닐까?(고민 필요)