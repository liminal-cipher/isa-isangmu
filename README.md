# 이사이상무

> AI 기반 이사 도우미. 계약 전 위험 점검부터 이사 후 행정 처리까지 이사 여정 전체를 한 곳에서.

![React Native](https://img.shields.io/badge/React%20Native-Expo-61DAFB?logo=expo&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-Backend-009688?logo=fastapi&logoColor=white)
![Azure AI Search](https://img.shields.io/badge/Azure%20AI%20Search-3--index%20RAG-0078D4?logo=microsoftazure&logoColor=white)
![Azure OpenAI](https://img.shields.io/badge/Azure%20OpenAI-GPT--4o-412991)

Microsoft AI School 9기 2차 프로젝트 · 팀 이문세 (6인) · 2026.04.10 ~ 2026.04.27

[발표 자료](docs/presentation.pdf)

## Motivation

이사는 수많은 행정 처리의 연쇄다. 전입신고·확정일자, 공과금 명의변경, 인터넷 이전, 금융 주소변경, 우편물 전송이 기본이고, 조건에 따라 학교 전학·자동차 변경등록·반려동물 등록사항 변경이 추가된다. 상당수는 법적 기한이 있고, 놓치면 과태료가 붙는다.

문제는 **무엇을 해야 하는지가 사람마다 다르다는 것**이다. 자취냐 가족이냐, 월세냐 전세냐, 반려동물이 있느냐, 자녀가 있느냐에 따라 챙겨야 할 절차가 다르다.

기존 서비스는 실행은 있고 안내는 없는 구조다. 정부24·KT무빙 등은 신고를 처리해주지만 어떤 신고가 필요한지 알려주지 않는다. 블로그·LLM은 안내를 시도하지만 개인 맞춤이 부족하거나 출처가 없다. 이사이상무는 이 맞춤형 안내의 공백을 메우는 도구다. 실행은 검증된 정부·기관 도구로 연결한다.

## What It Does

**계약 전 위험 점검부터 이사 후 행정 처리까지** 이사 여정 전체를 한 곳에서 안내하는 모바일 서비스다. 개인 조건(자취/가족, 월세/전세, 반려동물, 자녀 등)에 따라 절차가 달라지는 한국 이사 행정의 복잡성을 **법령·공공 데이터에 근거한 맞춤형 안내**로 다룬다.

- React Native (Expo) 모바일 앱
- AI는 필요한 자리에만 두는, 결정성을 살리는 하이브리드 설계
- 비로그인 + 업로드 즉시 삭제

### 1. 이사 체크리스트 (핵심)

조건을 입력하면 D-day 기준 타임라인을 만들어준다. 체크리스트 파이프라인은 프로젝트 중 두 단계로 발전했다.

- **2026-04-20 Golden Query 30건 평가 시점**: `LLM query planning → RAG/hybrid search → LLM checklist structuring → deterministic Python post-processing`. RAG+LLM으로 초안을 만들고 정형 조건과 필수 항목은 Python 규칙으로 보정했다.
- **2026-04-21 later static optimization**: `free_text`가 없으면 LLM/search를 건너뛰는 static/rule path를 사용하고, `free_text`가 있을 때만 LLM path를 유지했다.
- 각 항목은 법적 기한 기준 D-day로 정렬하고, 공휴일이면 다음 평일로 자동 조정했다.
- 항목별 상세: 설명·신청 방법·연락처·법적 근거(외부 링크)·내 메모(로컬 저장)

<br>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture/checklist-dark.png">
  <img src="docs/architecture/checklist-light.png" alt="체크리스트 아키텍처" width="100%">
</picture>

<br>

### 2. 등기부등본 해석기

PDF를 올리면 위험 요소를 분석해 위험·주의·안전 등급으로 분류한다. **추출·판정·해석을 분리**해 LLM이 최종 판정권을 갖지 않도록 했다.

- **추출**: `prebuilt-layout → GPT-4o Structured Output` 경로와 Azure Document Intelligence **Custom Neural** 구조화 추출을 병렬 실행
- **결합**: Custom Neural의 고신뢰 식별·수치 필드는 confidence gate 후 보강하고, 신탁·경매·가처분 등 위험 플래그는 OR merge
- **판정**: Python 룰 엔진 (공개된 임계값) → LLM이 판정 권한을 갖지 않음
- **해석**: GPT-4o → 사용자가 이해할 수 있게 풀어서 설명
- Custom Neural 호출 실패 시 Layout+LLM 경로로 fallback
- 종합 점수는 의도적으로 만들지 않음 → 가중치 책임을 사용자에게 떠넘기지 않기 위해

최종 Custom Neural 통합 코드는 [`feat/3-index-rag-transition`](https://github.com/liminal-cipher/isa-isangmu/tree/feat/3-index-rag-transition)에 보존되어 있다. 이 브랜치에서 `AZURE_DOCINTEL_CUSTOM_MODEL_ID`, `_extract_custom_fields()`, confidence 기반 merge 로직을 확인할 수 있다.

<br>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture/registry-dark.png">
  <img src="docs/architecture/registry-light.png" alt="등기부등본 아키텍처" width="100%">
</picture>

<br>

### 3. 꽉꽉봇 (RAG 챗봇)

이사·전월세 관련 자유 질문에 답한다. **3개 인덱스 병렬 호출**로 법령·해설·기관 정보를 종합한다.

- 3-index 통합 RAG: `law-index` (법령) · `guide-index` (해설) · `mapping-index` (기관 매핑)
- 음성 입력 지원 (Azure Speech)
- 인라인 출처 토큰 + 하단 출처 목록 이중 노출
- 도메인 외 질문은 LLM 호출 전 사전 필터링

<br>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture/chatbot-dark.png">
  <img src="docs/architecture/chatbot-light.png" alt="꽉꽉봇 아키텍처" width="100%">
</picture>

<br>

## Architecture

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture/overview-dark.png">
  <img src="docs/architecture/overview-light.png" alt="시스템 아키텍처" width="100%">
</picture>

### 데이터 전처리

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture/data-preprocessing-dark.png">
  <img src="docs/architecture/data-preprocessing-light.png" alt="데이터 전처리 파이프라인" width="100%">
</picture>

데이터는 법제처 API · easylaw · 정부 기관 가이드 · 공공데이터포털 등 공식·공공 출처를 기반으로 수집했으며, 사용 시 각 원 출처의 이용조건과 출처 표시 요건을 따른다.

### 스택

| 계층 | 구성 |
| --- | --- |
| 프론트엔드 | React Native (Expo) |
| 백엔드 | FastAPI (Python) |
| AI·검색 | Azure OpenAI (GPT-4o · text-embedding-3-small) · Azure AI Search |
| 문서 처리 | Azure Document Intelligence (Custom Neural + Layout) |
| 음성 | Azure Speech (STT) |
| 저장 | Azure Blob Storage |

## Tech Decisions

| 영역 | 선택 | 이유 |
| --- | --- | --- |
| 인덱스 구조 | **3-index 분리** (단일 통합 대신) | 글을 찾아 읽는 데이터(법령·해설)와 정확한 값을 그대로 꺼내 쓰는 데이터(기관 연락처)는 성격이 달랐다. 데이터 종류별로 자체 인덱스를 갖게 하니 환각 없이 정확한 연락처를 제공하고, 기능마다 필요한 인덱스만 호출할 수 있게 됐다 |
| 체크리스트 파이프라인 | **RAG+LLM + Python post-processing**, 이후 static fast path | 4/20 평가 시점에는 LLM query planning·hybrid search·LLM structuring 뒤 Python 규칙으로 정형 조건과 필수 항목을 보정했다. 4/21에는 no-`free_text` 요청에서 LLM/search를 건너뛰는 static/rule path를 추가했다 |
| 등기부 추출 | **Layout+LLM + Custom Neural 병렬 보강** | Layout+LLM 경로를 baseline/fallback으로 유지하고, Custom Neural의 고신뢰 필드와 위험 플래그를 confidence 기반으로 merge해 양식 변형에 대응했다 |
| 등기부등본 판정 | **Python 룰 엔진** (LLM 판정 대신) | 위험·주의·안전 등급은 공개된 임계값으로 정할 수 있다. LLM에 판정 권한을 주면 근거를 설명할 수 없고 같은 문서에 다른 답이 나온다. LLM은 결과를 풀어 설명하는 자리에만 둔다 |
| 종합 위험 점수 | **만들지 않음** | 점수를 내려면 근저당 규모와 지역 특성의 상대적 위험도에 가중치를 정해야 하는데, 그 근거가 없었다. 임의 가중치로 한 숫자를 만들면 그 판단의 책임이 사용자에게 넘어간다 |

## Results & Limitations

체크리스트 평가는 **서로 다른 두 stage**를 구분해 본다.

- **2026-04-20 · Golden Query 30건**: `mean_recall 0.969`, **26/30 scenarios perfect recall**, **predefined must-not violations 0건**. 당시 구조는 `LLM query planning → RAG/hybrid search → LLM checklist structuring → deterministic Python post-processing`였다. citation coverage와 deadline accuracy는 별도 지표이며 완벽하지 않았다.
- **2026-04-21 · later static optimization, 60건**: `free_text`가 없으면 LLM/search를 건너뛰는 static/rule path를 사용하고, `free_text`가 있을 때만 LLM path를 유지했다. 이 stage의 결과는 **recall 0.963, predefined must-not violations 0건**이다.
- `0.434 → 0.969 (v1 → v4)`는 retrieval·prompt·context·post-processing 등이 함께 바뀐 **전체 시스템 개선**이다. 특정 규칙이나 인덱스 변경 하나의 효과로 귀속하지 않는다. 현재 upstream 기록에는 v2/v3의 정확한 recall 값이 남아 있지 않다.
- **predefined must-not violations 0건은 모든 종류의 hallucination이 0이었다는 뜻이 아니다.** 그래서 문서에서는 `hallucination 0` 대신 해당 평가 지표명을 그대로 쓴다.
- **정량 평가는 있었지만 아키텍처 ablation은 아니었다.** unified index와 3-index를 같은 조건에서 A/B 비교하지는 않았다.
- **최종 구현과 team `main`은 다르다.** 3-index 전환과 Custom Neural 병렬 통합 코드는 feature branch에 보존됐지만 프로젝트 종료 전 team `main`에 머지되지 않았다. 따라서 `main`만 보면 unified-index·`prebuilt-layout` 중심의 이전 경로가 보인다.
- **Azure AI Search 인덱스는 코드로 재생성되지 않는다.** 최종 law·guide·mapping 인덱스는 포털에서 직접 구성하고 데이터를 올렸기 때문에, repo를 clone해도 인덱스 자체는 따라오지 않는다. 인프라를 코드로 관리하지 않은 상태이고, 지금은 구독 접근이 끊겨 원본 구성을 다시 확인할 수도 없다.
- **등기부등본 추출은 학습된 Custom Neural 모델에 의존한다.** feature branch에는 model ID 주입과 merge 로직이 남아 있지만 학습된 Azure 모델과 라벨링 데이터 자체는 repo에 없다.
- **종합 위험 점수를 만들지 않은 것은 의도된 한계다.** 가중치를 정하는 순간 그 판단의 책임이 서비스로 넘어오는데, 근저당 규모와 지역 특성의 상대적 위험도를 근거 있게 정할 수 없었다.

## Getting Started

**현재 `main` 실행**: `.env.example`의 키와 엔드포인트를 채우면 이전 unified-index / Layout 중심 경로를 기준으로 실행할 수 있다.

**최종 프로젝트 구성 재현**: AI Search(`law-index`·`guide-index`·`mapping-index`), OpenAI(GPT-4o, text-embedding-3-small), Document Intelligence Custom Neural 학습 모델, Speech(STT), Blob Storage가 필요하다. Custom Neural 통합 코드는 `feat/3-index-rag-transition`의 `AZURE_DOCINTEL_CUSTOM_MODEL_ID` 설정과 `safecontract_service.py`에 남아 있다.

```bash
# 백엔드
docker compose up

# 모바일 앱
cd frontend && npm install && npx expo start
```

최종 AI Search 인덱스와 학습된 Custom Neural 모델은 repo에 포함되지 않는다. 따라서 feature branch 코드만 checkout해도 최종 Azure 환경이 자동으로 복원되지는 않는다.

## Responsible AI

인공지능 6대 원칙(책임성·투명성·공정성·신뢰성·개인정보·포용성)을 평가 기준으로 다섯 가지 실천을 적용했다: 개인정보 최소 수집, 종합 점수 미제공, 위험 분류 룰 공개, 출처 공개, 도메인 한정.

## Team & Contributions

데이터 수집·전처리, Azure Document Intelligence 학습 데이터 라벨링, 앱 테스트, AI 윤리 6대 원칙 정리, 발표는 6인 전원이 함께 했다. 아래는 그 외에 각자 맡은 일이다.

| 이름 | GitHub | 담당 |
| --- | --- | --- |
| **조윤재** (Team Lead) | [@liminal-cipher](https://github.com/liminal-cipher) | 기획서 · 시스템 아키텍처와 RAG 파이프라인 설계 · Azure AI Search 인덱스 스키마 설계와 3-index 구축 · 멀티 인덱스 전환 프로토타입 구현 · 등기부등본 테스트 시나리오 설계 · 시스템 아키텍처 발표 |
| **김시언** | [@happybluebird](https://github.com/happybluebird) | 발표 슬라이드 구성 · UI 제작과 디자인 · 도메인 리서치 · 문제 정의와 서비스 소개 발표 |
| **노지현** | [@Jihyun-KR](https://github.com/Jihyun-KR) | 시연 영상 기획·촬영·편집 · 이용자 가상 시나리오 설계 · Azure AI Search 통합 인덱스 구현 · 시연 파트 발표 |
| **이승아** | [@wes0031-rgb](https://github.com/wes0031-rgb) | React Native 앱 개발 · Azure 서비스 연동(AI Search · OpenAI · Document Intelligence) · 도메인 리서치 · 성과와 한계 발표 |
| **이재모** | [@imjml](https://github.com/imjml) | Azure Speech Studio 커스텀 음성 모델 학습 · 도메인 리서치 · 발표 구성 검토 · 데이터 파이프라인 발표 |
| **이정우** | [@jwoo9711-rgb](https://github.com/jwoo9711-rgb) | 도메인 리서치 · STT 기능 도입 제안 · 발표 구성 검토 · 수행 과정과 주요 의사결정 발표 |

## My Role (조윤재)

| 담당 | 산출물 |
| --- | --- |
| 아키텍처·RAG 설계 | 시스템 아키텍처와 최종 `law` · `guide` · `mapping` 3-index RAG 구조 설계, 기획서 작성 |
| 멀티 인덱스 검색 프로토타입 | [`feat/3-index-rag-transition`](https://github.com/liminal-cipher/isa-isangmu/tree/feat/3-index-rag-transition)의 [`search_service.py`](https://github.com/liminal-cipher/isa-isangmu/blob/feat/3-index-rag-transition/backend/app/search_service.py)에서 law · guide · video 병렬 hybrid search 서비스와 `/chat` 연동을 구현 |
| mapping-index 구축 | [`feat/4-index-rag-transition-mapping`](https://github.com/liminal-cipher/isa-isangmu/tree/feat/4-index-rag-transition-mapping)에서 mapping 전용 스키마·Azure 리소스와 122개 청크를 구축 |
| 등기부등본 검증 | 위험·주의·안전 판정을 확인하는 테스트 시나리오 설계 |

> 멀티 인덱스 전환과 Custom Neural 통합안은 팀 `main`의 병행 개발 경로와 합쳐지지 않은 채 프로젝트가 끝나 최종 `main`에는 머지되지 않았다. 위 feature branch에 전환 코드와 검증 흔적이 남아 있다.

## Retrospective

**인덱스를 포털에서 만든 것이 가장 아쉽다.** 2주 일정에서는 그게 빨랐지만, 그 결정 때문에 지금 이 repo만으로는 챗봇을 되살릴 수 없다. 다시 한다면 인덱스 스키마와 적재 스크립트를 코드로 먼저 두고 포털은 확인용으로만 썼을 것이다.

**체크리스트는 한 번에 static/rule 구조로 간 것이 아니었다.** 4/20 평가 stage에서는 query planning과 checklist structuring에 LLM을 쓰고 Python 후처리로 정형 조건과 필수 항목을 보정했다. 그 다음날 no-`free_text` 요청을 static/rule path로 우회하는 최적화를 추가했다. 결정성이 필요한 영역을 더 일찍 분리했다면 호출 비용과 재작업을 줄일 수 있었을 것이다.

**정량 평가는 했지만 아키텍처 ablation이 없었다.** 4/20 30Q에서는 `mean_recall 0.969`와 predefined must-not violations 0건, 4/21 60Q에서는 `recall 0.963`과 predefined must-not violations 0건을 확인했다. 하지만 `0.434 → 0.969` 동안 여러 요소가 함께 바뀌었고 unified index와 3-index도 같은 조건에서 직접 비교하지 않았다. 다시 한다면 동일한 정답셋으로 각 변경을 분리해 ablation/A-B 평가할 것이다.

## Status

완료. Microsoft AI School 9기 2차 프로젝트로 2026.04.10 ~ 2026.04.27 진행. Azure 구독 접근이 끊겨 현재는 실행할 수 없고, 코드와 발표 자료만 남아 있다. 마지막 갱신 2026-09-21.
