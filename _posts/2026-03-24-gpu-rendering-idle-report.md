---
layout: post
title: "GPU 렌더링 성능 개선 - Idle 상태의 화면에서..."
date: 2026-03-24
categories: [Android]
description: ""
---

## 개요

메인 화면이 별도의 사용자 입력 없는 **Idle 상태**에 머무르는 동안에도 프레임이 지속적으로 발생하는 현상이 관찰되었다.  
그중 일부 프레임은 16ms를 초과하고 있었고, 이는 불필요한 렌더링 작업이 수행되고 있음을 보여주고 있었다.
이번 분석은 메인 화면 Idle 상태에서 발생하는 GPU 렌더링의 원인을 좁혀가고, 실제로 어떤 로직이 프레임 생성을 유발하는지 확인한 뒤, 이를 줄이기 위한 개선 작업을 정리한 것이다.

## 1. 문제 배경

메인 화면은 사용자 입력이 없고 화면 구성에도 변화가 거의 없는 상태라면 비교적 안정적으로 유지되어야 한다.  
하지만 실제로는 Idle 상태에서도 프레임이 계속 생성되고 있었고, 일정 간격으로 긴 프레임까지 발생하고 있었다.

이 문서에서는 다음 두 가지 관점에서 문제를 추적했다.

- 화면 변화와 직접 관련 없는 **불필요한 UI 업데이트**가 발생하고 있는지
- 화면에 보이지 않는 영역까지 포함해 **주기적인 작업이 계속 실행**되고 있는지

## 2. 관찰된 이상 징후

![프레임 발생 예시](./images/현상_이미지.png)

### 2.1 Idle 상태에서도 지속적으로 프레임이 발생함

메인 화면이 멈춰 있는 상태에서도 렌더링이 계속 발생했다.
- 약 10초 동안 120개의 프레임이 생성됨
- 사용자 인터랙션이 없는 상태라는 점을 고려하면 비정상적인 동작이라 판단됨

### 2.2 주기적으로 16ms를 초과하는 프레임이 발생함

단순히 프레임 수가 많은 것뿐 아니라, 일정 간격으로 긴 프레임도 확인되었다.
- 약 10초에 한 번씩 16ms를 초과하는 프레임이 발생
- HWUI 렌더링 프로파일 기준으로 적색 가로선을 넘는 프레임이 관찰됨

---

## 3. 원인 추적 과정

### 3.1 위치 오버레이 업데이트 로직 점검

먼저 메인 화면의 특정 UI 요소가 지속적인 렌더링에 영향을 주는지 확인하기 위해 메인화면을 구성하는 컴포넌트들을 리스트업 하여 차례로 비활성화 하는 작업을 시행하였다.

이 때 `container_map_btn_bundle`라는 뷰를 비활성화한 뒤에는 지속적으로 발생하던 렌더링이 관찰되지 않았다.  
이를 통해 위치 오버레이와 관련된 업데이트 로직이 주요 원인 후보 중 하나임을 확인할 수 있었다.

#### 확인된 원인

위치 오버레이 로직에서는 `location`, `bearing`, `locationMode`를 `combine`으로 함께 수집하고 있었다.

```kotlin
collectWhenCreated(viewModel.locationOverlay) { (location, bearing, locationMode) ->
    val icon = when(locationMode) {
        LocationMode.NONE -> R.drawable.ic_map_tracking_none
        LocationMode.FOLLOW -> R.drawable.ic_map_tracking_follow
        LocationMode.FACE -> R.drawable.ic_map_tracking_face
    }

    binding.mainOnRecommend.btnLocation.setImageResource(icon)
}
```

- 이 구조에서는 실제로 아이콘 변경과 직접 관련이 없는 `location`이나 `bearing` 값만 바뀌더라도, `setImageResource()`가 반복 호출될 수 있었다.

-> 즉, 필요한 상태 변화와 무관한 UI 업데이트가 계속 발생하면서 불필요한 렌더링을 유발하고 있었던 것이다.

- 또한 위치와 베어링 센서에서는 미세한 움직임까지 모두 반영하고 있었기 때문에, 실제 사용자 관점에서는 의미가 거의 없는 작은 변화도 계속적인 UI 갱신으로 이어질 수 있었다.

### 3.2 주기적 프레임 발생 원인 점검

다음으로 일정한 간격으로 긴 프레임이 반복된다는 점에 주목해, `ViewPager`나 `delay` 기반 `Job`처럼 주기적으로 실행되는 작업을 의심했다.

이를 확인하기 위해 다음 요소를 비활성화해 비교했다.

- 메인 화면의 `ViewPager`
- `DrawerLayout` 내부의 `ViewPager`

비활성화 후에는 주기적으로 발생하던 프레임이 사라졌고, 해당 작업들이 긴 프레임의 원인 후보로 좁혀졌다.

#### 확인된 원인

두 `ViewPager` 모두 일정 주기로 동작하는 스케줄성 `Job`을 사용하고 있었다.  
문제는 데이터 존재 여부를 확인하는 조건은 있었지만, **실제로 해당 뷰가 화면에 보이는 상태인지**에 대한 조건은 없었다는 점이다.

그 결과 다음과 같은 문제가 발생했다.

- 화면에 보이지 않는 상태에서도 Job이 계속 실행됨
- Drawer가 닫혀 있어도 내부 로직이 주기적으로 동작함
- 불필요한 프레임 생성과 긴 작업 시간으로 이어짐

### 3.3 dirty view 추적으로 영향 범위 확인

주기적으로 발생하는 프레임 시점에 어떤 뷰가 다시 그려지고 있는지 확인하기 위해 `OnPreDrawListener`를 사용해 dirty view를 추적했다.

```kotlin
window.decorView.viewTreeObserver.addOnPreDrawListener {
    fun findDirtyViews(view: View, depth: Int = 0) {
        val isDirty = view.isDirty
        val prefix = " ".repeat(depth)
        if (isDirty) {
            Logger.e(
                "FrameDebug",
                "${prefix}DIRTY: ${view.javaClass.simpleName} id=${view.id} vis=${view.visibility} ${view.width}x${view.height}"
            )
        }
    }
}
```

이 방식은 프레임이 발생하는 시점에 어떤 뷰가 invalidation에 관여하고 있는지 좁혀보는 데 도움이 되었고, 스케줄성 작업이 실제 렌더링과 연결되고 있음을 확인하는 보조 근거로 활용할 수 있었다.

## 4. 개선 내용

### 4.1 아이콘 갱신 로직을 `locationMode` 전용 collector로 분리

아이콘 변경은 `locationMode` 변화에만 반응하도록 collector를 분리했다.

```kotlin
collectWhenCreated(viewModel.locationMode) { locationMode ->
    val icon = when(locationMode) {
        LocationMode.NONE -> R.drawable.ic_map_tracking_none
        LocationMode.FOLLOW -> R.drawable.ic_map_tracking_follow
        LocationMode.FACE -> R.drawable.ic_map_tracking_face
    }

    binding.mainOnRecommend.btnLocation.setImageResource(icon)
}
```

이를 통해 `location`이나 `bearing` 값이 바뀌더라도 아이콘과 무관한 경우에는 불필요한 UI 갱신이 발생하지 않도록 했다.

### 4.2 위치 업데이트에 threshold 적용

10m 이내의 위치 변화는 무시하도록 하여 의미 없는 위치 갱신을 줄였다.

```kotlin
currentLocationFlow
    .distinctUntilChanged { old, new ->
        old != null && new != null && old.distanceTo(new) < 10f
    }
```

### 4.3 베어링 업데이트에 threshold 적용

5도 이하의 방향 변화는 방출하지 않도록 하여 잦은 베어링 갱신을 제한했다.

```kotlin
if (lastBearing >= 0) {
    val diff = abs(adjustedAzimuth - lastBearing)
    val circularDiff = minOf(diff, 360 - diff)
    if (circularDiff < 5.0) return
}
lastBearing = adjustedAzimuth
```

### 4.4 visible/open 상태를 기준으로 스케줄 실행 제한

스케줄성 Job은 단순히 데이터 존재 여부뿐 아니라, **실제로 사용자에게 보이는 상태인지**를 기준으로만 실행되도록 수정했다.

- ViewPager 관련 작업은 visible 여부를 기준으로 실행
- Drawer 내부 작업은 drawer open 상태를 기준으로 실행
- 스케줄 함수 내부에도 조건을 추가해 예외 케이스를 방지

이를 통해 화면에 드러나지 않는 UI 때문에 백그라운드성 렌더링이 발생하는 문제를 줄일 수 있었다.

## 5. 개선 결과

### 5.1 지속적 프레임 발생 감소

위치 오버레이 업데이트 구조와 스케줄성 작업을 정리한 뒤, Idle 상태에서 발생하는 프레임 수가 크게 줄었다.

- **AS-IS**: 10초간 프레임 140개 발생
- **TO-BE**: 10초간 프레임 1개 발생(주기적인 긴 프레임)

이는 Idle 상태에서 계속 발생하던 렌더링의 대부분이 실제 화면 변화가 아니라 애플리케이션 로직에 의해 유발되고 있었음을 보여준다.

### 5.2 주기적인 긴 프레임 제거

일정 간격으로 발생하던 약 30ms 수준의 프레임도 더 이상 관찰되지 않았다.

- **AS-IS**: 10초간 약 30ms 작업 시간의 프레임 1개 발생
- **TO-BE**: 동일 조건에서 해당 프레임 발생 없음

## 6. 결론

메인 화면 Idle 상태에서 발생하던 불필요한 GPU 렌더링은 크게 두 가지 원인으로 정리할 수 있었다.

1. 상태 변화와 직접 관련이 없는 **불필요한 UI 업데이트**
2. 화면에 보이지 않는 상태에서도 계속 실행되던 **스케줄성 작업**

이번 개선을 통해 Idle 상태에서의 프레임 발생 빈도를 크게 낮췄고, 주기적으로 발생하던 긴 프레임도 제거할 수 있었다.

즉, 이번 문제는 렌더링 성능 자체보다도, **어떤 로직이 언제 UI를 갱신하도록 만들고 있는지**를 정리하는 과정에서 상당 부분 개선할 수 있었던 사례라고 볼 수 있다.

## 7. 추후 확인할 항목

현재 개선 범위는 메인 화면의 **Idle 상태**에 한정된다.  
이후에는 다음 구간에 대한 추가 분석이 필요하다.

- 이용자 이동중 `Location` 실시간 수신 과정에서의 갱신 비용
- 실시간 데이터 반영 과정에서 발생하는 UI 변경에 따른 렌더링 부담

## 8. 측정 방법

### 8.1 카운터 초기화

```bash
adb shell dumpsys gfxinfo com.rocketmoon.picker.dev reset
```

### 8.2 테스트 수행

- 메인 화면 진입
- Idle 상태 유지
- 특정 UI 활성/비활성 비교
- 개선 전후 동일 조건으로 측정

### 8.3 결과 확인

```bash
adb shell dumpsys gfxinfo com.rocketmoon.picker.dev
```

## 9. 참고 자료

- https://developer.android.com/topic/performance/vitals/render?hl=ko
- https://developer.android.com/topic/performance/rendering/inspect-gpu-rendering?hl=ko#profile_rendering
- https://developer.android.com/topic/performance/rendering/overdraw?hl=ko

## 10. 넋두리
- Compose로도 위와 같이 복잡한 뷰로 구성된 화면을 다뤄봐야 해야 하는데... 
- Compose였으면 또 고민하는 방식이 많이 달랐을까? `recomposition`이라던지...