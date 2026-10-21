# 1. Mini Project Object Document

### Information

* **Dev Date :** 10/17/2026 ~ 10/24/2026 23:59:59
* **Member :**
  - 김영민 : Backend (`Server application develop`) 
  - 승진 : Frontend (`Web event handler`)
  - 최건우 : Infra, Frontend, Backend
      - Infra (`Docker -> Docker Compose`)
      - Frontend (`React build UI`)
      - Backend (`Dockerfile management`)
  - 최민기 : Backend (`OCR`)

### Objective

OpenAI API를 통해 PDF 파일의 이미지에서 텍스트 (Image-Text-to-Text)을 추출하여 결과 데이터통해 UI에 출력되도록 하는 방식 설계를 진행하고 AI Agent의 Parameter Size가 1B (100,000,000) 와 14B (1,400,000,000) 두 개의 모델을 선정하여 구성 가능한 환경에서 병목 현상이 일어나는지 테스트를 진행해 정확도가 높은 Application을 구축한다.

# 2. Project Feature Requirement

<details>

<div align="center">
    <img width="40%" alt="" src="./img/miniproject-requirement.jpg">
</div>
  
</details>

이 프로젝트는 PDF 및 이미지 기반 문서에서 텍스트를 추출하고, 이를 AI 엔진을 통해 처리한 뒤 사용자에게 결과를 보여주는 시스템을 구축하는 것을 목표로 한다. 사용자는 React 기반 웹 애플리케이션을 통해 문서를 업로드하고, Spring Boot 서버가 이를 받아 FastAPI OCR 서버로 전달한다. OCR 서버는 PDF를 이미지로 변환하거나 전처리한 뒤, Tesseract, PaddleOCR, OpenAI, Ollama 등 다양한 OCR 엔진을 선택하거나 자동 모드로 실행한다. 추출된 텍스트는 JSON 형태로 변환되어 백엔드로 전달되고, DB에 저장되며 최종적으로 웹 화면에서 문서 목록, 상세 결과, 추출 텍스트를 확인할 수 있다.

* 사용자 인증 및 문서 관리
  * 회원가입, 로그인 기능
  * 문서 업로드 및 저장
  * 업로드된 문서 목록 조회
  * 문서 상세 정보 및 추출 결과 확인

* OCR 및 AI 처리 기능
  * PDF 파일의 이미지 기반 텍스트 추출
  * 이미지 전처리 및 OCR 엔진 선택
  * OpenAI API 기반 텍스트 추출 및 결과 정제
  * OCR 엔진 성능 비교 및 평가

* 시스템 구성 및 배포
  * Frontend: React + Vite
  * Backend: Spring Boot
  * OCR API: FastAPI
  * Database: MySQL
  * Infra: Docker Compose, Kubernetes 기반 환경 구성

* 개발 구조
  * web: 사용자 인터페이스
  * server/backend: 비즈니스 로직 및 API 관리
  * server/ocr-api: OCR 처리 서비스
  * infra: DB와 서비스 실행 환경
  * test-lab: OCR 엔진 성능 평가

* 기술적 제약 및 설계 원칙
  * 각 레이어별 책임 분리
  * Controller → Service → Repository 구조 유지
  * Backend와 OCR API 간 명확한 API 계약 준수
  * DTO 기반 데이터 변환
  * 도커 환경에서 서비스 통합 배포

## 2.1 Applied Skill Stacks

| **Category**              | **Proposed Technology Stack**                         |
| ------------------------- | ----------------------------------------------------- |
| **Programing Language**   | JavaScript, Python (`v3.12.10`), Go (`v1.27.0`), Java |
| **Other Script Language** | Shell, YML                                            |
| **Database**              | MySQL (Docker Image with v8.4.11-oracle)              |
| **Infra & Delivery**      | Docker (v4.93.0), Kubernetes (v1.36.1)                |
| **Website Type**          | Dynamic Website (React with Vite)                     |

## 2.2 Develop Directory Struct

<details>
  
```md
projects/                         
│
├─ web/                            [FE 담당]
│   ├─ package.json
│   ├─ vite.config.js
│   └─ src/
│       ├─ api/                    # axios 인스턴스, API 호출 함수
│       ├─ pages/                  # Login, Signup, Upload, DocumentList, DocumentDetail
│       ├─ components/
│       └─ App.jsx
│
├─ server/                         [BE 공용 레포, 폴더별 담당 분리]
│   ├─ backend/                    ★ Spring Boot
│   │   ├─ build.gradle
│   │   ├─ http/api.http
│   │   └─ src/
│   │       ├─ main/java/com/example/ocr/   ← 아래에서 상세
│   │       ├─ main/resources/application*.yml`
│   │       └─ test/`
│   ├─ ocr-api/                    [BE-B, BE-C 담당] (FastAPI)
│   │   ├─ Dockerfile
│   │   ├─ requirements.txt
│   │   └─ app/
│   │       ├─ main.py
│   │       ├─ routers/            # /health, /ocr/engines, /ocr/extract
│   │       ├─ services/           # 엔진 선택(AUTO), 망 모드 차단
│   │       ├─ engines/            # base, pymupdf, paddle, tesseract, ollama, openai
│   │       ├─ preprocess/
│   │       └─ schemas.py
│   ├─ .gitignore                  # Spring + FastAPI 공통
│   └─ README.md
│
├─ infra/                          [환경/인프라]
│   ├─ docker-compose.yml          # mysql, ocr-api, ollama(profile)
│   ├─ .env.example
│   ├─ db/schema.sql
│   └─ README.md                   # "같은 부모 폴더에 clone" 안내
│
├─ document/                       [문서 전체 공유]
│   ├─ api/
│   │   - signin-api.md
│   │   - register-api.md
│   │   - FastAPI.md
│   │   - OCR-api.md
│   │   - docker-api.md
│   ├─ 02-db-schema.md
│   ├─ 03-schedule.md
│   ├─ 04-spring-guide.md
│   └─ presentation/               # 발표 자료
│
└─ test-lab/                       [OCR 평가, BE-C 중심]
    ├─ samples/                    # 샘플 PDF
    ├─ ground_truth/               # 정답 텍스트
    ├─ scripts/                    # CER/WER, 엔진 비교
    └─ results/                    # 비교표, CSV

### 데이터 흐름과 레포 경계 

[web]  ──HTTP/JSON──>  [server/backend]  ──HTTP/multipart──>  [server/ocr-api]
 React                  Spring Boot                             FastAPI
                         │                                       │
                         ▼                                       ▼
                       MySQL                              Ollama / OpenAI
                      [infra]                              (엔진 어댑터)

계약:   web ↔ backend        = document/01-api-spec.md §1~3
        backend ↔ ocr-api    = document/01-api-spec.md §4
        backend ↔ DB         = document/02-db-schema.md + infra/db/schema.sql
평가:   test-lab → ocr-api (직접 호출) 또는 backend 경유

### 계층 규칙
Controller ──> Service ──> Repository ──> DB
                  ├──────> OcrClient ──> FastAPI
                  └──────> Storage
(위→아래만. 경계마다 DTO로 변환)
```
</details>

---

# ETC. 참조

* [1. Markdown Content Element Style](https://markdific.com/)
