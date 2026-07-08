# 제주 지니 진행상황 HANDOFF (2026-07-08, 에이전트 개편 1·2라운드 반영)

세션이 바뀌어도 이 문서만 읽으면 이어서 작업할 수 있다.

## 현재 상태: 스트리밍 에이전트 + UI 전면 개편 완료

| 항목 | 상태 |
|---|---|
| 백엔드 (스트리밍 tool-calling 에이전트 루프) | 완료 — `app/agent.py`, pytest 51개 통과 |
| 도구 11종 (실시간 8 + 검색 2 + RAG 1) | 완료 — 전부 async, refs/map_points 계약 |
| 신규 API 3종 (/api/now, /api/live/summary, /api/suggest) | 완료 — 로컬 E2E 검증 |
| 프롬프트 (하이브리드 페르소나) | 완료 — "자료에 없다" 금지, 비제주 질문만 거절 |
| 프론트엔드 (Ocean Teal 팔레트, 사이드바, 지도, 후속 칩) | 완료 — `npm run build` 통과 |
| 상태별 마스코트 5종 | 완료 — idle/thinking/answering/answered/sorry (`FloatingMascot.tsx`) |
| 로컬 Docker Compose | 완료 — http://localhost:8090 E2E 5종 검증 (2026-07-08) |
| Cloud Run 배포 | https://jeju-genie-1030073717906.asia-northeast3.run.app |
| CI/CD | 가동 중 — `oiehnow/jeju-genie` main 푸시 → lint/test → 빌드 → AR push → Cloud Run 자동 배포 |

- 배포 이미지: `asia-northeast3-docker.pkg.dev/project-65d06dcc-5794-483b-b6f/mlops-quicklab/jeju-genie:latest`
- Chroma 인덱스 GCS: `gs://project-65d06dcc-5794-483b-b6f-mlops-quicklab/jeju-genie/chroma/` (서비스 시작 시 자동 pull)
- LLM: .env의 `OPENAI_API_KEY` + `OPENAI_MODEL=gpt-5-mini` (추론 강도 low, max_completion_tokens 4096)
- 로컬 8080은 Airflow 웹서버가 점유 중이라 **로컬 포트는 8090** 사용.

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

## 검증 결과 (2026-07-08 로컬 E2E)

- pytest 51개 통과, ruff 통과, `npm run build` 통과
- /api/health: llm_provider=openai, llm_available=true, index_chunks=2936
- /api/now: temp=30, /api/live/summary: 날씨/유가/교통 3항목
- SSE E2E 5종:
  1. "지금 제주 날씨 어때?" — status(weather) → live → 관측 수치 인용 답변 → sources → done
  2. "성산일출봉 근처 추천" — visitjeju_search status → map 8포인트 → visitjeju 출처 링크 → 마크다운 목록 답변
  3. "서울 맛집" — 도구 없이 ㅠㅠ 톤 거절 + 제주 맛집으로 화제 전환
  4. "안녕!" — 즉시 토큰 시작, sources 네이버 폴백 1개
  5. /api/suggest — 후속 질문 3개 생성

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
