# 통근권 (tonggeun) · 수도권 통근 등시권

> 출발지 하나를 찍으면 **수도권 전철 + 도보(+선택적 버스)** 기준으로 "30 / 60 / 90분 안에 닿을 수 있는 범위"를 지도 위에 등시권(等時圈, isochrone)으로 그려 주는 단일 파일 웹앱입니다. 집·직장을 고를 때 "여기서 통근 가능한 동네가 어디까지인가"를 한눈에 가늠하는 용도.

🔗 **라이브:** https://clayborneyeounjunlee.github.io/tonggeun/

---

## ✨ 주요 기능

- 🗺️ **출발지 지정** — 지도를 탭하거나, 주소·지명을 검색(예: `강남역`, `판교`)하거나, 도로명 주소 팝업(📮)으로 선택. 마커를 **드래그**해 미세 조정도 가능.
- ⏱️ **등시권 그리기** — 지정한 출발지에서 **지하철 + 도보** 소요시간을 계산해, 도달 가능한 영역을 색으로 칠한 원(円)들의 합집합으로 시각화.
- 🎚️ **30 / 60 / 90분 밴드 선택** — 세 개의 시간대 버튼 중 하나를 눌러 표시. 각 밴드 임계값(분)은 직접 숫자로 편집 가능.
- 🚶 **도보권 / 🚌 버스권 2겹 표시** — 도보만으로 닿는 진한 영역과, 버스를 포함(근사)했을 때 넓어지는 연한 영역을 구분해서 겹쳐 그림.
- 📍 **도달역 점 표시** — 도달 가능한 역에 점을 찍고, 점을 누르면 `역이름 · 약 NN분` 형태로 예상 소요시간을 InfoWindow로 표시.
- ⚙️ **계산 파라미터 조정** — 도보 속도, 버스 포함 여부·평균속도·대기시간, 환승 패널티, 역에서의 최대 도보시간을 슬라이더로 조절. 조절 즉시 재계산.
- 🌗 **다크 모드** + 🌐 **한/영 전환(EN⇄한)** — 상단 아이콘 버튼으로 토글. 테마는 모아 허브 앱들과 공유.
- 💾 **상태 자동 저장** — 마지막 출발지, 설정값, 언어, 테마를 브라우저에 저장해 다음 방문 시 복원.
- ◈ **모아(moa) 허브로 돌아가기** 버튼.

> ⚠️ 결과는 **역간 표준 소요시간 기반 추정치**입니다. 실제 배차간격·환승 도보·혼잡·실시간 버스 노선은 반영되지 않습니다. 통근 범위를 대략 가늠하는 용도로만 사용하세요.

---

## 🧱 기술 스택 / 언어

| 항목 | 내용 |
|---|---|
| 언어 | 순수 **HTML + CSS + Vanilla JavaScript** (프레임워크·빌드도구 **없음**) |
| JS 형식 | 클래식 스크립트 (`<script>` 직접 임베드, IIFE + `"use strict"`). **ES 모듈 아님**, `import`/`export` 없음 |
| 파일 수 | **단 2개** — `index.html`, `subway-data.js` (전역 `window.SUBWAY`로 데이터 주입) |
| 지도 SDK | **Kakao Maps JavaScript SDK** — `//dapi.kakao.com/v2/maps/sdk.js`, `autoload=false` + `libraries=services`로 **런타임 동적 로드** |
| 주소 팝업 | **Kakao(다음) 우편번호 서비스** — `//t1.kakaocdn.net/mapjsapi/bundle/postcode/prod/postcode.v2.js` (키 불필요·무료) |
| 지오코딩/검색 | Kakao `services.Geocoder`(주소·좌표변환), `services.Places`(키워드 검색) |
| 폰트 | 시스템 폰트 스택 — `Pretendard`, `Apple SD Gothic Neo`, `Malgun Gothic`, `Noto Sans KR` (웹폰트 임베드 없이 로컬 폴백) |
| 스타일 | CSS 커스텀 프로퍼티(디자인 토큰) + `data-theme` 다크모드, `env(safe-area-inset-*)` 대응(모바일/노치) |
| i18n | 자체 구현 딕셔너리(`I18N.ko` / `I18N.en`) + `T()` 헬퍼 + `data-i18n*` 속성 스캔 |
| 알고리즘 | 직접 구현한 **이진 최소 힙(MinHeap)** 기반 **Dijkstra** + Haversine 거리 |
| 백엔드 | **없음** (완전 정적 · 클라이언트 온리) |

CDN 라이브러리는 위 Kakao SDK 2종이 전부이며, 그 외 jQuery/차트/폴리필 등 외부 의존성은 없습니다.

---

## 🏗️ 시스템 구조

### 파일 구성

- **`index.html`** — UI 마크업 + 전체 CSS + 전체 애플리케이션 로직(하나의 IIFE)을 모두 담은 단일 파일.
- **`subway-data.js`** — `window.SUBWAY = { stations:[...], edges:[...] }` 형태의 전역 데이터. `index.html`의 로직보다 **먼저** `<script src="subway-data.js">`로 로드됨.

### 부팅 흐름

```
index.html 로드
  └─ (head) hub-theme 즉시 적용 → 다크모드 깜빡임 방지
  └─ postcode.v2.js (우편번호 SDK) 로드
  └─ subway-data.js 로드 → window.SUBWAY 주입
  └─ 메인 IIFE 실행
       ├─ localStorage에서 언어·설정·출발지 복원
       ├─ COORD 맵 구성 (역이름 → [위도, 경도])
       ├─ applyI18n() : data-i18n* 요소 텍스트/placeholder/title/aria 채우기
       ├─ 컨트롤 바인딩 (검색·밴드·슬라이더·토글·다크·언어·키모달)
       └─ startMap()
            ├─ localStorage["tg_kakaokey"] 없으면 → 키 입력 모달 표시
            └─ 있으면 → loadKakao(key) 로 SDK 동적 주입 → initMap()
                 └─ 지도 생성 → 클릭/타일로드 리스너 → 마지막 출발지 복원
```

### 핵심 모듈·함수 (index.html 내부)

| 영역 | 함수 | 역할 |
|---|---|---|
| 그래프 구축 | `buildAdj(transferPenalty)` | `edges`로 `역␟호선` 노드 인접리스트 생성. 같은 역의 서로 다른 호선 노드 사이에 **환승 패널티** 간선 추가 |
| 거리 | `haversine(la1,lo1,la2,lo2)` | 두 좌표 간 대권 거리(m) |
| 접근시간 | `accessMin(d)` | 집·직장 ↔ 역 한 다리(분). 도보 vs (대기+버스) 중 **빠른 쪽** |
| 도달반경 | `walkReach(t)` / `busReach(t)` | 남은 시간으로 마지막 다리에서 걸어/버스로 뻗는 반경(m) |
| 최단경로 | `computeArrival(oLat,oLng)` | **Dijkstra** — 출발지 → 모든 역까지의 도달시간(분) `Map` 반환 |
| 자료구조 | `MinHeap` | Dijkstra용 이진 최소 힙(push/pop 직접 구현) |
| 렌더 | `render()` | 밴드별로 `drawLayer()` 호출해 도보권·버스권 원을 지도에 그리고, 도달역 점 표시 |
| 렌더 | `drawLayer(Tmin, radiusFn, opacity, z)` | 원들의 합집합을 격자 스냅으로 정리해 **겹침 불투명도 누적을 제한**하며 그림 |
| 지도 | `initMap` / `loadKakao(key)` / `startMap` | 지도 초기화, SDK 동적 로드, 키 유무 분기 |
| 검색 | `doSearch()` / `openPostcode()` / `reverseGeo()` | 주소검색→키워드검색 폴백, 우편번호 팝업, 역지오코딩 |
| 계산 트리거 | `setOrigin()` / `recompute()` / `fitToReach()` | 출발지 설정 → 재계산 → 화면을 도달범위에 맞춤 |

### 상태관리 & 라우팅

- **상태**는 IIFE 클로저 내부의 단일 객체 `S`(설정) + 모듈 변수(`lastArrival`, `lastOrigin`, `map`, `marker` 등)로 관리. 리액티브 프레임워크 없이 이벤트 → 상태 갱신 → `render()`/`recompute()` 직접 호출.
- **라우팅 없음** — 단일 화면. URL 파라미터/해시 라우팅을 쓰지 않고, 상태는 전부 `localStorage`로 지속.

### 등시권 계산 원리 (요약)

1. 출발지 좌표에서 각 역까지 직선거리를 구하고 `accessMin()`으로 "역 진입 시간"을 계산. 최대 접근시간 이내인 역들을 Dijkstra 시작점으로 큐에 넣음(모두 너무 멀면 가장 가까운 역 하나에라도 연결).
2. `역␟호선` 노드 그래프에서 Dijkstra로 모든 역의 **최소 도달시간(분)**을 구함(간선 가중치 = 역간 소요분, 환승 시 패널티 추가).
3. 각 밴드 임계값 `Tmin`에 대해, 출발지 자신과 도달역들에서 "남은 시간(`Tmin - 도달시간`)"만큼 도보/버스로 뻗을 수 있는 반경의 원을 그림. 이 원들의 합집합이 곧 등시권.

---

## 🗂️ 데이터

모든 노선 데이터는 `subway-data.js` 한 파일에 하드코딩되어 있습니다. 출처는 파일 상단 주석에 명시됨:

> `/* 수도권 전철 데이터 · 출처: github.com/ledyx/KoreaMetroGraph (서울교통공사/위키 크롤링) */`

### 규모 (코드로 실측)

| 항목 | 개수 |
|---|---|
| 역(stations) | **604** |
| 구간(edges) | **684** (기본 680 + 파일 끝 누락 보정 4) |
| 노선 코드(line) | **23** |

### 스키마

**stations** — `[역이름, 위도, 경도]` 튜플 배열:

```js
window.SUBWAY = {
  stations: [
    ["녹천", 37.644799, 127.051269],
    ["고속터미널", 37.50481, 127.004943],
    ["양재", 37.484147, 127.034631],
    // … 총 604개
  ],
  ...
};
```

**edges** — `[출발역, 도착역, 소요분, 노선코드]` 튜플 배열:

```js
  edges: [
    ["서동탄", "병점", 5, "1"],
    ["강남", "역삼", 2, "2"],
    ["양재", "남부터미널", 3, "3"],
    // … 총 684개 (무방향으로 해석)
  ]
```

파일 맨 끝에는 원본 크롤링에서 빠진 실제 인접 구간을 **런타임 패치**로 덧붙입니다:

```js
/* ── 누락 보정: 원본 크롤링 데이터에 빠진 실제 인접 구간 ── */
window.SUBWAY.edges.push(
  ["부천","소사",3,"1"],
  ["강남","신논현",2,"S"],
  ["신논현","논현",2,"S"],
  ["논현","신사",2,"S"]
);
```

### 노선 코드 → 노선 (구간 수)

`edges`의 네 번째 필드(노선코드)와 실제 데이터로 확인한 구간 수:

| 코드 | 노선 | 구간 수 |
|---|---|---|
| `1` | 1호선 | 96 |
| `2` | 2호선 | 49 |
| `3` | 3호선 | 42 |
| `4` | 4호선 | 47 |
| `5` | 5호선 | 50 |
| `6` | 6호선 | 38 |
| `7` | 7호선 | 47 |
| `8` | 8호선 | 16 |
| `9` | 9호선 | 29 |
| `A` | 공항철도 | 11 |
| `K` | 경의·중앙선 | 54 |
| `G` | 경춘선 | 23 |
| `B` | 수인·분당선 | 35 |
| `S` | 신분당선 | 15 |
| `SU` | 수인선 구간 | 12 |
| `KK` | 경강선 | 10 |
| `I` | 인천 1호선 | 28 |
| `I2` | 인천 2호선 | 26 |
| `U` | 의정부 경전철 | 14 |
| `E` | 용인 에버라인 | 14 |
| `UI` | 우이신설선 | 12 |
| `W` | 서해선 | 11 |
| `M` | 인천공항 자기부상/자기부상철도 구간 | 5 |

> 노선 코드는 데이터 파일에 담긴 것이며, 코드 안에서는 그래프 간선을 호선별로 구분(같은 역이라도 `역␟호선` 노드로 분리 → 환승 패널티 부여)하는 데만 쓰입니다. UI에 노선명 라벨을 별도로 표시하지는 않습니다.

### 자료구조 변환 (런타임)

- 앱 시작 시 `COORD = Map<역이름, [위도, 경도]>` 로 좌표를 인덱싱.
- `buildAdj()`가 `역␟호선` 문자열을 노드 키로 쓰는 인접리스트(`Map`)와, 각 역이 속한 호선 집합(`stationLines: Map<역, Set<호선>>`)을 만듦. 구분자는 유니코드 `␟`(U+241F).

---

## 💾 저장소 / DB

**서버 DB·Firebase 없음.** 모든 상태는 브라우저 **localStorage**에만 저장됩니다. (게스트/오프라인 상태라도 데이터 파일과 저장된 키만 있으면 계산은 동작. 단, 지도 타일·주소검색은 Kakao SDK와 인터넷이 필요.)

### localStorage 키

| 키 | 용도 | 형식/예시 |
|---|---|---|
| `tg_kakaokey` | **Kakao JavaScript 키** (이 기기에만 저장) | 문자열 |
| `hub-theme` | 라이트/다크 테마 (모아 허브 앱들과 **공유**) | `"light"` / `"dark"` |
| `tg_lang` | 인터페이스 언어 | `"ko"` / `"en"` |
| `tg_settings` | 계산 설정 객체 | JSON — 아래 참조 |
| `tg_origin` | 마지막 출발지 | JSON `{lat, lng, label}` |

`tg_settings` 기본값(코드 기준):

```js
{ walk:4, transfer:4, maxwalk:15, dots:true,
  bus:false, busSpeed:12, busWait:5,
  bands:[30,60,90], on:[true,true,true] }
```

> 코드상 `S`의 초기값 `on`은 `[true,true,true]`지만, 앱 시작 시 "단일 선택" 규칙을 강제하므로(켜진 밴드가 정확히 1개가 아니면) 실제 초기 표시는 `[false,true,false]`(=60분)로 보정됩니다.

| 필드 | 의미 | 기본 | 범위(UI) |
|---|---|---|---|
| `walk` | 도보 속도 (㎞/h) | 4 | 3–6 |
| `transfer` | 환승 패널티 (분) | 4 | 0–10 |
| `maxwalk` | 역에서 최대 도보 (분) | 15 | 5–30 |
| `dots` | 도달역 점 표시 | true | on/off |
| `bus` | 버스 포함(근사) | false | on/off |
| `busSpeed` | 버스 평균속도 (㎞/h) | 12 | 8–20 |
| `busWait` | 버스 평균 대기 (분) | 5 | 0–12 |
| `bands` | 3개 시간대(분) | `[30,60,90]` | 자유 입력 |
| `on` | 어떤 밴드를 켤지(단일선택 강제) | 초기값 `[true,true,true]` → 보정 후 `[false,true,false]` | — |

> 밴드는 **단일 선택**만 허용(겹침·과도한 불투명도 방지). 저장값이 규칙에 안 맞으면 앱이 `[false,true,false]`(=60분)로 보정합니다.

---

## 🌐 외부 API·의존성

이 앱은 형제 앱들과 달리 **외부 지도 API 키가 필요한 유일한** 앱입니다.

### 1) Kakao Maps JavaScript SDK — **키 필요**

- 로드: `//dapi.kakao.com/v2/maps/sdk.js?appkey=<KEY>&autoload=false&libraries=services`
- 쓰이는 기능: 지도, 마커(드래그), 원(Circle) 오버레이, InfoWindow, ZoomControl, `services.Geocoder`(주소↔좌표), `services.Places`(키워드 검색).
- **키 입력 위치:** 앱 첫 실행 시(또는 ⚙️ 설정 → "카카오 키 변경") 뜨는 모달에 **JavaScript 키**를 붙여넣으면 `localStorage["tg_kakaokey"]`에 저장됩니다. 키는 코드에 하드코딩되어 있지 않으며, 각 사용자가 자기 키를 넣습니다.
- **발급·설정 절차(모달 안내 그대로):**
  1. [developers.kakao.com](https://developers.kakao.com) 로그인 → 내 애플리케이션 → 애플리케이션 추가
  2. 앱 선택 → **제품 설정 → 카카오맵 → 활성화 ON** (2024.12부터 필수)
  3. 앱 키에서 **JavaScript 키** 복사 (REST 키 아님)
  4. **플랫폼 → Web → 사이트 도메인**에 배포 주소 등록 (예: `https://clayborneyeounjunlee.github.io`)
  5. 모달에 키 붙여넣기 → 저장
- 지도가 회색으로만 뜬다면 대개 ① 카카오맵 제품 미활성화 ② 사이트 도메인 미등록 문제(앱이 토스트로 안내).

### 2) Kakao(다음) 우편번호 서비스 — **키 불필요**

- `postcode.v2.js`를 `<head>`에서 정적 로드. 📮 버튼으로 도로명 주소 팝업을 띄우고, 선택 주소를 다시 지오코딩해 출발지로 설정.

> 그 외 TravelTime/Google/Skyscanner/환율/Web Speech 등은 사용하지 않습니다.

---

## ▶️ 로컬 실행 방법

빌드 과정이 전혀 없습니다. `package.json`도 없으므로 **정적 파일 서버**로 열기만 하면 됩니다. (`file://`로 직접 열면 Kakao SDK 도메인 제약 때문에 동작이 불안정할 수 있어 로컬 서버 권장.)

```bash
# 저장소 폴더에서 — 아무 정적 서버나 사용
python -m http.server 8000
#   또는
npx serve .
#   또는
npx http-server -p 8000
```

브라우저에서 `http://localhost:8000` 접속 → 첫 실행 시 Kakao **JavaScript 키** 입력.
로컬에서 지도를 쓰려면 Kakao 개발자 콘솔의 **Web 사이트 도메인**에 `http://localhost:8000`(사용 포트)도 등록해야 합니다.

---

## 🚀 배포

**GitHub Pages** 정적 호스팅.

1. 이 저장소를 GitHub에 push (기본 브랜치 `main`).
2. 저장소 **Settings → Pages** → Source: `main` 브랜치 루트(`/`).
3. 배포 URL(예: `https://<사용자명>.github.io/tonggeun/`)을 Kakao 개발자 콘솔의 **Web 사이트 도메인**에 등록해야 지도 타일이 정상 렌더됨.

빌드 스텝·CI 없음 — 파일을 그대로 서빙합니다.

---

## 📁 파일 구조

```
tonggeun/
├── index.html        # 앱 전체 — UI 마크업 + CSS 디자인토큰/다크모드 + 로직(IIFE)
│                     #   · i18n(ko/en), Dijkstra+MinHeap, Kakao 지도/검색/우편번호,
│                     #   · 등시권 렌더(도보권/버스권 2겹), localStorage 상태관리
└── subway-data.js    # 수도권 전철 데이터: window.SUBWAY = { stations[604], edges[684] }
                      #   · 파일 끝에서 누락 구간 4개를 push로 보정
```

- 이 저장소에는 `package.json`, `firebase.json`/`.firebaserc`, `.gitignore`, 서버/라우트 코드가 **없습니다** (순수 정적 2파일).

---

## 🔗 관련 앱 (모아 허브)

- 헤더의 **◈** 버튼 → 모아 허브: https://clayborneyeounjunlee.github.io/moa/
- 형제 앱들과 **디자인 토큰 시스템**(색 변수, `--radius`, 그림자 등)과 다크모드 키(`hub-theme`)를 공유합니다. 코드 주석상 `moa/haru/kanade/dday`와 동일한 공통 UI 시스템을 사용.
