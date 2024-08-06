# MultiStore

### MultiStore의 강점

1. 자식 `State`의 부재를 UI에서 우아하게 처리할 수 있다.
    - 부모가 자식의 State를 Optional로 가지고 있어서 자식의 State가 없을 때에도 UI에서 적절한 처리가 가능함.
    Loading, Placeholder처럼 특정 조건에서 자식 `View`를 렌더링 해야할 때 유용함.
2. 작게 나눈 자식 `Reducer`는 독립적으로 작동하고 관리될 수 있음.
    - `Reducer`를 작게 분리하고 독립적으로 관리하는 것은 코드의 모듈성을 높여줌.
3. 특정 부분만 업데이트 하고 싶을 때, 그 부분의 작은 `store`를 통해 성능 최적화를 할 수 있음.
    - 큰 앱에서 일부만 업데이트 하는 경우 유용.

## ifLet

부모 `State`의 `Optional` 속성에서 작동하는 자식 `Reducer`를 부모 도메인이 포함하는 기능
즉, 부모의 기능이 자식 `State`를 `Optional`로 가지고 있어야 함.

### 예제

```swift
struct Parent: Reducer {
    struct State {
        var child: Child.State?
        /* code */
    }
    enum Action {
        case child(Child.Action)
        /* code */
    }
    
    var body: some Reducer<State, Action> {
        Reduce { state, action in
        }
        .ifLet(\.child, action: /Action.child) {
            Child()
        }
    }
}
```

- 부모(`Parent`)가 자식의 State(`State.child`)를 보유하고 있음.
- Reducer에 `ifLet`을 달아서 Action을 감지하고 있다가, 자식 State가 `nil`이 아닐 때 자식의 Reducer를 실행함.

### ifLet 생성자

```swift
@inlinable
@warn_unqualified_access
public func ifLet<WrappedState, WrappedAction, Wrapped: Reducer>(
    _ toWrappedState: WritableKeyPath<State, WrappedState?>,
    action toWrappedAction: CaseKeyPath<Action, WrappedAction>,
    @ReducerBuilder<WrappedState, WrappedAction> then wrapped: () -> Wrapped,
    fileID: StaticString = #fileID,
    line: UInt = #line
) -> _IfLetReducer<Self, Wrapped>
where WrappedState == Wrapped.State, WrappedAction == Wrapped.Action {
    .init(
        parent: self,
        child: wrapped(),
        toChildState: toWrappedState,
        toChildAction: AnyCasePath(toWrappedAction),
        fileID: fileID,
        line: line
    )
}
```

파라미터

- `toWrappedState`: 부모의 State에서 Optional인 자식 State에 대한 `KeyPath`
- `toWrappedAction`: 부모의 Action에서 자식의 Action에 대한 `KeyPath`
- `wrapped`: 자식 State가 nil이 아닐 때 자식 Action에 대해서 호출될 Reducer

반환값 → `_IfLetReducer<Self, Wrapped>`

- `where`절
    1. `WrappedState == Wrapped.State`
    2. `WrappedAction == Wrapped.Action`
    3. `Wrapped`(자식의 Reducer): `Reducer`
        - 자식의 Reducer가 Reducer 프로토콜을 준수

<aside>
💡 `ifLet`의 여러가지 기능

- 부모와 자식간 작업 순서를 자식먼저 실행하고, 그 다음 부모를 실행하는 방식으로 강제한다. → 순서가 뒤바뀌면 부모가 자식의 동작에 영향을 줄 수 있게 되므로..
- 자식 State가 `nil`이 된 것을 감지하면 모든 자식의 `Effect`를 취소함.
- 알림 및 확인 대화 상자에 대한 Action이 전송될 때 자동으로 자식의 State를 무효화함.
</aside>

### IfLetStore(⚠️Deprecated)

- 두 가지 뷰중 하나를 표시하기 위해 Store를 안전하게 unwrapping 함.
- State가 `nil`인지 여부에 따라 동일하게 Store도 `nil` or non `nil` 결정됨.

```swift
IfLetStore(store.scope(state: \.results, action: Child.Action.results)) {
    SearchResultsView(store: $0)
} else: {
    Text("검색 결과 로딩 중...")
}
```

- 자식의 State가 `nil`일 경우 `SearchResultsView`를 보여주고, 아닌 경우 로딩중 `Text`를 보여줌

⚠️ 현재 `IfLetStore`는 Deprecated 상태로 그냥 if let 을 사용하면 됨.

```swift
if let childStore = store.scope(state: \.child, action: \.child) {
    ChildView(store: childStore)
} else {
    Text("Nothing to show")
}
```

참조: [https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/migratingto1.7/#Replacing-IfLetStore-with-if-let](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/migratingto1.7/#Replacing-IfLetStore-with-if-let)

## forEach

- 부모 도메인에 부모 State내에 컬렉션 요소에 사용하는 자식 Reducer들을 포함시킬 때 사용함.
다시 말해, 부모 기능이 자식 State의 배열을 보유하는 경우 사용.

```swift
struct Parent: Reducer {
    struct State {
        var rows: IdentifiedArrayOf<Child.State>
        // ...
    }
    enum Action {
        case row(id: Child.State.ID, action: Child.Action)
        // ...
    }
    
    var body: some Reducer<State, Action> {
        Reduce { state, action in
        }
        .forEach(\.rows, action: /Action.row) {
            Row()
        }
    }
}
```

- 부모(`Parent`)가 자식(`Child`)의 `State`를 배열로 가지고 있음.
- 위치 인덱스가 아닌 안정적인 `ID`로 요소에 접근하기 위해서 `IdentifiedArray`를 사용함.
- 마찬가지로, `forEach`는 자식 및 부모 기능에 대해 특정 작업 순서를 강제함.
    - 자식을 먼저 실행한 후 부모를 실행함.

### forEach 생성자

```swift
@inlinable
@warn_unqualified_access
public func forEach<ElementState, ElementAction, ID: Hashable, Element: Reducer>(
    _ toElementsState: WritableKeyPath<State, IdentifiedArray<ID, ElementState>>,
    action toElementAction: CaseKeyPath<Action, IdentifiedAction<ID, ElementAction>>,
    @ReducerBuilder<ElementState, ElementAction> element: () -> Element,
    fileID: StaticString = #fileID,
    line: UInt = #line
) -> _ForEachReducer<Self, ID, Element>
where ElementState == Element.State, ElementAction == Element.Action {
    _ForEachReducer(
        parent: self,
        toElementsState: toElementsState,
        toElementAction: AnyCasePath(toElementAction.appending(path: \.element)),
        element: element(),
        fileID: fileID,
        line: line
    )
}
```

파라미터

- `toElementsState`: 부모 State로부터 자식 상태를 가지고 있는 `IdentifiedArray`에 대한 `KeyPath`
- `toElementAction`: 부모 Action에서 자식 식별자 및 자식 액션으로의 `KeyPath`
- `element`: 자식 State 및 Action을 처리할 Reducer

반환값 → `_ForEachReducer<Self, ID, Element>`

- `where`절
    1. `ElementState == Element.State`
    2. `ElementAction == Element.Action`
    3. `ID: Hashable`
    4. `Element`(자식의 Reducer): `Reducer`
        - 자식의 Reducer가 Reducer 프로토콜을 준수
- 1, 2, 3, 4를 준수하면 자식 Reducer와 부모 Reducer를 결합한 Reducer를 반환

### ForEachStore(⚠️Deprecated)

- 상태의 배열 또는 `Collection`을 순회하고, 각 항목에 대한 뷰를 생성하는데 사용.
    - 즉, 배열의 요소를 기반으로 동적으로 UI를 생성.

```swift
ForEachStore(
    self.store.scope(state: \.rows, action: { .row(id: $0, action: $1) })
) { childStore in
    ChildView(store: childStore)
}
```

⚠️ 현재 `ForEachStore`는 Deprecated 상태로 그냥 `ForEach`를 사용하면 됨.

```swift
ForEach(
    store.scope(state: \.rows, action: \.rows),
    id: \.state.id
) { childStore in
    ChildView(store: childStore)
}
```

참조: [https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/migratingto1.7/#Replacing-ForEachStore-with-ForEach](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/migratingto1.7/#Replacing-ForEachStore-with-ForEach)

## ifCaseLet

- 부모의 State를 여러 자식 State의 열거형(`enum`)으로 사용하는 경우에서 자식 Reducer를 부모 도메인에 포함 시킬 때 사용.
    - 특정 State 케이스에 대해서 자식 Reducer를 실행시킬 수 있다.

```swift
struct Parent: Reducer {
    enum State {
        case loggedIn(Authenticated.State)
        case loggedOut(Unauthenticated.State)
    }
    enum Action {
        case loggedIn(Authenticated.Action)
        case loggedOut(Unauthenticated.Action)
        // ...
    }
    
    var body: some Reducer<State, Action> {
        Reduce { state, action in
            // Core logic for parent feature
        }
        .ifCaseLet(/State.loggedIn, action: /Action.loggedIn) {
            Authenticated()
        }
        .ifCaseLet(/State.loggedOut, action: /Action.loggedOut) {
            Unauthenticated()
        }
    }
}
```

<aside>
💡 `ifCaseLet`의 기능

- 부모와 자식간 작업 순서를 자식먼저 실행하고, 그 다음 부모를 실행하는 방식으로 강제한다. → 순서가 뒤바뀌면 부모가 자식의 동작에 영향을 줄 수 있게 되므로..
- 자식 열거형의 변경을 감지하면 모든 자식 효과가 자동으로 취소.
</aside>

### ifCaseLet의 생성자

```swift
@inlinable
@warn_unqualified_access
public func ifCaseLet<CaseState, CaseAction, Case: Reducer>(
    _ toCaseState: CaseKeyPath<State, CaseState>,
    action toCaseAction: CaseKeyPath<Action, CaseAction>,
    @ReducerBuilder<CaseState, CaseAction> then case: () -> Case,
    fileID: StaticString = #fileID,
    line: UInt = #line
) -> _IfCaseLetReducer<Self, Case>
```

파라미터

- `toCaseState`: 부모 State로 부터 자식 상태를 가지고 있는 케이스에 대한 `KeyPath`
- `toCaseAction`: 부모 Action에서 자식 식별자 및 자식 액션으로의 `KeyPath`
- `case`: 자식 Action이 존재할 때 자식 State에 대해 자식 Action과 함께 호출될 Reducer

반환값 → `_IfCaseLetReducer<Self, Case>`

- 부모 Reducer와 자식 Reducer를 결합한 Reducer를 반환.

### SwitchStore(⚠️Deprecated)

- `enum` 형식의 State를 관찰하고 해당 상태의 각 경우에 대해 `CaseLet`으로 전환하기 위해 사용.

```swift
public struct AppView: View {
    let store: StoreOf<Parent>
    
    public init(store: StoreOf<Parent>) {
        self.store = store
    }
    
    public var body: some View {
        SwitchStore(self.store) { state in
            switch state {
            case .loggedIn:
                CaseLet(/Parent.State.loggedIn, action: Parent.Action.loggedIn) { store in
                    NavigationStack { LoginView(store: store) }
                }
            case .loggedOut:
                CaseLet(/Parent.State.loggedOut, action: Parent.Action.loggedOut) { store in
                    NavigationStack { LogoutView(store: store) }
                }
            }
        }
    }
}
```

⚠️ 현재 `SwitchStore`는 Deprecated 상태로 그냥 switch-case를 사용하면 됨.

```swift
switch store.state {
case .activity:
    if let store = store.scope(state: \.activity, action: \.activity) {
        ActivityView(store: store)
    }
case .settings:
    if let store = store.scope(state: \.settings, action: \.settings) {
        SettingsView(store: store)
    }
}
```

참조: [https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/migratingto1.7/#Replacing-SwitchStore-and-CaseLet-with-switch-and-case](https://pointfreeco.github.io/swift-composable-architecture/main/documentation/composablearchitecture/migratingto1.7/#Replacing-SwitchStore-and-CaseLet-with-switch-and-case)