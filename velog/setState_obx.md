## 개요

플러터의 가장 기본 상태 관리 기법인 setState. 이것이 어떻게 동작하는지 알고 계신가요?
setState가 상태의 변화를 만들고 이를 화면에 보여주기까지의 과정을 정리하고, 나아가 GetX의 Obx와도 비교 해보겠습니다.

### setState부터 쭉 따라가보는 동작 원리

플러터 소스코드를 쭉 따라가보면서 setState가 어떻게 동작하는지 확인해보겠습니다.

먼저 찾을 코드는 setState 메소드를 선언한 부분입니다.
packages/flutter/lib/src/widgets/framework.dart 파일에 선언되어 있습니다.
수 많은 주석과 소스코드 덕분에 분량이 아주 많은데, setState와 State 클래스 관련 내용만 확인해보겠습니다.

```dart
// lib/src/widgets/framework.dart
abstract class State<T extends StatefulWidget> with Diagnosticable {
  T get widget => _widget!;
  T? _widget;

  _StateLifecycle _debugLifecycleState = _StateLifecycle.created;

  bool _debugTypesAreRight(Widget widget) => widget is T;

  BuildContext get context {
    assert(() {
      if (_element == null) {
        throw FlutterError();
      }
      return true;
    }());
    return _element!;
  }

  StatefulElement? _element;

  @protected
  void setState(VoidCallback fn) {
    assert(() {
      if (_debugLifecycleState == _StateLifecycle.defunct) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('setState() called after dispose(): $this'),
          ...
        ]);
      }
      if (_debugLifecycleState == _StateLifecycle.created && !mounted) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('setState() called in constructor: $this'),
          ...
        ]);
      }
      return true;
    }());
    final Object? result = fn() as dynamic;
    assert(() {
      if (result is Future) {
        throw FlutterError.fromParts(<DiagnosticsNode>[
          ErrorSummary('setState() callback argument returned a Future.'),
          ...
        ]);
      }
      return true;
    }());
    _element!.markNeedsBuild();
  }
  ...
}
```

_주석이 너무 많아 삭제했는데, 클래스 전반의 이해를 위해 읽는 것을 추천드립니다._

State 클래스에는 widget, StatefulElement \_element, 그리고 setState() 등의 메소드가 선언되어 있습니다.
여기서 setState()의 마지막 줄을 보면 \_element!.markNeedsBuild()을 호출하는 것을 알 수 있습니다.
함수명에서 알 수 있듯 이 element가 빌드가 필요하다는 것을 표시하겠다는 뜻인데, 이 메소드의 docstring에는 다음과 같은 내용이 있습니다.

> Marks the element as dirty and adds it to the global list of widgets to rebuild in the next frame.
> Since it is inefficient to build an element twice in one frame, applications and widgets should be structured so as to only mark widgets dirty during event handlers before the frame begins, not during the build itself.

주요 내용은 해당 element가 dirty하다고 표시하며, 이를 전역 list에 추가하여 다음 프레임에서 rebuild하도록 한다는 것입니다.

markNeedsBuild() 메소드 코드도 한번 볼까요??
markNeedsBuild()는 StatefulElement 클래스의 메소드이며, StatefulElement는 State 클래스 내에 \_element 객체로 사용됩니다.
마찬가지로 관련 부분만 간단히 살펴보면,

```dart
abstract class StatefulElement... {
  ...
  bool _dirty = true;
  bool _inDirtyList = false;
  bool _debugBuiltOnce = false;
  ...
  void markNeedsBuild() {
    ...
    if (dirty) {
      return;
    }
    _dirty = true;
    owner!.scheduleBuildFor(this);
  }
}
```

역시 마지막 부분에 scheduleBuildFor()라는 메소드를 호출합니다.
해당 메소드의 설명은,

> Adds an element to the dirty elements list so that it will be rebuilt when [WidgetsBinding.drawFrame] calls [buildScope].

앞선 코드와 유사하게 흐름이 진행됩니다.
여기서 결국엔 WidgetsBinding.drawFrame에서 리빌드를 실행하도록 한다는 것을 알 수 있습니다.

그럼 최종적으로 WidgetsBinding.drawFrame이 어떻게 구현되어있는지를 알아야 플러터가 dirty로 만들어놓은 element들을 리빌드 하는지 알 수 있겠네요.

WidgetsBinding.drawFrame()은 다음과 같습니다.

```dart
  @override
  void drawFrame() {
    assert(!debugBuildingDirtyElements);
    assert(() {
      debugBuildingDirtyElements = true;
      return true;
    }());

    TimingsCallback? firstFrameCallback;
    bool debugFrameWasSentToEngine = false;
    if (_needToReportFirstFrame) {
      assert(!_firstFrameCompleter.isCompleted);

      firstFrameCallback = (List<FrameTiming> timings) {
        assert(debugFrameWasSentToEngine);
        if (!kReleaseMode) {
          // Change the current user tag back to the default tag. At this point,
          // the user tag should be set to "AppStartUp" (originally set in the
          // engine), so we need to change it back to the default tag to mark
          // the end of app start up for CPU profiles.
          developer.UserTag.defaultTag.makeCurrent();
          developer.Timeline.instantSync('Rasterized first useful frame');
          developer.postEvent('Flutter.FirstFrame', <String, dynamic>{});
        }
        SchedulerBinding.instance.removeTimingsCallback(firstFrameCallback!);
        firstFrameCallback = null;
        _firstFrameCompleter.complete();
      };
      // Callback is only invoked when FlutterView.render is called. When
      // sendFramesToEngine is set to false during the frame, it will not be
      // called and we need to remove the callback (see below).
      SchedulerBinding.instance.addTimingsCallback(firstFrameCallback!);
    }

    try {
      if (rootElement != null) {
        buildOwner!.buildScope(rootElement!);
      }
      super.drawFrame();
      assert(() {
        debugFrameWasSentToEngine = sendFramesToEngine;
        return true;
      }());
      buildOwner!.finalizeTree();
    } finally {
      assert(() {
        debugBuildingDirtyElements = false;
        return true;
      }());
    }
    if (!kReleaseMode) {
      if (_needToReportFirstFrame && sendFramesToEngine) {
        developer.Timeline.instantSync('Widgets built first useful frame');
      }
    }
    _needToReportFirstFrame = false;
    if (firstFrameCallback != null && !sendFramesToEngine) {
      // This frame is deferred and not the first frame sent to the engine that
      // should be reported.
      _needToReportFirstFrame = true;
      SchedulerBinding.instance.removeTimingsCallback(firstFrameCallback!);
    }
  }
```

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

\_executeFrame()을 통해 본격적인 렌더링 처리 단계로 진입합니다.

#### 3. 프레임 처리: \_executeFrame() → handleDrawFrame()

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
