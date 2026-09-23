`Composable` 함수는 기본적으로 순수(pure)해야 합니다. 즉, **동일한 입력에 항상 동일한 출력**을 내야 하며, **네트워크 요청·타이머·외부 상태 변경** 같은 **부수 효과(Side Effect)**를 컴포지션 내부에 직접 포함해서는 안 됩니다.

Compose는 이런 Side Effect를 안전하게 처리하기 위해 전용 Effect API를 제공합니다.

---

## LaunchedEffect — 코루틴 기반 비동기 작업
컴포저블이 [[컴포저블 생명주기|컴포지션에 진입]]할 때 **코루틴을 실행**하고, 지정된 `key` 가 변경되면 기존 코루틴을 취소 후 재실행합니다. 컴포저블이 **컴포지션에서 나가면 코루틴도 자동으로 취소**됩니다.

```kotlin
// userId가 바뀔 때마다 자동으로 재실행됩니다
@Composable
fun UserProfile(userId: String) {
    var profile by remember { mutableStateOf<Profile?>(null) }

    LaunchedEffect(userId) {          // key = userId
        profile = fetchUserProfile(userId)   // suspend 함수 직접 호출 가능
    }

    profile?.let { ProfileCard(it) }
}
```

>[!note] LaunchedEffect Key 전략
>	- `key=Unit` (또는 `true`): 최초 1회만 실행, **초기 데이터 로딩**에 주로 사용합니다.
>	- `key = someState`: `someState` 가 바뀔 때마다 재실행. **입력값 연동에 적합**합니다.
>	- key를 여러 개 전달하면(`vararg`), 하나라도 바뀌면 재실행됩니다.

## DisposableEffect — 정리(cleanup)가 필요한 부수 효과
리소스를 등록(구독)하고 컴포저블이 사라질 때 반드시 해제해야 하는 경우에 사용합니다.`onDispose` 블록에 정리 코드를 작성합니다.

```kotlin
@Composable
fun LifecycleObserverEffect(lifecycleOwner: LifecycleOwner) {
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_RESUME) doSomething()
        }
        lifecycleOwner.lifecycle.addObserver(observer)

        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)  // 반드시 해제!
        }
    }
}
```

## SideEffect — 컴포지션 성공 후 실행
Compose 상태를 Compose가 관리하지 않는 외부 객체(예: Analytics, 비Compose SDK)와 동기화할 때 씁니다. Recomposition이 성공할 때마다 실행됩니다.

```kotlin
@Composable
fun AnalyticsScreen(screenName: String, analytics: Analytics) {
    SideEffect {
        analytics.setCurrentScreen(screenName)   // 매 성공 컴포지션마다 동기화
    }
}
```

## rememberCoroutineScope - 이벤트 기반 코루틴 실행
`LaunchedEffect`는 컴포지션 생명주기에 맞춰 자동으로 실행되지만, 버튼 클릭처럼 사용자 이벤트에 응답해서 코루틴을 실행하고 싶을 때는 `rememberCoroutineScope`를 사용합니다.

```kotlin
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    val scope = rememberCoroutineScope()

    Button(onClick = {
        scope.launch {                        // 클릭 시 코루틴 실행
            listState.animateScrollToItem(0)
        }
    }) {
        Text("맨 위로 ↑")
    }
}
```

---
## Side Effect API 한눈에 비교
| API                      | 실행 시점                    | 주 사용처                 |
| ------------------------ | ------------------------ | --------------------- |
| `LaunchedEffect`         | 진입 시, key 변경 시           | 비동기 데이터 로드, 타이머       |
| `DisposableEffect`       | 진입 시 + 종료 시(`onDispose`) | 리스너 등록/해제, 브로드캐스트 수신  |
| `SideEffect`             | Recomposition 성공마다       | Analytics, 외부 SDK 동기화 |
| `rememberCoroutineScope` | 이벤트(클릭 등) 발생 시           | 버튼 클릭 → 스크롤, 애니메이션    |
