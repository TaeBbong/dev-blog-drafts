## 개요

플러터의 기본 상태 관리 도구는 아무래도 State와 setState()입니다.
문득 setState()를 통해 상태 변화를 만들었을 때 어떻게 화면에 그 변화가 적용되는지 궁금해졌습니다.
이를 알아보기 위해 setState()부터 플러터 엔진의 화면 그리기 과정을 소스코드를 통해 확인, 분석해보았습니다.

_핵심 코드만을 보기 위해 많은 부분을 삭제하였으니, 자세한 코드와 주석을 확인하시려면 원본 파일을 찾아보시는 것을 추천드립니다._

### 1. setState: setState()를 호출했을 때 일어나는 일

먼저 찾을 코드는 setState 메소드를 선언한 부분입니다.
packages/flutter/lib/src/widgets/framework.dart 파일에 선언되어 있습니다.

```dart
// lib/src/widgets/framework.dart
abstract class State<T extends StatefulWidget> with Diagnosticable {
  T get widget => _widget!;
  T? _widget;
  ...
  StatefulElement? _element;

  @protected
  void setState(VoidCallback fn) {
    ...
    final Object? result = fn() as dynamic;
    ...
    _element!.markNeedsBuild();
  }
  ...
}
```

State 클래스에는 widget, StatefulElement \_element, 그리고 setState() 등의 메소드가 선언되어 있습니다.
여기서 setState()의 마지막 줄을 보면 \_element!.markNeedsBuild()을 호출하는 것을 알 수 있습니다.
함수명에서 알 수 있듯 이 element가 빌드가 필요하다는 것을 표시하겠다는 뜻인데, 이 메소드의 docstring에는 다음과 같은 내용이 있습니다.

> Marks the element as dirty and adds it to the global list of widgets to rebuild in the next frame.
> Since it is inefficient to build an element twice in one frame, applications and widgets should be structured so as to only mark widgets dirty during event handlers before the frame begins, not during the build itself.

주요 내용은 해당 element가 dirty하다고 표시하며, 이를 전역 list에 추가하여 다음 프레임에서 rebuild하도록 한다는 것입니다.

StatefulElement \_element의 메소드인 `markNeedsBuild()`도 한번 볼까요??

```dart
// lib/src/widgets/framework.dart
abstract class StatefulElement... {
  ...
  bool _dirty = true;
  bool _inDirtyList = false;
  bool _debugBuiltOnce = false;
  BuildOwner owner;
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

여기서는 BuildOwner owner.scheduleBuildFor(this)를 호출합니다.
this를 통해 element 자체를 전달하는 것을 알 수 있습니다.
코드를 그대로 읽어보면 BuildOwner에게 이 element를 위한 빌드 스케쥴을 예약해라 정도로 해석할 수 있겠네요.
해당 메소드의 docstring은,

> Adds an element to the dirty elements list so that it will be rebuilt when [WidgetsBinding.drawFrame] calls [buildScope].

결국 scheduleBuildFor(this)는 해당 element를 dirty로 표시, dirtyElements 리스트에 넣어주는 것까지 수행합니다.
그 이후에 아직 보지 않았지만 WidgetsBinding.drawFrame이 buildScope을 호출하면서 리빌드가 실행되고, 이 과정에서 해당 element가 새로 빌드될거라는 것을 알 수 있습니다.

우선 여기까지의 과정을 통해 setState()가 호출되었을 때 어떤 일이 일어나는지 모두 확인해보았습니다.
현재까지 확인한 것은, setState를 호출하면 해당 element를 dirty로 표시하고, BuildOwner에게 해당 element에 대한 빌드 스케쥴을 예약한다는 것입니다.

### 2. drawFrame: 그럼 실제로 리빌드는 어떻게 이뤄지는가?

앞선 과정에서 리빌드를 실제로 수행하는 것은 WidgetsBinding.drawFrame()과 buildScope이었습니다.
WidgetsBinding.drawFrame()부터 시작해봐야겠죠?

lib/src/widgets/binding.dart에 drawFrame() 메소드가 선언되어 있는데, docstring이 상당히 잘 작성되어 있었습니다.
총 10개의 step으로 나눠서 작성해놓았는데, 이를 먼저 간단히 정리해보겠습니다.

1. 새로운 프레임을 만드는 것은 handleBeginFrame과 handleDrawFrame이 담당하며, 이들은 플러터 엔진이 호출한다.
2. handleDrawFrame은 drawFrame을 호출하며, 여기서 앞서 dirty로 표시된 element들이 리빌드된다.

좀 더 깊은 곳까지 닿은 것 같습니다. 이에 대한 실제 drawFrame 메소드를 확인해보면,

```dart
// lib/src/widgets/binding.dart
@override
void drawFrame() {
  assert(!debugBuildingDirtyElements);
  assert(() {
    debugBuildingDirtyElements = true;
    return true;
  }());
  ...
  TimingsCallback? firstFrameCallback;
  bool debugFrameWasSentToEngine = false;
  ...
  try {
    if (rootElement != null) {
      buildOwner!.buildScope(rootElement!);
    }
    super.drawFrame();
    buildOwner!.finalizeTree();
  }
  ...
}
```

buildScope에게 rootElement를 인자로 전달하여, 최종적으로 element들을 빌드하는 것을 확인할 수 있습니다.
buildScope은 어떤식으로 구성되어있냐면,

```dart
// lib/src/widgets/framework.dart
void buildScope(Element context, [VoidCallback? callback]) {
  final BuildScope buildScope = context.buildScope;
  if (callback == null && buildScope._dirtyElements.isEmpty) { // dirtyElements가 없다? => 새로 빌드할 필요가 없다!
    return;
  }
  ...
  try {
    _scheduledFlushDirtyElements = true;
    buildScope._building = true;
    if (callback != null) {
      ...
      Element? debugPreviousBuildTarget;
      ...
      try {
        callback();
      } ...
    }
    buildScope._flushDirtyElements(debugBuildRoot: context);
  }
  ...
}
```

flushDirtyElements를 호출하여 dirtyElements에 대한 빌드 처리를 진행합니다.
여기서 조건문을 통해 dirtyElements가 없으면 그냥 return을 하는데, 이를 통해 실제로 빌드할 필요가 없으면 안하는 방식으로 효율을 챙긴 모습입니다.

flushDirtyElements도 마저 볼까요?

```dart
// lib/src/widgets/framework.dart
void _flushDirtyElements({required Element debugBuildRoot}) {
  _dirtyElements.sort(Element._sort);
  _dirtyElementsNeedsResorting = false;
  try {
    for (int index = 0; index < _dirtyElements.length; index = _dirtyElementIndexAfter(index)) {
      final Element element = _dirtyElements[index];
      if (identical(element.buildScope, this)) {
        assert(_debugAssertElementInScope(element, debugBuildRoot));
        _tryRebuild(element);
      }
    }
  } finally {
    for (final Element element in _dirtyElements) {
      if (identical(element.buildScope, this)) {
        element._inDirtyList = false;
      }
    }
    _dirtyElements.clear();
    _dirtyElementsNeedsResorting = null;
    _buildScheduled = false;
  }
}
```

결국 dirty로 표시된 모든 dirtyElements 리스트를 기반으로 tryRebuild(element)를 통해 각 element를 리빌드함을 알 수 있습니다.

여기까지의 분석 결론은 결국,

1. setState()를 통해 해당 element를 dirty로 표시
2. dirty로 표시된 element들은 플러터 엔진이 호출하는 drawFrame을 통해 리빌드가 됨

과정 대비 결론이 간단해서 허무하기도 하지만..

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
