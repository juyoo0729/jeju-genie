# 제주 지니 진행상황 HANDOFF (2026-07-03, UI 개편 반영)

세션이 바뀌어도 이 문서만 읽으면 이어서 작업할 수 있다.

## 현재 상태: 플랫폼 구축 + 배포 완료

| 항목 | 상태 |
|---|---|
| 백엔드 (FastAPI, SSE 챗, RAG, 커넥터 플러그인) | 완료 — pytest 7개 통과 |
| 시드 지식 (제주 개요/관광지/음식/팁 4개 md) | 완료 — ingest 5청크 |
| 프론트엔드 (React 말풍선 채팅, 마스코트 슬롯, 테마 토큰) | 완료 — 이모지 금지 반영 |
| UI 시안 반영 (DLton\UI.png) | 완료 — 흰 헤더 + mascot_2 원형 액자, 좌측 mascot_1 플로팅 애니메이션, "제주에 관한 모든 것" 문구 |
| 상태별 마스코트 전환 | 완료 — 평상시 mascot_1 / 대답 준비 중 mascot_think / 대답 중 mascot_answer (`FloatingMascot.tsx`의 MascotState) |
| 로컬 Docker Compose | 완료 — http://localhost:8090 E2E 검증 |
| Cloud Run 배포 | 완료 — https://jeju-genie-1030073717906.asia-northeast3.run.app |
| CI/CD | **가동 중** — `oiehnow/jeju-genie`(비공개) main 푸시 → lint/test → 빌드 → AR push → Cloud Run 자동 배포 (런 28665679774 전 잡 success, 리비전 00004) |

- 배포 이미지: `asia-northeast3-docker.pkg.dev/project-65d06dcc-5794-483b-b6f/mlops-quicklab/jeju-genie:latest`
- Chroma 인덱스 GCS: `gs://project-65d06dcc-5794-483b-b6f-mlops-quicklab/jeju-genie/chroma/` (서비스 시작 시 자동 pull)
- 현재 배포본은 GPT 키가 없어 챗이 "지니가 아직 준비 중" 안내 반환 (설계된 폴백). 페이지/헬스/인덱스는 정상.
- 로컬 8080은 Airflow 웹서버가 점유 중이라 **로컬 포트는 8090** 사용.

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

## 사용자 제공 항목 도착 시 재개 절차

### 1. GPT(OpenAI) API 키
1. `DLton\jeju-genie\.env` 생성(.env.example 복사) 후 `OPENAI_API_KEY=` 채우기
2. **임베딩 모델이 nomic → OpenAI로 바뀌므로 인덱스 재구축 필수**:
   `GOOGLE_APPLICATION_CREDENTIALS`(Pipeline\solution의 SA 키) + `GCS_BUCKET` 설정 후
   `python -m app.ingest --reset --push`
3. Cloud Run에 키 주입:
   `gcloud run services update jeju-genie --region asia-northeast3 --update-env-vars OPENAI_API_KEY=<키> --project project-65d06dcc-5794-483b-b6f`
4. 배포 URL에서 챗 질문으로 검증

### 2. 제주 정보 API 키 (visitjeju 등)
1. `.env`에 `VISITJEJU_API_KEY=` (새 소스면 `backend/app/connectors/`에 커넥터 파일 추가 —
   README의 "커넥터 추가하는 법" 참조)
2. `python -m app.ingest --push` → `GET /api/sources`로 활성 확인
3. visitjeju.py의 응답 필드 파싱은 실제 API 응답 보고 조정 필요할 수 있음

### 3. 마스코트 이미지 / UI 시안 — 완료 (2026-07-03)
- mascot_1/2.png 적용 완료 (`frontend/src/assets/` — ffmpeg으로 축소본, 원본은 `DLton\` 루트).
- 헤더 원형 액자: `MascotSlot.tsx`, 플로팅 애니메이션: `FloatingMascot.tsx`,
  색/크기 토큰: `theme.css` (`--mascot-size`, `--floating-mascot-width` 등).
- 추후 디자인 변경 시: 에셋 교체 또는 theme.css 수정 → `npm run build` →
  `docker compose up -d --build`(로컬) / 이미지 재빌드+push+`gcloud run deploy`(배포)

### 4. GitHub / CI-CD — 완료 (2026-07-03)
- gh 인증 완료(oiehnow), 비공개 레포 `oiehnow/jeju-genie` 생성·푸시.
- Secrets 등록: `GCP_SA_KEY`, `GCP_PROJECT_ID`. SA에 `roles/run.admin` 부여(사용자 승인).
- 이후 배포는 **main에 push만 하면 자동** (lint/test 실패 시 배포 차단).
- GPT 키 도착 시: `gh secret set OPENAI_API_KEY` 후 워크플로가 Cloud Run 환경변수로 주입.
- UI 2차 수정 반영됨: 새 마스코트 에셋, 플로팅 마스코트 3배(600px), 헤더 그라데이션·타이틀 확대.

## 주의사항
- 도커 빌드는 항상 `--provenance=false --sbom=false --platform linux/amd64` (Vertex/AR 검증된 함정)
- Chroma 컬렉션은 임베딩 모델별로 분리됨(`jeju-<provider>-<model>`) — 키 도입 시 재-ingest 필수인 이유
- Cloud Run은 min-instances 0이라 유휴 과금 없음. GCP 비용은 사용자가 자유 사용 승인함
- 이모지 사용 금지 (UI 문구, 프롬프트, 문서, 대화 전부)
