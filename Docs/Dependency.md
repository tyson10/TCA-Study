# Dependency

# Dependency

- Dependency란 현재 도메인과 다른 기능을 가진 객체를 참조하는 것을 의미함.
    - `UserDefaults`, 네트워크 통신등이 있음.
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

- `init`에 파라미터 추가 해줘야 하고, 그러면 해당 함수를 사용하는 곳에서 수정이 필요함.
- 연쇄적인 컴파일 오류로 개발자가 수동으로 다 고쳐줘야 하는 문제가 있음. 🤮

## ReducerProtocol도입과 변화된 Dependency 관리 방식

- 기존 `Environment`로 의존성을 관리할 떄 생기는 문제점을 보완하기 위해 `Dependency` 라이브러리를 추가함.

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

- `Dependency`라는 Property Wrapper는 `DependencyValues`에 저장된 특정 의존성에 `KeyPath`를 통해서 접근하도록 정의되어 있음.

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
- 위와 같이 특정 `Dependency`에 전역적으로 접근하기 위해서는 `DependencyValues`에 필요한 의존성을 등록하는 과정이 필요함

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

- `liveValue`
    - 실제 기기나 시뮬레이터에서 동작할 때 사용되는 의존성. 반드시 구현되어야 함.
- `previewValue`
    - SwiftUI의 Preview 환경에서 사용될 의존성.
    - `extension`으로 Default 값이 `liveValue`로 되어있기 떄문에, 항상 구현해야 할 필요는 없음.
- `testValue`
    - Test 환경에서 사용될 의존성
    - `extension`으로 Default 값이 `liveValue`로 되어있기 떄문에, 항상 구현해야 할 필요는 없음.
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

- `DependencyKey`를 통해서 의존성을 `return`하고 의존성을 관리하는 역할을 함.

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

- `currentDependency`
    - 현재 `DependencyKey`를 통해 사용중인 의존성
- `subscript`
    - `DependencyKey`를 통해서 storage에 저장된 의존성 탐색 및 접근
- `storage`
    - `DependencyKey`를 통해서 `storage`에 의존성 저장

### @Dependency 적용

```swift
// ✅ "import Dependencies" 과정은 필수!
// 물론 "import ComposableArchitecture"를 통해 대체될 수 있음
import Dependencies

public struct UserDefaultsClient {
    public var boolForKey: @Sendable (String) -> Bool
    public var dataForKey: @Sendable (String) -> Data?
    public var doubleForKey: @Sendable (String) -> Double
    public var integerForKey: @Sendable (String) -> Int
    public var remove: @Sendable (String) async -> Void
    public var setBool: @Sendable (Bool, String) async -> Void
    public var setData: @Sendable (Data?, String) async -> Void
    public var setDouble: @Sendable (Double, String) async -> Void
    public var setInteger: @Sendable (Int, String) async -> Void
    /* code */
}
```

- `UserDefaults`를 통해서 데이터를 저장, 삭제, 탐색하는 역할을 수행하는 `UserDefaultsClient`

```swift
extension UserDefaultsClient: DependencyKey {
    public static let liveValue: Self = {
        let defaults = { UserDefaults(suiteName: "group.isowords")! }
        return Self(
            boolForKey: { defaults().bool(forKey: $0) },
            dataForKey: { defaults().data(forKey: $0) },
            doubleForKey: { defaults().double(forKey: $0) },
            integerForKey: { defaults().integer(forKey: $0) },
            remove: { defaults().removeObject(forKey: $0) },
            setBool: { defaults().set($0, forKey: $1) },
            setData: { defaults().set($0, forKey: $1) },
            setDouble: { defaults().set($0, forKey: $1) },
            setInteger: { defaults().set($0, forKey: $1) }
        )
    }()
}
```

- `UserDefaultsClient`가 `DependencyKey`를 준수하여야 한다.

```swift
extension DependencyValues {
    public var userDefaults: UserDefaultsClient {
        get { self[UserDefaultsClient.self] }
        set { self[UserDefaultsClient.self] = newValue }
    }
}
```

- `DependencyValues`의 `extension`에서 `UserDefaultsClient`를 가져오는 계산 프로퍼티 선언
- `UserDefaultsClient`를 `subscript` Key로 사용함.

## Overriding Dependency

특정 부분(Preview, Test…)에서 서로 다른 종속성을 사용할 수 있도록 런타임 시점에 종속성를 변경하는 방법.

```swift
struct ImageApiClient {
    // 임의의 Image Data를 얻어오는 Api요청을 담당할 프로퍼티
    var getRandomImage: () async throws -> Data
}

extension ImageApiClient: DependencyKey {
    static var liveValue = ImageApiClient(
        getRandomImageData: {
            let urlString = "<https://picsum.photos/200/300>"
            guard let url = URL(string: urlString)
            else { throw URLError(.badURL) }
            
            if let data = try? Data(contentsOf: url) {
                return data
            }
            throw URLError(.badServerResponse)
        }
    )
}
```

- `ImageApiClient`는 이미지 관련 API Client임.
- `DependencyKey`를 준수하고 있음.

```swift
extension DependencyValues {
    var imageApiClient: Self {
        get { self[ImageApiClient.self] }
        set { self[ImageApiClient.self] = newValue }
    }
}
```

- `DependencyValues`에 `imageApiClient` 생성

```swift
struct PracticeReducer: Reducer {
    struct State: Equatable {
        var imageData: Data
        var isRequestingImage = false
    }

    enum Action: Equatable {
        case getImageButtonTapped
        case imageDataResponse(Data)
    }

    @Dependency(\.imageApiClient) var imageApiClient

    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .getImageButtonTapped:
          state.isRequestingImage = true
          return .run { send in
            let imageData = try await imageApiClient.getRandomImageData()
            await send(.imageDataResponse(imageData))
          }

        case .imageDataResponse(let newImageData):
          state.isRequestingImage = false
          state.imageData = newImageData
            return .none
        }
    }
}
```

- `getImageButton`을 누르면 `imageApiClient`로 이미지를 가져오는 `Reducer`
- 만약, Test를 위해서 실제로 api 요청을 하는 것이 아니라, mock 데이터를 사용해야 한다면?

```swift
func testRandomImageData() async {
    let dummyData = Data(count: 0)
    let store = TestStore(initialState: PracticeReducer.State()) {
        PracticeReducer()
    } withDependencies: {
        $0.imageApiClient.getRandomImageData = { dummyData }
    }

    await store.send(.getImageButtonTapped) {
        $0.isRequestingImage = true
    }
    await store.receive(.imageDataResponse(dummyData)) {
        $0.imageData = dummyData
        $0.isRequestingImage = false
    }
}
```

### ObservableObject에서 Dependency사용

- TCA를 사용하지 않고 `ObservableObject`를 사용한 경우에도 TCA Dependency를 활용할 수 있음.

```swift
final class AppModel: ObservableObject {
    @Published var onboardingTodos: TodosModel?
    
    func tutorialButtonTapped() {
        self.onboardingTodos = withDependencies(from: self) {
            $0.apiClient = .mock
            $0.fileManager = .mock
            $0.userDefaults = .mock
        } operation: {
            TodosModel()
        }
    }
}
```

- `withDependencies`함수내에서 `apiClient`, `fileManager`, `userDefaults`는 mock 객체로 재정의되어, `TodosModel`은 실제 네트워킹등의 Side Effect가 발생하지 않도록 구성되어 있다.
- 다만, 위와 같이 새로 정의된 종속성을 자식 모델까지 전파하고 싶다면, 생성된 모든 하위 모델이 `withDependencies(from:Operation:file:line:)` 함수내에서 생성되어야 한다.
    - `DependencyValue._current`가 `@TaskLocal`로 래핑되어 특정 스코프에서만 값 변화가 유효하기 떄문임.

```swift
final class TodosModel: ObservableObject {
    @Published var todos: [Todo] = []
    @Published var editTodo: EditTodoModel?
    
    @Dependency(\.apiClient) var apiClient
    @Dependency(\.fileManager) var fileManager
    @Dependency(\.userDefaults) var userDefaults
    
    func tappedTodo(_ todo: Todo) {
        // 다시 liveValue를 참조함
        self.editTodo = EditTodoModel(todo: todo)
    }
}
```

- 위와 같이 그냥 하위 모델을 생성하게 되면 `liveValue`를 참조한다.
- 상위 모델에서 `withDependencies`를 통해 설정한 종속성을 하위 모델에도 동일하게 참조하려면 아래와 같이 `withDependencies`함수 내에서 하위 모델을 생성한다.

```swift
// withDependencies 함수내에서 자식 모델을 생성했으므로 기존 종속성을 유지함.
func tappedTodo(_ todo: Todo) {
    self.editTodo = withDependencies(from: self) {
        EditTodoModel(todo: todo)
    }
}
```

## Dependency의 LifeTime

### @TaskLocal

- 어플리케이션 모든 곳에 값을 전달하는 방법중 하나로 쓰임
- 전역변수와 비슷하지만 아래와 같은 차별점을 가짐

<aside>
💡 TaskLocal은 동시 컨텍스트에서 사용할 때, `안전성`을 보장함.

- 여러 작업이 경쟁 조건에 대한 걱정 없이 로컬에서 동일한 작업에 액세스할 수 있다.

TaskLocal은 정의된 `특정 범위(scope)에서만 변경`할 수 있음.

- 애플리케이션의 모든 부분이 변경 사항을 관찰하는 방식으로 값을 변경하는 것은 허용되지 않는다.

TaskLocal은 기존 Task에서 생성된 `Task에 의해 상속`됩니다.

</aside>

예시

```swift
enum Locals {
    @TaskLocal static var value = 1
}

print(Locals.value)  // value: 1

Locals.$value.withValue(42) {
    print(Locals.value)  // value: 42
    
}
print(Locals.value)  // value: 1
```

- `Locals`는 `@TaskLocal` 프로퍼티 래퍼로 래핑된 `value`라는 `static` 변수를 가지고 있음.
- `withValue`함수를 통해서만 `value`를 변경할 수 있으며, 변경된 내용은 해당 클로저(non-escaping) 내에서만 유효함.
    - `@TaskLocal`로 래핑된 변수는 get-only property임.
- 아무데서나 직접 값을 수정할 수 있다면?
    - 값을 캡쳐하는 시점마다 값이 변화될 수 있고, 결과를 예상하기 어려워질 수 있음.

```swift
enum Locals {
    @TaskLocal static var value = 1
}

print(Locals.value)  // 1

Locals.$value.withValue(42) {
    print(Locals.value)  // 42

    Task {
        try await Task.sleep(for: .seconds(1))
        print(Locals.value)  // 42
    }

    print(Locals.value)  // 42
}
```

- 작업 로컬 상속을 통해서 withValue내에서 생성된 Task 또는 TaskGroup에도 TaskLocal을 상속함.
    - 심지어, withValue블록이 끝나고 Task의 탈출 클로저가 수행돼도 TaskLocal을 상속한 상태이므로 변경된 값이 참조됨.
- Task, TaskGroup에서만 TaskLocal이 상속되고 일반적으론 withValue scope를 벗어나면 상속되지 않음.
    - 예를 들어, 아래 코드와 같이 DispatchQueue.main.asyncAfter를 사용한 경우 탈출클로저에서 변경한 value의 값을 상속받지 못함.
    
    ```swift
    print(Locals.value)  // 1
    
    Locals.$value.withValue(42) {
        print(Locals.value)  // 42
    
        DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
            print(Locals.value)  // 1
    
        }
        print(Locals.value)  // 42
    }
    ```
    
- @Dependency의 withDependencies가 @TaskLocal의 withValue과 동일한 개념이다.

### withDependencies의 여러가지 사용법

withDependencies의 후행 non-escaping 클로저에서 변경된 의존성을 유지한채로 어떤 작업이 가능하다.

```swift
class FeatureModel: ObservableObject {
    @Dependency(\.apiClient) var apiClient

    func onAppear() async {
        do {
            self.user = try await self.apiClient.fetchUser()
        } catch {
            /* code */
        }
    }
}
```

- 위 모델에서 테스트를 하는 상황을 생각해본다면, apiClient 실제로 api를 요청하는 것이 아니라 mock 객체를 반환하는 방식으로 구현이 필요하다.
1. withDependency(_:Operation:) → model 반환 ❌
    
    ```swift
    func testOnAppear() async {
        await withDependencies {
            // (1) 통제된 테스트를 위해 특정 종속성을 재정의할 수 있는 Scope
            $0.apiClient.fetchUser = { _ in User(id: 42, name: "Blob") }
        } operation: {
            // (2) 종속성이 재정의된 범위안에서 로직테스트를 진행
            let model = FeatureModel()
            XCTAssertEqual(model.user, nil)
            await model.onAppear()
            XCTAssertEqual(model.user, User(id: 42, name: "Blob"))
        }
    }
    ```
    
    - withDependencies의 첫번째 클로저
        - 종속성 재정의
    - operation 클로저
        - 종속성이 재정의 된 상태에서 로직을 테스트 함
2. withDependency(_:Operation:) → model 반환 ⭕
    
    ```swift
    func testOnAppear() async {
        let model = withDependencies {
            $0.apiClient.fetchUser = { _ in User(id: 42, name: "Blob") }
        } operation: {
            FeatureModel()
        }
        
        XCTAssertEqual(model.user, nil)
        await model.onAppear()
        XCTAssertEqual(model.user, User(id: 42, name: "Blob"))
    }
    ```
    
    - 모든 테스트 코드를 operation에 담을수는 없음.
    - operation에서 FeatureModel 객체를 return하도록 해서 테스트 의존성을 가지고 있는 model을 생성하여 사용하는 방법도 있음.
    - 이런 방식으로 model을 생성하면 model내부의 모든 apiClient 종속성은 mock을 참조하게 됨.
3. withDependency(from:Operation:file:line:)
    
    ```swift
    let onboardingModel = withDependencies(from: self) {
        $0.apiClient = .mock
    } operation: {
        FeatureModel()
    }
    ```
    
    - 상위 모델에서 하위 모델을 생성하는 경우
        - from: self를 통해서 상위 모델의 종속성을 하위 모델에 상속할 수 있음
    - 클로저에서는 추가로 재정의할 종속성을 정의할 수 있음.
