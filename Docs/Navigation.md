# Navigation

# Navigation이란

- 일반적으로 SwiftUI의 `NavigationStack`과 UIKit의 `UINavigationController`를 떠올릴 수 있음.
    - drill-down 스타일
- drill-down 스타일이 Navigation이라면 sheet나 fullscreen covers View 또한 Navigation 이라고 할 수 있음.
    - 화면 전환이 옆으로 되느냐 위 아래로 되느냐의 차이 정도.
    - popovers, alert, confirmation dialogs 또한 Navigation이라고 볼 수 있다.

<aside>
💡 즉, Navigation은 앱에서 모드를 변경하는 것으로 볼 수 있다.
모드 변경이란 어떤 상태가 새로 생기거나, 사라지는 것을 의미한다.

</aside>

### TCA에서의 Navigation

- TCA는 State와 Action을 가지고 Navigation을 처리하므로, SwiftUI의 명령형 방식으로 Navigation을 처리할 필요가 없다.
    - 상태의 존재 or 비 존재에 의해서 Navigation이 동작
- 상태 기반의 Navigation에는 2가지 주요한 형태가 있다.
    1. 트리 기반 Navigation
    2. 스택 기반 Navigation

## 트리 기반 Navigation

```swift
struct InventoryFeature: Reducer {
    struct State {
        @PresentationState var detailItem: DetailItemFeature.State?
        /* code */
    }
    /* code */
}

struct DetailItemFeature: Reducer {
    struct State {
        @PresentationState var editItem: EditItemFeature.State?
        /* code */
    }
    /* code */
}

struct EditItemFeature: Reducer {
    struct State {
        @PresentationState var alert: AlertState<AlertAction>?
        /* code */
    }
    /* code */
}

InventoryView(
    store: Store(
        initialState: InventoryFeature.State(
            detailItem: DetailItemFeature.State(      // Drill-down to detail screen
                editItem: EditItemFeature.State(        // Open edit modal
                    alert: AlertState {                   // Open alert
                        TextState("This item is invalid.")
                    }
                                               )
                                               )
        )
    ) {
        InventoryFeature()
    }
)
```

- `Inventory` → `Detail` → `Edit` 순서로 drill-down 형태의 Navigation을 구성한 예제.
    - 인벤토리에서 특정 Item의 디테일을 확인할 수 있고, 디테일에서는 해당 Item을 수정할 수도 있다.
- 각 Feature의 State에 `@PresentaionState` 프로퍼티 래퍼로 선언된 하위 State을 가지고 있다.
    - `InventoryFeature.State` ← `DetailItemFeature.State` ← `EditItemFeature.State`

## 스택 기반 Navigation

- State들을 스택에 넣고 빼는 방식으로 Navigation을 하는 방식.
- 일반적으로 스택에 들어갈 수 있는 모든 케이스를 정의한 `enum`을 사용한다.

```swift
enum Path {
    case detail(DetailItemFeature.State)
    case edit(EditItemFeature.State)
    /* code */
}

let path: [Path] = [
    .detail(DetailItemFeature.State(item: item)),
    .edit(EditItemFeature.State(item: item)),
    /* code */
]
```

- `Path`
    - 스택에 들어갈 수 있는 케이스를 정의함.
- `path`라는 스택에 상태를 추가
    - 현재 State → Detail → Edit 순으로 drill-down

## 트리 기반 Navigation 🆚 스택 기반 Navigation

### 트리 기반 Navigation의 장점

- 앱에서 가능한 Navigation을 정적으로 관리할 수 있으므로, 유효하지 않은 Navigation을 사용하지 못함.
    
    ```swift
    struct State {
        @PresentationState var editItem: EditItemFeature.State?
        /* code */
    }
    ```
    
    - 정적으로 선언된 형태이기 때문에 State에서는 Edit 이외의 Navigation을 실행할 수 없음.
- Navigation을 정적으로 관리하므로 Navigation의 경로의 수를 제한할 수 있음.
- Preview를 사용하기 쉽다.
    - 이미 정적으로 하위 기능을 포함한 상태이기 때문에 Preview 사용에 좋다.
- 통합에 대한 단위 테스트가 용이하다.
    - 기능한 강하게 결합된 형태이기 때문에, 각 기능이 어떻게 통합되어 상호 작용을 하는지 증명하기 쉽다.

### 트리 기반 Navigation의 단점

- 복잡하거나 재귀적인 Navigation을 사용하는 것이 번거롭다.
    - 영화 정보 → 출연 배우 → 특정 배우 → 영화 정보
    - 위와 같은 경우는 재귀적인 의존성을 가지므로 좋지 않다.
- 컴파일 시간이 늦다.
    - 각 기능이 강하게 결합되어 있기 때문에, 하나를 컴파일 하기 위해서 다른 기능까지 모두 컴파일 해야되는 경우가 생길 수 있음.

### 스택 기반 Navigation의 장점

- 재귀적인 Navigation 경로를 쉽게 처리할 수 있다.
    
    ```swift
    let path: [Path] = [
        .movie(/* code */),
        .actors(/* code */),
        .actor(/* code */)
        .movies(/* code */),
        .movie(/* code */),
    ]
    ```
    
    - `movie` 기능에서 시작해서 `movie` 기능에서 끝이 나지만, `path`는 스택이기 때문에 재귀적 참조가 없다.
- 스택에 포함될 각 기능은 다른 모든 화면으로부터 완벽히 분리될 수 있다. 즉, 각 기능이 서로 의존성 없이 존재할 수 있고, 컴파일 타임이 빨라질 수 있다.

### 스택 기반 Navigation의 단점

- 완전히 비논리적인 Navigation 경로를 표현할 수 있어짐.
    
    ```swift
    let path: [Path] = [
        .edit(/* code */),
        .detail(/* code */)
    ]
    
    let path: [Path] = [
        .edit(/* code */),
        .edit(/* code */),
        .edit(/* code */),
    ]
    ```
    
    - Edit → Detail 경로는 말이 안됨
    - Edit → Edit → Edit 경로도 말이 안됨
- Preview 사용이 제한됨
    - 스택 기반의 Navigation에서는 기능을 모듈화하면 완벽히 분리가 가능하기 때문에, Preview 사용시 다른 기능 사용을 못하게 될 수 있음.
    - 앱 전체를 빌드해야 동작 확인이 가능해짐.
- 통합에 대한 단위 테스트가 힘들어진다.
    - 기능간 완벽한 분리로 통합에 대한 테스트가 어려워진다.

# 트리 기반 Navigation 살펴보기

- 옵셔널과 열거형 상태를 사용하여 Navigation을 모델링 함.
- State가 깊은 영역까지 중첩되어 있으므로 간편하게 구성될 수 있고, State의 구성에 따른 실제 Navigation은 SwiftUI가 담당하도록 함.

### 1. 기능의 통합

```swift
@Reducer
struct InventoryFeature {
    @ObservableState
    struct State: Equatable {
        @Presents var addItem: ItemFormFeature.State?
        var items: IdentifiedArrayOf<Item> = []
    }
    
    enum Action: Equatable {
        case addButtonTapped
        case addItem(PresentationAction<ItemFormFeature.Action>)
    }
    
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .addButtonTapped:
                state.addItem = .init()
            case .addItem(let presentationAction):
                switch presentationAction {
                case .dismiss:
                    print("Child dismissed")
                case .presented(let formAction):
                    print("Child action received:", formAction)
                }
            }
            return .none
        }.ifLet(\.$addItem, action: \.addItem) {
            ItemFormFeature()
        }
    }
}

@Reducer
struct ItemFormFeature {
    struct State: Equatable {}
    enum Action: Equatable {
        case buttonTapped
    }
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        print(action)
        return .none
    }
}

struct Item: Identifiable, Equatable {
    var id: String = ""
}
```

- 아이템을 보여주는 인벤토리에서 새 아이템을 추가하는 flow임.
- State가 `@ObservableState`로 선언된 경우에는 자식 State에 `@Presents`를 추가함.
    - State가 그냥 선언된 경우엔 자식 State에 `@PresentationState`를 추가함.
- `State.addItem`이 `nil`이 아닌 경우 `ifLet` 구문에서 `ItemFormFeature` 객체를 생성하여 액션을 처리하도록 함.
    - 새로운 scope의 탄생.
    - 부모와 자식의 Reducer를 통합함.

### 2. View의 통합

```swift
struct InventoryView: View {
    @State var store: StoreOf<InventoryFeature>
    
    var body: some View {
        Button("add", action: {
            store.send(.addButtonTapped)
        })
        .sheet(
            item: $store.scope(state: \.addItem, action: \.addItem)
        ) { store in
            ItemFormView(store: store)
        }
    }
}

struct ItemFormView: View {
    let store: StoreOf<ItemFormFeature>
    
    var body: some View {
        Button("button", action: {
            store.send(.buttonTapped)
        })
    }
}
```

- `sheet` 함수를 통해서 scope를 감시하고 상태에 따라 `ItemFormView`를 띄움.
    - `addItem` State가 nil 이 아닌 경우에 `sheet` 표시됨.
- TCA에는 sheet 뿐 아니라 다른 오버 로딩도 많음.
    - `alert(store:)`
    - `confirmationDialog(store:)`
    - `sheet(store:)`
    - `popover(store:)`
    - `fullScreenCover(store:)`
    - `navigationDestination(store:)`
    - `NavigationLinkStore`

### Optional State를 사용한 Navigation의 한계

```swift
struct State {
    @PresentationState var detailItem: DetailFeature.State?
    @PresentationState var editItem: EditFeature.State?
    @PresentationState var addItem: AddFeature.State?
    /* code */
}
```

- 하나의 기능에서 여러 화면으로 Navigation이 가능할 때 생기는 문제.
    - 한 번에 2개의 Navigation이 동작하는 문제가 있을 수 있음. 즉, 뭐가 어떻게 떠있는지 파악이 어려움.
    - SwiftUI에서는 단일 View에 여러개의 View를 동시에 표시하는 것을 지원하지 않음.
- 열거형 State를 선언함으로써 해결 가능함.

## 열거형 State

- 옵셔널 State를 활용해서 Navigation을 하는 방법은 위와 같은 한계가 있음.
- State를 열거형으로 선언하면 한 번에  1가지 상태만 가질 수 있으므로 한계 해결 가능.

```swift
struct InventoryView: View {
    @State var store: StoreOf<InventoryFeature>
    
    var body: some View {
        VStack {
            Button("add", action: {
                store.send(.addButtonTapped)
            })
            
            Button("edit", action: {
                store.send(.editButtonTapped)
            })
            
            Button("detail", action: {
                store.send(.detailButtonTapped)
            })
        }
        .sheet(
            item: $store.scope(
                state: \.destination?.addItem,
                action: \.destination.addItem
            )
        ) { store in
            AddView(store: store)
        }
        .popover(
            item: $store.scope(
                state: \.destination?.editItem,
                action: \.destination.editItem
            )
        ) { store in
            EditView(store: store)
        }
        .fullScreenCover(
            item: $store.scope(
                state: \.destination?.detailItem,
                action: \.destination.detailItem
            )
        ) { store in
            DetailView(store: store)
        }
    }
}

@Reducer
struct InventoryFeature {
    @ObservableState
    struct State: Equatable {
        @Presents var destination: Destination.State?
        var items: IdentifiedArrayOf<Item> = []
    }
    
    enum Action: Equatable {
        case addButtonTapped
        case editButtonTapped
        case detailButtonTapped
        
        case destination(PresentationAction<Destination.Action>)
    }
    
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .addButtonTapped:
                state.destination = .addItem(.init())
                
            case .editButtonTapped:
                state.destination = .editItem(.init())
                
            case .detailButtonTapped:
                state.destination = .detailItem(.init())
                
            case .destination(let presentationAction):
                switch presentationAction {
                case .dismiss:
                    print("Child dismissed")
                case .presented(let destinationAction):
                    switch destinationAction {
                    case .addItem(let addAction):
                        print(addAction)
                    case .detailItem(let detailAction):
                        print(detailAction)
                    case .editItem(let editAction):
                        print(editAction)
                    }
                }
            }
            return .none
        }.ifLet(\.$destination, action: \.destination) {
            Destination()
        }
    }
    
    @Reducer
    struct Destination {
        enum State: Equatable {
            case addItem(AddFeature.State)
            case detailItem(DetailFeature.State)
            case editItem(EditFeature.State)
        }
        
        enum Action: Equatable {
            case addItem(AddFeature.Action)
            case detailItem(DetailFeature.Action)
            case editItem(EditFeature.Action)
        }
        
        var body: some ReducerOf<Self> {
            Scope(state: \.addItem, action: \.addItem) {
                AddFeature()
            }
            Scope(state: \.editItem, action: \.editItem) {
                EditFeature()
            }
            Scope(state: \.detailItem, action: \.detailItem) {
                DetailFeature()
            }
        }
    }
}

#Preview {
    InventoryView(store: .init(initialState: .init(), reducer: { InventoryFeature() }))
}
```

- `InventoryFeature`내에 `Destination` Reducer를 생성
    - 일반적으로 해당 부모 Reducer 내부에 직접 선언하는 것이 좋다.
- `Destination`의 `State`를 열거형으로 선언하고, `InventoryFeature.State`에는 `Destination.State`를 옵셔널로 선언(`destination`).
    - 자식의 State를 직접 참조하지 않음.
    - `Destination.State`를 직접 참조하고, `Destination.State`는 `enum`이므로 `destination`은 한가지의 State만 가질 수 있게 됨.
    - 실제 자식 State에는 `enum`의 associated value로 접근함.
- `InventoryFeature`에서 각 자식의 Action에 반응하여 뭔가를 처리할 수도 있음.
    - 호출 순서는 자식의 Reducer 다음임.
- View단에서 해당 `destination`을 여러가지 방식으로 Navigation 할 수 있음.

<aside>
💡 포인트는, enum으로 선언된 자식 State 사용으로 트리 기반 Navigation의 한계를 극복할 수 있다는 점이다.

</aside>

## Dismissal

```swift
case .closeButtonTapped:
    state.destination = nil
    return .none
```

- 위와 같이 부모 Reducer에서 자식 State를 nil로 만들어주면 해당 자식은 dismiss 됨.
- 하지만 자기 자신을 dismiss 하고 싶은 경우는?

### @Dependency(\.dismiss)

```swift
struct Feature: Reducer {
    struct State { /* code */ }
    enum Action {
        case closeButtonTapped
        /* code */
    }
    @Dependency(\.dismiss) var dismiss
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .closeButtonTapped:
            return .run { _ in await dismiss() }
        }
    }
}
```

- 자기 자신을 dismiss 하는 경우에는 `dismiss` Dependency를 사용함.

```swift
return .run { send in
    await self.dismiss()
    await send(.tick)  // Warning
}
```

- dismiss 한 이후에 `send`를 호출하면 런타임 경고가 발생함.
    - dismiss 한 이후에는 Action을 실행할 수 없으므로..

## 트리 기반 Navigation Testing

### Test 코드 작성할 Reducer

```swift
@Reducer
struct Feature {
    struct State: Equatable {
        @PresentationState var counter: CounterFeature.State?
    }
    enum Action: Equatable {
        case counter(PresentationAction<CounterFeature.Action>)
    }
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            return .none
        }
        .ifLet(\.$counter, action: \.counter) {
            CounterFeature()
        }
    }
}

@Reducer
struct CounterFeature {
    struct State: Equatable {
        var count = 0
    }
    enum Action: Equatable {
        case decrementButtonTapped
        case incrementButtonTapped
    }
    
    @Dependency(\.dismiss) var dismiss
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .decrementButtonTapped:
            state.count -= 1
            return .none
            
        case .incrementButtonTapped:
            state.count += 1
            return state.count >= 5
            ? .run { _ in await dismiss() }
            : .none
        }
    }
}
```

- `Feature`: 부모 Reducer
- `CounterFeature`: 자식 Reducer. 카운트를 올리거나 내리는 역할을 하고, 카운트가 5이상이면 자기 자신을 dismiss 함.

### Test Code

```swift
@MainActor
func testDismissal() async {
    let store = TestStore(
        initialState: Feature.State(
            counter: CounterFeature.State(count: 3)
        )
    ) {
        Feature()
    }
    
    await store.send(.counter(.presented(.incrementButtonTapped))) { state in
        state.counter?.count = 4
    }
    await store.send(.counter(.presented(.incrementButtonTapped))) { state in
        state.counter?.count = 5
    }
    await store.receive(.counter(.dismiss)) { state in
        state.counter = nil
    }
}
```

- `TestStore` 생성
    - `CounterFeature`를 처음부터 주입해줌.
    - `count` 값은 3.
- `incrementButtonTapped` 할 때 마다 `state.counter.count`가 1씩 증가함.
- `state.counter.count`가 5가 된 시점에는 `.counter(.dismiss)`가 호출되었을 것으로 예상되므로, `receive`함수를 사용해서 `state.counter`가 `nil`이 되었는지 확인.
    - 만약 `.counter(.dismiss)`가 수신되지 않았다면 테스트 실패.

```swift
@MainActor
func testDismissal2() async {
    let store = TestStore(
        initialState: Feature.State(
            counter: CounterFeature.State(count: 3)
        )
    ) {
        Feature()
    }
    
    store.exhaustivity = .off
    
    await store.send(.counter(.presented(.incrementButtonTapped)))
    await store.send(.counter(.presented(.incrementButtonTapped)))
    await store.receive(.counter(.dismiss))
}
```

- `exhaustivity` 변수를 `off`로 설정해서 State 검증은 하지 않을수도 있음.
    - 비완전한 테스트

```swift
@Reducer
struct Feature {
    @ObservableState
    struct State: Equatable {
        @Presents var destination: Destination.State?
    }
    @CasePathable
    enum Action: Equatable {
        case destination(PresentationAction<Destination.Action>)
    }
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            return .none
        }
        .ifLet(\.$destination, action: \.destination) {
            Destination()
        }
    }
    
    @Reducer
    struct Destination {
        @CasePathable
        enum State: Equatable {
            case counter(CounterFeature.State)
            case counter2(CounterFeature2.State)
        }
        
        @CasePathable
        enum Action: Equatable {
            case counter(CounterFeature.Action)
            case counter2(CounterFeature2.Action)
        }
        
        var body: some ReducerOf<Self> {
            Scope(state: \.counter, action: \.counter) {
                CounterFeature()
            }
            
            Scope(state: \.counter2, action: \.counter2) {
                CounterFeature2()
            }
        }
    }
}

@Reducer
struct CounterFeature {
    struct State: Equatable {
        var count = 0
    }
    enum Action: Equatable {
        case decrementButtonTapped
        case incrementButtonTapped
    }
    
    @Dependency(\.dismiss) var dismiss
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .decrementButtonTapped:
            state.count -= 1
            return .none
            
        case .incrementButtonTapped:
            state.count += 1
            return state.count >= 5
            ? .run { _ in await dismiss() }
            : .none
        }
    }
}

@Reducer
struct CounterFeature2 {
    struct State: Equatable {
        var count = 0
    }
    enum Action: Equatable {
        case decrementButtonTapped
        case incrementButtonTapped
    }
    
    @Dependency(\.dismiss) var dismiss
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .decrementButtonTapped:
            state.count -= 1
            return .none
            
        case .incrementButtonTapped:
            state.count += 1
            return state.count >= 5
            ? .run { _ in await dismiss() }
            : .none
        }
    }
}
```

- 위와 같이 State가 열거형으로 선언된 경우엔?

```swift
@MainActor
func testModify1() async {
    let store = TestStore(
        initialState: Feature.State(destination: .counter(.init(count: 3)))
    ) {
        Feature()
    }
    
    await store.send(.destination(.presented(.counter(.incrementButtonTapped)))) {
        $0.destination?.modify(\.counter, yield: { state in
            state.count = 4
        })
    }
}

@MainActor
func testModify2() async {
    let store = TestStore(
        initialState: Feature.State(destination: .counter2(.init(count: 3)))
    ) {
        Feature()
    }
    
    await store.send(.destination(.presented(.counter2(.incrementButtonTapped)))) {
        $0.destination?.modify(\.counter2, yield: { state in
            state.count = 4
        })
    }
}
```

- `CasePathable`의 `modify`를 통해 특정 State 케이스(`counter` or `counter2`)에 접근해서 State 값을 expected 값으로 설정하여 검증함.

# 스택 기반 Navigation 살펴보기

- State를 컬렉션에 담아서 Navigation을 모델링하는 방식
- 단순한 1차원 데이터 컬렉션으로 앱의 어느 상태든지 직접 접근할 수 있는 딥링크를 가능하게 함.
- 복잡하고 재귀적인 Navigation 경로를 사용할 수 있음.

### 1. 기능의 통합

```swift
@Reducer
struct RootFeature {
    @ObservableState
    struct State {
        var path = StackState<Path.State>()
        var title = "RootView"
    }
    
    @CasePathable
    enum Action {
        case path(StackAction<Path.State, Path.Action>)
        case drillDown
    }
    
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .path(let stackAction):
                switch stackAction {
                default:
                    break
                }
                
            case .drillDown:
                state.path.append(
                    contentsOf: [
                        .addItem(.init()),
                        .detailItem(.init()),
                        .editItem(.init())
                    ]
                )
            }
            return .none
        }
        .forEach(\.path, action: \.path) {
            Path()
        }
    }
    
    @Reducer
    struct Path {
        @ObservableState
        enum State {
            case addItem(AddFeature.State)
            case detailItem(DetailFeature.State)
            case editItem(EditFeature.State)
        }
        
        @CasePathable
        enum Action {
            case addItem(AddFeature.Action)
            case detailItem(DetailFeature.Action)
            case editItem(EditFeature.Action)
        }
        
        var body: some ReducerOf<Self> {
            Scope(state: \.addItem, action: \.addItem) {
                AddFeature()
            }
            Scope(state: \.editItem, action: \.editItem) {
                EditFeature()
            }
            Scope(state: \.detailItem, action: \.detailItem) {
                DetailFeature()
            }
        }
    }
}
```

- `RootFeature.State`에 `NavigtaionStack`에 쌓일 `Path`들을 담은 `StackState`변수(`path`)를 가지고 있음.
- `Path`는 `RootFeature`내에 선언되어 있으며, Navigation이 가능한 모든 케이스를 열거형으로 정의함.
    - 기본적으로 트리 기반 Navigation에서 봤던 `Destination`과 동일함.
- `RootFeature`의 `body`에서는 `StackState`에 쌓인 `Path`들에 사용할 Reducer들을 생성함.
    - `forEach` 함수 사용.
- drillDown 액션 전송시에 Add → Detail → Edit 경로로 Navigation 됨.
- `RootFeature`에서 `Path`들의 Action을 통합하여 처리할 수 있음.

### 2. View의 통합

```swift
struct RootView: View {
    @State var store: StoreOf<RootFeature>
    
    var body: some View {
        NavigationStack(
            path: $store.scope(state: \.path, action: \.path)
        ) {
            VStack {
                Text(store.title)
                
                Button("DeepLink: Add -> Detail -> Edit") {
                    store.send(.drillDown)
                }
            }
        } destination: { store in
            switch store.state {
            case .addItem:
                if let store = store.scope(state: \.[case: \.addItem], action: \.addItem) {
                    TitleView(title: "add item")
                }
                
            case .detailItem:
                if let store = store.scope(state: \.[case: \.detailItem], action: \.detailItem) {
                    TitleView(title: "detail item")
                }
                
            case .editItem:
                if let store = store.scope(state: \.[case: \.editItem], action: \.editItem) {
                    TitleView(title: "edit item")
                }
            }
        }
    }
}
```

- `NavigationStack`을 가지고 특정 scope의 Stack에 있는 모든 뷰들을 표현함.
    - `path`: 참조할 Path의 scope
    - `root`: Root에 표시할 뷰 빌더
    - `destination`: Navigation시 보여줄 View
        - 각 Scope(Add, Edit…)별로 store가 존재하면 해당 뷰 반환함.

## Dismissal

```swift
case .closeButtonTapped:
    state.popLast()
    return .none
```

- `state.popLast()` 하면 마지막에 push된 화면이 dismiss됨
- 자기 자신을 dismiss 하고 싶으면?

```swift
struct Feature: Reducer {
    struct State { /* code */ }
    enum Action {
        case closeButtonTapped
        /* code */
    }
    @Dependency(\.dismiss) var dismiss
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .closeButtonTapped:
            return .run { _ in await dismiss() }
        }
    }
}
```

- 트리 기반 Navigation에서와 동일하게 `@Dependency(\.dismiss)`를 사용함.

```swift
return .run { send in
    await self.dismiss()
    await send(.tick)  // Warning
}
```

- dismiss 이후에는 Action 전송에 실패하는 것도 동일함.

### StackState 🆚 NavigationPath

- SwiftUI에서는 이미 스택 기반의 Navigation을 지원할 수 있는 `NavigationPath`라는 타입이 존재함.
    
    ```swift
    var path = NavigationPath()
    path.append(1)
    path.append("Hello")
    path.append(false)
    
    struct RootView: View {
        @State var path = NavigationPath()
        
        var body: some View {
            NavigationStack(path: self.$path) {
                Form {
                    /* code */
                }
                .navigationDestination(for: Int.self) { integer in
                    /* code */
                }
                .navigationDestination(for: String.self) { string in
                    /* code */
                }
                .navigationDestination(for: Bool.self) { bool in
                    /* code */
                }
            }
        }
    }
    ```
    
    - navigation destination을 타입으로 구분함.
- `StackState`와 유사하지만 Element 선언 및 사용성에서 차이가 있음.
    - `StackState`의 Element: 정적인 타입(ex. `Path.State`)
    - `NavigationPath`의 Element: type-erased 된 타입(`Hashable`만 준수하면 됨.)
- **NavigationPath의 단점**
    1. type-erased에서 오는 단점들..
        - 타입이 명확하지 않기 때문에, 지원하는 API가 많지 않다.
        - `removeLast()`, `count`, `isEmpty` 정도 지원함.
    2. 순회 불가능
        - `NavigationPath`는 `Sequence` 프로토콜을 준수하지 않으므로 요소들을 순회하는 것도 불가능
            
            ```swift
            let path: NavigationPath = …
            for element in path {  // 🛑 For-in loop requires 'NavigationPath' to conform to 'Sequence'
            }
            ```
            

## 스택 기반 Navigation Testing

### Test 코드 작성할 Reducer(트리 기반과 거의 동일)

```swift
@Reducer
struct Feature {
    @ObservableState
    struct State: Equatable {
        var path = StackState<Path.State>()
    }
    @CasePathable
    enum Action: Equatable {
        case path(StackAction<Path.State, Path.Action>)
    }
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            return .none
        }
        .forEach(\.path, action: \.path) {
            Path()
        }
    }
    
    @Reducer
    struct Path {
        @CasePathable
        enum State: Equatable {
            case counter(CounterFeature.State)
            case counter2(CounterFeature2.State)
        }
        
        @CasePathable
        enum Action: Equatable {
            case counter(CounterFeature.Action)
            case counter2(CounterFeature2.Action)
        }
        
        var body: some ReducerOf<Self> {
            Scope(state: \.counter, action: \.counter) {
                CounterFeature()
            }
            
            Scope(state: \.counter2, action: \.counter2) {
                CounterFeature2()
            }
        }
    }
}

@Reducer
struct CounterFeature {
    struct State: Equatable {
        var count = 0
    }
    enum Action: Equatable {
        case decrementButtonTapped
        case incrementButtonTapped
    }
    
    @Dependency(\.dismiss) var dismiss
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .decrementButtonTapped:
            state.count -= 1
            return .none
            
        case .incrementButtonTapped:
            state.count += 1
            return state.count >= 5
            ? .run { _ in await dismiss() }
            : .none
        }
    }
}

@Reducer
struct CounterFeature2 {
    struct State: Equatable {
        var count = 0
    }
    enum Action: Equatable {
        case decrementButtonTapped
        case incrementButtonTapped
    }
    
    @Dependency(\.dismiss) var dismiss
    
    func reduce(into state: inout State, action: Action) -> Effect<Action> {
        switch action {
        case .decrementButtonTapped:
            state.count -= 1
            return .none
            
        case .incrementButtonTapped:
            state.count += 1
            return state.count >= 5
            ? .run { _ in await dismiss() }
            : .none
        }
    }
}
```

### Test Code

```swift
@MainActor
func testDismissal() async {
    let store = TestStore(
        initialState: Feature.State(
            path: StackState([
                .counter(.init(count: 3))
            ])
        )
    ) {
        Feature()
    }
    
    await store.send(.path(.element(id: 0, action: .counter(.incrementButtonTapped)))) { state in
        state.path[id: 0]?.modify(\.counter, yield: { state in
            state.count = 4
        })
        
        // $0.path[id: 0, case: \.guessMyAge]?.isGuessAgeButtonTapped = true
        // $0.path[id: 0, case: \.guessMyAge]?.age = nil
        // $0.path[id: 0, case: \.guessMyAge]?.isGuessAgeIncorrect = false
    }
    
    await store.send(.path(.element(id: 0, action: .counter(.incrementButtonTapped)))) { state in
        state.path[id: 0]?.modify(\.counter, yield: { state in
            state.count = 5
        })
    }
    
    await store.receive(.path(.popFrom(id: 0))) {
        $0.path[id: 0] = nil
    }
}
```

- `TestStore` 생성
    - `path`에 `.counter(.init(count: 3))` 1개를 가지도록 초기화.
- `StackState`는 `id`로 관리됨.
    - 해당 ID는 주로 외부에서는 볼 수 없음.
    - `TestStore`를 구성하는 경우에는 `id`가 0부터 1씩 증가하는 형태로 생성됨. 고로, 1개만 들어간 현 상태에서는 `id`가 0임.
        - ❗`path[id: 0]`가 case path와 일치하지 않아도 테스트 실패.
    - `id` 값으로 subscription 가능
- `CasePathable`의 `modify`를 통해 특정 State 케이스(`counter` or `counter2`)에 접근해서 State 값을 expected 값으로 설정하여 검증함.(트리 기반과 동일)
- `receive`함수를 통해서 특정 Action을 받았는지 확인 및 dismiss 됐는지(State가 없어졌는지) 확인도 가능.

```swift
func testDismissal() {
    let store = TestStore(
        initialState: Feature.State(
            path: StackState([
                .counter(.init(count: 3))
            ])
        )
    ) {
        Feature()
    }
    
    store.exhaustivity = .off
    
    await store.send(.path(.element(id: 0, action: .incrementButtonTapped)))
    await store.send(.path(.element(id: 0, action: .incrementButtonTapped)))
    await store.receive(.path(.popFrom(id: 0)))
}
```

- `exhaustivity` 변수를 `off`로 설정해서 state 검증은 하지 않을 수도 있음.
    - 비완전한 테스트