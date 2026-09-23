# Android Firebase Analytics 연동과 공통 로깅 구조

- 기준 버전: `firebase-bom` 34.19.0 (2026-09-09 릴리스), `firebase-analytics` 23.2.0, `google-services` Gradle 플러그인 4.5.0, minSdk 23
- 조사일: 2026-09-21
- 참고 사례: LinkU_Android `Feature/301 Firebase Analytics 연동 (#309)` — 마지막 [[#5. 참고 사례 LinkU PR 309]] 참고

## 요약

- Firebase Analytics는 `logEvent(이름) { param(키, 값) }`으로 이벤트를 보낸다. 이벤트명·파라미터 키/값은 **한 번 정하면 바꾸지 않는 계약**이다. 바꾸면 GA4에서 별개 이벤트로 집계된다.
- 이 계약을 코드에서 한곳에 모으기 위해 이벤트 정의(`AnalyticsEvent`), 파라미터 변환(`AnalyticsEventMapper`), 파라미터 값 `enum`, 로깅 추상화(`AnalyticsLogger` interface)를 **`:core:domain`** 에 둔다.
- Firebase SDK는 **`:core:common`** 의 `FirebaseAnalyticsLogger`에만 둔다. `:core:common`이 `:core:domain`에 의존해서 인터페이스를 구현한다.
- **`:feature:*` 는 `:core:domain`만 안다.** 각 feature는 인터페이스(`AnalyticsLogger`)만 보고 `log(event)`를 호출하며, Firebase도 `:core:common`도 모른다.
- 특정 앱에 종속되지 않는 구조이며, 아래 예시는 일반적인 예시 이벤트를 쓴다.

## 내용

### 1. Firebase Analytics 기본 사용법 (Android)

#### 1-1. 설정

Firebase 콘솔에서 앱을 등록하고 `google-services.json`을 내려받아 앱 모듈에 둔다.

```kotlin
// 루트 build.gradle.kts
plugins {
    id("com.google.gms.google-services") version "4.5.0" apply false
}

// app/build.gradle.kts
plugins {
    id("com.google.gms.google-services")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:34.19.0"))
    implementation("com.google.firebase:firebase-analytics") // BoM 사용 시 버전 생략
}
```

- `google-services` 플러그인은 **앱 모듈에만** 적용한다. 라이브러리 모듈에는 `firebase-analytics` 의존성만 추가한다.
- Kotlin이라도 `-ktx` 아티팩트는 필요 없다. 아래 [[#최신 변경 사항]] 참고.

#### 1-2. 이벤트 로깅

```kotlin
// 인스턴스 (공식 문서 예시)
val firebaseAnalytics = Firebase.analytics

// 권장(predefined) 이벤트
firebaseAnalytics.logEvent(FirebaseAnalytics.Event.SELECT_ITEM) {
    param(FirebaseAnalytics.Param.ITEM_ID, id)
    param(FirebaseAnalytics.Param.ITEM_NAME, name)
    param(FirebaseAnalytics.Param.CONTENT_TYPE, "image")
}

// 커스텀 이벤트
firebaseAnalytics.logEvent("share_image") {
    param("image_name", name)
    param("full_text", text)
}

// 모든 이후 이벤트에 붙는 기본 파라미터 (개별 이벤트 파라미터가 우선)
firebaseAnalytics.setDefaultEventParameters(
    Bundle().apply {
        putString("level_name", "Caverns01")
        putInt("level_difficulty", 4)
    }
)
```

- 권장 이벤트가 있으면 그것을 우선 쓴다. 리포트에서 더 상세하게 보인다. 맞는 권장 이벤트가 없을 때 커스텀 이벤트를 만든다.
- 이벤트는 앱당 **서로 다른 이름 500개**까지, 총 전송량은 무제한이다.
- `Firebase.analytics`를 쓸 때의 import(`com.google.firebase.Firebase`, `com.google.firebase.analytics.analytics`)는 이번 조사에서 공식 페이지로 확인하지 못했다. **확인 필요**
- `logEvent { param() }` 블록 형태의 확장 함수는`com.google.firebase.analytics.logEvent`를 import한다.

#### 1-3. 이름·크기 규칙 (GA4 수집 한도)

| 항목        | 한도/규칙                                                      |
| --------- | ---------------------------------------------------------- |
| 이벤트 이름    | 40자 이하                                                     |
| 파라미터 이름   | 40자 이하                                                     |
| 파라미터 값    | 100자 이하 (`page_title` 등 일부 예외)                             |
| 이벤트당 파라미터 | 25개                                                        |
| 문자        | 영문·숫자·`_`, 반드시 문자로 시작, 공백 불가                               |
| 대소문자      | 구분한다. `my_event`와 `My_Event`는 다른 이벤트                       |
| 예약 접두사    | `_`, `firebase_`, `ga_`, `google_`, `gtag.` 사용 금지          |
| 예약 파라미터명  | `user_id`, `session_id`, `gclid`, `dclid` 등은 커스텀 차원에 사용 불가 |

#### 1-4. 커스텀 파라미터는 "커스텀 정의" 등록이 있어야 리포트에 나온다

- 커스텀 파라미터는 Firebase 콘솔 **Analytics > Custom Definitions**에서 등록해야 리포트의 차원/측정항목으로 쓸 수 있다.
- 이벤트 범위(event-scoped) 커스텀 차원 한도는 일반 속성 기준 50개, 360 속성 125개다.
- 등록하기 전에 수집된 값이 소급 적용되는지는 이번 조사에서 확인하지 못했다. **확인 필요**

#### 1-5. 전송 확인 (DebugView)

```bash
adb shell setprop debug.firebase.analytics.app <PACKAGE_NAME>
adb shell setprop debug.firebase.analytics.app .none.   # 해제
```

- 해제하기 전까지 디버그 모드가 유지된다. Firebase 콘솔 DebugView에서 이벤트를 실시간으로 볼 수 있다.

### 2. 공통 로깅 구조

1장의 방식(`Firebase.analytics` + `logEvent { param(...) }`)을 **한 클래스에 가두고**, 나머지 코드는 그 존재를 모르게 만드는 구조다. 아래 예시의 패키지·이벤트는 일반적인 예시다.

#### 2-1. 모듈 배치와 의존 방향

| 모듈             | 파일                                     | 역할                                      |
| -------------- | -------------------------------------- | --------------------------------------- |
| `:core:domain` | `analytics/AnalyticsEvent.kt`          | 이벤트 정의. 이벤트명과 필요한 값을 가진 sealed class    |
| `:core:domain` | `analytics/AnalyticsEventMapper.kt`    | 이벤트 → 파라미터(`Map<String, String>`) 변환    |
| `:core:domain` | `analytics/AnalyticsParams.kt`         | 파라미터 값 `enum`과 도메인 → 분석 값 변환 함수         |
| `:core` (test) | `analytics/AnalyticsEventTest.kt`      | 이름·키·값이 스펙과 같은지 고정하는 테스트                |
| `:core:domain` | `analytics/AnalyticsLogger.kt`         | Analytics 기록을 위한 추상화(interface)         |
| `:core:common` | `analytics/FirebaseAnalyticsLogger.kt` | Firebase SDK 호출. 유일한 SDK 접점             |
| `:feature:*`   | 각 ViewModel/Screen                     | 행동 완료 시점에 `AnalyticsLogger.log(...)` 호출 |

의존 방향은 아래와 같다. 화살표는 "의존한다"이다.

```
:feature:*   ───────▶ :core:domain   (AnalyticsLogger, AnalyticsEvent만 사용)
:core:common ───────▶ :core:domain   (AnalyticsLogger를 구현)
:app         ───────▶ :core:common   (Hilt 그래프에 구현체 바인딩이 포함되도록)
```

- `:feature:*` → `:core:common` 의존이 **없다.** feature가 Firebase 구현을 직접 볼 수 없다.
- `:core:domain`에는 Firebase 의존성이 없다. 그래서 Mapper·enum·이벤트 정의를 Firebase 없이 JVM 단위 테스트로 검증할 수 있다.
- `:app`이 `:core:common`(직·간접)을 의존해야 Hilt가 `FirebaseAnalyticsLogger` 바인딩을 그래프에 포함한다. feature는 몰라도 되지만 앱 조립 지점에서는 필요하다. (일반적인 Hilt 동작이며 이번 조사에서 공식 문서로 다시 확인하지는 않았다.)

```kotlin
// :core:domain/build.gradle.kts — Firebase 의존성 없음

// :core:common/build.gradle.kts
dependencies {
    implementation(project(":core:domain"))
    implementation(platform("com.google.firebase:firebase-bom:34.19.0"))
    implementation("com.google.firebase:firebase-analytics")
    // + Hilt
}

// :feature:xxx/build.gradle.kts
dependencies {
    implementation(project(":core:domain"))   // Firebase 의존성 없음
    // + Hilt
}
```

#### 2-2. AnalyticsEvent — 이벤트 정의 (`:core:domain`)

```kotlin
package com.example.core.domain.analytics

/**
 * 분석 이벤트 정의. 파라미터 변환은 [AnalyticsEventMapper]가 담당한다.
 * 이벤트명과 파라미터 키·값은 배포 후 변경하지 않는다.
 */
sealed class AnalyticsEvent(val name: String) {

    /** 회원가입 완료 */
    data class SignUp(val method: SignUpMethod) : AnalyticsEvent("sign_up")

    /** 콘텐츠 저장 완료 */
    data class ContentSaved(val category: AnalyticsCategory) : AnalyticsEvent("content_saved")

    /** 공유 완료 */
    data class ShareCompleted(val method: ShareMethod) : AnalyticsEvent("share_completed")
}
```

- 이벤트마다 필요한 값을 **타입이 있는 프로퍼티**로 강제한다. 문자열을 직접 넘길 수 없다.
- `:core:domain`은 Firebase를 모르므로 `FirebaseAnalytics.Event.SIGN_UP` 같은 상수를 쓸 수 없다. 권장 이벤트를 쓸 때는 **같은 문자열을 리터럴로** 적는다(`"sign_up"`). 권장 이벤트·파라미터 이름은 공식 이벤트 목록에서 확인한다.

#### 2-3. AnalyticsEventMapper — 이벤트 → 키-값 Map (`:core:domain`)

```kotlin
object AnalyticsEventMapper {

    private const val METHOD = "method"
    private const val CATEGORY = "category"

    // Firebase의 param()은 키-값 쌍이므로 이벤트별로 (키 to 값) Map으로 변환한다.
    fun toParams(event: AnalyticsEvent): Map<String, String> = when (event) {
        is AnalyticsEvent.SignUp -> mapOf(METHOD to event.method.value)
        is AnalyticsEvent.ContentSaved -> mapOf(CATEGORY to event.category.value)
        is AnalyticsEvent.ShareCompleted -> mapOf(METHOD to event.method.value)
    }
}
```

- `when`이 sealed class 전체를 다뤄야 하므로 **새 이벤트를 추가하고 매핑을 빠뜨리면 컴파일 오류**가 난다.
- 파라미터 키 문자열은 `private const val`에 모아 오타·중복을 막는다.
- 반환 타입이 `Map<String, String>`이라 **문자열 값만** 보낼 수 있다. Firebase의 `param()`은 문자열 외 숫자 값도 받으므로(공식 문서 예시에서 `putInt` 사용), 숫자 파라미터가 필요해지면 반환 타입을 넓혀야 한다. **추론이며 `ParametersBuilder`의 오버로드 목록은 이번 조사에서 확인하지 못했다.**

#### 2-4. 파라미터 값 enum (`:core:domain`)

```kotlin
// value는 분석 플랫폼에 그대로 기록되므로 변경하지 않는다.
enum class SignUpMethod(val value: String) {
    EMAIL("email"),
    GOOGLE("google"),
    KAKAO("kakao");

    companion object {
        /** 가입 수단을 알 수 없으면 null → 호출부에서 이벤트를 보내지 않는다. */
        fun from(loginType: LoginType): SignUpMethod? = when (loginType) {
            LoginType.EMAIL -> EMAIL
            LoginType.GOOGLE -> GOOGLE
            LoginType.KAKAO -> KAKAO
            LoginType.NONE -> null
        }
    }
}

enum class ShareMethod(val value: String) {
    LINK("link"),
    QR("qr"),
}

enum class AnalyticsCategory(val value: String) {
    NEWS("news"),
    TECH("tech"),
    ETC("etc"),
}

// 도메인 모델 → 분석 값
fun Category.toAnalyticsType(): AnalyticsCategory = when (this) {
    Category.NEWS -> AnalyticsCategory.NEWS
    Category.DEVELOPMENT -> AnalyticsCategory.TECH
    Category.OTHER -> AnalyticsCategory.ETC
}
```

- **도메인 enum과 분석 enum을 분리**한다. 도메인 모델의 이름이 리팩터링으로 바뀌어도 분석 플랫폼에 나가는 값(`"tech"`)은 유지된다.
- 여러 도메인 값을 하나의 분석 값으로 **묶을 수도** 있다(예: 서로 다른 화면·역할의 같은 행동). 그러면 값의 종류가 줄어 리포트 분석이 쉬워진다.
- 매핑할 수 없는 값은 `null`로 두고 **이벤트를 생략**하거나 `ETC` 같은 명시적 대체 값을 쓴다.

#### 2-5. AnalyticsLogger — 로깅 추상화 (`:core:domain`)

```kotlin
/** 사용자 행동 이벤트를 분석 플랫폼으로 전송한다. 구현체는 `:core:common`에 있다. */
interface AnalyticsLogger {
    fun log(event: AnalyticsEvent)
}
```

- feature가 보는 유일한 로깅 API다. 메서드는 `log(event)` 하나로 충분하다.

#### 2-6. FirebaseAnalyticsLogger — 구현과 DI 바인딩 (`:core:common`)

1장의 `Firebase.analytics` + `logEvent { param(...) }` 방식을 여기에 그대로 사용한다.

```kotlin
package com.example.core.common.analytics

@Singleton
class FirebaseAnalyticsLogger @Inject constructor(
    private val firebaseAnalytics: FirebaseAnalytics,
) : AnalyticsLogger {

    // AnalyticsEvent를 Firebase 이벤트로 변환하여 기록
    override fun log(event: AnalyticsEvent) {
        // 블록 안의 this는 ParametersBuilder이므로 param()을 호출할 수 있다.
        firebaseAnalytics.logEvent(event.name) {
            // Mapper가 만든 (키 → 값) Map을 하나씩 꺼내 
            // param(키, 값)으로 이벤트에 담는다.
            // 예: { "method" = "email" } → param("method", "email")
            AnalyticsEventMapper.toParams(event).forEach { (key, value) ->
                param(key, value)
            }
        }
    }
}
```

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class AnalyticsModule {

    // 인터페이스 ← 구현체 바인딩
    @Binds
    abstract fun bindAnalyticsLogger(impl: FirebaseAnalyticsLogger): AnalyticsLogger

    companion object {
        // 1장의 Firebase.analytics를 그대로 사용
        @Provides
        @Singleton
        fun provideFirebaseAnalytics(): FirebaseAnalytics = Firebase.analytics
    }
}
```

- `FirebaseAnalytics`를 생성자로 주입받으므로 구현체에서 `Context`를 직접 다루지 않는다. 구현체 테스트에서 가짜 `FirebaseAnalytics`로 바꾸기도 쉽다.
- `:core:common`이 `:core:domain`의 `AnalyticsLogger`를 구현하는 것이 의존 방향(`core:common → core:domain`)과 맞는다.
- 두 번째 분석 도구(예: 다른 SDK)가 필요하면 `AnalyticsLogger` 구현체를 하나 더 만들고 바인딩만 바꾸면 된다. feature 코드는 그대로다.

#### 2-7. 호출부 (`:feature:*`) — 인터페이스만 보고 로깅

```kotlin
@HiltViewModel
class SignUpViewModel @Inject constructor(
    private val authRepository: AuthRepository,
    private val analyticsLogger: AnalyticsLogger,   // :core:domain의 인터페이스
) : ViewModel() {

    fun onSignUpSucceeded() {
        // 행동이 완료된 시점에 기록
        analyticsLogger.log(AnalyticsEvent.SignUp(SignUpMethod.EMAIL))
    }
}
```

```kotlin
// 소셜 로그인: 가입 수단을 알 수 없으면 이벤트를 보내지 않는다.
SignUpMethod.from(loginType)?.let { method ->
    analyticsLogger.log(AnalyticsEvent.SignUp(method))
}
```

```kotlin
// 도메인 값은 호출부에서 분석 값으로 변환한다.
analyticsLogger.log(AnalyticsEvent.ContentSaved(saved.category.toAnalyticsType()))
```

- feature의 import는 `com.example.core.domain.analytics.*`뿐이다. `firebase`나 `core.common` import가 있으면 구조가 깨진 것이다.
- Composable에서는 `AnalyticsLogger`를 직접 다루지 않고 ViewModel의 메서드(`onXxxClicked()`)를 통해 호출한다.
- 화면 진입마다가 아니라 **사용자의 행동이 완료된 시점**에 호출한다. 같은 요청의 재조회로 노출이 중복 집계되지 않도록 ViewModel에서 한 번만 기록하게 막는 처리가 필요할 수 있다. (5장 사례 참고)

#### 2-8. 테스트

**(1) 이벤트 계약 테스트** — 이름·키·값을 바꾸면 깨지게 한다. Firebase 없이 JVM에서 돈다.

```kotlin
class AnalyticsEventTest {

    @Test
    fun `event names and param keys follow the spec`() {
        assertEvent("sign_up", mapOf("method" to "email"), AnalyticsEvent.SignUp(SignUpMethod.EMAIL))
        assertEvent("content_saved", mapOf("category" to "tech"), AnalyticsEvent.ContentSaved(AnalyticsCategory.TECH))
    }

    @Test
    fun `all analytics values are snake case`() {
        val snakeCase = Regex("^[a-z]+(_[a-z]+)*$")
        val values = SignUpMethod.entries.map { it.value } + ShareMethod.entries.map { it.value }
        values.forEach { assertTrue("$it is not snake_case", snakeCase.matches(it)) }
    }

    private fun assertEvent(name: String, params: Map<String, String>, event: AnalyticsEvent) {
        assertEquals(name, event.name)
        assertEquals(params, AnalyticsEventMapper.toParams(event))
    }
}
```

- 이 테스트가 실패하면 분석 데이터가 분리될 수 있는 변경이므로 의도한 변경인지 확인하라는 취지를 KDoc에 적어 둔다.

**(2) feature 테스트** — 인터페이스이므로 가짜 구현으로 "이벤트가 호출됐는가"를 검증할 수 있다.

```kotlin
class FakeAnalyticsLogger : AnalyticsLogger {
    val events = mutableListOf<AnalyticsEvent>()
    override fun log(event: AnalyticsEvent) { events += event }
}

// ViewModel 테스트에서
assertEquals(listOf(AnalyticsEvent.SignUp(SignUpMethod.EMAIL)), fakeLogger.events)
```

#### 2-9. 새 이벤트 추가 절차

1. `AnalyticsEvent`에 sealed 하위 클래스 추가 (이름은 snake_case, 40자 이하)
2. 새 파라미터 값이 필요하면 `AnalyticsParams.kt`에 enum 추가 (`value`는 snake_case)
3. `AnalyticsEventMapper.toParams`의 `when`에 분기 추가 (빠뜨리면 컴파일 오류)
4. `AnalyticsEventTest`에 기대값 추가
5. feature에서 행동이 **완료된 시점**에 `analyticsLogger.log(...)` 호출
6. 커스텀 파라미터를 리포트에서 쓸 거면 콘솔의 Custom Definitions에 등록

### 3. 설계상 주의점

- **이름 불변**: 이벤트명·파라미터 키·값은 배포 후 바꾸지 않는다. 바꿔야 하면 새 이벤트를 추가하고 이전 것은 유지한다.
- **행동 완료 시점에만 로깅**: 화면 진입마다 보내면 노출/CTR이 왜곡된다.
- **PII를 파라미터에 넣지 않는다.** 이메일, 닉네임, URL 원문 등은 넣지 않는다.
- **값의 카디널리티를 통제한다.** 자유 입력 문자열 대신 enum으로 제한한다. 파라미터 값 100자 제한과 커스텀 차원 수 한도(50개)도 있다.
- **의존 방향을 지킨다.** `:core:domain`에 Firebase 의존성을 넣지 않고, feature가 `:core:common`을 의존하지 않는다.

### 4. 인터페이스를 두는 이유

| 구분                        | 인터페이스 있음 (이 노트의 구조) | 구체 클래스 직접 주입        |
| ------------------------- | ------------------- | ------------------- |
| feature의 의존               | `:core:domain`만     | 구현체가 있는 모듈까지 의존해야 함 |
| SDK 교체·추가 (Amplitude 등)   | 구현체만 추가/교체          | feature 코드 수정 필요    |
| 빌드 변형별 동작 (debug에서 no-op) | 바인딩 교체로 가능          | 구현체 내부 분기 필요        |
| 코드량                       | 인터페이스 + Hilt 모듈     | 최소                  |

- 이 노트의 목표(feature는 `:core:domain`만 안다)는 인터페이스가 있어야 성립한다. 구체 클래스를 주입받으려면 feature가 그 클래스가 있는 모듈을 의존해야 하기 때문이다.
- 앱 규모가 작고 분석 도구를 바꿀 일이 없다면 직접 주입도 실용적인 선택이다. 이 비교는 조사자의 해석이며 어느 쪽이 정답이라고 단정하지 않는다.

### 5. 참고 사례: LinkU PR 309

`Feature/301 Firebase Analytics 연동 (#309)`(머지 커밋 `b7d6ab4f`, 2026-09-20)의 실제 변경이다. 위 예시는 이 PR의 구조를 일반화한 것이며 차이가 있다.

| 항목                     | PR #309의 실제 코드                                                   | 이 노트의 구조                           |
| ---------------------- | ---------------------------------------------------------------- | ---------------------------------- |
| 이벤트·Mapper·enum 위치     | `:core`의 `analytics/`                                            | `:core:domain`                     |
| Firebase 구현체 위치        | `:data`의 `FirebaseAnalyticsLogger`                               | `:core:common`                     |
| 인터페이스                  | **최종 상태에 없음**                                                    | `:core:domain`의 `AnalyticsLogger`  |
| ViewModel 주입 대상        | `FirebaseAnalyticsLogger` 직접                                     | `AnalyticsLogger`                  |
| `FirebaseAnalytics` 획득 | `FirebaseAnalytics.getInstance(context)` (`@ApplicationContext`) | `Firebase.analytics` (`@Provides`) |

- PR의 커밋 흐름: 첫 커밋(`5ca5f68a`)은 `AnalyticsLogger` 인터페이스와 `AnalyticsModule`(`@Binds`)을 두었다. 두 번째 커밋(`4aabcccd`)은 `AnalyticsEvent`의 `params`를 `AnalyticsEventMapper`로 분리했다. 세 번째 커밋(`e057e52d`)에서 인터페이스와 `AnalyticsModule`을 삭제하고 ViewModel이 구현체를 직접 주입받게 했다.
- PR에서 적용한 이벤트: `sign_up`(`method`), `link_saved`(`category`), `ai_summary_view`(`summary_source`), `emotion_selected`(`emotion_type`), `situation_selected`(`situation_type`), `recommendation_shown`, `recommended_link_click`, `folder_shared`(`share_method`).
- 도메인 → 분석 값 변환 예: 직업별로 나뉜 서버 상황 ID(`HIGH_SCHOOL_COMMUTE`, `OFFICE_COMMUTE` 등)를 직업과 무관한 분석 값 하나(`commute`)로 묶었다. 감정 enum도 도메인 이름(`JOY`)과 분석 값(`happy`)을 분리했다.
- 중복 노출 방지 예: `recommendation_shown`은 `lastShownRecommendationRequestId`로 같은 추천 요청의 재조회(삭제 후 새로고침, 재시도)에 중복 기록되지 않게 하고, 목록이 비어 있으면 노출로 집계하지 않는다.
- 누락 방지 예: 서버가 카테고리를 내려주지 않아도 저장 이벤트가 빠지지 않도록 `ETC`로 기록한다. 가입 수단을 알 수 없는(`LoginType.NONE`) 경우는 이벤트를 생략한다.
- 이벤트 계약 테스트(`AnalyticsEventTest`)로 이름·키·값과 snake_case 규칙을 고정했다.
- `:core`에는 Firebase 의존성이 없고, `data/build.gradle.kts`에 `implementation(libs.firebase.analytics)`가 추가되었다. 이 프로젝트의 BoM은 34.18.0이다.

## 최신 변경 사항

- **KTX 모듈 제거**: 2025-07부터 Firebase는 새 KTX 모듈 버전을 내지 않는다. BoM v32.5.0 이상에서는 메인 모듈에 Kotlin 확장이 포함되어 있고, KTX 라이브러리는 BoM v34.0.0에서 제거되었다. 그래서 `firebase-analytics-ktx` 없이 Kotlin 확장을 쓸 수 있다. 이전 블로그·Stack Overflow 글의 `-ktx` 의존성 예시는 낡은 것이다.
- **인앱 결제 로깅** (Analytics 23.2.0): `logEvent()`로 수동 인앱 결제 기록 지원이 추가되었다.
- 조사 중 확인한 최신 BoM은 34.19.0(2026-09-09)이다. Analytics 관련 breaking change는 확인하지 못했다.

## 확인 필요

- `Firebase.analytics`의 import 경로(`com.google.firebase.Firebase`, `com.google.firebase.analytics.analytics`)와 `FirebaseAnalytics.getInstance(context)`와의 차이 — 공식 Get started 페이지에서 import 줄을 확인하지 못했다.
- 등록하기 전에 수집된 커스텀 파라미터 값이 Custom Definitions 등록 후 소급 반영되는지
- Android용 사용자 속성(`setUserProperty`, `setUserId`) 문서 — 해당 URL이 404여서 이번에 확인하지 못했다. iOS 문서 기준 사용자 속성은 프로젝트당 25개까지이며 `Age`, `Gender`, `Interest`는 이름으로 쓸 수 없다.
- `:app`이 `:core:common`을 의존해야 Hilt 바인딩이 그래프에 들어간다는 설명은 일반적인 Hilt 동작 기준이며 이번에 공식 문서로 확인하지 않았다.

## 출처

| 자료 | 종류 | 버전/작성일 |
|---|---|---|
| [Get started with Google Analytics (Android)](https://firebase.google.com/docs/analytics/android/get-started) | 공식 문서 | BoM 34.19.0 / 2026-09-17 |
| [Log events (Android)](https://firebase.google.com/docs/analytics/android/events) | 공식 문서 | 2026-09-17 |
| [Log events (공통)](https://firebase.google.com/docs/analytics/events) | 공식 문서 | 2026-09-17 |
| [DebugView](https://firebase.google.com/docs/analytics/debugview) | 공식 문서 | 2026-09-17 |
| [Add Firebase to your Android project](https://firebase.google.com/docs/android/setup) | 공식 문서 | google-services 4.5.0 / 2026-09-17 |
| [Firebase Android SDK release notes](https://firebase.google.com/support/release-notes/android) | 공식 릴리스 노트 | BoM 34.19.0 / 2026-09-09 |
| [GA4 collection limits](https://support.google.com/analytics/answer/9267744) | 공식 문서 (Analytics 고객센터) | 조회 2026-09-21 |
| [Event naming rules](https://support.google.com/analytics/answer/13316687) | 공식 문서 (Analytics 고객센터) | 조회 2026-09-21 |
| [Custom dimensions and metrics](https://support.google.com/analytics/answer/14240153) | 공식 문서 (Analytics 고객센터) | 조회 2026-09-21 |
| [User properties (iOS+)](https://firebase.google.com/docs/analytics/user-properties?platform=android) | 공식 문서 | 2026-09-17 |
| LinkU_Android PR #309 (`b7d6ab4f`, `5ca5f68a`, `4aabcccd`, `e057e52d`) | 프로젝트 코드 | 2026-09-20 |
