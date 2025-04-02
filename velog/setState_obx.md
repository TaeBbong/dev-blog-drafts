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

### 3. frame이란?

여기서 또 하나 중요한 사실은, 우리의 앱이 돌아가는 기기들은 1초에 60번 이상 새로운 프레임을 그립니다.
플러터에서는 60fps를 기준으로 앱을 실행할 때 1초에 최대 60번 화면을 그릴 수 있는 프레임 기반의 렌더링 루프를 운용합니다.
즉 1초에 60번씩 화면을 새로 렌더링할지 말지를 결정하게 되며, 앞서 보았던 것처럼 dirty elements가 있으면 해당 프레임에서 element를 리빌드하는 것입니다.

이 렌더링 루프의 핵심 함수가 바로 handleBeginFrame, handleDrawFrame입니다.

```dart
// lib/src/scheduler/binding.dart
void handleBeginFrame(Duration? rawTimeStamp) {
  ...
  try {
    // TRANSIENT FRAME CALLBACKS
    _schedulerPhase = SchedulerPhase.transientCallbacks;
    final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
    _transientCallbacks = <int, _FrameCallbackEntry>{};
    callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
      if (!_removedIds.contains(id)) {
        _invokeFrameCallback(
          callbackEntry.callback,
          _currentFrameTimeStamp!,
          callbackEntry.debugStack,
        );
      }
    });
    _removedIds.clear();
  } finally {
    _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
  }
}
```

handleBeginFrame의 역할은 간단합니다.
일시적인(transient) 콜백들을 전부 실행하는 것입니다.
여기서 일시적인 콜백들이라는 것은 애니메이션이 진행될 때마다 호출되는 콜백들인데, 프레임 단위의 실행을 위해 \_transientCallbacks에 등록되는 것 같아 보입니다.
여기서는 우리가 원하는 dirty element들의 리빌드와 관련 없으니 간단히 넘어가겠습니다.

다음은 handleDrawFrame 입니다.

```dart
// lib/src/scheduler/binding.dart
void handleDrawFrame() {
  assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
  _frameTimelineTask?.finish(); // end the "Animate" phase
  try {
    // PERSISTENT FRAME CALLBACKS
    _schedulerPhase = SchedulerPhase.persistentCallbacks;
    // 모든 persistent callbacks 실행
    for (final FrameCallback callback in List<FrameCallback>.of(_persistentCallbacks)) {
      _invokeFrameCallback(callback, _currentFrameTimeStamp!);
    }

    // POST-FRAME CALLBACKS
    _schedulerPhase = SchedulerPhase.postFrameCallbacks;
    final List<FrameCallback> localPostFrameCallbacks = List<FrameCallback>.of(
      _postFrameCallbacks,
    );
    _postFrameCallbacks.clear();
    if (!kReleaseMode) {
      FlutterTimeline.startSync('POST_FRAME');
    }
    try {
      // 모든 post callbacks 실행
      for (final FrameCallback callback in localPostFrameCallbacks) {
        _invokeFrameCallback(callback, _currentFrameTimeStamp!);
      }
    } finally {
      if (!kReleaseMode) {
        FlutterTimeline.finishSync();
      }
    }
  } finally {
    _schedulerPhase = SchedulerPhase.idle;
    _frameTimelineTask?.finish(); // end the Frame
    assert(() {
      if (debugPrintEndFrameBanner) {
        debugPrint('▀' * _debugBanner!.length);
      }
      _debugBanner = null;
      return true;
    }());
    _currentFrameTimeStamp = null;
  }
}
```

위와 같이 동작은 마찬가지로 등록된 persistentCallbacks와 postFrameCallbacks를 실행합니다.
그럼 여기서 언제 drawFrame을 실행할까요?
바로 persistentCallbacks에 drawFrame이 예약되어있습니다.

앞선 drawFrame이 선언된 파일을 자세히 보면,

```dart
// lib/src/rendering/binding.dart
mixin RendererBinding
    on
        BindingBase,
        ServicesBinding,
        SchedulerBinding,
        GestureBinding,
        SemanticsBinding,
        HitTestable {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _rootPipelineOwner = createRootPipelineOwner();
    platformDispatcher
      ..onMetricsChanged = handleMetricsChanged
      ..onTextScaleFactorChanged = handleTextScaleFactorChanged
      ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged;
    addPersistentFrameCallback(_handlePersistentFrameCallback);
    initMouseTracker();
    if (kIsWeb) {
      addPostFrameCallback(_handleWebFirstFrame, debugLabel: 'RendererBinding.webFirstFrame');
    }
    rootPipelineOwner.attach(_manifold);
  }

  ...
  void _handlePersistentFrameCallback(Duration timeStamp) {
    drawFrame();
    _scheduleMouseTrackerUpdate();
  }
```

이렇게, drawFrame이 \_handlePersistentFrameCallback에서 실행되고, 이는 결국 addPersistentFrameCallback을 통해 등록되어 방금 확인한 handleDrawFrame에서 실행되는 것입니다.

handleBeginFrame과 handleDrawFrame은 플러터 엔진에서 프레임마다 실행해주기 때문에, 결론적으로는 1초에 60번 상태 변화를 만들 수 있는 기회가 있는 것입니다.
