## 개요
2025년 3월 기준 플러터 앱 개발자들이 거쳐야 하는 배포 과정에 대해 차근차근 정리합니다.
공통적으로 준비할 사항과 Play Store(안드로이드)와 AppStore(iOS) 각각에 대해 준비할 내용을 시간 순서에 맞게 정리했습니다.

### 0. 각종 이름 바꾸기/버전 업데이트
1. 패키지 이름 변경(한방에)
```bash
$ dart pub add change_app_package_name --dev
$ dart pub run change_app_package_name:main com.new.name
```

2. 앱 라벨 이름 변경
실제 기기 화면에서 보여지게될 앱 이름
* 안드로이드
android/app/src/main/AndroidManifest.xml
  ```xml
  android:label="My App Name"
  ```
  ![안드로이드 라벨 네임](https://velog.velcdn.com/images/taebbong/post/2663674a-e1c9-422d-bbfb-9b093f9bd445/image.png)
* iOS
  ios/Runner/Info.plist
  ```plist
  <key>CFBundleDisplayName</key>
  <string>My App Name</string>
  ```
  ![iOS DisplayName](https://velog.velcdn.com/images/taebbong/post/efddc130-e0ac-4b5c-b79e-60155cd86615/image.png)
  
3. 앱 버전 업데이트
버전 관리는 pubspec.yaml의 값을 다른 파일에서 참조하기 때문에 여기만 수정하면 문제 없이 적용된다.
  ```yaml
  name: chungmo
  description: "A new Flutter project."
  publish_to: 'none'
  version: 1.0.0+1
  ```
버전 규칙 : 1(major).0(minor).0(maintain)+1(version_code)
major, minor, maintain의 경우 통상적인 개념을 적용하면 되며, 안드로이드의 경우 매번 배포마다 version_code를 하나씩 증가시켜야 한다. 이전과 같은 version_code를 사용하면 배포가 안된다.

### 1. 앱 아이콘 변경
1. 아이콘 만들기
* 디자인이 없다면
[https://www.iconikai.com/](https://www.iconikai.com/) 사이트를 활용해 AI로 아이콘 생성

* 아이콘 이미지 디자인이 있다면
[https://www.appicon.co/](https://www.appicon.co/)
을 통해서 이미지를 아이콘 양식에 맞게 변경
![appicon.co](https://velog.velcdn.com/images/taebbong/post/85295222-9d0a-4cbf-9afa-c9889800572e/image.png)

2. 아이콘 변경
압축 풀고 나서 폴더에 있는 리소스들을 적절히 복사/덮어쓰기
![res](https://velog.velcdn.com/images/taebbong/post/62840672-dea7-4776-986b-48b49c387f4c/image.png)
* 안드로이드
![android_res](https://velog.velcdn.com/images/taebbong/post/267a8396-06fd-439e-9299-a8bfa6d0da87/image.png)
* iOS
![ios_res](https://velog.velcdn.com/images/taebbong/post/228126e9-5752-4e32-8009-11d570e6e16c/image.png)
> 아이콘 제너레이터 서비스에 따라 파일명이 다를 수 있으며, 이런 경우 대부분 Contents.json에 관련 코드를 첨부해주니 Contents.json까지 복사/덮어쓰기

### 2. 스토어 업로드 용 디자인 리소스 만들기
* [previewed.app](previewed.app) 페이지에서 스크린샷을 활용해 예쁜 이미지 만들기
* 귀찮으면 에뮬레이터/기기에서 스크린샷만 찍어놔도 무방

### 3. (안드로이드) 앱 서명

업로드 키 생성

```bash
  keytool -genkey -v -keystore %userprofile%\upload-keystore.jks -storetype JKS -keyalg RSA -keysize 2048 -validity 10000 -alias upload // 기본 jks 타입 키 생성
  keytool -importkeystore -srckeystore C:\upload-keystore.jks -destkeystore C:\upload-keystore.jks -deststoretype pkcs12 // pkcs12 방식 키로 migrate
```

android/key.properties 생성

```properties
storePassword=paaassswoord
keyPassword=passsswowwword
keyAlias=upload
storeFile=C:\\upload-keystore.jks
```

android/app/build.gradle 수정

```gradle
plugins {
    id "com.android.application"
    id "kotlin-android"
    // The Flutter Gradle Plugin must be applied after the Android and Kotlin Gradle plugins.
    id "dev.flutter.flutter-gradle-plugin"
}

def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    namespace = ...
    compileSdk = flutter.compileSdkVersion
    ndkVersion = flutter.ndkVersion
    ...

    defaultConfig {
        ...
    }

    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            // TODO: Add your own signing config for the release build.
            // Signing with the debug keys for now, so `flutter run --release` works.
            signingConfig = signingConfigs.release
        }
    }
}

flutter {
    source = "../.."
}
```

### 4. (안드로이드) 앱 빌드

```bash
$ flutter clean
$ dart pub get
$ flutter build appbundle
```

### 5. (안드로이드) Play Store 등록
[https://play.google.com/console](https://play.google.com/console)에 접속
![](https://velog.velcdn.com/images/taebbong/post/2289f97d-1036-4fc1-a358-ba6b9cfe1877/image.png)

앱 만들기 선택

![](https://velog.velcdn.com/images/taebbong/post/56dd5d85-82d3-4ad5-9960-bc0437b20b5c/image.png)

내용에 맞게 입력

![](https://velog.velcdn.com/images/taebbong/post/62030834-8125-4697-b144-1969f2f17ffa/image.png)

여기서 앱 설정 할 일 보기 토글하여 조치 시작

![](https://velog.velcdn.com/images/taebbong/post/849bece8-87f4-4c77-8963-86fc9bdd2de5/image.png)

1. 개인정보처리방침

![](https://velog.velcdn.com/images/taebbong/post/d77b0967-3073-4ceb-9fe4-6338da704e7f/image.png)

2. 앱 액세스 권한(구글 앱 심사팀이 테스트할 때 로그인이 필요한 경우 계정 정보 입력)

![](https://velog.velcdn.com/images/taebbong/post/d72305ff-5c7e-4d9f-9879-9946ffd88efc/image.png)

3. 광고

![](https://velog.velcdn.com/images/taebbong/post/731fa712-5713-442f-99cd-569c18ca2d09/image.png)

4. 콘텐츠 등급(설문지 입력)

![](https://velog.velcdn.com/images/taebbong/post/3a07fb54-0504-488e-95a5-f395b92c4086/image.png)

5. 타겟층 및 콘텐츠

![](https://velog.velcdn.com/images/taebbong/post/e8cbaeba-89c6-484a-959a-07031333c941/image.png)

6. 뉴스 앱

![](https://velog.velcdn.com/images/taebbong/post/9d5003af-4aa3-492e-8e1c-299db9cbcf79/image.png)

7. 데이터 보안

![](https://velog.velcdn.com/images/taebbong/post/5a06cda4-0959-4f08-96ba-817187a46181/image.png)

8. 정부 앱

![](https://velog.velcdn.com/images/taebbong/post/c1a213cf-0b91-4ce3-9a61-08853400b172/image.png)

9. 금융 기능

![](https://velog.velcdn.com/images/taebbong/post/fd901970-3750-45e3-97a3-24b0a7343074/image.png)

10. 건강 기능

![](https://velog.velcdn.com/images/taebbong/post/d8074d29-d072-4d6f-a329-12ed61642d48/image.png)

11. 스토어 설정

![](https://velog.velcdn.com/images/taebbong/post/a3cef763-93db-4a38-b6c0-d8909cd41834/image.png)

12. 기본 스토어 등록정보 만들기

일단 여기까지 입력하고 대시보드로 돌아가면 아래와 같은 화면이 나옴.

![](https://velog.velcdn.com/images/taebbong/post/6e293593-66c8-4a41-8fd7-9635727f678e/image.png)

이제 테스트 출시를 차근차근 진행하면 된다.
테스트는 내부 테스트 / 비공개 테스트 / 정식 출시로 구성되며 각각 진행하면 안전한 배포가 가능하다.
내부테스트부터 진행해보면,

#### 1. 내부 테스트 등록

![](https://velog.velcdn.com/images/taebbong/post/8a4389f6-86cf-46c0-9ab0-74312eeaa962/image.png)

테스터 선택을 눌러 진행

![](https://velog.velcdn.com/images/taebbong/post/9cf28b96-488b-4fe2-ac6d-777750daee38/image.png)

본인, 지인, 동료 등 말그대로 내부 테스트를 위한 등록으로, 이메일을 넣어서 관리할 수 있다.
여기서는 본인만 넣어도 무방하긴 함.
테스터를 설정한 후 새 버전 만들기를 선택.

![](https://velog.velcdn.com/images/taebbong/post/d3591267-e7f5-48f3-82a9-848aec815ebb/image.png)

앞서 빌드한 앱 번들(aab) 파일을 첨부.

![](https://velog.velcdn.com/images/taebbong/post/5361a8e0-fc5c-48a6-bf65-c102da47c622/image.png)

저장하면 위와 같이 출시 완료!



### 6. (iOS) App Store 등록
개발자 계정은 준비되었다고 가정하고...


