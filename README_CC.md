# 📰 동AI일보 (DongAI Ilbo)

> **한국 경제부 기자를 위한 실데이터 기반 AI 기사작성 어시스턴트**
> FastAPI · React 19 · ChromaDB · OpenAI GPT-4o · 7개 공공·금융 API RAG 파이프라인

<!-- 이미지: 메인 대시보드 스크린샷 또는 대표 이미지 (선택) -->

<br>

## 📌 목차

- [프로젝트 소개](#-프로젝트-소개)
- [기획 배경](#-기획-배경)
- [기술 스택](#️-기술-스택)
- [핵심 기능](#-핵심-기능)
- [시스템 아키텍처](#️-시스템-아키텍처)
- [실행 화면](#-실행-화면)
- [기술적 의사결정](#-기술적-의사결정)
- [프로젝트 구조](#-프로젝트-구조)
- [데이터베이스 구조](#️-데이터베이스-구조)
- [실행 방법](#-실행-방법)
- [발표 자료](#-발표-자료)
- [팀 & 기여](#-팀--기여)

<br>

---

## 📖 프로젝트 소개

동AI일보는 단순 GPT 래퍼가 아닙니다.

**한국은행 ECOS · 통계청 KOSIS · 금융감독원 DART · 한국거래소 · 미국 FRED · 미국 EIA**의 공공·금융 데이터를 실시간으로 끌어오고, **ChromaDB 기반 RAG**로 경제 용어·작성 규칙·실제 뉴스 맥락을 GPT-4o에 결합해 **근거 있고 출처가 명확한 경제 기사**를 자동 생성하는 기자 전용 워크플로우 도구입니다.

여기에 한국 언론 규정(인터넷신문 기사심의규정·생성형 AI 활용기사 자율심의 준칙·한국기자협회 윤리강령)에 근거한 **규정 인용 방식의 윤리 검증**을 더해, AI 생성 기사가 안고 있는 신뢰성 문제를 정면으로 다뤘습니다.

<br>

---

## 🎯 기획 배경

생성형 AI가 기사를 쓰는 시대에도, 현장 기자가 신뢰하고 쓸 수 있는 도구는 드뭅니다. 기존 AI 글쓰기 도구는 세 가지 한계를 안고 있습니다.

- **출처 없는 수치** — AI가 그럴듯한 숫자를 만들어내지만 근거가 없습니다.
- **맥락 없는 생성** — 최신 경제 상황과 뉴스 흐름을 반영하지 못합니다.
- **검증 불가능한 윤리성** — "이 기사가 언론 규정에 맞는가"를 판단할 방법이 없습니다.

동AI일보는 이 세 가지를 각각 **실시간 공공 데이터 연동**, **RAG 기반 뉴스 맥락 주입**, **규정 인용 윤리 체크리스트**로 해결합니다.

설계의 일관된 기준은 하나입니다 — 기자가 신뢰할 수 있어야 한다는 것. 그래서 AI가 기사에 점수를 매기는 방식 대신, 어느 규정의 어느 조항을 통과하고 미통과했는지를 명시하는 방향을 택했습니다.

<br>

---

## 🛠️ 기술 스택

| 구분 | 기술 | 선택 이유 |
|---|---|---|
| Frontend | React 19 (Vite) | SPA, 빠른 HMR, 컴포넌트 재사용 |
| Backend | FastAPI (Python) | 비동기 처리, 자동 문서화(Swagger), 외부 API 연동 용이 |
| Database | PostgreSQL (asyncpg) | Docker 팀 공유, 동시 접속 안정성 (초기 SQLite에서 전환) |
| Vector DB | ChromaDB | 경량 로컬 벡터 DB, RAG 파이프라인 구현 |
| AI 생성 | OpenAI GPT-4o | 한국어 경제 기사 품질, 긴 컨텍스트 처리 |
| 임베딩 (참고자료) | OpenAI text-embedding-3-small | 검색 정확도 우선 — 경제 용어·규칙·샘플 |
| 임베딩 (뉴스) | ChromaDB 기본 임베딩 (로컬) | 대용량 뉴스 — 비용 절감 위해 로컬 처리 |
| 인증 | bcrypt + JWT + OAuth2(카카오·구글) | 소셜 로그인 + 자체 회원가입 병행 |
| 배포 | Docker (docker-compose) | DB·백엔드·프론트 3개 컨테이너 통합 실행 |
| 프록시 | nginx | React SPA 라우팅 + `/api` → FastAPI 프록시 |

> 임베딩을 두 가지로 나눈 것은 의도된 설계입니다. 정확도가 중요한 참고자료(소량)는 OpenAI 임베딩으로, 검색 빈도 대비 양이 많은 뉴스 데이터(대량)는 로컬 임베딩으로 분리해 품질과 비용을 동시에 확보했습니다.

<br>

---

## ✨ 핵심 기능

### 1. 실데이터 + RAG 기반 기사 자동 생성

주제만 입력하면 완성된 기사(제목·부제·본문)를 생성합니다. 주제 키워드를 분석해 해당하는 실시간 데이터만 선택 조회하고(ECOS·KOSIS·DART), ChromaDB에서 경제 용어·작성 규칙·샘플 기사와 관련 최신 뉴스를 검색해 GPT-4o에 함께 전달합니다.

주제와 무관한 데이터가 섞이지 않도록, 키워드가 있을 때만 그 데이터를 조회하는 것이 핵심입니다.

지원 문체: 보도체 · 해설체 · 심층분석체 · 분석체 · 쉬운말체

### 2. 스마트 인터뷰 어시스턴트

취재 유형(단독취재·공식 브리핑·전문가 논평)에 맞춰 인터뷰 질문 리스트를 생성합니다. 취재원에 기업명이 포함되면 DART 최근 공시를 자동 조회해 질문에 반영합니다.

### 3. 규정 인용 윤리 검증

AI 생성 기사에 점수를 매기지 않습니다. 인터넷신문 기사심의규정·생성형 AI 활용기사 자율심의 준칙·한국기자협회 윤리강령 세 규정의 조항별로 통과/미통과를 명시합니다.

- 수치에 출처가 없으면 → 출처 명시 조항 미통과로 분류하고 수정 방향 제안
- 선정적·과장 표현 감지 → 해당 조항 미통과
- AI 생성 고지문 누락 → 고지문 자동 삽입

경제 저널리즘에서 정상적으로 쓰이는 표현("역대 최고/최저" 등)은 탐지에서 제외해 오탐을 줄였습니다.

### 4. AI 취재 파트너 챗봇

기능 안내 도우미가 아니라 경제부 동료 기자 페르소나로 동작합니다.

- URL을 붙여넣으면 본문을 자동 추출·분석 (최대 8,000자)
- 경제 키워드 감지 시 금리·환율·물가·기업 공시 실시간 조회
- 답변 하단에 참고 뉴스 출처(언론사·날짜·링크) 자동 표시
- 모든 페이지 우측 하단 고정, 전체화면 모드 전환 지원

### 5. 기사 보조 도구

| 기능 | 설명 | 엔드포인트 |
|---|---|---|
| 초안 생성 | 취재 포인트 + 기사 구조 + 제목 후보 3개 | `POST /api/article/draft` |
| 문체 변환 | 수치·사실은 그대로, 문체만 변경 | `POST /api/article/rewrite` |
| 교정 | 맞춤법·문법 검사 및 수정 | `POST /api/article/proofread` |
| 요약 | 보도자료 핵심 불릿 요약 | `POST /api/article/summary` |

### 6. 실시간 경제 데이터 대시보드

7개 공공·금융 데이터 소스를 직접 조회합니다.

| 소스 | 제공 지표 |
|---|---|
| 한국은행 ECOS | 기준금리, 원달러 환율, 위안·엔 환율, 생산자물가(PPI), 국고채 금리, 소비자심리지수(CCSI), GDP 성장률, 주택매매가격지수 |
| 통계청 KOSIS | 소비자물가(CPI), 고용지표(실업률·취업자·경활참가율), 경기선행지수 |
| 금융감독원 DART | 주요 기업 공시 (기업명 → corp_code 매핑) |
| 한국거래소 KRX | 코스피·코스닥 지수, 외국인 순매수 |
| 미국 FRED | 미 연준 기준금리, 달러인덱스(DXY) |
| 미국 EIA | WTI 유가, 브렌트유 유가 |

### 7. 부가 기능

- **STT** — 음성을 텍스트로 변환 (인터뷰 녹취 보조)
- **이미지 생성** — 기사용 이미지 생성
- **트렌드 분석** — 뉴스 트렌드 분석
- **기사 이력** — 작성 기사 저장·조회 (로그인 필수)

<br>

---

## 🏗️ 시스템 아키텍처

```mermaid
flowchart TD
    U["사용자 · React 19 Frontend<br/>localhost 3000 또는 5173"]
    NX["nginx<br/>API 요청은 FastAPI로, 그 외는 React SPA로"]
    BE["FastAPI Backend :8000<br/>article · chat · data · rag · history<br/>auth · stt · image · trend · naver"]

    subgraph RAG["RAG 파이프라인 · ChromaDB"]
        R1["newsai_rag<br/>OpenAI 임베딩<br/>경제용어 · 규칙 · 샘플"]
        R2["newsai_news<br/>로컬 임베딩<br/>실제 경제 뉴스"]
    end

    subgraph API["실시간 데이터 API · 선택 조회"]
        D1["ECOS · KOSIS · DART"]
        D2["KRX · FRED · EIA · 네이버"]
        D3["키워드 감지 시에만 호출"]
    end

    GPT["OpenAI GPT-4o<br/>RAG 자료 + 실데이터 + 문체 규칙으로 기사 생성"]
    ETH["윤리 검증 · 규정 체크리스트"]
    DB[("PostgreSQL<br/>users · articles")]

    U --> NX --> BE
    BE --> RAG
    BE --> API
    RAG --> GPT
    API --> GPT
    GPT --> ETH --> DB
```

<br>

---

## 🎥 실행 화면

### 메인 화면
![메인화면](gif/메인화면.gif)

### AI 기사 생성
주제 입력 → 실시간 데이터 조회 → RAG 결합 → 문체별 기사 자동 생성
![AI 기사 생성](gif/AI%20기사%20생성.gif)

### 스마트 인터뷰 어시스턴트
취재 유형별 맞춤 질문 + 취재원 기업의 DART 공시 자동 반영
![인터뷰 어시스턴트](gif/인터뷰%20어시스턴트.gif)

### 규정 인용 윤리 검증
세 개 언론 규정의 조항별 통과/미통과 체크리스트
![AI 기사 검토](gif/AI%20기사%20검토.gif)

### AI 기사 교정
맞춤법·문법 교정 및 수정 항목 표시
![AI 기사 교정](gif/AI%20기사%20교정.gif)

### AI 취재 파트너 챗봇
URL 자동 분석 + 실시간 데이터 + 참고 뉴스 출처 표시
![챗봇](gif/챗봇.gif)

### 실시간 경제 데이터 조회
7개 공공·금융 소스의 지표를 페이지에서 직접 조회
![데이터조회 페이지](gif/데이터조회%20페이지.gif)

### 네이버 API · 뉴스 데이터 연동
실시간 뉴스 검색 및 데이터 활용
![네이버 api 및 뉴스 데이터](gif/네이버%20api%20및%20뉴스%20데이터.gif)

<br>

<details>
<summary><b>📂 전체 기능 시연 더보기 (클릭)</b></summary>

<br>

### AI 기사 요약 및 취재 초안
![AI 기사 요약 및 취재 초안](gif/AI%20기사%20요약%20및%20취재%20초안.gif)

### AI 기사 문체 변경
![AI 기사 문체 변경](gif/AI%20기사%20문체%20변경.gif)

### AI 기사 보도자료
![AI 기사 보도자료](gif/AI%20기사%20보도자료.gif)

### AI 기사 이미지 생성
![AI 기사 이미지 생성](gif/AI%20기사%20이미지%20생성.gif)

### 기사·보도자료 요약
![기사 보도자료 요약](gif/기사,보도자료%20요약.gif)

### 기사 작성 페이지
![기사작성 페이지](gif/기사작성%20페이지.gif)

### 작성 페이지 데이터 활용 (1)
![작성 페이지 데이터](gif/작성%20페이지%20데이터.gif)

### 작성 페이지 데이터 활용 (2)
![작성 페이지 데이터2](gif/작성%20페이지%20데이터2.gif)

### 교정 페이지
![교정 페이지](gif/교정%20페이지.gif)

### 음성 변환 페이지 (STT)
![음성변환 페이지](gif/음성변환%20페이지.gif)

### 이미지 제작
![이미지 제작](gif/이미지%20제작.gif)

### 챗봇 데이터 조회
![챗봇 데이터조회](gif/챗봇%20데이터조회.gif)

### 기사 저장
![기사 저장](gif/기사%20저장.gif)

### 기사 저장 시 비회원 로그인 이동
![기사 저장 시 비회원 로그인창 이동](gif/기사%20저장%20시%20비회원%20로그인창%20이동.gif)

### 회원가입
![회원가입](gif/회원가입.gif)

### 구글 로그인
![구글 로그인](gif/구글%20로그인.gif)

### 카카오 로그인
![카카오 로그인](gif/카카오%20로그인.gif)

</details>

<br>

---

## 🔧 기술적 의사결정

### SQLite에서 PostgreSQL로 전환

SQLite는 파일 기반이라 Docker 컨테이너를 재시작하면 데이터가 초기화되고, 동시 접속과 비동기(asyncpg) 환경에 취약했습니다.

PostgreSQL로 전환하고 `docker-compose`의 `db` 서비스로 통합해, 팀원이 `docker-compose up` 한 번으로 DB를 포함한 전체 스택을 띄울 수 있게 했습니다. 이 과정에서 Docker 내부 통신을 위해 `DATABASE_URL`의 호스트를 `localhost`가 아닌 서비스명 `db`로 오버라이드하는 처리가 필요했습니다.

### passlib 제거 후 bcrypt 직접 사용

`chromadb`가 요구하는 `bcrypt 4.x`와 비밀번호 암호화에 쓰던 `passlib 1.7.x`가 충돌해 `AttributeError`가 발생했습니다.

두 라이브러리가 한 환경에서 공존할 수 없는 의존성 문제였기에, passlib을 제거하고 bcrypt를 직접 호출하는 방식으로 해결했습니다.

### 임베딩 이원화로 비용과 정확도 양립

참고자료는 양이 적고 검색 정확도가 중요해 OpenAI 임베딩을, 뉴스 데이터는 양이 많아 비용 부담이 커 로컬 임베딩을 적용했습니다. 하나의 ChromaDB 안에 두 컬렉션을 목적에 맞게 분리 운용하는 구조입니다.

### AI 점수제를 규정 인용 체크리스트로 재설계

초기에는 GPT가 기사에 0~100점을 매겼으나, AI가 기자를 점수로 평가하는 방식은 현장의 신뢰를 얻기 어렵다고 판단했습니다.

점수를 버리고 실제 언론 규정의 조항을 인용해 통과/미통과를 보여주는 방식으로 재설계했습니다.

### 챗봇 키워드 오탐 제거

`"기사"`, `"뉴스"` 같은 단일 키워드가 일상적인 질문("기사 어떻게 써요?")에도 뉴스 검색을 발동시켜 관련 없는 결과가 따라붙었습니다.

단일어를 복합어(`"경제 기사"`, `"경제 뉴스"`)로 교체해 경제 맥락이 있을 때만 검색이 실행되도록 조정했습니다.

<br>

---

## 📁 프로젝트 구조

```
TEAMPROJECT_DONGAIILBO/
├── backend/
│   ├── app/
│   │   ├── api/routes/
│   │   │   ├── article.py    기사 생성·초안·문체변환·교정·요약·인터뷰질문
│   │   │   ├── chat.py       AI 취재 파트너 챗봇 (URL읽기 + RAG + 실데이터)
│   │   │   ├── data.py       실시간 경제 데이터 조회 (7개 소스, 20+ 엔드포인트)
│   │   │   ├── rag.py        ChromaDB 인덱싱·검색
│   │   │   ├── history.py    기사 저장·조회 (로그인 필수)
│   │   │   ├── auth.py       카카오·구글 OAuth + 이메일/JWT
│   │   │   ├── stt.py        음성 → 텍스트
│   │   │   ├── image.py      이미지 생성
│   │   │   ├── trend.py      트렌드 분석
│   │   │   └── naver.py      네이버 뉴스 검색
│   │   ├── services/
│   │   │   ├── ai.py             GPT-4o 기사생성·교정·요약·초안·인터뷰
│   │   │   ├── vector_store.py   ChromaDB RAG 핵심 로직 (이원화 임베딩)
│   │   │   ├── ecos.py / kosis.py / dart.py    한국 공공 데이터
│   │   │   ├── krx.py / fred.py / eia.py        증시·미국 지표
│   │   │   └── ethics.py        규정 기반 윤리 검증
│   │   ├── core/        config · database · auth(JWT)
│   │   ├── models/      user · article
│   │   ├── data/        ethics_rules.json (규정 사전)
│   │   └── rag_data/    economy_terms · news_writing_rules · sample_articles (.txt)
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   └── src/
│       ├── pages/       Dashboard · Editor · Data · Trend · History · Chat
│       │                Summary · Proofread · STT · Image · Login · ...
│       ├── components/  Layout(Header·Sidebar) · FloatChat(전역 챗봇)
│       └── api/index.js
├── docker-compose.yml   db + backend + frontend
└── README.md
```

<br>

---

## 🗄️ 데이터베이스 구조

```sql
CREATE TABLE users (
    id            SERIAL PRIMARY KEY,
    email         VARCHAR(255) UNIQUE NOT NULL,
    name          VARCHAR(100),
    password_hash VARCHAR(255),   -- 이메일 가입 시 bcrypt 해시
    provider      VARCHAR(50),    -- 'email' | 'kakao' | 'google'
    provider_id   VARCHAR(255),   -- OAuth 식별자
    created_at    TIMESTAMP DEFAULT NOW()
);

CREATE TABLE articles (
    id         SERIAL PRIMARY KEY,
    user_id    INTEGER REFERENCES users(id),
    title      VARCHAR(500),
    content    TEXT,
    style      VARCHAR(50),       -- '보도체' | '해설체' 등
    type       VARCHAR(50),       -- 'generated' | 'draft' 등
    created_at TIMESTAMP DEFAULT NOW()
);
```

<br>

---

## 🚀 실행 방법

### Docker (권장)

```bash
git clone https://github.com/ponyo911/TEAMPROJECT_DONGAIILBO.git
cd TEAMPROJECT_DONGAIILBO

# backend/.env 파일 생성 후 키 입력 (아래 환경변수 참고)

docker-compose up --build      # 최초 실행
# → http://localhost:3000
```

### 로컬 개발 환경

```bash
# 백엔드
cd backend
python -m venv venv
venv\Scripts\activate            # Windows
# source venv/bin/activate       # Mac/Linux
pip install -r requirements.txt
uvicorn app.main:app --reload    # → http://localhost:8000

# 프론트엔드 (별도 터미널)
cd frontend
npm install
npm run dev                      # → http://localhost:5173
```

### RAG 인덱싱 (최초 1회, 서버 실행 후)

```
POST http://localhost:8000/api/rag/index        # 참고자료(.txt) 인덱싱
POST http://localhost:8000/api/rag/index-news   # 뉴스 데이터 인덱싱
```

뉴스 인덱싱에는 뉴스 데이터 파일(.xlsx)이 필요합니다. 용량 문제로 저장소에서 제외되어 있어, 별도로 전달받아 `backend/app/rag_data/`에 위치시켜야 합니다.

### 환경 변수 (`backend/.env`)

```
OPENAI_API_KEY=
ECOS_API_KEY=
KOSIS_API_KEY=
DART_API_KEY=
FRED_API_KEY=
EIA_API_KEY=
NAVER_CLIENT_ID=
NAVER_CLIENT_SECRET=
KAKAO_CLIENT_ID=
KAKAO_REDIRECT_URI=http://localhost:3000/auth/kakao
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=http://localhost:3000/auth/google
SECRET_KEY=
DATABASE_URL=postgresql+asyncpg://postgres:비밀번호@localhost:5432/newsai
```

`.env`는 `.gitignore`로 제외되어 저장소에 올라가지 않습니다. `SECRET_KEY`에는 추측 불가능한 랜덤 문자열을 사용합니다.

<br>

---

## 📊 발표 자료

<br>

![슬라이드1](slides/슬라이드1.PNG)
![슬라이드2](slides/슬라이드2.PNG)
![슬라이드3](slides/슬라이드3.PNG)
![슬라이드4](slides/슬라이드4.PNG)
![슬라이드5](slides/슬라이드5.PNG)
![슬라이드6](slides/슬라이드6.PNG)
![슬라이드7](slides/슬라이드7.PNG)
![슬라이드8](slides/슬라이드8.PNG)
![슬라이드9](slides/슬라이드9.PNG)
![슬라이드10](slides/슬라이드10.PNG)
![슬라이드11](slides/슬라이드11.PNG)
![슬라이드12](slides/슬라이드12.PNG)
![슬라이드13](slides/슬라이드13.PNG)
![슬라이드14](slides/슬라이드14.PNG)
![슬라이드15](slides/슬라이드15.PNG)
![슬라이드16](slides/슬라이드16.PNG)
![슬라이드17](slides/슬라이드17.PNG)
![슬라이드18](slides/슬라이드18.PNG)
![슬라이드19](slides/슬라이드19.PNG)
![슬라이드20](slides/슬라이드20.PNG)
![슬라이드21](slides/슬라이드21.PNG)
![슬라이드22](slides/슬라이드22.PNG)
![슬라이드23](slides/슬라이드23.PNG)
![슬라이드24](slides/슬라이드24.PNG)
![슬라이드25](slides/슬라이드25.PNG)
![슬라이드26](slides/슬라이드26.PNG)
![슬라이드27](slides/슬라이드27.PNG)
![슬라이드28](slides/슬라이드28.PNG)
![슬라이드29](slides/슬라이드29.PNG)
![슬라이드30](slides/슬라이드30.PNG)
![슬라이드31](slides/슬라이드31.PNG)
![슬라이드32](slides/슬라이드32.PNG)
![슬라이드33](slides/슬라이드33.PNG)
![슬라이드34](slides/슬라이드34.PNG)
![슬라이드35](slides/슬라이드35.PNG)
![슬라이드36](slides/슬라이드36.PNG)
![슬라이드37](slides/슬라이드37.PNG)
![슬라이드38](slides/슬라이드38.PNG)
![슬라이드39](slides/슬라이드39.PNG)
![슬라이드40](slides/슬라이드40.PNG)
![슬라이드41](slides/슬라이드41.PNG)
![슬라이드42](slides/슬라이드42.PNG)
![슬라이드43](slides/슬라이드43.PNG)
![슬라이드44](slides/슬라이드44.PNG)
![슬라이드45](slides/슬라이드45.PNG)
![슬라이드46](slides/슬라이드46.PNG)
![슬라이드47](slides/슬라이드47.PNG)
![슬라이드48](slides/슬라이드48.PNG)


---

## 👥 팀 & 기여

동아일보 AI 마스터클래스 1조 팀 프로젝트입니다.

### 단독 설계·구현

**외부 데이터 연동 (7개 공공·금융 API)**
- ECOS·KOSIS·DART·KRX·FRED·EIA·네이버 7개 API 연동 및 기관별 응답 포맷 정규화
- 기업명 → `corp_code` 매핑으로 자연어 주제에서 기업을 감지해 DART 공시를 자동 조회
- 주제 키워드를 분석해 관련 데이터만 선택 조회하는 로직 (무관한 데이터 혼입 방지)
- 통계 코드 탐색용 디버그 엔드포인트 구축 및 API 키 마스킹 처리

**규정 인용 윤리 검증 시스템**
- 한국 언론 3대 규정(인터넷신문 기사심의규정·생성형 AI 활용기사 자율심의 준칙·한국기자협회 윤리강령) 기반 검증 로직 설계·구현
- AI 점수제를 규정 조항별 통과/미통과 체크리스트로 재설계
- 규정 사전을 `ethics_rules.json`으로 외부화해 규정 개정에 유연하게 대응
- 필수/권고 조항 등급 구분, 정규식 기반 수치-출처 누락 탐지

**RAG 임베딩 이원화 설계**
- 참고자료는 OpenAI 임베딩, 대용량 뉴스는 로컬 임베딩으로 분리해 정확도와 비용을 동시에 확보
- ChromaDB 2개 컬렉션 분리 운용 및 재인덱싱 중복 방지 로직

**인증 시스템**
- 이메일 가입(bcrypt + JWT)과 카카오·구글 OAuth를 `provider` 필드로 통합 관리하는 로직 설계

**챗봇 핵심 아이디어**
- 경제 키워드 화이트리스트 기반 선택적 RAG·실데이터 발동 (불필요한 호출·오탐 제거)
- 답변에 참고 뉴스 출처(언론사·날짜·링크)를 함께 반환하는 추적 가능성 설계

### 팀 공동 작업

이 프로젝트가 실제 언론 현장에 맞도록 만든 기반 작업은 팀 전체가 함께 수행했습니다.

- **현직 경제기자 인터뷰** — 실제 기자의 업무 흐름과 요구사항을 직접 청취해 기능 방향에 반영
- **서비스 전체 기획** — 도구가 풀어야 할 문제 정의와 기능 범위 설계
- **UI/UX 디자인 기획** — 기자 사용 시나리오 기반의 화면 흐름 설계
- **경제 뉴스 데이터 수집 (약 2만 건)** — RAG 학습용 데이터 확보 및 분류

발표 자료 제작과 발표는 팀원이 담당했습니다.

### 역할 요약

- **본인** — 백엔드·AI·데이터 연동·RAG·윤리 검증·인증 등 시스템 개발 전담, 핵심 아키텍처 설계
- **팀 공동** — 현직 기자 인터뷰, 서비스·UI/UX 기획, 2만 건 데이터 수집
- **팀원** — 발표 자료 및 발표

<br>

---

## 📄 License

This project was created for educational purposes as part of the DongA AI Masterclass.
