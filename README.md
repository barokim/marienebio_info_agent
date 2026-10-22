# Sea-Info Agent · FastAPI 대시보드 뼈대

Day6 수업 자료의 Pydantic·FastAPI 개념을 해양 정보 프로젝트에 연결한 로컬 학습 예제입니다. 기존 `marine_identifier` 코드는 수정하지 않았습니다.

## 시작하기

Python 3.12 이상이 설치된 PowerShell에서 실행합니다.

```powershell
cd C:\KBR\sea_info_dashboard
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m uvicorn main:app --reload --host 127.0.0.1 --port 8001
```

- 대시보드: http://127.0.0.1:8001
- Swagger API 실습: http://127.0.0.1:8001/docs
- 처음에는 `예시로 연습하기` 모드가 열립니다. 외부 API나 키 없이 화면과 CRUD를 연습할 수 있습니다.
- `실제 API 조회하기`를 선택하고 조회하면 파고·해수면 온도만 Open-Meteo Marine API에서 가져옵니다. 한국 시간 기준 오늘~3일 뒤까지 지원합니다.
- 물때는 모든 모드에서 **직접 작성한 고정 예시**입니다. 실제 만조·간조가 아닙니다.

## 화면 뼈대

```text
왼쪽 메뉴: 대시보드 / 물때·예보 / 생물 도감 / 관찰 기록 / 참고 링크

상단     지역 선택 + 날짜 선택 + 예시/실제 API 선택 + 조회
요약     첫 만조 | 첫 간조 | 하루 최대 파고 | 평균 해수면 온도
중단     시간별 파고 차트 | 하루 만조·간조 목록
하단     사진 미리보기 | 생물 도감 검색
기록     관찰 기록 생성 / 조회 / 수정 / 즐겨찾기 / 삭제
실습     실제로 반환받은 JSON 응답 살펴보기
참고     참고한 서비스·공식 API 링크
```

PC에서는 왼쪽 메뉴와 여러 열로, 모바일에서는 한두 열로 배치됩니다. 메뉴는 같은 페이지의 해당 영역으로 이동합니다. 지도·회원가입·배포·복잡한 AI 프레임워크는 포함하지 않았습니다.

## 파일별 읽는 순서

```text
sea_info_dashboard/
├─ models.py       # 1. 데이터의 모양과 검증 규칙
├─ services.py     # 2. 예시 데이터 및 외부 API 호출·변환
├─ main.py         # 3. 주소와 함수 연결, 메모리 CRUD
├─ static/
│  ├─ index.html   # 4. 화면 요소 배치
│  ├─ style.css    # 5. 색상·크기·모바일 배치
│  └─ app.js       # 6. 버튼 → fetch 요청 → 결과 표시
├─ test_main.py    # 입력 검증과 API 동작 테스트
└─ requirements.txt
```

먼저 Python 파일 3개에 집중하세요. HTML/CSS/JavaScript는 준비된 화면으로 사용하고 Python 응답을 바꿔서 화면 변화를 확인하면 됩니다. FastAPI는 데이터를 검증·처리하는 서버이며, HTML 화면은 별도로 필요합니다. 현재는 FastAPI가 HTML과 API를 같은 주소에서 제공합니다. 별도 CORS 설정이나 React 개발 서버는 필요하지 않습니다.

## 수업 개념과 프로젝트의 연결

| 수업 개념 | 적용 부분 | 연습할 내용 |
|---|---|---|
| `BaseModel` | `RecordCreate`, `MarinePoint` | 관찰 기록과 예보 데이터 형식 정의 |
| `Field` | 제목 2~80자, 메모 500자 제한 | 공백 제목·긴 메모 보내기 |
| `Literal` | 지역 ID, demo/live | 알 수 없는 지역 요청 시 422 확인 |
| `field_validator` | 제목 정규화 | 앞뒤 공백을 제거한 뒤 길이 검사 |
| `model_validator` | 외부 예보의 배열 길이·단위·시간 기준 | 정상 JSON이라도 의미가 맞는지 검사 |
| 중첩 모델 | `DashboardResponse` 안의 지역·물때·해양 예보 | 여러 정보를 하나의 응답으로 묶기 |
| 외부 API 호출 | `fetch_live_marine()` | URL, params, timeout, HTTP 오류 처리 |
| 응답 모델 | 각 라우트의 `response_model` | `/docs`에서 응답 구조 확인 |
| 직렬화 | `payload.model_dump()` | 모델을 딕셔너리로 변환 |
| 부분 수정 | `model_dump(exclude_unset=True)` | 보낸 필드만 변경하고 생략 필드는 유지 |
| 재검증 | `RecordOut.model_validate(merged)` | 명시적 null이 필수 필드에 저장되지 않도록 검사 |
| 메모리 CRUD | `_records` 딕셔너리 | POST·GET·PATCH·DELETE 실습 |

PDF 자체의 중심 내용은 검증·모델·CRUD입니다. 이 예제는 여기에 사용자가 설명한 외부 API 호출 수업을 연결합니다. 메모리에 저장한 기록은 재시작하면 사라지며, 여러 서버 프로세스가 공유하지 않습니다. 로컬 단일 사용자 실습을 위한 구조입니다.

## 요청이 화면까지 오는 과정

```text
브라우저에서 '실제 API 조회하기' 선택
→ app.js가 GET /api/dashboard?region=incheon&mode=live 요청
→ main.py가 지역·날짜·모드 검증
→ services.py가 Open-Meteo를 httpx로 호출
→ ExternalMarine 모델로 응답 검증
→ DashboardResponse 모델로 형태 통일
→ JSON 응답
→ app.js가 카드·차트를 갱신
```

외부 API에서 문자열이나 JSON이 왔다는 사실만으로 올바른 데이터라고 판단하지 않습니다. 시간 배열과 값 배열의 길이, 한국 시간 오프셋, m/°C 단위, 조회 날짜를 확인합니다. 결측값은 `None`/`null`로 유지합니다. 외부 API 실패 시 예시값을 실제값처럼 반환하지 않습니다.

현재 차트는 시간별 파고입니다. 물때는 고·저조 사건 목록만 준비했으므로 실제 조위 곡선으로 그리지 않습니다. 요약 카드는 당일 첫 만조·간조이며 '현재 시각 이후 다음 만조'와 다릅니다. 파고와 해수면 온도는 모델 예보이며 실시간 현장 관측값이 아닙니다. 대표 좌표는 특정 갯벌이나 관측소 위치가 아닙니다.

## API 목록

| 메서드 | 주소 | 역할 |
|---|---|---|
| GET | `/` | HTML 대시보드 |
| GET | `/api/regions` | 지역·대표 좌표 목록 |
| GET | `/api/dashboard?region=incheon&target_date=2026-10-22&mode=demo` | 중첩 대시보드 응답 |
| GET | `/api/species?q=바지락` | 연습용 생물 항목 검색 |
| GET | `/api/records` | 관찰 기록 목록 |
| GET | `/api/records/{record_id}` | 기록 단건 조회 |
| POST | `/api/records` | 기록 생성, 201 |
| PATCH | `/api/records/{record_id}` | 일부 수정, 200 |
| DELETE | `/api/records/{record_id}` | 삭제, 204 |

생성 예시:

```json
{
  "title": "인천 갯벌 관찰",
  "region": "incheon",
  "observed_on": "2026-10-22",
  "note": "조개와 게를 관찰했습니다.",
  "favorite": false
}
```

수정 예시:

```json
{"favorite": true}
```

`{}`는 모든 값을 유지합니다. `{"note": null}`은 메모를 비웁니다. `{"title": null}`은 최종 모델 재검증에서 422입니다. 잘못된 지역·날짜·타입도 422, 없는 기록은 404, 외부 API의 HTTP/데이터 오류는 502, 응답 시간 초과는 504입니다.

## 현재 구현과 다음 실습

| 기능 | 현재 상태 | 다음에 추가할 코드 |
|---|---|---|
| 지역·날짜별 화면 | 구현 | 지역 목록 확장 |
| 파고·해수면 온도 | 예시 + 외부 API 연결 | 공공 해양 관측/예보 데이터와 비교 |
| 만조·간조 | 모든 지역·날짜에 고정 예시값 | 국립해양조사원 API 인증키, 예보지점 매핑, 응답 변환 |
| 생물 도감 검색 | 직접 입력한 연습 항목 3종 | 검토한 국명·학명·출처 데이터 확대 |
| 사진 | 내 기기 미리보기 | FastAPI 파일 검증 → 이미지 분석 API → 종 후보 |
| 식용 정보 | 미연결 | 종 식별과 분리한 출처 기반 자료 조회 |
| 기록 | 메모리 CRUD | SQLite 저장 |
| Agent/RAG | 미연결 | 위 함수가 안정적으로 동작한 다음 추가 |

권장 실습 순서는 `/docs`로 CRUD를 이해하기 → `services.py`의 외부 API 호출 읽기 → KHOA 물때 연결 → SQLite → 사진 분석입니다. 기존 `C:\KBR\marine_identifier\app.py`는 이후 이미지 업로드 구현의 참고 자료로 사용할 수 있습니다.

국립해양조사원 API는 [조석예보(고, 저조)](https://www.data.go.kr/data/15156018/openapi.do) 활용신청과 가이드를 먼저 확인하세요. 실제 API의 예보지점 코드를 받아 지역과 매핑하고, `demo_tides()` 대신 호출하는 함수를 추가하면 됩니다. 현재 대표 위경도를 그대로 예보지점 코드로 취급하면 안 됩니다. 조석값만으로 갯벌 출입 안전 시간을 판정하지 않습니다.

## 참고한 서비스·공식 문서

확인일: 2026-10-22. 기존 서비스의 화면을 복제하거나 데이터를 수집하지 않았으며 기능 배치와 이용 흐름을 참고했습니다.

| 링크 | 참고한 내용 | 현재 연동 여부 |
|---|---|---|
| [바다타임](https://g.badatime.com/) | 지역 중심의 물때·해양 정보 묶음 | 화면 아이디어만 참고 |
| [Seek by iNaturalist](https://www.inaturalist.org/pages/seek_app) | 사진 기반 식별·관찰 학습 | 기능 흐름만 참고 |
| [국립해양조사원 조석예보 API](https://www.data.go.kr/data/15156018/openapi.do) | 고·저조 시각과 조위 조회 | 다음 실습용, 미연결 |
| [Open-Meteo Marine API](https://open-meteo.com/en/docs/marine-weather-api) | 시간별 파고·해수면 온도 | 실제 연결 |
| [MBRIS](https://www.mbris.kr/) | 해양생물 자료를 직접 찾아볼 링크 | 검색 화면으로 이동하는 일반 링크, 자동 조회 아님 |
| [FastAPI 응답 모델](https://fastapi.tiangolo.com/tutorial/response-model/) | 응답 검증·문서화 | 코드에 적용 |
| [FastAPI 부분 수정](https://fastapi.tiangolo.com/tutorial/body-updates/) | PATCH와 `exclude_unset` | 코드에 적용 |
| [Pydantic Validators](https://docs.pydantic.dev/latest/concepts/validators/) | 전처리·모델 관계 검증 | 코드에 적용 |

Open-Meteo 공개 API는 비상업 학습용으로 사용하고 있으며, 외부 공개·상업 이용 전 현재 [이용약관](https://open-meteo.com/en/terms)과 호출 한도를 확인해야 합니다. 현재 예제는 버튼을 누를 때마다 호출합니다. 캐시는 다음 단계에서 추가할 수 있습니다. UI 기본 필터 값은 서버가 아니라 브라우저의 한국 날짜를 사용합니다.

## 검증

```powershell
.\.venv\Scripts\python.exe -m unittest -v
```

테스트는 대시보드 응답, 잘못된 입력, 기록 CRUD, PATCH 생략/null 처리, 외부 API 응답 변환·오류를 확인합니다. 실제 외부 API 접근성과 예보 정확도는 모의 응답 테스트와 구분해야 합니다.

작성 시 검증: 테스트 8개 통과, JavaScript 구문 검사 통과. 브라우저에서 기록 생성·즐겨찾기·제목 수정과 Open-Meteo 실제 예보 조회를 확인했습니다. `preview.png`는 작은 화면에서 확인한 미리보기입니다. 예보 모델의 정확도와 현장 적합성은 이 검증에 포함하지 않습니다.
