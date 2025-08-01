## Ch1. Flutter의 매력 Hot Reload의 원리

### 들어가며

플러터를 처음 접한 사람들이 가장 놀란 와우 포인트는 아마 Hot Reload, Hot Restart 일 것입니다.
코드를 수정하고 저장만 하면 실행 중인 기기에 바로 반영이 된다는 것은 분명 엄청난 일이 맞습니다.
이 덕분에 플러터로 UI를 개발할 때 그 생산성이 아주 높아진 것은 부정할 수 없습니다.
Hot Reload 뿐만 아니라 앱 자체를 다시 실행하는 Hot Restart 역시 다른 도구들보다 한참 빠른 속도를 자랑합니다.

이번 글에서는 이런 Hot Reload와 Hot Restart의 근간이 되는 기술들을 설명하고, 이것이 어떻게 가능한지에 대해 설명합니다.
플러터 엔진이 어떻게 동작하는지가 아닌, 내가 작성한 Dart 코드가 어떻게 기기로 전달되어 실행되는지, Hot Reload와 Hot Restart는 어떻게 그렇게 빨리 기기로 전달되는지를 이해할 것입니다.

### Dart VM이 컴파일 하는 두가지 방식

첫번째 단계는 Dart에서 시작합니다. 플러터 코드는 다트를 기반으로 작성되는데, 다트는 컴파일 언어로 대부분의 언어들과 비슷한 컴파일 과정을 거치게 됩니다. 이때 다트는 두가지의 컴파일 방식(AOT, JIT)을 제공합니다.

먼저 요약하자면, **release / profile 환경에서는 AOT 방식으로 컴파일**이 되고, **debug 환경에서는 JIT 방식**으로 컴파일됩니다.

> profile 모드는 AOT 코드에 디버그 서비스를 일부 남겨 두는 점만 다릅니다. 아래 내용에서는 profile 모드에 대한 설명은 하지 않고 release(AOT), debug(JIT)에 대한 내용만 담습니다.

#### Dart - AOT 컴파일

**AOT(Ahead of Time)** 컴파일은 우리가 대부분의 상황에서 컴파일이라고 하면 생각나는 방식과 거의 동일합니다.
우리가 작성한 Dart 코드는 플러터 엔진에서 돌아가기 위해 기계어로 변환되는 컴파일 과정이 당연 필요합니다.
이 과정에서 Dart 코드는 1차적으로 Kernel IR이라고 불리는 중간단계로 한번 변환되고, 2차적으로 Kernel IR이 바이너리(이하 `스냅샷`)으로 변환됩니다. Kernel IR은 빌드하고자 하는 플랫폼에 관계 없이 동일한 결과물을 갖습니다. 따라서 Kernel IR은 어떤 빌드에서도 동일하게 생성되며, 동일한 Kernel IR 파일을 기준으로 각 플랫폼에 맞는 스냅샷을 만드는 것입니다. 이런 역할을 수행하는 변환기를 Common Frontend(CFE)라고 부릅니다. 공통 파일을 만드는 앞단이라는 표현이 아주 직관적이군요.
또한 AOT 컴파일은 단순히 Dart 코드를 기계어로 변환하는 것뿐만 아니라 _Tree-Shaking, Inlining, 레지스터 할당 최적화_ 등 코드를 최적화하는 여러 단계를 거칩니다. 이를 통해 좀 더 용량이 작은, 최적화된 스냅샷이 생성됩니다.

> Tree-Shaking : 사용되지 않는 클래스·함수·필드를 분석 후 제거해 바이너리 크기를 줄이는 과정
> Inlining : 짧은 호출을 함수 몸체로 직접 교체해 호출 오버헤드를 없애고 추가 최적화를 가능하게 하는 과정
> 레지스터 할당 최적화 : 자주 쓰는 변수를 CPU 레지스터에 배치해 메모리 접근을 최소화하는 과정

이 모든 과정을 거친 스냅샷은 그 자체로 기기에서 실행될 수 없습니다. 컴파일된 기계어 코드는 말그대로 기계어 코드일 뿐이고, 이것이 원래 작성된 Dart와 Flutter 기반의 동작을 수행하려면 Flutter 엔진과 Dart 실행 의존성(runtime)이 필요하기 때문입니다.
여기서 필요한 모든 실행 관련 의존성들은 스냅샷과 함께 앱 파일로 패키징됩니다. 뒤에서 설명하겠지만, AOT 방식으로 컴파일된 패키지는 더 적은 실행 의존성만으로도 충분합니다. 앞서 최적화된 스냅샷과 더 적은 실행 의존성 파일 덕분에 앱 용량이 많이 작아집니다.

AOT 컴파일 방식으로 최종 빌드된 앱은 아래와 같은 구성을 하고 있습니다.

```bash
## Android APK
lib/arm64-v8a/
 ├─ libflutter.so            ← Flutter Engine + precompiled runtime
 ├─ libapp.so*               ← AOT 스냅샷(단일 파일로 합쳐진 버전)
assets/flutter_assets/
 ├─ vm_snapshot_data         ← AOT 스냅샷(쪼개진 버전) #1
 ├─ vm_snapshot_instr        ← AOT 스냅샷(쪼개진 버전) #2
 ├─ isolate_snapshot_data    ← AOT 스냅샷(쪼개진 버전) #3
 ├─ isolate_snapshot_instr   ← AOT 스냅샷(쪼개진 버전) #4
 ├─ AssetManifest.json …
classes.dex, resources.arsc, AndroidManifest.xml
```

```bash
## iOS IPA
Payload/MyApp.app/
 ├─ Frameworks/
 │   ├─ Flutter.framework/Flutter     ← Flutter Engine + precompiled runtime
 │   └─ App.framework/App             ← AOT 스냅샷
 ├─ flutter_assets/
 ├─ Info.plist, entitlements, icon
```

> AOT 스냅샷은 원래 4개의 파일로 구성됩니다. 컴파일 과정을 통해 총 4개의 스냅샷이 생성된다고 보는게 더 정확하겠네요. 굳이 4개로 생성하는 이유는 각각의 역할이 구분되어 있기 때문입니다.
>
> vm_snapshot_data : Dart, Flutter 런타임을 위한 메타 데이터
> vm_snapshot_instruction : Dart, Flutter 런타임 실행을 위한 네이티브 코드
> isolate_snapshot_data : 내 앱 실행을 위한 전역변수, 정적필드 값 등 메타 데이터
> isolate_snapshot_instruction : 내가 작성한 Dart 코드 기반의 기계어 코드(main 함수 등)

여기까지의 내용을 시간 순서대로, 단계별로 정리한 설명은 아래와 같습니다.

| 단계                     | 호스트(빌드 머신)                                                                                                                | 디바이스(실제 사용자 기기)                                                  |
| ------------------------ | -------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| **1. 빌드 명령**         | `flutter build apk` / `flutter build ios` (Release)                                                                              | —                                                                           |
| **2. 프론트엔드 컴파일** | `CFE`가 모든 Dart 파일을 **Kernel IR** 로 컴파일                                                                                 | —                                                                           |
| **3. gen_snapshot**      | 동일 Kernel IR을 아키텍처별 **머신코드(스냅샷)** 로 AOT 컴파일                                                                   | —                                                                           |
| **4. 최적화**            | **트리 쉐이킹** 등 다양한 기법으로 코드·리소스 최소화                                                                            | —                                                                           |
| **5. 패키징**            | Android → 스냅샷을 `assets/flutter_assets` 또는 `libapp.so`에 포함<br>iOS → `App.framework` 내부 심볼(`kDartVmSnapshot…`)로 포함 | —                                                                           |
| **6. 설치 & 로딩**       | 실 기기에 앱 설치, 실행                                                                                                          | OS가 엔진 라이브러리를 로드 → **precompiled runtime** 초기화                |
| **7. Isolate 부팅**      | —                                                                                                                                | 엔진이 스냅샷을 **Read+Execute** 로 맵핑 → `main()` 네이티브 코드 직접 실행 |
| **8. 실행**              | —                                                                                                                                | 추가적인 코드 생성 없이 실행됨                                              |

#### Dart - JIT 컴파일

두번째는 **JIT(Just in Time)** 컴파일입니다. 여기서도 마찬가지로 **CFE**가 Dart 코드를 **Kernel IR**로 변환하는 1차 중간단계를 거칩니다. AOT 컴파일과 다른 점은, 이렇게 만들어진 중간 결과물이 그대로 실행할 기기로 바로 전송되며 기기 내에서 **스크립트 스냅샷(`snapshot_blob.bin`)** 형태의 바이너리로 저장된다는 것입니다. 앞선 스냅샷들과 달리 중간 결과물이 담긴 스냅샷을 스크립트 스냅샷이라고 합니다.
중간단계 결과물이 앱에 그대로 전송되는 방식이므로, 기기에서 실행 가능한 기계어 코드로 변환하는 기능이 AOT 처럼 개발자 PC에서 진행되는게 아니라 실행 기기에서 진행됩니다. 따라서 이를 위한 Dart VM JIT 컴파일로도 앱에 함께 패키징 되어야 합니다.
AOT 컴파일에서는 이미 기계어로 변환된 코드 스냅샷이 앱으로 들어가기 때문에 경량화된 precompiled runtime만 앱에 함께 넣으면 실행하는데 충분했지만, JIT 컴파일에서는 함수 호출 시점에 기계어로 변환하기 위한 의존성도 앱에 포함됩니다.
뿐만 아니라 디버깅 단계에서 동작하기 위한 다양한 디버깅 도구도 포함되어야 합니다. 그러다보니 AOT 컴파일 단계를 거친 릴리즈 용 앱보다 용량이 더 커지게 되죠.

JIT 컴파일 방식으로 빌드된 앱은 다음 구조를 갖습니다.

```bash
## Android
lib/arm64-v8a/
 ├─ libflutter.so            # Engine + full Dart VM (JIT, DevTools)
 ├─ libplugins_xyz.so        # 네이티브 플러그인 so
assets/flutter_assets/
 ├─ kernel_blob.bin          # 전체 Dart 코드의 Kernel IR
 ├─ vm_snapshot_data         # 자리표시자 stub (몇 KB)
 ├─ vm_snapshot_instr        # ─〃─
 ├─ isolate_snapshot_data    # ─〃─
 ├─ isolate_snapshot_instr   # ─〃─
 ├─ icudtl.dat               # 국제화 데이터
 ├─ AssetManifest.json
 ├─ FontManifest.json
 └─ ...                      # 이미지·폰트 등 앱 자산
classes.dex                  # Flutter bootstrap Java 코드
AndroidManifest.xml
resources.arsc
res/ ...
```

```bash
## iOS
Payload/Runner.app/
 ├─ Frameworks/
 │   ├─ Flutter.framework/Flutter     # Engine + full Dart VM (JIT)
 │   └─ App.framework/
 │       └─ App                       # Debug stub (엔트리 포인트만)
 ├─ flutter_assets/
 │   ├─ kernel_blob.bin               # Kernel IR
 │   ├─ vm_snapshot_data              # 자리표시자 stub
 │   ├─ vm_snapshot_instr             # ─〃─
 │   ├─ isolate_snapshot_data         # ─〃─
 │   ├─ isolate_snapshot_instr        # ─〃─
 │   ├─ icudtl.dat
 │   ├─ AssetManifest.json
 │   ├─ FontManifest.json
 │   └─ ...                           # 이미지·폰트 등 자산
 ├─ PlugIns/
 │   └─ *.framework / *.dylib         # 네이티브 플러그인 코드
 ├─ Info.plist
 ├─ AppIcon60x60@2x.png …
 └─ _CodeSignature/ …
```

대신 그 덕분에 **Hot Reload**가 가능해집니다.
개발자가 디버깅 과정에서 앱을 실행시켰다고 가정해봅시다. 그렇다면 개발자 PC에는 CFE 프로세스가 구동됩니다.
실행 기기에서는 앱 파일과 함께 패키징된 **Flutter Engine, Dart VM JIT 컴파일러 및 디버그 도구**가 동작합니다.
이때 개발자가 코드를 수정하고 저장, Hot Reload를 실행시키면 IDE는 바뀐 `.dart` 파일을 개발자 PC의 flutter-tools에 전달합니다.
이를 통해 CFE 프로세스는 변화한 코드에 대해 부분적인 컴파일을 즉시 실시합니다. 이 과정은 수 ms밖에 걸리지 않습니다.
그렇게 컴파일된 **Kernel IR** 파일은 실행 기기에서 돌고 있는 Dart VM으로 전송됩니다.
실행 기기 내의 Dart VM은 필요한 클래스 부분들을 교체하고 필요한 함수만 즉시 JIT 재컴파일을 실시합니다.
이와 같은 과정을 거쳐 아주 작은 시간 내에 변화한 부분만 재컴파일 과정을 거쳐 실행 기기에 전달되는 것입니다.

그렇다면 **Hot Restart**는 어떨까요?
Hot Restart를 실행시키면 실행 기기에서 돌고 있던 main 함수를 비롯한 Isolate 프로세스들이 종료됩니다.
여러 객체들을 포함한 말그대로 State들이 전부 지워지게 되는 것입니다.
그와 동시에 개발자 PC의 CFE는 부분적으로 컴파일했던 Hot Reload와 달리, 전체 Kernel IR을 컴파일합니다.
컴파일된 Kernel IR 파일은 실행 기기의 Dart VM으로 전달됩니다. 이때 다시 실행하기 위해 새로운 Isolate 프로세스를 시작합니다.
이미 Dart VM, Flutter engine은 무언가를 실행시킬 준비가 되어있기 때문에 앱을 아예 새로 시작하는 것보다 한참 빠르게 동작합니다.

자세한 단계별 설명은 다음과 같습니다.

| 단계                     | 호스트(개발자 PC)                                                                                                                           | 디바이스(에뮬레이터·실기기)                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **1. 빌드 명령**         | `flutter run` (Debug) 실행                                                                                                                  | —                                                                                 |
| **2. 프론트엔드 컴파일** | `CFE`가 모든 Dart 파일을 **Kernel IR** 로 컴파일 (캐시 유지)                                                                                | —                                                                                 |
| **3. 패키징**            | `libflutter.so`/`Flutter.framework`(엔진 + Dart VM JIT) 포함<br>`app.dill` → `kernel_blob.bin`<br>서비스 프로토콜·DevTools 디버그 심볼 포함 | —                                                                                 |
| **4. 설치 & 로딩**       | adb / Xcode를 통해 APK·IPA 설치                                                                                                             | OS가 엔진 라이브러리 로드 → **Dart VM JIT 초기화**                                |
| **5. Isolate 부팅**      | —                                                                                                                                           | `kernel_blob.bin` 읽어 **스크립트 스냅샷** 생성 → `main()` Isolate 실행           |
| **6. Hot Reload**        | 저장(Ctrl-S) → 수정 파일 목록 전달 → `frontend_server`가 **증분 .dill** 생성 → VM Service로 푸시                                            | VM이 클래스 바이트코드 갱신 → 변경 함수 **즉시 JIT** → `reassemble()` → 새 프레임 |
| **7. Hot Restart**       | `R` 키 → 새 스냅샷 전체 전송                                                                                                                | 기존 Isolate 종료 → 새 Isolate 부팅 (상태 초기화)                                 |
| **8. 디버깅**            | DevTools, Observatory, 로그, 메모리·CPU 프로파일링                                                                                          | VM Service로 실시간 통계 제공                                                     |
