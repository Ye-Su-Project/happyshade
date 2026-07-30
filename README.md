# ☀️ 해피해요 (Happyshade)

> **"버스 탈 때, 햇빛 없는 시원한 좌석을 알려드려요!"**  
> **해피해요**는 이동 경로상의 태양 위치를 실시간으로 계산하여, 버스 내에서 햇빛을 피할 수 있는 최적의 좌석(왼쪽/오른쪽)을 추천해 주는 앱인토스(Apps in Toss) 기반 미니앱입니다.

---

## 📌 주요 기능

* **📍 현재 위치 및 목적지 기반 경로 탐색**
  * 위치 권한(GPS)을 통한 현재 위치 자동 설정 (브라우저 Fallback 지원)
  * 디바운스(400ms) 적용된 자동 검색으로 목적지 주소 및 좌표 변환
* **☀️ 실시간 태양 위치 및 경로 계산**
  * 출발 시각 기준 태양의 방위각(Azimuth) 및 고도(Altitude) 계산
  * 실제 도로 경로(Polyline) 구간별 진행 방향(Bearing)과 태양 위치 분석
* **🚌 최적 좌석 추천**
  * 전체 이동 거리 중 햇빛이 덜 드는 좌석(왼쪽 vs 오른쪽) 자동 판정
  * 야간 시간대(태양 고도 ≤ 0) 안내 처리

---

## ⚙️ 시스템 구조 & 로직 요약

### 1. 위치 및 검색 수집 (Location & Search)
* **출발지**: 앱인토스 네이티브 브릿지 (`getCurrentLocation`) 우선 호출 $\rightarrow$ 실패 시 브라우저 `navigator.geolocation`으로 자동 대체
* **목적지**: Nominatim API를 활용한 키리스(Keyless) 주소-좌표 변환 (400ms 디바운스 적용)

### 2. 실제 도로 경로 탐색 (Routing)
* **OSRM(Open Source Routing Machine)** `driving` 프로필을 사용해 실제 도로 경로 좌표 배열 수집
* 경로 수신 실패 시 출발지 $\rightarrow$ 목적지 직선 방위각 계산으로 자동 전환(Fallback)

### 3. 태양 위치 계산 (Sun Position)
* `suncalc` 라이브러리를 통해 현재 시각·위도·경도 기반 **태양 방위각**과 **고도** 산출
* SunCalc 방위각(남쪽 0°)을 표준 나침반 방식(북쪽 0°, 시계방향)으로 보정:  
  $$\text{Standard Azimuth} = (\text{Azimuth} + 180) \bmod 360$$
* 고도 $\le 0$일 경우 해가 진 상태로 판단하여 예외 처리

### 4. 좌석 판정 로직 (Seat Algorithm)
1. 전체 도로 경로를 여러 구간(Segment)으로 분할
2. 각 구간의 **진행 방향(Bearing)** 계산
3. 상대 각도(`relativeAngle = (태양방위각 - 진행방향 + 360) % 360`) 산출:
   * `relativeAngle < 180`: 오른쪽에서 햇빛 들어옴 $\rightarrow$ **왼쪽 좌석 추천**
   * `relativeAngle ≥ 180`: 왼쪽에서 햇빛 들어옴 $\rightarrow$ **오른쪽 좌석 추천**
4. 하버사인(Haversine) 공식을 활용해 구간별 거리를 누적 계산한 후, 전체 이동 거리 기준 햇빛 노출이 적은 좌석 최종 추천

---

## 📱 화면 흐름 (UI Flow)

```
[HOME] (GPS/목적지 입력)
![메인 화면](images/main-screen.png)
  └──> [SEARCH] (전체화면 목적지 검색)
        └──> [LOADING] (OSRM 경로 분석 & 계산)
              ├──> [RESULT] (버스 비주얼 + 방향 뱃지 & 추천 문구)
              ![결과 화면](images/result-screen.png)
              └──> [ERROR] (경로/위치 불러오기 실패 시 안내)
```

---

## 🛠 기술 스택 (Tech Stack)

* **Framework**: React, TypeScript, Apps in Toss Web Framework (Granite)
* **Design System**: `@toss/tds-mobile` (Toss Design System)
* **Library**: `suncalc`
* **API**:
  * [OpenStreetMap Nominatim](https://nominatim.openstreetmap.org/) (주소 검색)
  * [OSRM Router](https://project-osrm.org/) (도로 경로)

---

* **도로 경로 근사**: API Key 발급 없이 운영 가능한 오픈 API 사용을 위해 정밀 버스 노선 대신 일반 도로 주행 경로를 활용합니다. (버스 전용 차로 등에 따른 미세한 차이가 존재할 수 있음)
* **단일 태양 위치 적용**: 이동 시간이 짧은 버스 이용 특성을 고려하여, 출발 시각의 태양 위치 1회를 기준값으로 설정합니다.
