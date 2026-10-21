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

<div align="center">
    <img width="40%" alt="" src="./img/miniproject-requirement.jpg">
</div>

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
