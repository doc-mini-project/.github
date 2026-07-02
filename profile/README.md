# 📄 AI 문서 OCR·요약·분류 RAG 시스템

## 1. 프로젝트 소개

본 프로젝트는 사용자가 업로드한 문서(PDF, HWP/HWPX, DOCX, PPT/PPTX)를 분석하여 **요약**과 **카테고리(대분류·소분류)** 를 자동으로 생성해 주는 **문서 처리 RAG 파이프라인**입니다.

단순히 텍스트를 추출해서 LLM에 넘기는 구조가 아니라,
**형식별 전용 추출 → OCR 폴백 → 벡터화(RAG) → 로컬 LLM 요약·분류 → DB 저장**까지 이어지는 파이프라인을 목표로 합니다.

프론트엔드에서 파일을 업로드하면 `job_id`가 발급되고, 이 `job_id`로 진행률과 현재 단계, 최종 결과(요약·분류)를 실시간으로 조회할 수 있습니다.

---

## 2. 핵심 목표

* PDF / HWP / HWPX / DOCX / PPT / PPTX 문서 업로드
* 문서 형식별 텍스트 추출 및 OCR 폴백 처리
* 청킹 및 임베딩을 통한 벡터 저장소(Chroma) 구성
* 로컬 LLM(Ollama)을 이용한 문서 요약
* 문서 대분류·소분류 자동 분류
* `job_id` 기반 비동기 처리 및 진행 상태 조회
* 처리 결과(텍스트, 요약, 분류)의 PostgreSQL 저장

---

## 3. Repository 구성

프로젝트는 프론트엔드와 백엔드를 분리하여 관리합니다.

| Repository | Description | 비고 |
| --- | --- | --- |
| [frontend](https://github.com/doc-mini-project/frontend) | 업로드 · 진행 상태 · 결과 확인 UI | React 18 + Vite 6 |
| [backend](https://github.com/doc-mini-project/backend) | 추출 · RAG · LLM 요약/분류 API 서버 | FastAPI |

### Frontend Repository

* 클릭 또는 드래그 앤 드롭 문서 업로드
* 업로드 확장자 검증 (`.hwp .hwpx .pdf .docx .ppt .pptx`)
* `job_id` 기반 진행률(progress) · 단계(stage) · 메시지(message) 폴링 표시
* 요약 결과 확인 및 텍스트 다운로드
* 대분류·소분류 결과 표시

### Backend Repository

* 파일 업로드 및 확장자 검증
* 문서 형식별 텍스트 추출기 (PDF / HWP·HWPX / DOCX / PPT)
* 텍스트 추출 실패 시 OCR(PaddleOCR) 폴백
* 텍스트 청킹 및 임베딩(BGE-M3) 후 Chroma 저장
* Ollama 기반 요약 및 대분류·소분류 생성
* 처리 결과 PostgreSQL 저장
* 작업 상태 조회 API 제공 (in-memory)

---

## 4. 전체 서비스 구조

```text
사용자
  ↓
React Frontend (Vite)
  ↓  POST /api/process/start
FastAPI Backend
  ↓
문서 형식별 추출기 (PDF · HWP/HWPX · DOCX · PPT)
  ↓ (텍스트 부족 시)
PaddleOCR 폴백
  ↓
청킹 · BGE-M3 임베딩
  ↓
Chroma 벡터 저장소
  ↓
Ollama LLM (요약 · 대분류 · 소분류)
  ↓
PostgreSQL 저장
  ↓  GET /api/process/{job_id}
React Frontend (진행률 · 결과 표시)
```

---

## 5. 주요 기술 스택

### Frontend

* React 18
* Vite 6
* Axios

### Backend

* Python 3.10+
* FastAPI, Uvicorn, Pydantic
* PostgreSQL, SQLAlchemy

### 문서 추출 / OCR

* PaddleOCR
* PyMuPDF, pdfplumber
* python-docx, python-pptx
* olefile (HWP)

### RAG / LLM

* LangChain, LangChain Text Splitters
* Sentence Transformers
* Chroma (벡터 저장소)
* Ollama (기본 모델 `qwen3:8b`)
* 임베딩 모델: `BAAI/bge-m3`

---

## 6. 문서 업로드 처리 흐름

사용자가 문서를 업로드하면 다음 흐름으로 처리됩니다.

```text
1. 파일 업로드 (POST /api/process/start)
2. 확장자 검증 (.hwp .hwpx .pdf .docx .ppt .pptx)
3. job_id 발급, 상태 "queued" 로 초기화
4. [stage: upload]  파일 서버 저장 (progress 5%)
5. [stage: extract] 문서 형식별 텍스트 추출 (progress 20%)
   - 텍스트 추출 실패 시 처리 중단, 상태 failed
6. [stage: rag]      청킹 + 임베딩 + Chroma 저장 (progress 55%)
7. [stage: llm]      Ollama 기반 요약 · 대분류 · 소분류 생성 (progress 70%)
8. [stage: db]       Category / Document / Job 테이블에 저장 (progress 90%)
9. [stage: completed] 처리 완료 (progress 100%)
   - 예외 발생 시 상태 failed 로 전환, 에러 메시지 기록
```

---

## 7. 진행 상태 조회 흐름

프론트엔드는 업로드 직후 `job_id`를 받아 약 0.9초 간격으로 상태를 폴링합니다.

```text
1. GET /api/process/{job_id} 호출
2. status / progress / stage / message 수신
3. status == "completed" → 결과(result) 표시 후 폴링 종료
4. status == "failed"    → 에러 메시지 표시 후 폴링 종료
5. 그 외 (queued / running) → 폴링 유지
```

---

## 8. 주요 상태값

### status

| 상태값 | 설명 |
| --- | --- |
| `queued` | 작업 대기 중 |
| `running` | 파이프라인 처리 중 |
| `completed` | 처리 완료 |
| `failed` | 처리 실패 |

### stage

| 단계값 | 설명 |
| --- | --- |
| `upload` | 파일 저장 |
| `extract` | 문서 텍스트 추출 |
| `rag` | 청킹·임베딩·Chroma 저장 |
| `llm` | 요약·분류 생성 |
| `db` | PostgreSQL 저장 |
| `completed` / `failed` | 종료 상태 |

프론트엔드는 이 값을 기반으로 다음과 같이 진행 상태를 표시합니다.

```text
파일 저장 중
문서 텍스트 추출 중
벡터 DB 저장 중
요약/분류 생성 중
DB 저장 중
처리 완료
```

---

## 9. 데이터 모델

PostgreSQL에는 3개의 테이블이 사용됩니다.

**category** — 문서 분류 정보

| 컬럼 | 설명 |
| --- | --- |
| `cat_id` | 분류 고유번호 (PK) |
| `main` | 대분류 (예: 계약서) |
| `sub` | 소분류 (예: 임대차계약) |
| `extension` | 파일 확장자 |

**document** — 업로드 문서 및 처리 결과

| 컬럼 | 설명 |
| --- | --- |
| `doc_id` | 문서 고유번호 (PK) |
| `file_name` / `file_type` / `file_size` | 파일 메타데이터 |
| `cat_id` | category 참조 (FK) |
| `content_full` | 추출된 전체 텍스트 |
| `content_sum` | LLM 요약문 |
| `saved_time` | 저장 시각 |
| `visible` | 삭제 여부 (soft delete) |

**job** — 작업 이력

| 컬럼 | 설명 |
| --- | --- |
| `job_id` | 작업 고유번호 (PK) |
| `job_start` / `job_finish` | 작업 시작·종료 시각 |
| `doc_id` | document 참조 (FK) |
| `status` | 완료 여부 (Boolean) |

> 실시간 진행 상태(`progress`, `stage`, `message`)는 DB가 아닌 **서버 메모리**에서 관리되며, 최종 완료된 결과만 위 테이블에 저장됩니다.

---

## 10. API 문서

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `GET` | `/health` | 서버 상태 확인 |
| `POST` | `/api/process/start` | 문서 업로드 및 처리 시작 (`job_id` 반환) |
| `GET` | `/api/process/{job_id}` | 진행 상태 및 처리 결과 조회 |

FastAPI 자동 문서: `http://localhost:8000/docs`

---

## 11. 실행 방법

### Backend

```bash
git clone https://github.com/doc-mini-project/backend.git
cd backend

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# DATABASE_URL, OLLAMA_MODEL 등을 환경에 맞게 수정

ollama pull qwen3:8b
ollama serve

uvicorn main:app --reload --port 8000
```

> PostgreSQL과 Ollama가 먼저 실행되어 있어야 합니다. HWP 변환은 환경에 따라 LibreOffice 등 별도 변환 도구가 필요할 수 있습니다.

### Frontend

```bash
git clone https://github.com/doc-mini-project/frontend.git
cd frontend

npm install
cp .env.example .env
# VITE_API_BASE_URL이 백엔드 주소와 다르면 수정

npm run dev
```

개발 서버: `http://localhost:5173`

---

## 12. 환경 변수 예시

**Backend (`.env`)**

```env
DATABASE_URL=postgresql+psycopg2://postgres:YOUR_PASSWORD@localhost:5432/rag_db

APP_NAME=Document OCR and RAG Pipeline
ALLOWED_EXTENSIONS=.hwp,.hwpx,.pdf,.docx,.ppt,.pptx
UPLOAD_DIR=./data/uploads
WORK_DIR=./data/work
CHROMA_DIR=./data/chroma_db
OCR_OUTPUT_DIR=./data/ocr_output

OLLAMA_URL=http://localhost:11434
OLLAMA_MODEL=qwen3:8b
OLLAMA_TIMEOUT_SEC=300

EMBEDDING_MODEL_NAME=BAAI/bge-m3
EMBEDDING_PREFIX_MODE=bge
```

**Frontend (`.env`)**

```env
VITE_API_BASE_URL=http://localhost:8000
```

실제 `.env` 파일은 Git에 올리지 않고, `.env.example` 파일만 공유합니다.

---

## 13. 프로젝트 구조

**backend**

```text
main.py                 # FastAPI 진입점, 작업 상태 관리
document_pipeline.py    # 문서 형식별 추출 오케스트레이션
rag_pipeline.py         # 청킹·임베딩·Chroma 저장
llm_chain.py             # Ollama 요약·분류
database.py / models.py  # PostgreSQL 연결 및 SQLAlchemy 모델
app/                      # 설정·스키마
extractors/               # PDF · HWP/HWPX · DOCX · PPT 추출기
utils/                     # 공통 유틸리티
```

**frontend**

```text
src/App.jsx      # 업로드 · 진행 상태 · 결과 화면
src/api.js       # 백엔드 API 호출
src/main.jsx     # React 진입점
src/styles.css   # 화면 스타일
```

---

## 14. 현재 한계점 및 참고 사항

* 작업 진행 상태(`progress`, `stage`)는 서버 메모리에 저장되므로, 백엔드 재시작 시 진행 중이던 작업 상태는 초기화됩니다.
* 별도의 사용자 인증(로그인/JWT) 기능은 포함되어 있지 않으며, 단일 사용자 기준의 문서 처리 파이프라인입니다.
* Docker 구성은 아직 포함되어 있지 않습니다.
* `.env` 파일은 저장소에 커밋하지 않습니다.
* 업로드 원본, OCR 결과, Chroma 데이터는 실행 중 생성되며 Git에서 제외됩니다.

---

## 15. 프로젝트 완성 기준

* 사용자가 문서를 업로드할 수 있다.
* 지원 확장자가 아닌 파일은 업로드가 차단된다.
* 업로드 즉시 `job_id`가 발급되고, 진행 상태를 실시간으로 조회할 수 있다.
* 문서 형식에 맞는 텍스트 추출이 수행되고, 필요 시 OCR로 보완된다.
* 추출된 텍스트가 청킹·임베딩되어 Chroma에 저장된다.
* Ollama 기반으로 요약과 대분류·소분류가 생성된다.
* 처리 결과가 PostgreSQL에 저장된다.
* 처리 실패 시 사용자에게 실패 사유가 표시된다.

---

## 16. 최종 목표

이 프로젝트는 단순 텍스트 추출 도구가 아니라, **문서 형식 다양성 대응 + OCR 폴백 + RAG 기반 요약·분류**를 함께 다루는 문서 처리 파이프라인 구현을 목표로 합니다.

핵심은 다음과 같습니다.

* 문서 형식별 전용 추출기 설계
* OCR 폴백을 통한 텍스트 추출 안정성 확보
* 벡터DB(Chroma) 기반 RAG 파이프라인 구성
* 로컬 LLM(Ollama)을 통한 요약·분류 자동화
* 비동기 작업 상태 관리 및 실시간 진행률 제공
