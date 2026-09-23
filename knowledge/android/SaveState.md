# SavedState 관련 개념 정리

> 출처: [Android SavedState 관련 개념 정리 - Velog](https://velog.io/@hyunwoomoon/Android-SavedState-%EA%B4%80%EB%A0%A8-%EA%B0%9C%EB%85%90-%EC%A0%95%EB%A6%AC)

## 1. SavedState

```kotlin
public expect class SavedState
```

네이티브 플랫폼에서 **저장하고 복원할 수 있는 값을 담는 공통 타입**이다. 직렬화 메커니즘을 통해 상태 데이터를 지속적으로 유지한다.

플랫폼별 구현:
- **Android:** `Bundle`의 `typealias`
- **KMP 및 기타 플랫폼:** `Map<String, Any>` 인스턴스

---

## 2. SavedStateRegistryOwner

SavedState 기능의 시작점. **SavedStateRegistry를 소유하는 스코프(범위)**를 가리킨다.

```kotlin
public interface SavedStateRegistryOwner : androidx.lifecycle.LifecycleOwner {
    public val savedStateRegistry: SavedStateRegistry
}
```

- `LifecycleOwner`를 상속하므로 라이프사이클을 가진 컴포넌트여야 한다
- 대표 구현체: `ComponentActivity`, `Fragment`, `NavBackStackEntry`

### ComponentActivity

```kotlin
open class ComponentActivity() :
    androidx.core.app.ComponentActivity(),
    SavedStateRegistryOwner {
    
    final override val savedStateRegistry: SavedStateRegistry
}
```

### Fragment

```kotlin
open class Fragment : LifecycleOwner,
    ...
    SavedStateRegistryOwner
}
```

### NavBackStackEntry

```kotlin
public actual class NavBackStackEntry
private constructor(
    internal actual val context: NavContext?,
    public actual var destination: NavDestination,
    internal actual val immutableArgs: SavedState? = null,
    internal actual var hostLifecycleState: Lifecycle.State = Lifecycle.State.CREATED,
    internal actual val viewModelStoreProvider: NavViewModelStoreProvider? = null,
    public actual val id: String = randomUUID(),
    internal actual val savedState: SavedState? = null
) : 
    LifecycleOwner,
    ViewModelStoreOwner,
    HasDefaultViewModelProviderFactory,
    SavedStateRegistryOwner
```

---

## 3. SavedStateRegistry

**SavedState를 사용하고 저장하는 컴포넌트를 연결하기 위한 인터페이스.** Activity나 Fragment가 재생성될 때 해당 객체의 새 인스턴스도 함께 생성된다.

```kotlin
public expect class SavedStateRegistry internal constructor(
    impl: SavedStateRegistryImpl
) {
    // SavedState에 기여하는 컴포넌트를 표시
    public fun interface SavedStateProvider {
        public fun saveState(): SavedState
    }
    
    // 상태가 복원되었고 안전하게 사용 가능한지 여부
    public val isRestored: Boolean
    
    // 이전에 제공된 저장된 상태를 사용
    @MainThread 
    public fun consumeRestoredStateForKey(key: String): SavedState?
    
    // 주어진 키로 SavedStateProvider를 등록
    @MainThread 
    public fun registerSavedStateProvider(
        key: String, 
        provider: SavedStateProvider
    )
    
    // 이전에 등록된 SavedStateProvider를 반환
    public fun getSavedStateProvider(key: String): SavedStateProvider?
    
    // 주어진 키로 이전에 등록된 컴포넌트를 등록 해제
    @MainThread 
    public fun unregisterSavedStateProvider(key: String)
}
```

---

## 4. SavedStateRegistryController

`SavedStateRegistryOwner` 구현체가 `SavedStateRegistry`를 제어하기 위한 API.

```kotlin
public actual class SavedStateRegistryController
// 외부에서 직접 생성 불가 — companion object의 create()를 통해서만 인스턴스화
private actual constructor(private val impl: SavedStateRegistryImpl) {
    
    // 이 컨트롤러가 관리하는 SavedStateRegistry
    // Owner(Activity, Fragment 등)는 이 프로퍼티를 통해 Registry에 접근
    public actual val savedStateRegistry: SavedStateRegistry = 
        SavedStateRegistry(impl)
    
    // 초기의 일회성 연결 작업 — Lifecycle.State.INITIALIZED 시점에 호출
    // Registry를 Owner의 라이프사이클에 바인딩한다
    @MainThread
    public actual fun performAttach() {
        impl.performAttach()
    }
    
    // 이전에 저장된 상태를 복원 — onCreate()에서 savedInstanceState를 넘겨 호출
    // savedState가 null이면 복원할 상태가 없는 것 (최초 실행 등)
    @MainThread
    public actual fun performRestore(savedState: SavedState?) {
        impl.performRestore(savedState)
    }
    
    // 등록된 모든 SavedStateProvider의 상태를 수집하여 outBundle에 저장
    // onSaveInstanceState()에서 호출되어 프로세스 종료 전 상태를 보존
    @MainThread
    public actual fun performSave(outBundle: SavedState) {
        impl.performSave(outBundle)
    }
    
    public actual companion object {
        @JvmStatic
        // SavedStateRegistryController의 유일한 생성 경로
        // owner: SavedStateRegistryOwner (Activity, Fragment 등)
        public actual fun create(
            owner: SavedStateRegistryOwner
        ): SavedStateRegistryController {
            val impl = SavedStateRegistryImpl(
                owner = owner,
                // attach 시 Recreator를 라이프사이클 옵저버로 등록
                // → 프로세스 재생성 시 자동으로 상태 복원 로직이 트리거됨
                onAttach = { 
                    owner.lifecycle.addObserver(Recreator(owner)) 
                },
            )
            return SavedStateRegistryController(impl)
        }
    }
}
```

### 라이프사이클에 따른 동작

```
Lifecycle.State          SavedStateRegistryController
─────────────────        ────────────────────────────
INITIALIZED ──────────→ performAttach (일회성 attach)
                        │
                        performRestore (기존 상태 복원)
                        │
CREATED ────────────────+
                        │
STARTED ───────────────→ SavedStateProvider 등록, 복원된 상태 소비
                        │
CREATED ───────────────→ performSave (상태 수집 및 저장)
                        │
DESTROYED ◄─────────────+
```

---

## 5. ComponentActivity 구현 확인

```kotlin
open class ComponentActivity() :
    androidx.core.app.ComponentActivity(),
    SavedStateRegistryOwner {
    
    // SavedStateRegistryController 객체 생성
    private val savedStateRegistryController: SavedStateRegistryController =
        SavedStateRegistryController.create(this)
    
    init {
        savedStateRegistry.registerSavedStateProvider(ACTIVITY_RESULT_TAG) {
            val outState = Bundle()
            activityResultRegistry.onSaveInstanceState(outState)
            outState
        }
        
        addOnContextAvailableListener {
            val savedInstanceState =
                savedStateRegistry.consumeRestoredStateForKey(
                    ACTIVITY_RESULT_TAG
                )
            if (savedInstanceState != null) {
                activityResultRegistry.onRestoreInstanceState(
                    savedInstanceState
                )
            }
        }
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        // SavedStateRegistryController에 savedInstanceState Bundle 보관
        savedStateRegistryController.performRestore(savedInstanceState)
        super.onCreate(savedInstanceState)
    }
    
    @CallSuper
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        // SavedStateRegistry에 등록된 컴포넌트들의 상태를 Bundle에 저장
        savedStateRegistryController.performSave(outState)
    }
    
    final override val savedStateRegistry: SavedStateRegistry
        get() = savedStateRegistryController.savedStateRegistry
}
```

---

## 요약

- **상태 복원:** `onCreate`에서 `SavedStateRegistryController.performRestore(savedState)` 수행
- **상태 저장:** `onSaveInstanceState`에서 `SavedStateRegistryController.performSave(outBundle)` 수행

[[Activity 생명주기]]의 `onCreate()`에서 `savedInstanceState`를 통해 복원되는 상태가 바로 이 SavedState 메커니즘을 통해 저장된 것이다.