# 제주 지니 진행상황 HANDOFF (2026-07-08, 에이전트 개편 1·2·3·4라운드 반영)

세션이 바뀌어도 이 문서만 읽으면 이어서 작업할 수 있다.

## 현재 상태: 스트리밍 에이전트 + UI 전면 개편 + 3·4라운드 품질개선 완료

| 항목 | 상태 |
|---|---|
| 백엔드 (스트리밍 tool-calling 에이전트 루프) | 완료 — `app/agent.py`, pytest 67개 통과 |
| 도구 11종 (실시간 8 + 검색 2 + RAG 1) | 완료 — 전부 async, refs/map_points 계약 |
| 신규 API 3종 (/api/now, /api/live/summary, /api/suggest) | 완료 — 로컬 E2E 검증 |
| 3라운드 라이브 API 3종 (/api/news, /api/live/detail, /api/live/density) | 완료 — `app/live_api.py`, 로컬 E2E 검증 |
| 3라운드 UI (사이드바 세분화, 밀집 지도, 뉴스/코스 카드, 대형 마스코트) | 완료 — `npm run build` 통과 |
| 프롬프트 (하이브리드 페르소나) | 완료 — "자료에 없다" 금지, 비제주 질문만 거절 |
| 프론트엔드 (Ocean Teal 팔레트, 사이드바, 지도, 후속 칩) | 완료 — `npm run build` 통과 |
| 상태별 마스코트 7종 | 완료 — idle/thinking/answering/answered/sorry/english/easteregg (`FloatingMascot.tsx`) |
| 4라운드 (이스터에그 2종, 사이드바 마스코트, 주유소 링크, 대화 기록, density 폴백) | 완료 — pytest 72개, 브라우저 E2E 검증 |
| 로컬 Docker Compose | 완료 — http://localhost:8090 E2E 5종 검증 (2026-07-08) |
| Cloud Run 배포 | https://jeju-genie-1030073717906.asia-northeast3.run.app |
| CI/CD | 가동 중 — `oiehnow/jeju-genie` main 푸시 → lint/test → 빌드 → AR push → Cloud Run 자동 배포 |

- 배포 이미지: `asia-northeast3-docker.pkg.dev/project-65d06dcc-5794-483b-b6f/mlops-quicklab/jeju-genie:latest`
- Chroma 인덱스 GCS: `gs://project-65d06dcc-5794-483b-b6f-mlops-quicklab/jeju-genie/chroma/` (서비스 시작 시 자동 pull)
- LLM: .env의 `OPENAI_API_KEY` + `OPENAI_MODEL=gpt-5-mini` (추론 강도 low, max_completion_tokens 4096)
- 로컬 8080은 Airflow 웹서버가 점유 중이라 **로컬 포트는 8090** 사용.

## 2026-07-09 변경

- 헤더 서브타이틀 문구 변경: "제주 여행 AI 컨시어지" → "Everything about Jeju" (`frontend/src/App.tsx`). 커밋 56a66cb 푸시 → CI/CD 배포 성공, 프로덕션 번들에서 새 문구 확인 완료.
- GitHub 토큰 만료로 푸시가 멈추는 문제가 있었음 → `gh auth login -w` 재인증 + `gh auth setup-git`으로 해결 (자격 증명 헬퍼는 gh 경유).

## 아키텍처 (2026-07-08 개편판)

### 에이전트 루프 (`backend/app/agent.py`)
기존 3단계(RAG 프리페치 → 도구선택 비스트리밍 호출 → 답변 스트리밍)를 **단일 스트리밍
tool-calling 루프**로 교체. 도구가 필요 없으면 LLM 왕복 1번에 바로 토큰이 나오고,
필요하면 status 이벤트를 쏜 뒤 asyncio.gather로 병렬 실행하고 재호출한다(최대 3라운드,
`agent_max_rounds`). RAG도 도구(`search_jeju_knowledge`)로 편입. Ollama 폴백은 기존
방식(RAG 프리페치 + 단순 스트리밍) 유지.

### SSE 이벤트 계약 (agent.py ↔ frontend/src/api.ts — 변경 금지)
`status`{tool,label,theme} (도구 실행마다) → `live`{[{name,label}]} (최종 답변 전 1회)
→ `token`{content} → `map`{points:[{name,lat,lng}]} (마커 있을 때만 1회)
→ `sources`{[{title,source,url}]} (**항상 1개 이상** — 비면 네이버 검색 링크 폴백) → `done`

### 도구 11종 (전부 async run, BaseTool의 refs/map_points 수집기 계약)
| 도구 | 파일 | theme | 키 |
|---|---|---|---|
| jeju_airport_weather (제주공항 METAR) | tools/jeju_live.py | weather | KMA_APIHUB_KEY |
| jeju_fuel_price (도 평균 유가) | tools/jeju_live.py | fuel | OPINET_API_KEY |
| jeju_gas_station (최저가 주유소) | tools/jeju_live.py | fuel | OPINET_API_KEY |
| jeju_realtime_traffic (구간 통행속도) | tools/jeju_live.py | traffic | JEJUITS_ITS_CODE |
| jeju_ev_charger (충전소 검색/집계) | tools/jeju_live.py | ev | CITS 또는 데이터허브 |
| jeju_place_search (VWorld 지오코딩) | tools/jeju_live.py | place | VWORLD_API_KEY |
| jeju_statistics (데이터허브 시계열) | tools/jeju_live.py | stats | JEJUDATAHUB_PROJECT_KEY |
| jeju_dialect (방언사전) | tools/jeju_live.py | dialect | 불요 |
| visitjeju_search (관광 콘텐츠 검색) | tools/search_live.py | place | VISITJEJU_API_KEY |
| jeju_wiki_search (한국어 위키백과) | tools/search_live.py | knowledge | 불요 |
| search_jeju_knowledge (RAG 벡터검색) | tools/knowledge.py | knowledge | store 주입 |

- visitjeju_search가 장소 질문 1순위 (title 부분일치 + category c1~c6 필터, latitude/longitude로 지도 마커).
- 도구가 `self.refs`(출처 칩)와 `self.map_points`(지도 마커)를 채우면 에이전트가 라운드마다 회수해 SSE로 내보낸다.

### 신규 API 3종 (`backend/app/main.py`)
- `GET /api/now` — 헤더 기온 칩 (METAR 기온 숫자, 10분 캐시)
- `GET /api/live/summary` — 사이드바 실시간 패널 (날씨/유가/교통 병렬, 5분 캐시, 실패 항목 제외)
- `POST /api/suggest` — 문답 기반 후속 질문 3개 (gpt-5-mini, 실패 시 빈 배열로 500 금지)

### 프롬프트 (`backend/app/prompts.py`) — 하이브리드 페르소나
- SYSTEM_PROMPT: "모든 제주 자료를 이미 갖춘 안내 요정" 콘셉트. 제주 질문에
  "자료에 없다"/"검색해 보니" 금지, 실시간 수치 필수 인용("지금 기준"), 수치 지어내기 금지.
- 비제주 질문(다른 지역/코딩 등)만 "그건 제 자료함에 없어요, 죄송해요 ㅠㅠ" 톤으로 거절
  후 제주로 화제 전환 (텍스트 이모티콘 허용, 이모지 금지).
- AGENT_TOOL_GUIDE: 도구 라우팅 지침 (예보/항공편은 도구 없음 — 지어내지 말고 확인 안내).

### 프론트엔드 UI 콘셉트
- Ocean Teal 팔레트 + Pretendard 폰트 (`theme.css`), 좌측 `Sidebar.tsx`
  (실시간 정보 패널 = /api/live/summary, 추천 질문), 헤더 기온 칩 (= /api/now)
- `MapCard.tsx` — react-leaflet 지도 (map 이벤트의 points 마커)
- `WaitingIndicator.tsx` — 5초마다 회전하는 테마별 대기 멘트 (status 수신 시 테마 전환)
- `MessageBubble.tsx` — react-markdown + remark-gfm 렌더, 답변 후 후속 질문 칩 (= /api/suggest)
- 마스코트: idle/thinking/answering/answered/sorry — sorry는 거절 답변("자료함에 없어요" 감지) 시

## 3라운드 개편 (2026-07-08, 에이전트 3개 + 수동 수정)

### 신규 라이브 API 3종 (`backend/app/live_api.py`, main.py 라우터 include)
- `GET /api/news` — 구글 뉴스 RSS '제주' 검색 상위 4건 (30분 캐시, 실패 시 빈 목록·빈 결과 미캐시)
- `GET /api/live/detail?category=fuel|weather|traffic&refresh=1` — 카테고리별 실시간 상세
  (5분 캐시, refresh=1 무시. fuel은 도 평균 유가 + 최저가 주유소 둘 다, "조회 실패" 문구 제외)
- `GET /api/live/density` — 중국인 관광객 밀집 지역 상위 12곳 (30분 캐시).
  제주데이터허브 "읍면동 단위 외국인 단기체류 유동인구"를 nationality=중국으로 조회,
  읍면동 43곳 좌표표(`_EMD_COORDS`)에 매핑해 level 1(빨강)/2(주황)/3(노랑) 3등분.
  월간 데이터가 2~3개월 지연 게시라 최대 8개월 거슬러 최신월 탐색 (현재 2026-04).
- 신규 `tests/test_live_api.py` — 전체 pytest 67개 통과
- `prompts.py` — 지도 마커 관련 지침 1줄 추가

### 3라운드 UI
- `Sidebar.tsx` 세분화: 실시간 정보(유가/날씨/교통 패널 + 새로고침 버튼 = /api/live/detail),
  중국인 관광객 밀집(= /api/live/density), 추천 코스 6종 카드(클릭 시 프리셋 질문 전송),
  오늘의 제주 뉴스 카드(= /api/news)
- 신규 `DensityMap.tsx` — 지도 오버레이, CircleMarker 빨강/주황/노랑 + 범례
- `FloatingMascot.tsx` 대형화 + 신규 `mascot.css` (그림자 제거), `mascot_2.png` 에셋 교체(430x401)
- `MapCard.tsx` 개선 + 신규 `mapcard.css` (280px 높이 + 헤더바)

### 3라운드 데이터 발견사항
- 데이터허브 "중국인 카드소비" 계열 API 4종은 데이터가 2016~2018에서 멈춘 스테일 —
  밀집 지도는 "읍면동 단위 외국인 단기체류 유동인구"(최신 2026-04)로 대체함.
- 데이터허브 프록시는 필터와 number/page를 섞으면 페이지가 안 넘어감 —
  limit 파라미터(최대 100)로 한 번에 43개 읍면동 수신 (live_api.py 주석 참조).
- **기상청 API허브 상류 장애 중** (APPLICATION_ERROR): /api/now temp=null,
  /api/live/detail?category=weather items 빈 배열은 코드 문제가 아님. 복구되면 자동 정상화.

## 4라운드 개편 (2026-07-08 저녁, 사용자 피드백 반영)

### 이스터에그 2종 (App.tsx + FloatingMascot.tsx)
- 영어 질문(한글 없이 알파벳 포함) → 영어로 답변(prompts.py 답변 스타일 1줄 추가) +
  마스코트가 `mascot_english.png` 로 전환. 다음 일반 질문에서 평소 상태로 복귀.
- 정확히 "guten tag" 입력(대소문자 무관) → API 호출 없이 "Auf Wiedersehen!" 즉답 +
  `mascot_easteregg.png` 전환.
- 신규 에셋 2종 assets/ 추가 (english 원본 2400px 는 1200px 로 축소 최적화, 1.4MB→349KB)

### UI 변경
- 마스코트 위치 이동: 채팅 카드 우하단 플로팅 → **사이드바** 로그인/회원가입 메뉴와
  "오늘의 제주 뉴스" 카드 사이. 사이드바 폭 264→300px(theme.css). mascot.css 는
  `.sidebar-mascot` 로 재작성 (flex-shrink:0 — 서브패널이 열려도 찌그러지지 않고
  사이드바가 스크롤됨). app.css 의 구 .floating-mascot 규칙 제거.
- 명칭 변경(메커니즘 그대로): 사이드바 버튼 "실시간 외국인 관광객 밀집",
  오버레이 제목 "외국인 관광객 밀집 지도", 부제 "외국 카드 사용 내역 기반".
  asof 에 "기준"이 중복 표기되던 버그도 수정.
- 유가 패널: 제주시 휘발유 최저가 주유소 상위 7곳을 **카카오맵 검색 링크**로 제공.
  /api/live/detail fuel 응답에 links[{label,url,desc}] 추가 — live_api 의
  `_fuel_station_item()` 이 오피넷 lowTop10 직접 조회 (주유소 도구는 챗 에이전트용 유지).
- **대화 기록 탭**: 새 대화 시 이전 대화가 사라지는 문제 해결 — localStorage
  (`jeju-genie-chat-history`, 최대 20건)에 자동 보관, 사이드바 "대화 기록"에서
  복원(이어서 대화 가능)/삭제. 복원 시 현재 대화도 보관됨.

### density Cloud Run 폴백 (3라운드 배포 검증에서 발견)
- Cloud Run 에서 데이터허브 프록시가 **비JSON 응답**을 반환해 /api/live/density 가
  빈 지도({"asof": null, "points": []})가 되는 문제 발견 (로컬은 정상 —
  GCP 발신 IP 에 대한 상류 이슈로 추정, 로그의 JSONDecodeError 로 확인).
- live_api.py 대응: 월별 조회 실패 시 이전 달 계속 시도 + 전부 실패하면 번들 스냅샷
  (`_DENSITY_FALLBACK`, 2026-04 실측 상위 12곳)으로 폴백. 폴백은 캐시하지 않아
  상류 복구 시 자동 회복. 비JSON 본문 앞 200자를 로그로 남긴다 (원인 추적용).

## 검증 결과 (2026-07-08 로컬 E2E)

### 1·2라운드
- pytest 51개 통과, ruff 통과, `npm run build` 통과
- /api/health: llm_provider=openai, llm_available=true, index_chunks=2936
- /api/now: temp=30, /api/live/summary: 날씨/유가/교통 3항목
- SSE E2E 5종:
  1. "지금 제주 날씨 어때?" — status(weather) → live → 관측 수치 인용 답변 → sources → done
  2. "성산일출봉 근처 추천" — visitjeju_search status → map 8포인트 → visitjeju 출처 링크 → 마크다운 목록 답변
  3. "서울 맛집" — 도구 없이 ㅠㅠ 톤 거절 + 제주 맛집으로 화제 전환
  4. "안녕!" — 즉시 토큰 시작, sources 네이버 폴백 1개
  5. /api/suggest — 후속 질문 3개 생성

### 3라운드
- pytest 67개 통과, `npm run build` 통과
- docker compose E2E (http://localhost:8090):
  - /api/health ok (index_chunks=2936), /api/news items 4건
  - /api/live/detail?category=fuel items 2건(도 평균 유가 + 최저가 주유소),
    traffic 1건(구간 12081개), weather 빈 배열(기상청 API허브 상류 장애 — 코드 문제 아님)
  - /api/live/density: points 12곳, level 1~3 전부 존재, asof "2026-04 기준"
  - 챗 "애월에서 가볼 만한 카페 추천해줘" — status(visitjeju_search) → live → 마크다운
    목록 답변 토큰 → map 2포인트 → sources(visitjeju 링크 2건) → done

### 4라운드 (브라우저 실측 E2E — claude-in-chrome)
- pytest 72개 통과, ruff 통과, `npm run build` 통과
- localhost:8090 브라우저 검증:
  - 마스코트가 사이드바 로그인 메뉴와 뉴스 카드 사이에 표시, 유가 패널을 열어도 유지
  - "guten tag" → "Auf Wiedersehen!" 즉답 + 이스터에그 마스코트
  - "What is the best beach in Jeju?" → 영어 마크다운 답변 + 영어 마스코트
  - 새 대화 → 대화 기록에 저장(제목=첫 질문) → 클릭 복원(전체 메시지 유지) 확인
  - 유가 패널: 평균 유가 + 최저가 주유소 TOP 7 링크(카카오맵) 표시
  - 밀집 지도: "외국인 관광객 밀집 지도 / 2026-04 기준 · 외국 카드 사용 내역 기반"

## 자주 쓰는 명령

```bash
# 로컬 컨테이너 (권장)
cd DLton/jeju-genie && docker compose up -d --build     # http://localhost:8090
curl -X POST "http://localhost:8090/api/ingest?reset=true"

# 로컬 개발 서버
cd backend && ./.venv/Scripts/python.exe -m uvicorn app.main:app --port 8090

# 테스트 / 인덱스 재구축
./.venv/Scripts/python.exe -m pytest tests/ -q
./.venv/Scripts/python.exe -m app.ingest --reset [--push]

# 수동 재배포 (빌드부터 Cloud Run까지)
./scripts/deploy_cloudrun.ps1
```

주의: docker compose의 chroma-data 볼륨이 비어 있으면 index_chunks=0 —
호스트 인덱스를 `docker cp backend/chroma_data/. jeju-genie-jeju-genie-1:/srv/chroma_data/`
후 재시작하거나 `/api/ingest`로 재구축.

## 보류 항목 (공공데이터포털 활용신청 필요)

- **기상청 단기예보** (`VilageFcstInfoService_2.0`): DATA_GO_KR 키로 403 —
  포털에서 해당 API "활용신청" 승인 후 jeju_weather_forecast 도구 추가
  (좌표: 제주시 53,38 / 서귀포 52,33 / 성산 60,37 / 고산 46,35, 발표시각 02/05/08/11/14/17/20/23시)
- **한국공항공사 운항정보**: SERVICE KEY IS NOT REGISTERED — KAC 운항정보 활용신청 후
  jeju_flight_info 도구 추가
- 그때까지 예보/항공편 질문은 AGENT_TOOL_GUIDE 지침대로 "지금 기준" 관측 + 일반 지식 +
  확인 안내로 답한다 (수치 지어내기 금지).

## 주의사항
- 도커 빌드는 항상 `--provenance=false --sbom=false --platform linux/amd64` (Vertex/AR 검증된 함정)
- Chroma 컬렉션은 임베딩 모델별로 분리됨(`jeju-<provider>-<model>`)
- Cloud Run은 min-instances 0이라 유휴 과금 없음. GCP 비용은 사용자가 자유 사용 승인함
- 이모지 사용 금지 (UI 문구, 프롬프트, 문서, 대화 전부), .env 커밋 금지
- SSE 이벤트 계약(agent.py 독스트링)은 프론트와 합의된 것 — 변경 금지
