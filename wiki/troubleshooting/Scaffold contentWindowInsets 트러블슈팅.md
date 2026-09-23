
> 출처: [Android Scaffold에서의 contentWindowInsets](https://velog.io/@hyunwoomoon/Android-Scaffold%EC%97%90%EC%84%9C%EC%9D%98-contentWindowInsets)

## 배경

엣지-투-엣지 디자인 적용 중 Scaffold의 `contentWindowInsets` 기본 동작으로 인해 두 가지 문제가 발생했다.

`contentWindowInsets`은 Scaffold가 내부 콘텐츠에 패딩으로 적용할 WindowInsets을 정의하며, 기본값은 `systemBarsForVisualComponents`(시스템 바 + 디스플레이 컷아웃)이다.

---

## 문제 1: 시스템 바 영역에 배경색 미적용

### 증상

각 화면의 배경색이 상태바 영역까지 확장되지 않았다.

### 원인

Scaffold의 기본 `contentWindowInsets`이 시스템 바만큼 padding을 적용하므로, 내부 컴포넌트가 시스템 바 영역에서 제외되었다.

### 해결

상단 인셋을 제외하고, 각 화면에서 `.statusBarsPadding()`으로 개별 조정했다.

```kotlin
contentWindowInsets = WindowInsets.safeDrawing
    .only(WindowInsetsSides.Bottom + WindowInsetsSides.Horizontal)
```

---

## 문제 2: 키보드 올라올 때 레이아웃이 밀려 올라감

### 증상

TextField 클릭 시 채팅 화면이 키보드 높이만큼 밀려 올라갔다.

### 원인

`contentWindowInsets`에 IME(Input Method Editor) 인셋이 포함되어 Scaffold가 키보드 높이를 자동으로 padding으로 적용했다.

### 해결

키보드 인셋을 명시적으로 제외했다.

```kotlin
contentWindowInsets = WindowInsets.safeDrawing
    .only(WindowInsetsSides.Bottom + WindowInsetsSides.Horizontal)
    .exclude(WindowInsets.ime)
```

---

## 관련 API 정리

| API                        | 설명                          |
| -------------------------- | --------------------------- |
| `WindowInsets.safeDrawing` | 시스템 바 + 컷아웃 + 소프트 키보드 전체 포함 |
| `.only(sides)`             | 지정된 방향의 인셋만 활성화             |
| `.exclude(insets)`         | 특정 인셋 타입 제외                 |

## 핵심 교훈

Scaffold의 `contentWindowInsets` 기본값이 자동으로 시스템 바와 IME 패딩을 적용한다는 점을 인지해야 한다. 엣지-투-엣지 디자인에서는 기본값을 그대로 사용하지 말고, 필요한 방향과 인셋 타입만 명시적으로 지정해야 의도한 레이아웃을 구현할 수 있다.
