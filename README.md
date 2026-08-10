# 이사이상무

> AI 기반 이사 도우미. 계약 전 위험 점검부터 이사 후 행정 처리까지 이사 여정 전체를 한 곳에서.

![React Native](https://img.shields.io/badge/React%20Native-Expo-61DAFB?logo=expo&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-Backend-009688?logo=fastapi&logoColor=white)
![Azure AI Search](https://img.shields.io/badge/Azure%20AI%20Search-3--index%20RAG-0078D4?logo=microsoftazure&logoColor=white)
![Azure OpenAI](https://img.shields.io/badge/Azure%20OpenAI-GPT--4o-412991)

Microsoft AI School 9기 2차 프로젝트 · 팀 이문세 (6인) · 2026.04.13 ~ 04.26

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

조건을 입력하면 D-day 기준 타임라인을 만들어준다. **하이브리드 구조**로 결정성과 유연성을 동시에 확보했다.

- 정형 조건(토글)은 정적 쿼리 매핑으로 처리 → 같은 입력 → 같은 결과
- 자유 텍스트("기타 특이사항")는 LLM으로 처리 → 토글로 잡히지 않는 엣지 케이스 대응
- 각 항목은 법적 기한 기준 D-day로 정렬, 공휴일이면 다음 평일로 자동 조정
- 항목별 상세: 설명·신청 방법·연락처·법적 근거(외부 링크)·내 메모(로컬 저장)

<br>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/architecture/checklist-dark.png">
  <img src="docs/architecture/checklist-light.png" alt="체크리스트 아키텍처" width="100%">
</picture>

<br>


### 2. 등기부등본 해석기

PDF를 올리면 위험 요소를 분석해 위험·주의·안전 등급으로 분류한다. **추출·판정·해석 3단 분리**로 환각을 구조적으로 차단했다.

- **추출**: Azure Document Intelligence (Custom Neural: 양식 변형에 강건한 모델로 학습) → 주소·면적·소유자·근저당·지역
- **판정**: Python 룰 엔진 (공개된 임계값) → LLM이 판정 권한을 갖지 않음
- **해석**: GPT-4o → 사용자가 이해할 수 있게 풀어서 설명
- 종합 점수는 의도적으로 만들지 않음 → 가중치 책임을 사용자에게 떠넘기지 않기 위해

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

데이터는 법제처 API · easylaw · 정부 기관 가이드 · 공공데이터포털 기반이며 모두 **공공누리 제1유형**(출처 표시) 라이선스를 따른다.

### 스택

| 계층 | 구성 |
| --- | --- |
| 프론트엔드 | React Native (Expo) |
| 백엔드 | FastAPI (Python) |
| AI·검색 | Azure OpenAI (GPT-4o · text-embedding-3-small) · Azure AI Search |
| 문서 처리 | Azure Document Intelligence (Custom Neural) |
| 음성 | Azure Speech (STT) |
| 저장 | Azure Blob Storage |

## Tech Decisions

| 영역 | 선택 | 이유 |
| --- | --- | --- |
| 인덱스 구조 | **3-index 분리** (단일 통합 대신) | 글을 찾아 읽는 데이터(법령·해설)와 정확한 값을 그대로 꺼내 쓰는 데이터(기관 연락처)는 성격이 달랐다. 데이터 종류별로 자체 인덱스를 갖게 하니 환각 없이 정확한 연락처를 제공하고, 기능마다 필요한 인덱스만 호출할 수 있게 됐다 |
| 체크리스트 파이프라인 | **하이브리드** (단일 LLM 대신) | 처음에는 모든 조건을 LLM에 넘겼는데 같은 입력에도 매번 다른 출력이 나왔다. 토글 조건은 옵션이 정해져 있어 LLM이 필요 없다. 정형 조건은 정적 매핑, 자유 텍스트만 LLM으로 분리해 결정성과 호출 절감을 동시에 얻었다 |
| 등기부등본 판정 | **Python 룰 엔진** (LLM 판정 대신) | 위험·주의·안전 등급은 공개된 임계값으로 정할 수 있다. LLM에 판정 권한을 주면 근거를 설명할 수 없고 같은 문서에 다른 답이 나온다. LLM은 결과를 풀어 설명하는 자리에만 둔다 |
| 종합 위험 점수 | **만들지 않음** | 점수를 내려면 근저당 규모와 지역 특성의 상대적 위험도에 가중치를 정해야 하는데, 그 근거가 없었다. 임의 가중치로 한 숫자를 만들면 그 판단의 책임이 사용자에게 넘어간다 |

## Results & Limitations

**정량 지표는 측정하지 않았다.** 체크리스트 정확도, 등기부등본 추출 정확도, 챗봇 답변 품질 어느 것도 정답셋을 만들어 재지 않았다. 2주 일정에서 기능 완성을 우선했고, 검증은 팀이 만든 시나리오를 손으로 돌려보는 수준이었다.

- **Azure AI Search 인덱스는 코드로 재생성되지 않는다.** 3개 인덱스를 포털에서 직접 구성하고 데이터를 올렸기 때문에, repo를 clone해도 인덱스는 따라오지 않는다. 인프라를 코드로 관리하지 않은 상태이고, 지금은 구독 접근이 끊겨 원본 구성을 다시 확인할 수도 없다.
- **등기부등본 추출은 Custom Neural 학습 모델에 묶여 있다.** 학습한 모델이 없으면 해당 기능이 동작하지 않는다. 학습 데이터는 팀이 라벨링한 것으로 repo에 없다.
- **종합 위험 점수를 만들지 않은 것은 의도된 한계다.** 가중치를 정하는 순간 그 판단의 책임이 서비스로 넘어오는데, 근저당 규모와 지역 특성의 상대적 위험도를 근거 있게 정할 수 없었다.
- 체크리스트의 자유 텍스트 경로는 LLM을 쓰므로 같은 입력에 같은 출력을 보장하지 않는다. 토글 경로만 결정적이다.

## Getting Started

**필요한 Azure 리소스**: AI Search(`law-index`·`guide-index`·`mapping-index`), OpenAI(GPT-4o, text-embedding-3-small), Document Intelligence(등기부등본 Custom Neural 학습 모델), Speech(STT), Blob Storage. 키와 엔드포인트는 `.env.example`을 복사해 채운다.

```bash
# 백엔드
docker compose up

# 모바일 앱
cd frontend && npm install && npx expo start
```

위 Limitations에 적었듯 인덱스와 커스텀 모델은 이 repo에 포함되지 않는다. 인덱스 스키마를 새로 만들고 데이터를 적재해야 챗봇이 동작한다.

## Responsible AI

인공지능 6대 원칙(책임성·투명성·공정성·신뢰성·개인정보·포용성)을 평가 기준으로 다섯 가지 실천을 적용했다: 개인정보 최소 수집, 종합 점수 미제공, 위험 분류 룰 공개, 출처 공개, 도메인 한정.

## Team & Contributions

데이터 수집·전처리, Azure Document Intelligence 학습 데이터 라벨링, 앱 테스트, AI 윤리 6대 원칙 정리, 발표는 6인 전원이 함께 했다. 아래는 그 외에 각자 맡은 일이다.

| 이름 | GitHub | 담당 |
| --- | --- | --- |
| **조윤재** (팀 리드) | [@liminal-cipher](https://github.com/liminal-cipher) | 기획서 · 시스템 아키텍처와 RAG 파이프라인 설계 · Azure AI Search 인덱스 스키마 설계와 3-index 구축 · 등기부등본 테스트 시나리오 설계 · 시스템 아키텍처 발표 |
| **김시언** | [@happybluebird](https://github.com/happybluebird) | 발표 슬라이드 구성 · UI 제작과 디자인 · 도메인 리서치 · 문제 정의와 서비스 소개 발표 |
| **노지현** | [@Jihyun-KR](https://github.com/Jihyun-KR) | 시연 영상 기획·촬영·편집 · 이용자 가상 시나리오 설계 · Azure AI Search 통합 인덱스 구현 · 시연 파트 발표 |
| **이승아** | [@wes0031-rgb](https://github.com/wes0031-rgb) | React Native 앱 개발 · Azure 서비스 연동(AI Search · OpenAI · Document Intelligence) · 도메인 리서치 · 성과와 한계 발표 |
| **이재모** | [@imjml](https://github.com/imjml) | Azure Speech Studio 커스텀 음성 모델 학습 · 도메인 리서치 · 발표 구성 검토 · 데이터 파이프라인 발표 |
| **이정우** | [@jwoo9711-rgb](https://github.com/jwoo9711-rgb) | 도메인 리서치 · STT 기능 도입 제안 · 발표 구성 검토 · 수행 과정과 주요 의사결정 발표 |

## My Role (조윤재)

| 담당 | 산출물 |
| --- | --- |
| 아키텍처·RAG 설계 | 시스템 아키텍처와 3-index RAG 파이프라인 설계, 기획서 작성 |
| Azure AI Search 인덱스 | `law-index` · `guide-index` · `mapping-index` 스키마 설계와 구축. 코드가 아닌 포털에서 구성했기 때문에 전환 과정은 [`feat/3-index-rag-transition`](https://github.com/liminal-cipher/isa-isangmu/tree/feat/3-index-rag-transition) · [`feat/4-index-rag-transition-mapping`](https://github.com/liminal-cipher/isa-isangmu/tree/feat/4-index-rag-transition-mapping) 브랜치에만 남아 있다 |
| 등기부등본 검증 | 위험·주의·안전 판정을 확인하는 테스트 시나리오 설계 |

> 인덱스 작업이 브랜치에만 있는 이유는 팀 정본에 머지되지 않은 채 프로젝트가 끝났기 때문이다. `main`에서는 찾을 수 없다.

## Retrospective

**인덱스를 포털에서 만든 것이 가장 아쉽다.** 2주 일정에서는 그게 빨랐지만, 그 결정 때문에 지금 이 repo만으로는 챗봇을 되살릴 수 없다. 다시 한다면 인덱스 스키마와 적재 스크립트를 코드로 먼저 두고 포털은 확인용으로만 썼을 것이다.

**하이브리드 전환은 늦게 깨달았다.** 처음부터 모든 조건을 LLM에 넘겼다가 같은 입력에 다른 출력이 나오는 걸 보고서야 토글은 LLM이 필요 없다는 걸 알았다. 결정성이 필요한 자리와 유연성이 필요한 자리를 먼저 나눴다면 재작업이 없었다.

**정량 평가를 아예 두지 않았다.** 시나리오를 손으로 돌려보는 것으로 갈음했는데, 작은 정답셋이라도 만들었다면 3-index 전환이 실제로 나아졌는지 말할 수 있었을 것이다.

## Status

완료. Microsoft AI School 9기 2차 프로젝트로 2026.04.13 ~ 04.26 진행. Azure 구독 접근이 끊겨 현재는 실행할 수 없고, 코드와 발표 자료만 남아 있다. 마지막 갱신 2026-08-11.
