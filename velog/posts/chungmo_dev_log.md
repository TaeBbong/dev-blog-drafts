### 들어가며

<img src="https://velog.velcdn.com/images/taebbong/post/232980df-0818-436a-be4b-be86d5d2c13e/image.png" width="100%">

모바일 청첩장 자동 저장 앱 **청모**를 개발하면서 고민하고 문제를 해결했던 과정을 공유합니다.

### 문제 인식 : 청모는 어떻게 탄생했는가?

<img src="https://velog.velcdn.com/images/taebbong/post/3b0cd276-4625-402b-8bb1-0db8f3cc8c7b/image.png" width="30%">

20대 후반이 되어가면서 주변에 결혼하는 분들이 많이 있습니다. 종이 청첩장도 좋지만, 받는 입장에선 일정을 확인할 때 보통 모바일 청첩장을 더 많이 보게 되더라구요.
청첩장을 받고나서 캘린더에 일정과 장소를 그때그때 저장하면 편하겠지만, 보통 저장하지 않고 매번 카톡 채팅방에 들어가서 모바일 청첩장을 찾아 일정을 확인하는 일이 참 많았습니다...
다른 분들도 저와 비슷한 경험을 많이 하셨을거라 생각합니다.

여기서 식별한 문제점은 2개가 있었는데,

> 1. 모바일 청첩장이 채팅방 마다 흩어져있으니, 찾는게 어렵다. 모여져 있으면 좋겠다.
> 2. 매번 모바일 청첩장을 받고 일정/장소 등 정보를 저장하기 귀찮다. 알아서 저장되면 좋겠다.

이런 문제점을 인식하여 **청모** 앱을 기획하게 되었습니다.

### 문제 해결 : 청모는 어떻게 문제를 해결하는가?

앞서 식별한 2개의 문제점을 **청모** 앱은 아래와 같이 해결합니다.

> 1. 모바일 청첩장 링크를 입력하여 저장, 모아서 볼 수 있다.
> 2. 모바일 청첩장을 "알아서" 파싱하여, 일정/장소 정보를 캘린더에 저장한다.

첫번째 문제에 대한 해결은 비교적 간단하고 뻔합니다. 모바일 청첩장 전용 메모앱을 만들겠다는 것이죠.
이 자체로도 누군가에겐 의미가 있을 수 있으나, 문제를 완전히 해결했다고 보기 어렵습니다.
모바일 청첩장의 내용을 **"알아서"** 파싱하여 일정 정보를 저장하는 것이 **"청모"**의 **"킥"**이라고 할 수 있겠습니다.

### 개념 증명 : GPT API가 "알아서" 모바일 청첩장을 파싱할 수 있을까?

![](https://velog.velcdn.com/images/taebbong/post/50e523f5-a499-4183-ace2-702d2727d6d5/image.png)

여기서 **"알아서"** 는 `GPT API`가 맡아주었습니다.
몇번의 프롬프트 테스트를 통해, `GPT API`에게 모바일 청첩장 웹 페이지 리소스를 전달하면,
100%에 가까운 정확도로 일정 정보를 파싱한다는 것을 확인했습니다.

![](https://velog.velcdn.com/images/taebbong/post/d535972e-c14b-426c-b486-d19c8b47f183/image.png)

초기에는 직접 모바일 청첩장 링크를 전달했습니다. 여기서 일정 정보를 파싱해달라고 했었죠.
하지만 `GPT API`는 웹 페이지 접근과 같은 인터렉션에 매우 제한적이었습니다.
~~이게 올해 초였는데, 요즘엔 웹 탐색 기능이 생기면서 이젠 링크만 던져줘도 알아서 잘 되더라구요...😅~~

```python
@https_fn.on_request(secrets=["OPENAI_API_KEY"])
def parse_voucher_handler(req: https_fn.Request) -> https_fn.Response:
    """POST로 link 데이터를 전달받아 html 파싱, gpt를 통해 적절한 데이터 추출 및 JSON(str)을 반환하는 API 핸들러"""
    try:
        data = req.get_json()
        OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY")
        voucher = data.get("link")
        parsed_result = HTML

        query = parsed_result + "\n\n"
        query += '''
        Extract the required wedding data from the given text and return it in pure JSON format, without any additional text or snippet tags. Ensure that the output follows this exact JSON structure:

        {
            "thumbnail": "",
            "groom": "",
            "bride": "",
            "datetime": "", // ex: 2025-04-26T14:00:00
            "location": ""
        }
         '''
        client = OpenAI(api_key=OPENAI_API_KEY)
        model = "gpt-4o-mini"
        messages = [{
            "role": "system",
            "content": "Parse the wedding voucher data into JSON format."
        }, {
            "role": "user",
            "content": query
        }]

        # GPT API 호출
        response = client.chat.completions.create(model=model, messages=messages, response_format={'type': 'json_object'})

        try:
            result = response.choices[0].message.content
            return https_fn.Response(json.dumps(result), status=200, mimetype="application/json", headers=headers)
        except Exception as e:
            return https_fn.Response(
            json.dumps({"error": str(e), "note": f"{result}"}), status=404, mimetype="application/json", headers=headers
        )

    except Exception as e:
        return https_fn.Response(
            json.dumps({"error": str(e) + str(req.get_json())}), status=500, mimetype="application/json", headers=headers
        )
```

그 다음 시도로는 `html` 페이지 리소스를 `python`의 `requests`를 통해 가져온 다음, 해당 데이터를 `GPT`에게 전달했습니다.
이때 `GPT`는 놀랍게도 일정 정보를 파싱할 수 있었습니다. 아주 높은 정확도로 말이죠.

여기까지만으로도 `GPT API`는 모바일 청첩장 내용 파싱이라는 임무를 수행할 수 있었지만, 좀 더 개선이 필요했습니다.
왜냐하면 `html` 페이지 리소스에는 불필요한 정보가 너무 많았고, 중복된 텍스트도 많이 있었습니다.
특히 여러 애니메이션, 디자인 리소스가 포함되는 모바일 청첩장의 특성상 `html` 페이지 전체를 `GPT`에게 `input`으로 전달하면 그 속도가 많이 느려진다는 것을 체감했습니다.

따라서 여기서 한번 더 개선을 해냈습니다. `html`에서 불필요한 태그, 스크립트를 미리 `BeautifulSoup`과 같은 전통적인 파서로 제거하고, 텍스트, 이미지 링크 등 주요 정보만 남겨서 `GPT`에게 `input`으로 전달했습니다.
이때 응답속도가 확연히 빨라지는 것을 확인했고, `GPT API`를 활용할 때 성능 개선을 할 수 있었습니다.

```python
def extract_text_content(soup: BeautifulSoup, content_set: Set[str]) -> None:
    """본문 텍스트 콘텐츠를 추출하여 content_set에 추가합니다."""
    for element in soup.find_all(["p", "h1", "h2", "h3", "h4", "h5", "h6", "li", "div", "script", "meta", "title"]):
        text = element.get_text(strip=True)
        if text and len(text) >= 2 and contains_korean(text):
            content_set.add(text)


def extract_media_content(soup: BeautifulSoup, content_set: Set[str]) -> None:
    """이미지 및 링크 콘텐츠를 추출하여 content_set에 추가합니다."""
    for img in soup.find_all("img"):
        if src := img.get("src"):
            content_set.add(f"[IMAGE] {src}")

    for link in soup.find_all("link"):
        if href := link.get("href"):
            content_set.add(f"[LINK] {href}")

    for anchor in soup.find_all("a"):
        href = anchor.get("href", "").strip()
        text = anchor.get_text(strip=True)
        if href:
            content_set.add(f"[ANCHOR] {text} → {href}" if text else f"[ANCHOR] {href}")


def extract_content_with_images(url: str) -> str:
    """주어진 URL에서 텍스트와 미디어 콘텐츠를 추출하여 출력합니다."""
    html = fetch_html(url)
    if not html:
        return

    soup = BeautifulSoup(html, "html.parser")

    for tag in soup(["style", "noscript", "link"]):
        tag.decompose()

    extracted_content: Set[str] = set()

    extract_text_content(soup, extracted_content)
    extract_media_content(soup, extracted_content)

    return "\n".join(sorted(extracted_content))
```

최근에 받은 모바일 청첩장 서비스 업체들이 모두 달라서 마침 잘 되었다 싶어 테스트 했을 때, 모든 종류의 웹 페이지에 대해 `GPT`가 잘 동작함을 확인했습니다.

이렇게 `GPT API`와 연동하여 동작하는 코드를 만들었고, 이를 `Firebase Functions`에 배포하여 아주 간단한 백엔드 서버를 구축했습니다.

### 클린 아키텍처 적용기

본격적인 구현에 앞서 프로젝트 아키텍처를 설계했습니다.
여기서는 **클린 아키텍처**를 적용하였는데, 그 이유는 다음과 같습니다.

> 1. 클린 아키텍처에 대한 이해와 적용 실습
> 2. 최소기능으로 시작하여 백엔드, DB 교체 가능성이 높음

클린 아키텍처에 대한 공부가 좀 더 메인 이유였는데, 이 과정에서 클린 아키텍처의 정답을 찾기보단 철학을 존중하고 따르는 것이 중요하다고 느꼈습니다.

여기서 핵심적으로 생각한 철학들은 다음과 같습니다.

> 1. 가장 안쪽의 레이어는 엔티티, 비즈니스 로직으로 구성되며 이들은 외부 의존성이 없거나 최소가 되어야 한다.
> 2. 이를 위해 추상화로 의존성 역전을 발생시킨다.
> 3. 각 파일/클래스는 한가지 책임을 가지게 하여 결합도를 낮춘다.

위와 같은 철학들을 우선적으로 지키게끔 설계하고 개발했습니다.

최종적으로 선택한 프로젝트 구조는 다음과 같은데,

```css
📂 core/
   ├── di/             (의존성 주입)
   ├── services/       (Push Notifications 서비스)
   ├── utils/          (공통 유틸 함수)
   ├── env.dart        (각종 설정 값)

📂 data/
   ├── mapper/         (데이터 모델 <-> 엔티티 변환기)
   ├── models/         (데이터 모델)
   ├── repositories/   (레포지토리 구현체)
   ├── sources/        (로컬, 원격 데이터 소스)

📂 domain/
   ├── entities/       (순수 도메인 모델)
   ├── repositories/   (레포지토리)
   ├── usecases/       (비즈니스 로직)

📂 presentation/
   ├── controllers/    (뷰모델)
   ├── pages/          (화면 UI)
   ├── widgets/        (재사용 가능한 위젯들)
   ├── themes/         (앱 테마 관리)

main.dart
```

아키텍처를 설계하고 개발에 들어가면서 대부분의 개발자들이 하는 고민을 비슷하게 겪었습니다.

#### 1. usecase는 왜 있는거지?

처음 공부할 땐 `usecase`가 비교적 불필요해보여서 빼고 구현하려 했습니다.
그럼에도 최종적으로는 `usecase`를 구현하게 되었습니다.

`usecase`가 있어야 했던 사례는 다음과 같습니다.

```dart
Future<List<Schedule>> getSchedules() async {
  final List<ScheduleModel> schedules = await localSource.getAllSchedules();
  final List<Schedule> entitySchedules =
      schedules.map((e) => ScheduleMapper.toEntity(e)).toList();
  return entitySchedules;
}
```

이 코드는 `data/repository`에 작성된 코드입니다.
로컬 데이터 소스, 즉 `sqflite`에서 데이터를 가져오는 레포지토리 기능입니다.
이 데이터 접근 기능은 여러 곳에서 쓰입니다.

> 1. Calendar 위젯에서 일정을 달력에 표시하기 위해
> 2. ListView 위젯에서 앞으로의 일정, 지난 일정을 표시하기 위해
> 3. 메인 페이지에서 등록된 일정 수를 표시하기 위해

이때 각 사용처에서는 필요한 데이터의 형태나 범위 등이 다릅니다.

> 1. Calendar 위젯 : 전체 일정 중 월 단위 데이터가 필요함
> 2. ListView 위젯 : 현재를 기준으로 전체 일정을 둘로 나눈 리스트가 필요함
> 3. 메인 페이지 : 전체 일정의 수가 필요함

클린 아키텍처의 관점에서 보면,
데이터를 가져오는 기능 자체는 **data 영역**에서 다뤄지는게 적절하지만, 이렇게 사용처마다 필요한 데이터를 가공하는 코드는 data 영역보단 **domain 영역**이 더 어울립니다.
왜냐하면 사용처에서 원하는 데이터의 양식은 사용처가 제일 잘 알기 때문입니다.

또한 위젯과 같은 사용처는 추가/수정이 잦게 일어납니다.
추가나 수정이 필요할 때마다 `repository`를 편집하는 것보다, 기존의 `repository`에서 제공하는 데이터를 활용해 새로운 `usecase`를 만드는게 더 비용이 적게 들겠다고 판단했습니다.

따라서 `usecase`는 사용하는 것으로 결정했고, 이를 적용한 예시는 다음과 같습니다.

```dart
/// 전체 일정을 가져와 제공하는 유즈케이스
class ListSchedulesUsecase {
  final ScheduleRepository repository;
  ListSchedulesUsecase(this.repository);

  Future<List<Schedule>> execute() {
    return repository.getSchedules();
  }
}

/// 전체 일정의 수를 제공하는 유즈케이스
class CountSchedulesUsecase {
  final ScheduleRepository repository;
  CountSchedulesUsecase(this.repository);

  Future<int> execute() async {
    List<Schedule> schedules = await repository.getSchedules();
    return schedules.length;
  }
}
```

#### 2. data/model과 domain/entity 둘다 필요한가?

**data/model**과 **domain/entity**는 그 코드가 유사하지만, 개념적으로 지향하는 바가 완전히 다릅니다.

`Entity`는 사용자에게 제공되는 데이터, 모델은 DB나 서버에서 처리되는 데이터를 구현한 것입니다.

개념적으로 다르지만 사실 하나만 써도 구현하는데 큰 문제는 없습니다.
그럼에도 패턴에서 자주 사용되는 이유가 있는데, 이 프로젝트에서도 그런 사례가 있었습니다.

```dart
@freezed
class ScheduleModel with _$ScheduleModel {
  factory ScheduleModel({
    required String link,
    required String thumbnail,
    required String groom,
    required String bride,
    required String date,
    required String location,
  }) = _ScheduleModel;

  factory ScheduleModel.fromJson(Map<String, dynamic> json) =>
      _$ScheduleModelFromJson(json);
}
```

이 코드는 프로젝트 초기에 사용된 `Schedule` **모델**입니다.
이땐 모델과 엔티티의 구분 필요를 느끼지 못하여 모델 하나만으로 데이터 저장, 서버 통신, UI 표현 등 모든 곳에 적용하여 왔습니다.

여기서 특이한 점은 `date`를 `String` 타입으로 선언한 것인데, 이는 로컬 DB인 `sqflite`에 `DateTime` 타입이 없기 때문입니다.

따라서 해당 모델로 DB에 저장하기 위해서는 반드시 `String` 타입으로 선언해야 했고, 나중에 `DateTime`을 편집해야 하는 UI 기능을 구현할 때에는,

```dart
ScheduleModel(date: today.toIso8601String());
...
DateTime date = DateTime.parse(schedule.date);
```

이렇게 **String <-> DateTime** 변환을 그때그때 사용해왔습니다.

너무 불편한 방법이죠?
이렇게 데이터를 변환하고 편집하는 기능이 여러곳에 흩어지면 데이터의 일관성을 지키기 어려웠습니다.

이때 엔티티의 필요성을 느끼고, 오롯이 비즈니스 로직과 UI를 위한 엔티티를 별도로 만들었습니다.

```dart
// domain/schedule.dart
@freezed
class Schedule with _$Schedule {
  const factory Schedule({
    required String link,
    required String thumbnail,
    required String groom,
    required String bride,
    required DateTime date,
    required String location,
  }) = _Schedule;
}
```

그리고 엔티티 <-> 모델 변환을 위한 **Mapper**를 만들었습니다.

```dart
// data/mapper/schedule_mapper.dart
import '../models/schedule/schedule_model.dart';
import '../../domain/entities/schedule.dart';

/// ScheduleMapper class converts Schedule(entity, domain) <-> ScheduleModel(model, data)
class ScheduleMapper {
  /// Converts Schedule(entity, domain) -> ScheduleModel(model, data)
  static ScheduleModel toModel(Schedule entity) {
    return ScheduleModel(
      link: entity.link,
      thumbnail: entity.thumbnail,
      groom: entity.groom,
      bride: entity.bride,
      date: entity.date.toIso8601String(),
      location: entity.location,
    );
  }

  /// Converts ScheduleModel(model, data) -> Schedule(entity, domain)
  static Schedule toEntity(ScheduleModel model) {
    return Schedule(
      link: model.link,
      thumbnail: model.thumbnail,
      groom: model.groom,
      bride: model.bride,
      date: DateTime.parse(model.date),
      location: model.location,
    );
  }
}
```

이처럼 모델과 엔티티를 별도로 구현하여 얻은 이점은 다음과 같습니다.

> 1. 순수한 엔티티 모델을 통해 사용자에게 일관된 데이터를 제공
> 2. UI와 DB 등 쓰이는 여러곳에서 데이터를 변환하지 않고 Mapper를 통해 데이터 변환 일관성 유지
> 3. 추후 DB, 서버 등 데이터 소스를 교체할 때 엔티티에 의존하는 UI, 비즈니스 로직은 유지할 수 있음(모델과 레포지토리만 수정하면 됨!)

### Trouble Shooting #1 : 일정 수정 후 달력으로 돌아왔을 때 상태 변화 적용 시키기

모바일 청첩장을 파싱하여 등록한 일정을 달력에 보여주게끔 구현했습니다.
플러터에서는 `table_calendar`라는 위젯을 주로 사용합니다.

![](https://velog.velcdn.com/images/taebbong/post/5aa63501-1ac2-4b62-9703-0f46ab7e9486/image.png)

현재 앱에서는 `CalendarView`, `ListView`에서 일정을 선택하면 `DetailPage`로 이동하고, `DetailPage`에서는 수정 기능이 제공됩니다.
여기서 일정을 수정하고 나서 돌아가면 `CalendarView`, `ListView`로 돌아오게 되는데요,
기본적으로 플러터는 뒤로 가기 액션에서는 화면이 새로 렌더링 되지 않습니다.
따라서 일정을 수정해도 그 상태에서는 캘린더에 반영이 안된 것처럼 보이게 됩니다.

이 문제를 해결하기 위해 처음 선택한 방법은 캘린더로 돌아올 때 캘린더 페이지를 새로고침하는 것이었습니다.
이 방법을 처음 떠올린 이유는 가장 구현하기 쉬운 방법이기 때문이었습니다.
라우터를 활용해 기존 라우팅 기록을 전부 지우고, 메인에서 캘린더 페이지로 라우팅 시키는 방법으로 구현할 수 있습니다.

```dart
Get.offNamedUntil('/calendar', (route) => route.settings.name == '/');
```

기존에는

<img src="https://velog.velcdn.com/images/taebbong/post/902c817e-2e62-47d4-9d50-81da6fc976d9/image.png" width="70%">
였다면,

<img src="https://velog.velcdn.com/images/taebbong/post/45fac4ac-5422-4eff-aa58-2bda3e53916b/image.png" width="70%">

이렇게 말이죠.

이는 뒤로가기 처럼 보이지만 실제로는 새로 페이지에 접근하는 것이기 때문에 새로고침이 되고, 수정사항이 반영되어 보입니다.

기능적으로 문제는 없지만, `DetailPage`를 보고 돌아올 때마다 새로고침되는 캘린더 페이지는 사용자에게 좋지 못한 경험입니다.

다음으로 찾은 해결방법은 `PopScope` 위젯과 `Routing Argument`을 사용하는 것이었습니다.
`PopScope` 위젯을 사용하면, 해당 위젯이 pop될 때의 콜백을 정의해줄 수 있습니다.
이를 통해 `DetailPage`를 `PopScope`로 감싸고, 여기서 pop할 때 수정된 `Schedule` 객체를 `Routing Argument`로 전달하여 최종적으로 캘린더 페이지의 `Controller`를 통해 상태 업데이트를 하는 구조를 작성했습니다.

먼저, `DetailPage`에서 `PopScope`로 위젯이 pop될 때의 이벤트를 정의합니다.

```dart
// DetailPage
return PopScope(
  canPop: false,
  onPopInvokedWithResult: (didPop, result) {
    if (didPop) return;
    Get.back(result: controller.schedule.value); // 돌아갈 때 변한 상태(Schedule)를 전달
  },
  child: Scaffold(),
);
```

`DetailPage`로 이동하는 버튼에서는 `Get.back()`으로부터 값을 수신할 수 있도록 처리하며,
`CalendarController`를 이용해 상태를 업데이트 합니다.

```dart
// ScheduleListTile from CalendarView
return GestureDetector(
  onTap: () async {
    final updated = await Get.toNamed('/detail', arguments: schedule); // Get.back()으로부터 값을 전달받고,
    final calendarController = Get.find<CalendarController>();
    calendarController.onUpdateSchedule(updatedSchedule: updated); // CalendarController에서 상태 업데이트 진행
  },
  ...
);
```

그리하여 `CalendarController`에서 수정된 `Schedule` 상태를 반영하며,

```dart
// CalendarController
Future<void> onUpdateSchedule({required Schedule updatedSchedule}) async {
  // Step 1. Update focusedDay, selectedDay for CalendarView.
  focusedDay.value = updatedSchedule.date;
  selectedDay.value = updatedSchedule.date;

  // Step 2. Update `allSchedules` by key `link`.
  final currentList = allSchedules.value ?? [];
  final updatedList = currentList.map((s) {
    if (s.link == updatedSchedule.link) {
      return updatedSchedule;
    }
    return s;
  }).toList();
  allSchedules.value = updatedList;
}
```

`Obx`로 상태 변화를 감지하는 위젯이 상태의 변화만을 화면에 적용, 리렌더링합니다.

```dart
// CalendarView
class CalendarView extends StatelessWidget {
  final CalendarController _controller = Get.find<CalendarController>();

  @override
  Widget build(BuildContext context) {
    return Obx(() {
      final normalizedSelectedDay = DateTime(
          _controller.selectedDay.value.year,
          _controller.selectedDay.value.month,
          _controller.selectedDay.value.day);
      final eventCounts = _controller.schedulesWithDate.value ?? {};
      return Column(
        children: [
          TableCalendar(
            key: ValueKey(eventCounts.hashCode),
            locale: 'ko_KR',
            firstDay: DateTime(2000, 1, 1),
            lastDay: DateTime(2100, 12, 31),
            focusedDay: _controller.focusedDay.value,
            ...
          )
        ]
      );
    });
  }
}
```

이를 통해 수정 후 돌아오더라도 캘린더 위젯이 새로고침되지 않는 자연스러운 UX를 구현할 수 있었습니다.

> 최근 **Flutter Seoul 4월 오픈 스테이지**에서 **실시간 상태 동기화 전략**에 대해 고민해볼 수 있었습니다. 연사님은 `Bloc`, `Stream`을 `usecase`가 구독하게끔 구현했다고 했는데, 역시 실시간 상태 동기화에는 `Stream`이 적절한 듯 하네요. 다음 리팩토링에서는 `Stream`을 적용해보겠습니다.

### Trouble Shooting #2 : 로컬 푸시 알림 예약 기능 구현

푸시 알림은 보내는 주체가 누구냐에 따라 로컬과 서버로 방식이 나뉘는데, 기본적으로 이 앱에서 스케쥴 데이터는 서버가 아닌 로컬 DB에 저장되기 때문에 당연히 로컬 푸시 알림을 먼저 고려하게 되었습니다.

![청모 앱에서 푸시 알림 기능 UI]()

제가 기획한 푸시 알림은 모바일 청첩장의 결혼식 일정 전날 11시쯤 **"다음날 OOO님의 결혼식이 있어요!"**와 같은 푸시 알림을 제공하는 것이었습니다.

당연히 이 푸시 알림은 앱이 종료되어도 발생하여야 합니다.

최초에 생각했던 방식은 매일 11시에 DB를 조회하여 다음날로 저장된 일정을 찾으면 푸시 알림을 제공하는 방식으로 생각했었어요.
이 방식을 구현하기 위해서는 **백그라운드로 돌아가는 "서비스"**를 구현해야 하고, 이 서비스가 정해진 시각에 **작업을 하도록 예약**해야 했습니다.

백그라운드 서비스를 구현하기 위해 알아본 플러터 레벨의 패키지는 `WorkManager`였습니다.
`WorkManager`는 플러터에서 네이티브 백그라운드 서비스를 구현하기 위해 최선의 선택지였습니다만, 패키지를 적용하면 빌드가 안되는 이슈가 있었습니다. 저만 안되는 줄 알았는데, 다들 겪고 있는 문제더라구요..

<img src="https://velog.velcdn.com/images/taebbong/post/90f7bb27-5159-45e2-a710-ccf105ccdc2d/image.png" width="70%">

이후에 직접 패키지를 활용해 구현했을 때에도 제대로 동작하지 않았습니다.
~~이때 푸시 알림을 때려칠까 고민 많이 했습니다만..~~
다른 방식의 푸시 알림을 고민하게 되었습니다.

이후에 선택한 방식은 **새로운 스케쥴이 등록될 때마다** 푸시 알림을 보낼 날짜를 계산하여 푸시 알림을 **예약**하는 것이었습니다.
이는 스케쥴 등록 뿐만 아니라 스케쥴 수정 때에도 기존 푸시 알림 예약을 취소하고 새로 날짜를 계산하여 등록하는 방식으로 동일하게 동작시킬 수 있었습니다.

여기서 푸시 알림을 예약하는 기능은 `flutter_local_notifications` 패키지로 구현했습니다.

<img src="https://velog.velcdn.com/images/taebbong/post/9dce35b1-d0c1-4bf5-ba36-d9b096cbce2b/image.png" width="70%">

해당 패키지는 각 네이티브 기기의 알림 스케쥴러를 활용할 수 있고, 개발하는 입장에서 백그라운드 서비스 구현 없이 네이티브한 방식으로 푸시 알림 예약 기능을 구현할 수 있었습니다.

이렇게 하면 백그라운드 서비스를 구현할 필요가 없고, 심지어 앱이 백그라운드에서 매일 같은 시각에*(푸시 알림을 보낼만한 스케쥴이 있을지 없을지도 모르는데)* 일하는 것을 방지할 수 있습니다.

푸시 알림 예약 기능은 `zonedSchedule()`을 활용해 구현할 수 있었습니다.

일정을 등록할 때에는 아래와 같은 로직이 실행됩니다.

> 1. 등록하려는 결혼식 일정의 (날짜 - 1일)의 오전 9시에 푸시 알림을 예약
> 2. 만약 결혼식 일정이 내일이라면 당일 오전 9시에 푸시 알림을 예약
> 3. 결혼식 일정이 오늘을 포함하여 과거라면 푸시 알림을 예약하지 않음

이를 확인하여 일정을 등록하는 코드를 다음과 같이 구현했습니다.

```dart
// NotificationService
Future<void> checkPreviousDayForNotify({
  required Schedule schedule,
}) async {
  final int id = await UrlHash.hashUrlToInt(schedule.link);
  String title = "내일 ${schedule.groom} & ${schedule.bride}님의 결혼식이 있습니다!";
  tz.TZDateTime scheduleDate =
      _timeZoneSetting(scheduleDate: schedule.date, hour: 9, minute: 0);

  final now = tz.TZDateTime.now(scheduleDate.location);
  final todayEleven = tz.TZDateTime(
      scheduleDate.location, now.year, now.month, now.day, 11, 0);

  if (scheduleDate.isAtSameMomentAs(todayEleven) &&
      now.isAfter(todayEleven)) {
    final DateTime tomorrow = DateTime.now().add(const Duration(days: 1));
    scheduleDate = tz.TZDateTime(scheduleDate.location, tomorrow.year,
        tomorrow.month, tomorrow.day, 9, 0);
    title = "오늘 ${schedule.groom} & ${schedule.bride}님의 결혼식이 있습니다!";
  } else if (scheduleDate.isBefore(todayEleven)) {
    return;
  }

  await addNotifySchedule(
      id: id, appName: '청모', title: title, scheduleDate: scheduleDate);
}
```

여기서 `addNotifySchedule()`은 `zonedSchedule()`을 활용해 푸시 알림을 예약하는 함수입니다.

```dart
Future<void> addNotifySchedule(
    {required int id,
    required String appName,
    required String title,
    required tz.TZDateTime scheduleDate}) async {

  await _localNotifyPlugin.zonedSchedule(
    id,
    "청모",
    title,
    scheduleDate,
    details,
    uiLocalNotificationDateInterpretation:
        UILocalNotificationDateInterpretation.absoluteTime,
    androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
    payload: scheduleDate.toIso8601String(),
  );
}
```

일정을 수정하거나 삭제하면 어떻게 해야할까요?

> 1. 기존 예약된 푸시 알림을 삭제
> 2. 결혼식 날짜가 수정되었다면 새롭게 푸시 알림을 예약

이 과정에서 기존 예약된 푸시 알림을 삭제하는 기능이 필요했고,
우리는 `Schedule` 모델에서 `link`를 키로 사용하고 있었기 때문에 이를 기반으로 푸시 알림 등록할 때 `id`를 설정, 찾아서 삭제할 수 있었습니다.

```dart
Future<void> cancelNotifySchedule({required String link}) async {
  final int id = await UrlHash.hashUrlToInt(link);
  await _localNotifyPlugin.cancel(id);
}
```

또한 앱이 꺼져있는 상태에서 푸시 알림을 눌러 앱을 키면 해당 일정이 나오는 페이지로 이동해야 합니다.
이 기능은 `onDidReceiveNotificationResponse`를 설정하여 구현했습니다.

```dart
Future<void> onDidReceiveNotificationResponse({required String link}) async {
  final GetScheduleByLinkUsecase getScheduleByLinkUsecase =
      getIt<GetScheduleByLinkUsecase>();
  final Schedule? targetSchedule =
      await getScheduleByLinkUsecase.execute(link);
  if (targetSchedule != null) {
    Get.toNamed('/');
    Get.toNamed('/detail', arguments: targetSchedule);
  } else {
    Get.toNamed('/');
  }
}
```

푸시 알림은 구현 자체는 간단하지만 이처럼 고려할 시나리오들이 많았습니다.
또한 네이티브 기능을 사용하는 만큼 각 OS별 권한 설정을 잘 알고 있어야 했습니다.
이번에 **"청모"** 프로젝트에서 로컬 푸시 알림 기능을 구현하여 다음에도 비슷한 방식의 구현을 할 수 있겠다는 자신감이 생겼습니다.

### Trouble Shooting #3 : 테스트를 위한 Mock-Up 레포지토리 만들기

플러터에서 테스트를 할 수 있는 방법은 많이 있지만, 의존성 주입을 함께 고려하여 테스트할 때에는 `Mockito` 만한게 없었습니다.
`Mockito`를 `injectable`과 함께 사용하면, `data sources`나 `repository` 등을 `Mock` 객체로 만들어서 의존성 주입까지 구현할 수 있습니다.
그러면 위젯/유닛 단위의 테스트를 할 때 해당 `Mock` 객체를 주입하여 사용할 수 있는 것입니다.

"청모" 프로젝트는 규모가 크지 않았기 때문에 UI 부분의 테스트는 크게 필요하지 않다고 느꼈고,
그래서 데이터 처리 및 푸시 알림 관련된 기능만 테스트하면 되겠다고 생각했습니다.
따라서 테스트 대상을 `data/repository`로 설정했고, 해당 `repository`를 테스트 하기 위해 필요한 의존성들을 `mockito`를 통해 구성했습니다.

먼저 `RemoteSource`를 구현할 때 `@lazySingleton` 어노테이션을 설정하여 `injectable`이 `build_runner`를 통해 객체를 생성하게끔 하였습니다. `LocalSource`, `NotificationService`도 마찬가지로 구현했습니다.

```dart
@LazySingleton(as: ScheduleRemoteSource)
class ScheduleRemoteSourceImpl implements ScheduleRemoteSource {
  final Dio dio = Dio();

  ScheduleRemoteSourceImpl();

  /// Fetch analyzed data in `json` type from Firebase functions API.
  ///
  /// If result, returns `ScheduleModel` type data.
  ///
  /// If not, throw error.
  @override
  Future<ScheduleModel> fetchScheduleFromServer(String link) async {
    try {
      final response = await dio.post('${Env.url}/', data: {'link': link});
      if (response.statusCode == 200) {
        return ScheduleModel.fromJson(
            jsonDecode(response.data)..addAll({'link': link}));
      } else {
        throw Exception('[-] Failed to fetch data from server');
      }
    } catch (e) {
      throw Exception('[-] Failed to fetch data from server');
    }
  }
}
```

이후 테스트 코드를 구현할 때 아래와 같이 `mocks.dart` 파일을 만들어서 **Mock 객체**를 생성하도록 설정했습니다.

```dart
@GenerateMocks([
  ScheduleLocalSource,
  ScheduleRemoteSource,
  NotificationService,
])
void main() {}
```

`build_runner`를 돌리면, 다음과 같이 `mocks.mocks.dart` 파일이 생성됩니다.

```dart
import 'dart:async' as _i5;

import 'package:chungmo/core/services/notification_service.dart' as _i7;
import 'package:chungmo/data/models/schedule/schedule_model.dart' as _i2;
import 'package:chungmo/data/sources/local/schedule_local_source.dart' as _i4;
import 'package:chungmo/data/sources/remote/schedule_remote_source.dart' as _i6;
import 'package:chungmo/domain/entities/schedule.dart' as _i8;
import 'package:flutter_local_notifications/flutter_local_notifications.dart'
    as _i3;
import 'package:mockito/mockito.dart' as _i1;
import 'package:timezone/timezone.dart' as _i9;

/// A class which mocks [ScheduleRemoteSource].
///
/// See the documentation for Mockito's code generation for more information.
class MockScheduleRemoteSource extends _i1.Mock
    implements _i6.ScheduleRemoteSource {
  MockScheduleRemoteSource() {
    _i1.throwOnMissingStub(this);
  }

  @override
  _i5.Future<_i2.ScheduleModel> fetchScheduleFromServer(String? url) =>
      (super.noSuchMethod(
            Invocation.method(#fetchScheduleFromServer, [url]),
            returnValue: _i5.Future<_i2.ScheduleModel>.value(
              _FakeScheduleModel_0(
                this,
                Invocation.method(#fetchScheduleFromServer, [url]),
              ),
            ),
          )
          as _i5.Future<_i2.ScheduleModel>);
}
```

이제 **Mock 객체**를 활용해 테스트 코드를 작성할 수 있습니다.

```dart
import '../../mocks/mocks.mocks.dart'; // build_runner로 생성된 파일

void main() {
  late ScheduleRepositoryImpl repository;
  late MockScheduleRemoteSource mockRemoteSource;
  late MockScheduleLocalSource mockLocalSource;
  late MockNotificationService mockNotificationService;

  setUp(() {
    mockRemoteSource = MockScheduleRemoteSource();
    mockLocalSource = MockScheduleLocalSource();
    mockNotificationService = MockNotificationService();
    repository = ScheduleRepositoryImpl(
        mockRemoteSource, mockLocalSource, mockNotificationService);
  });

  group('analyzeLink', () {
    test('should return Schedule when remote source call is successful',
        () async {
      // Given
      when(mockRemoteSource.fetchScheduleFromServer(any))
          .thenAnswer((_) async => tScheduleModel);

      // When
      final result = await repository.analyzeLink(tUrl);

      // Then
      expect(result, tSchedule);
      verify(mockRemoteSource.fetchScheduleFromServer(tUrl)).called(1);
    });
  });
}
```

이를 통해 의존성이 복잡하게 구현된 프로젝트에서도 간편하게 **Mock 객체**를 생성하여 안전한 테스트를 진행할 수 있었습니다.

### 그 외 사용한 기법과 근거

#### 1. 상태 관리도구를 왜 GetX로 했나?

![원피스 해군 3대장에 BloC, Provider, GetX 로고를 합성한 그림](https://velog.velcdn.com/images/taebbong/post/8a52f1de-8347-4781-9ff0-29ac4b57c929/image.jpg)

플러터에는 상태 관리 도구 3대장이 있습니다. `BloC`, `Provider`, `GetX`가 그것입니다.
~~Riverpod도 심심찮게 보이지만, 아직 제게 익숙치는 않습니다.~~
~~커뮤니티 활동을 안해서 몰랐는데, GetX는 좀 많이 배척되고 Riverpod이 들어가는 느낌이네요.~~

각 상태 관리 도구는 개성이 강하고 장단점이 명확한데,

먼저 `BloC`은 현재, 그리고 앞으로도 `Stream`을 사용할 계획이 없고, 프로젝트 규모를 고려했을 때 하나의 상태 관리를 위해 너무 많은 파일이 생긴다고 판단해 제외하였습니다.

`Provider`와 `GetX`는 둘다 상태 관리를 위해 사용하기에는 간단하고 좋은 방법들입니다.
둘 중에서는 이후 라우팅이나 다이얼로그, 스낵바 등 위젯 레벨에서 도움을 받을 수 있는 `GetX`를 채택하였습니다.
이는 어떻게 보면 상태 관리 도구로써 선택했다기보단 하나의 프레임워크로 선택했다고 보는게 더 적합하겠네요.
어디까지나 빠른 기능 개발을 목표로 했기에, 라우팅과 UI 영역에서의 `GetX`의 장점을 놓치기엔 아쉬웠습니다.

#### 2. 왜 GetX와 StatefulWidget을 함께 쓰는가?

`GetX`는 그 자체로 온전한 상태 관리도구로, 전역 및 지역 상태 관리를 충분히 수행할 수 있습니다.
보통 `GetX`에서 상태 관리를 할 때에는 **MVVM 패턴**을 적용하여 `view`에 해당하는 `viewmodel(controller)`를 선언하여 사용합니다.
이때 다뤄지는 상태가 많거나 `viewmodel`에서 수행해야 하는 작업이 많은 경우 `viewmodel`이 비대해질 수 있습니다.

비대한 `viewmodel`을 관리하는 방법은 여러가지가 있겠지만,
청모 프로젝트에서는 `viewmodel`은 비즈니스 로직 관리에 집중하고 UI와 관련된 단순 상태는 `StatefulWidget`으로 관리하도록 하였습니다.

```dart
class _DetailPageState extends State<DetailPage> {
  bool editMode = false; // 수정 상태인지 확인

  final DetailController controller = Get.put(DetailController()); // 비즈니스 로직을 담당하는 ViewModel
  final TextEditingController groomController = TextEditingController(); // 텍스트 입력 컨트롤러
  final TextEditingController brideController = TextEditingController();
  final TextEditingController locationController = TextEditingController();
  final TextEditingController linkController = TextEditingController();
  DateTime? selectedDate; // 현재 페이지에서 선택된 날짜
  ...
}
```

이와 같이 관심사를 분리하여 좀 더 명료한 코드 구조를 만들 수 있었습니다.

#### 3. 의존성 관리는 어떻게?

의존성 주입 및 관리에는 `injectable`과 `get_it`을 사용하고 있습니다,
`GetX`에는 `Get.put`, `Get.find`라는 아주 편리한 의존성 관리 기능이 있습니다.
그럼에도 `injectable`과 `get_it`을 사용한 것은 `GetX` 의존도를 낮추고, 다른 상태 관리도구와도 사용할 수 있도록 범용성을 확보하기 위함이었습니다.

사실 프로젝트에서 `injectable`과 `get_it`을 사용한 것은 처음이었는데, 생각보다 사용하기 편리했습니다.
무엇보다 `build_runner` 기반의 자동 생성 기능이 편리했습니다.

```dart
@LazySingleton(as: ScheduleRepository)
class ScheduleRepositoryImpl implements ScheduleRepository {
  ...
}

@injectable
class AnalyzeLinkUsecase {
  ...
}
```

위와 같이 의존성 관리가 필요한 대상 클래스에 `LazySingleton`, `injectable`과 같은 어노테이션을 넣어 선언합니다.

> `LazySingleton`은 해당 객체가 처음 사용될 때 싱글톤 방식으로 객체를 생성하는 방식입니다.
> `injectable`은 해당 객체를 사용할 때마다 새로 생성하는 방식입니다.

```dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';

import 'di.config.dart';

final GetIt getIt = GetIt.instance;

@InjectableInit()
Future<void> configureDependencies() async => getIt.init();
```

`di.dart` 파일을 위처럼 선언하고 `build_runner`를 실행하면 아래와 같이 `di.config.dart` 파일이 자동으로 생성됩니다.

```dart
import 'package:get_it/get_it.dart' as _i174;
import 'package:injectable/injectable.dart' as _i526;

extension GetItInjectableX on _i174.GetIt {
// initializes the registration of main-scope dependencies inside of GetIt
  _i174.GetIt init({
    String? environment,
    _i526.EnvironmentFilter? environmentFilter,
  }) {
    final gh = _i526.GetItHelper(
      this,
      environment,
      environmentFilter,
    );
    gh.lazySingleton<_i109.NotificationService>(
        () => _i109.NotificationServiceImpl());
    gh.lazySingleton<_i1014.ScheduleLocalSource>(
        () => _i1014.ScheduleLocalSourceImpl());
    gh.lazySingleton<_i153.ScheduleRemoteSource>(
        () => _i153.ScheduleRemoteSourceImpl());
    gh.lazySingleton<_i561.ScheduleRepository>(
        () => _i798.ScheduleRepositoryImpl(
              gh<_i153.ScheduleRemoteSource>(),
              gh<_i1014.ScheduleLocalSource>(),
              gh<_i109.NotificationService>(),
            ));
    gh.factory<_i407.ListSchedulesUsecase>(
        () => _i407.ListSchedulesUsecase(gh<_i561.ScheduleRepository>()));
        ...
  }
  ...
}
```

이후 필요한 의존성을 주입시켜 사용하고 싶을 땐 `getIt` 객체를 통해 접근하면 됩니다.

```dart
final ListSchedulesByDateUsecase listSchedulesByDateUsecase = getIt<ListSchedulesByDateUsecase>();
```

### 배포 완료 : 다음 버전은?

![데모 영상]()

아주 기본적인 기능만 구현한 현재 버전 1.0.0은 플레이스토어에 무사히 배포되었습니다.
배포할 때 쯤 맥북을 교체하느라 앱스토어 배포는 놓쳤는데, 다음 버전에 함께 배포하려고 합니다.

다음 버전에 넣고 싶은 기능은 다음과 같은데,

> 1. `showcaseview` 패키지 기반 온보딩 프로세스 추가
> 2. 메인 페이지 UI에 일정 데이터 추가로 보여주기
> 3. 계좌 정보 추가
> 4. 주소 복사 버튼 추가

비교적 간단한 기능들 위주로 다음 버전을 준비하려고 합니다.

### 개선할 점, 마치며

한번에 완벽할 수 없다는 것은 알지만, 그럼에도 볼 때마다 아쉬운 점이 생기는 것 같습니다.
최근 커뮤니티나 세미나를 통해 새롭게 알아가는 내용이 있는데, 이런 부분들을 적용하면 좀 더 좋은 프로젝트가 되겠다 느껴집니다.

> 1. `Isolate` 적용하여 성능 개선
> 2. `Stream` 적용하여 실시간 상태 관리, 구독
> 3. 적극적인 객체지향 개념 적용, 리팩토링

<img src="https://velog.velcdn.com/images/taebbong/post/98ce493e-05a3-4cc4-8638-01c00c451c95/image.png" width="70%">

여기까지 제 작고 소중한 **"청모"** 프로젝트였습니다. 지금은 많이 부족하지만 좀 더 시간이 지난 후, 더 멋진 프로젝트가 되어 새롭게 얻은 경험들을 공유하겠습니다. 긴 글 읽어주셔서 감사합니다!
