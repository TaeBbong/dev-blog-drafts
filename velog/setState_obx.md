## 개요

플러터의 가장 기본 상태 관리 기법인 setState. 이것이 어떻게 동작하는지 알고 계신가요?
setState가 상태의 변화를 만들고 이를 화면에 보여주기까지의 과정을 정리하고, 나아가 GetX의 Obx와도 비교 해보겠습니다.

### 프레임이란?

모니터 장비에 관심이 많거나 게임을 즐기시는 분들에겐 익숙한 개념일 수 있습니다.
**프레임(Frame)**은 화면이 그려지는 기본 단위라고 생각할 수 있습니다.

우리가 보고 있는 스마트폰이나 컴퓨터 화면은 아무것도 하지 않을 때 멈춰있는 것처럼 보이지만, 실제로는 1초에 수십 번씩 화면이 새로 그려지고 있습니다.
이렇게 화면을 새로 그리는 과정을 **"프레임 갱신"**이라고 부르며, **1초에 몇 번 화면이 그려지는지를 나타내는 지표가 바로 FPS(Frames Per Second)**입니다.

예를 들어, 1초에 60번 화면이 그려진다면 60fps라고 하며, 프레임 수가 높을수록 사용자 입장에서는 더 부드럽고 자연스러운 동작처럼 느껴집니다.

스마트폰에서는 일반적으로 초당 60프레임, 즉 60fps로 동작하며, 최근에는 90fps, 120fps를 지원하는 고주사율 디스플레이도 많아지고 있습니다.

### 플러터의 렌더링 루프

그럼 프레임이 플러터와 무슨 관련이 있을가요??
60fps 디스플레이를 기준으로 플러터에서는 앱을 실행할 때 1초에 최대 60번 화면을 그릴 수 있도록 프레임 기반 렌더링 루프를 운영합니다.
즉, 1초에 60번씩 화면을 새로 렌더링할지 말지를 결정하며 이 결정에 따라 상태 변화가 UI에 반영되기도 하고, 변화가 없다면 아무 작업도 하지 않기도 합니다.

이 렌더링 루프는 플러터의 핵심 클래스인 `SchedulerBinding`을 통해 관리됩니다.

#### 1. 프레임 요청: scheduleFrame()

화면에 어떤 변화가 발생하면 플러터는 다음 프레임을 예약합니다.

```dart
// packages/flutter/lib/src/scheduler/binding.dart
/// If necessary, schedules a new frame by calling
/// [dart:ui.PlatformDispatcher.scheduleFrame].
///
/// After this is called, the engine will (eventually) call
/// [handleBeginFrame]. (This call might be delayed, e.g. if the device's
/// screen is turned off it will typically be delayed until the screen is on
/// and the application is visible.) Calling this during a frame forces
/// another frame to be scheduled, even if the current frame has not yet
/// completed.
///
/// Scheduled frames are serviced when triggered by a "Vsync" signal provided
/// by the operating system. The "Vsync" signal, or vertical synchronization
/// signal, was historically related to the display refresh, at a time when
/// hardware physically moved a beam of electrons vertically between updates
/// of the display. The operation of contemporary hardware is somewhat more
/// subtle and complicated, but the conceptual "Vsync" refresh signal continue
/// to be used to indicate when applications should update their rendering.
///
/// To have a stack trace printed to the console any time this function
/// schedules a frame, set [debugPrintScheduleFrameStacks] to true.
///
/// See also:
///
///  * [scheduleForcedFrame], which ignores the [lifecycleState] when
///    scheduling a frame.
///  * [scheduleWarmUpFrame], which ignores the "Vsync" signal entirely and
///    triggers a frame immediately.
void scheduleFrame() {
    if (_hasScheduledFrame || !framesEnabled) {
        return;
    }
    assert(() {
        if (debugPrintScheduleFrameStacks) {
            debugPrintStack(label: 'scheduleFrame() called. Current phase is $schedulerPhase.');
        }
        return true;
    }());
    ensureFrameCallbacksRegistered();
    platformDispatcher.scheduleFrame();
    _hasScheduledFrame = true;
}
```

여기서 window.scheduleFrame()은 Flutter 엔진(C++ 레이어)에 프레임을 요청하는 API입니다.
→ 이후 실제로 프레임 타이밍이 되면, Flutter는 프레임 처리 콜백인 handleBeginFrame()을 호출합니다.

#### 2. 프레임 시작: handleBeginFrame()

```dart
@override
void handleBeginFrame(Duration rawTimeStamp) {
  _currentFrameTimeStamp = rawTimeStamp;
  _hasScheduledFrame = false;
  _frameScheduled = true;

  _executeFrame();
}
```

이 시점에서:

현재 프레임의 타임스탬프가 기록되고

_executeFrame()을 통해 본격적인 렌더링 처리 단계로 진입합니다.

#### 3. 프레임 처리: _executeFrame() → handleDrawFrame()

```dart
void _executeFrame() {
  handleDrawFrame();
  _frameScheduled = false;
}
```

handleDrawFrame()은 화면 렌더링의 핵심이 되는 메서드로, 실제로 UI를 갱신해야 하는 요소를 찾아 다시 그리는 작업을 수행합니다.

#### 4. 실제 렌더링: handleDrawFrame()

```dart
@override
void handleDrawFrame() {
  super.handleDrawFrame(); // BindingBase
  buildOwner.buildScope(_dirtyElements);
  // 이후 layout → paint → composite 단계가 이어짐
}
```

여기서 핵심은 buildOwner.buildScope()입니다.
이 메서드는 현재 프레임에서 다시 그려야 할(dirty) 위젯들을 찾아서 build()를 호출합니다.

### setState가 동작하는 원리