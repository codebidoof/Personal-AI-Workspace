# Activity 생명주기

> 출처: [Android Activity LifeCycle - Velog](https://velog.io/@hyunwoomoon/Android-Activity-LifeCycle)
> 참고: [Android 공식문서 - Activity 생명주기](https://developer.android.com/guide/components/activities/activity-lifecycle)

## 개요

Activity 클래스는 6가지 콜백으로 이루어진 핵심 집합을 제공한다: `onCreate()`, `onStart()`, `onResume()`, `onPause()`, `onStop()`, `onDestroy()`. 시스템은 Activity가 새 상태로 전환될 때 각 콜백을 호출한다.

---

## 1. onCreate()

- Activity 생성 시 **단 한 번만** 호출되며, Created 상태에 진입
- 주요 작업:
    - UI 레이아웃 설정 (`setContentView()`)
    - View 초기화 (findViewById 또는 ViewBinding)
    - [[SaveState|저장된 상태 복원]] (`savedInstanceState`)

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.main_activity)
    gameState = savedInstanceState?.getString(GAME_STATE_KEY)
    textView = findViewById(R.id.text_view)
}
```

`onCreate()` 이후 시스템은 즉시 `onStart()`와 `onResume()`을 순차적으로 호출한다.

## 2. onStart()

- Activity가 Started 상태로 전환될 때 호출
- **사용자에게 화면이 보이기 시작하는 단계**
- 데이터 갱신, 화면 표시 시 필요한 작업 수행

## 3. onResume()

- Activity가 Resumed 상태가 되면 **포그라운드에 위치**하며, 사용자와 직접 상호작용 가능
- **포그라운드 = Resumed 상태**
- `onPause()`에서 중단했던 작업을 재활성화하는 로직을 작성
- 다음 상황에서 Paused 상태로 전환:
    - 다른 Activity로 이동
    - 전화 등 외부 이벤트 발생
    - 화면 꺼짐

## 4. onPause()

- Activity가 **포그라운드에서 벗어남**을 의미
- 멀티 윈도우 환경에서 포커스를 잃은 윈도우도 Paused 상태
- 권장 작업 (일시적 중단):
    - GPS 끄기
    - 카메라 일시정지
    - 센서 해제
- **주의:** 실행 시간이 매우 짧으므로, 데이터 저장이나 네트워크 요청 같은 무거운 작업은 `onStop()`에서 수행해야 한다.

## 5. onStop()

- Activity가 **사용자에게 더 이상 보이지 않게 되면** Stopped 상태로 전환
- 발생 상황:
    - 새로운 Activity가 화면 전체를 차지할 때
    - 홈 버튼으로 화면이 완전히 가려질 때
- 권장 작업:
    - 불필요한 리소스 해제
    - 애니메이션이나 위치 업데이트 정리
    - 데이터베이스 저장 등 시간 소모 작업 수행
    - UI 관련 리소스 정리 (멀티 윈도우 환경에서 중요)
- Stopped에서 재개되면 `onRestart()`가 호출됨

## 6. onDestroy()

- Activity 소멸 전에 호출
- 호출 시점:
    1. 사용자가 Activity 종료 (`finish()` 호출 또는 뒤로가기)
    2. 시스템이 구성 변경(예: 화면 회전)으로 일시적 제거
- **주의:** `onDestroy()`가 "완전 종료"를 의미하지 않을 수 있다. 화면 회전 시 시스템은 즉시 새 인스턴스를 생성하고 `onCreate()`를 다시 호출한다.
- 실행이 무조건 보장되지 않으므로 필수 코드는 여기에 작성하면 위험하다.

---

## 실험 결과

```kotlin
class MainActivity : AppCompatActivity() {
    private val TAG = "LIFE_QUIZ"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Log.d(TAG, "onCreate")
    }

    override fun onStart() {
        super.onStart()
        Log.d(TAG, "onStart")
    }

    override fun onResume() {
        super.onResume()
        Log.d(TAG, "onResume")
    }

    override fun onPause() {
        super.onPause()
        Log.d(TAG, "onPause")
    }

    override fun onStop() {
        super.onStop()
        Log.d(TAG, "onStop")
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.d(TAG, "onDestroy")
    }

    override fun onRestart() {
        super.onRestart()
        Log.d(TAG, "onRestart")
    }
}
```

### 시나리오 1: 앱 최초 실행

`onCreate` → `onStart` → `onResume`

### 시나리오 2: 홈 버튼으로 백그라운드 전환

`onPause` → `onStop`

### 시나리오 3: 앱으로 복귀

`onRestart` → `onStart` → `onResume`

### 시나리오 4: 화면 회전

`onPause` → `onStop` → `onDestroy` → `onCreate` → `onStart` → `onResume`
