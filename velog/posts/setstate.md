## 개요

플러터의 기본 상태 관리 도구는 아마도 `State` 클래스와 `setState()` 메소드일 것입니다.
그동안 큰 고민 없이 이들을 사용해왔지만, 문득 `setState()`를 통해 상태 변화를 만들었을 때 내부적으로 어떤 과정을 통해 화면이 갱신되는지 궁금해졌습니다.
이번 글에서 플러터의 소스코드를 분석하며 `setState()` 호출부터 실제 화면이 리빌드되는 과정까지 단계별로 알아보겠습니다.

_핵심적인 흐름에 집중하기 위해 일부 코드는 생략했으므로, 전체 소스코드를 확인하고 싶은 분들은 플러터 공식 리포지토리를 직접 참고하시길 권장합니다._

### 1. setState()를 호출했을 때 일어나는 일

분석의 시작점은 `State` 클래스와 `setState()` 메소드의 구현체입니다.
플러터에서 `setState()` 메소드는 다음 경로에 선언되어 있습니다:

- **파일 경로:** `packages/flutter/lib/src/widgets/framework.dart`

```dart
abstract class State<T extends StatefulWidget> with Diagnosticable {
  T get widget => _widget!;
  T? _widget;

  StatefulElement? _element;

  @protected
  void setState(VoidCallback fn) {
    final Object? result = fn() as dynamic;
    _element!.markNeedsBuild();
  }
}
```

코드를 보면 `setState()`는 전달받은 콜백(`fn`)을 실행한 후, 연결된 `StatefulElement`의 `markNeedsBuild()`를 호출하는 것을 볼 수 있습니다.
이 콜백은 우리가 앱을 만들면서 setState()를 사용할 때 내부에 작성하는 기능일 것입니다.

예시:

```dart
IconButton(
  icon: Icon(_isCalendarView ? Icons.list : Icons.calendar_month),
  onPressed: () {
    setState(() {
      _isCalendarView = !_isCalendarView;
    });
  },
),
```

`markNeedsBuild()`는 이름에서도 알 수 있듯, "이 `element`가 다시 빌드되어야 한다는 것을 명시하는 메소드"입니다. 이 메소드의 주석을 살펴보면, 좀 더 명확히 이해할 수 있습니다.

> Marks the `element` as `dirty` and adds it to the global list of widgets to rebuild in the next frame.
>
> (요약) `element`를 `dirty`로 표시하고, 다음 프레임에서 재구성될 전역 위젯 목록에 추가한다.

그러면 이제 `markNeedsBuild()` 메소드를 조금 더 살펴보겠습니다.
`markNeedsBuild()`는 `StatefulElement`에서 구현된 메소드입니다.

- **파일 경로:** `packages/flutter/lib/src/widgets/framework.dart`

```dart
abstract class StatefulElement... {
  bool _dirty = true;
  BuildOwner owner;

  void markNeedsBuild() {
    if (dirty) return;
    _dirty = true;
    owner!.scheduleBuildFor(this);
  }
}
```

여기서 중요한 부분은 바로 `BuildOwner`에게 이 `element`를 `this` 키워드로 전달하여 빌드를 예약한다는 것입니다. 실제로 주석을 확인하면,

> Adds an element to the dirty elements list so that it will be rebuilt when [WidgetsBinding.drawFrame] calls [buildScope].
>
> (요약) `[WidgetsBinding.drawFrame]`이 호출될 때 `[buildScope]`를 통해 다시 빌드될 수 있도록 dirty 리스트에 엘리먼트를 추가한다.

현재까지 내용을 정리하자면, `setState()`는 `element`를 `dirty`로 표시하고, `BuildOwner`에게 해당 `element`에 대한 빌드를 예약합니다.

`dirty`로 표시된 `element`를 모아놓은 `dirty element list`에 대해 어떻게 빌드가 진행되는지 알아봐야겠습니다.

### 2. drawFrame: 그럼 실제로 리빌드는 어떻게 이뤄지는가?

이제 화면이 실제로 리빌드되는 과정인 `WidgetsBinding.drawFrame()`을 살펴보겠습니다. `WidgetsBinding` 클래스에 정의된 `drawFrame()`의 역할을 원본 소스코드의 주석을 토대로 요약하면 다음과 같습니다.

1. 새로운 프레임을 만드는 것은 `handleBeginFrame`과 `handleDrawFrame`이 담당하며, 이들은 플러터 엔진이 호출한다.
2. `handleDrawFrame`은 `drawFrame`을 호출하며, 여기서 앞서 `dirty`로 표시된 `element`들이 리빌드된다.

그럼 실제 구현된 코드를 살펴보겠습니다.

- **파일 경로:** `lib/src/widgets/binding.dart`

```dart
@override
void drawFrame() {
  assert(!debugBuildingDirtyElements);
  try {
    if (rootElement != null) {
      buildOwner!.buildScope(rootElement!);
    }
    super.drawFrame();
    buildOwner!.finalizeTree();
  }
}
```

여기서 중요한 것은 `buildOwner.buildScope(rootElement)`가 호출된다는 것입니다.

- **파일 경로:** `lib/src/widgets/framework.dart`

```dart
void buildScope(Element context, [VoidCallback? callback]) {
  final BuildScope buildScope = context.buildScope;
  if (callback == null && buildScope._dirtyElements.isEmpty) return;

  try {
    _scheduledFlushDirtyElements = true;
    buildScope._building = true;
    buildScope._flushDirtyElements(debugBuildRoot: context);
  }
}
```

`flushDirtyElements`를 호출하여 `dirtyElements`에 대한 빌드를 진행합니다.
이를 통해 실제 `dirty`로 표시된 `element`들이 리빌드됩니다.

여기서 조건문을 통해 `dirtyElements`가 없으면 그냥 `return`을 하는데, 이를 통해 실제로 빌드할 필요가 없으면 안하는 방식으로 효율을 챙긴 모습입니다.

`flushDirtyElements`도 마저 확인해보면,

- **파일 경로:** `lib/src/widgets/framework.dart`

```dart
void _flushDirtyElements({required Element debugBuildRoot}) {
  _dirtyElements.sort(Element._sort);
  try {
    for (int index = 0; index < _dirtyElements.length; index++) {
      final Element element = _dirtyElements[index];
      if (identical(element.buildScope, this)) {
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
  }
}
```

최종적으로 `dirty`로 표시된 모든 `dirtyElements` 리스트를 순회하여 `tryRebuild(element)`를 통해 각 `element`를 리빌드함을 알 수 있습니다.

### 3. frame이란?

앞서 새로운 프레임을 만든다, drawFrame과 같은 키워드가 나왔었습니다.
플러터에서 화면을 그릴 때 프레임이라는 단위로 구분한다는 것을 유추할 수 있는데, 프레임에 대해 이해하고 넘어가겠습니다.

플러터 앱은 기본적으로 초당 60프레임(fps)을 기준으로 동작합니다. 즉, 초당 최대 60번의 화면 갱신이 일어난다는 의미입니다.
플러터 엔진은 이를 활용해 프레임 기반의 렌더링 루프를 운용합니다.
즉, 1초에 60번씩 화면을 새로 렌더링할지 말지를 결정하게 되며, 앞서 보았던 것처럼 `dirty elements`가 있으면 해당 프레임에서 `element`를 리빌드하는 것입니다.

이 렌더링 루프의 핵심 함수가 바로 `handleBeginFrame`, `handleDrawFrame`입니다.

- **파일 경로:** `lib/src/scheduler/binding.dart`

```dart
void handleBeginFrame(Duration? rawTimeStamp) {
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

`handleBeginFrame()`의 역할은 **일시적인(transient) 콜백**들을 전부 실행하는 것입니다.

프레임 생성 과정에서 실행되는 콜백은 3종류가 있는데,

1. `TransientCallbacks` : 프레임 시작 시 애니메이션이나 시간에 따라 변하는 콜백
2. `PersistentCallbacks` : 매 프레임마다 반드시 실행되는 콜백(렌더링, 레이아웃 작업 등)
3. `PostFrameCallbacks` : 프레임이 끝난 후에 실행되는 콜백

이 중에서 `handleBeginFrame`에서는 `TransientCallbacks`이 실행됩니다.

다음은 `handleDrawFrame` 입니다.

- **파일 경로:** `lib/src/scheduler/binding.dart`

```dart
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
  }
}
```

여기서는 마찬가지로 등록된 `PersistentCallbacks`와 `PostFrameCallbacks`를 실행합니다.

여기까지 프레임 생성 관련 코드를 살펴보았는데, `dirty elements`를 다시 그리는 `drawFrame`은 두 단계 중 어느 단계에서 동작하게 될까요?

결론부터 말하면, `drawFrame`은 `PersistentCallbacks`에 등록되는 작업입니다.

앞선 `drawFrame`이 선언된 파일을 자세히 보면,

- **파일 경로:** `lib/src/rendering/binding.dart`

```dart
mixin RendererBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _rootPipelineOwner = createRootPipelineOwner();
    addPersistentFrameCallback(_handlePersistentFrameCallback); // _handlePersistentFrameCallback을 등록
  }

  void _handlePersistentFrameCallback(Duration timeStamp) {
    drawFrame(); // 반가운 얼굴 drawFrame()
    _scheduleMouseTrackerUpdate();
  }
```

이렇게, `drawFrame`이 `\_handlePersistentFrameCallback`에서 실행되고, 이는 결국 `addPersistentFrameCallback`을 통해 등록되어 방금 확인한 `handleDrawFrame`에서 실행되는 것입니다.

`handleBeginFrame`과 `handleDrawFrame`은 플러터 엔진에서 매 프레임마다 실행해주기 때문에, 결론적으로는 1초에 60번씩 상태가 변경된 `element`들을 리빌드하고 화면에 반영할 수 있는 기회가 있는 것입니다.

### 4. 종합 정리

`setState()`를 호출하면 다음과 같은 순서로 리빌드가 진행됩니다.

1. `setState()`가 실행된 후, `StatefulElement.markNeedsBuild()` 호출됨
2. 이 메소드는 `BuildOwner.scheduleBuildFor(element)`를 호출하여 해당 Element를 **dirtyElements** 리스트에 추가
3. 다음 프레임이 시작될 때 `SchedulerBinding.handleDrawFrame()` 호출, 등록된 `PersistentCallbacks`들을 실행
4. 등록된 `PersistentCallbacks` 중에는 `WidgetsBinding.drawFrame()`메소드를 실행하는 `_handlePersistentFrameCallback()`가 있어서 결국 `BuildOwner`에게 루트 Element(`rootElement`)로부터 시작하는 리빌드를 요청
5. `BuildOwner`는 `flushDirtyElements()`를 통해 `dirty`로 표시된 모든 `Element`를 순서대로 리빌드

이렇게 한 프레임 단위로 모든 상태 변화에 따른 변경사항이 반영됩니다.

`setState()`를 사용하는 입장에서는 크게 중요하지 않을 수도 있겠지만, 여기서 중요한 사실들을 알아갈 수 있는데,

> `dirty`로 표시된 `element`는 프레임 단위로 리빌드가 된다. `setState()`를 남발하거나 큰 `element`에서 `setState()`를 적용하면 해당 프레임에서 할 일이 많아지므로 성능의 저하가 일어날 수 있다.

우리는 추상적으로 `StatefulWidget`와 `setState()`가 `StatelessWidget`에 비해 성능에 부하가 많이 걸린다고 알고 있었습니다.
프레임 단위에서 상태 변화가 일어나고 `dirty`로 표시된 `element` 전체가 리빌드된다는 것을 확인한 만큼, 생각없이 작성한 `StatefulWidget`과 `setState()`가 성능에 큰 영향을 줄 것이라는 것을 느낄 수 있습니다.

`StatefulWidget`과 `setState()`를 잘 쓰려면 최소한의 단위로 위젯을 쪼개서 사용하는 것이 좋겠습니다.
`dirty`로 표시되는 위젯, `element`의 크기가 작을수록 다음 프레임에서 리빌드 해야하는 `element`가 적어지기 때문입니다.

### 5. 마치며

여기까지 우리가 흔히, 편하게 쓰던 `setState()`의 동작 원리와 플러터 엔진이 프레임을 구성하는 방법에 대해 알아보았습니다.
다음에는 `GetX`에서 사용하는 `Obx()`의 동작 원리를 분석하고 `setState()`와 비교해보겠습니다.
