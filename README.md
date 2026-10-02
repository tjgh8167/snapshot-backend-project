# Snapshot Backend

숙박업 소상공인이 대화를 통해 광고 기획서를 만들고, 객실·분위기·혜택을 강조한 광고 이미지 3종을 생성하는 서비스의 FastAPI 백엔드입니다.

React 클라이언트와는 REST API로 통신하고, AI 모델 서버와는 gRPC로 통신합니다. 기획 단계와 입력 데이터, 이미지 생성 상태를 PostgreSQL에 저장하며 모델 오류와 사용자 재생성을 구분해 처리합니다.

## 프로젝트 개요

| 구분 | 내용 |
| --- | --- |
| 기간 | 2026.09.03 - 2026.10.02 |
| 인원 | 3명 (Frontend, Backend, AI Model) |
| 대상 사용자 | 광고 제작 인력이 부족한 숙박업 소상공인 |
| 핵심 기능 | 대화형 광고 기획, 원본 이미지 업로드, 광고 초안 3종 생성, 선택·재생성·다운로드 |
| 담당 | FastAPI API, PostgreSQL 스키마, REST-gRPC 변환, 상태 및 오류 처리, 통합 테스트 |
| 전체 구성 | React REST ↔ FastAPI ↔ AI Model gRPC |

팀 전체 코드와 배포 구성은 [SnapshotMain](https://github.com/tjgh8167/snapshot-ai-ad-service)에서 확인할 수 있습니다.

## 문제 정의

소규모 숙박업체가 광고 콘텐츠를 만들려면 숙소의 특징과 혜택을 정리하고, 목적에 맞는 문구와 이미지를 반복해서 제작해야 합니다. Snapshot은 사용자가 챗봇의 질문에 답하면 광고 기획 정보를 구조화하고, 서로 다른 강조점을 가진 광고 초안을 한 번에 비교할 수 있도록 설계했습니다.

백엔드는 단순히 요청을 중계하는 데서 끝나지 않고 다음 조건을 만족해야 했습니다.

- 기획 대화의 단계와 저장 데이터가 어긋나지 않아야 합니다.
- 각 광고 방향에 필요한 정보가 명확히 구분되어야 합니다.
- REST 요청과 gRPC 응답 사이의 식별자와 상태를 검증해야 합니다.
- 시스템 오류 재시도와 사용자의 재생성 기회를 분리해야 합니다.
- 생성 이미지는 규격과 형식을 확인한 뒤 안전한 경로에 저장해야 합니다.

## 서비스 흐름

```text
기획 세션 생성
  → 챗봇 질문에 답하며 숙소 정보 입력
  → 원본 이미지 업로드
  → 광고 기획서 확정
  → 객실·분위기·혜택 광고 초안 생성
  → 초안 선택 또는 1회 재생성
  → PNG 다운로드
```

광고 기획 단계는 `lodging_type → lodging_information → selling_points → lodging_service → mood → color_preference → target_audience → ad_copy → complete` 순서로 진행됩니다.

## 담당 역할

- FastAPI 기반 REST API와 서비스 계층 설계 및 구현
- PostgreSQL 테이블과 Alembic 마이그레이션 관리
- React 요청을 모델 서버의 gRPC 계약으로 변환하는 클라이언트 구현
- 기획 단계 전환, 요청 식별자, 상태 버전 검증 로직 구현
- 원본 및 결과 이미지 검증·저장·다운로드 처리
- 실패한 초안 재시도와 사용자 재생성 정책 분리
- Frontend·AI Model 담당자와 V1/V2 통합 테스트 진행

세부 의사결정과 구현 범위는 [기여 기록](document/contribution.md)에 정리했습니다.

## 핵심 구현

### 1. 광고 방향별 입력 정보 분리

초기에는 하나의 숙소 장점 정보만으로 광고 3종을 생성했습니다. 입력이 짧으면 세 결과가 비슷하고 실제 숙소와 다른 표현이 만들어질 가능성이 있었습니다.

광고 방향별로 참고할 필드를 분리하고, 혜택과 서비스를 묻는 `lodging_service` 단계를 새로 추가했습니다.

| 광고 방향 | 사용하는 정보 | 목적 |
| --- | --- | --- |
| 객실·전망 | `selling_points` | 객실 내외부 특징과 공간 강조 |
| 감성·분위기 | `mood`, `color_preference` | 원하는 분위기와 색상 반영 |
| 서비스·혜택 | `lodging_service` | 조식, 이벤트, 부가 서비스 강조 |

스키마, DB 마이그레이션, Protobuf 계약, REST-gRPC 변환 로직을 함께 수정해 입력부터 생성 요청까지 같은 의미가 유지되도록 했습니다.

### 2. 기획 단계 일관성 검증

화면의 진행 단계와 서버가 판단한 다음 단계가 어긋나면서 잘못된 `current_step`이 후속 요청에 저장되는 문제가 있었습니다.

백엔드에서 현재 단계에 필요한 필드가 실제로 저장되었는지 확인하고, 현재 단계 유지 또는 정해진 다음 단계 이동만 허용했습니다. 잘못된 단계가 들어오면 즉시 오류를 반환해 연동 문제를 조기에 발견하도록 했습니다.

정상 답변과 부적합 답변을 각각 입력해 단계 이동과 단계 유지가 모두 의도대로 동작하는지 확인했습니다.

### 3. REST-gRPC 계약 검증

모델 응답의 `request_id`, `session_id`, `state_revision`, `current_step`을 원래 요청과 대조합니다. 다른 세션의 응답이나 오래된 상태가 반환되면 처리하지 않아 데이터가 잘못 갱신되는 것을 막았습니다.

gRPC 오류는 백엔드 내부 예외로 변환한 뒤 REST 상태 코드와 오류 메시지로 정리해 클라이언트가 실패 원인을 구분할 수 있도록 했습니다.

### 4. 시스템 오류와 사용자 재생성 분리

최초 생성 중 모델 또는 외부 서비스 오류가 발생하면 실패한 초안만 같은 `draft_id`로 다시 요청합니다. 이 재시도는 사용자의 1회 재생성 기회를 소모하지 않습니다.

사용자가 결과 3종을 확인한 뒤 재생성을 선택한 경우에만 두 번째 생성 라운드를 시작하며, 세 초안이 모두 완료되었을 때 재생성 사용 여부를 확정합니다.

```text
pending → processing → completed
                     ↘ failed → system retry

round 1 completed × 3 → user regeneration → round 2
```

### 5. 이미지 검증과 저장

- 원본 이미지: JPEG, PNG, WebP / 최대 25MiB
- 생성 결과: PNG / 1080 × 1350
- 세션·초안 ID를 기준으로 저장 경로 생성
- DB에는 파일 자체가 아닌 URL과 메타데이터 저장

## 검증 결과

- 정상 답변 입력 시 다음 기획 단계로 이동
- 필수 정보가 부족한 답변 입력 시 현재 단계 유지
- 잘못된 단계와 식별자 입력 시 오류 반환
- 객실·분위기·혜택별 입력 데이터가 각각의 생성 요청에 반영
- 실패한 초안만 동일 ID로 재시도
- 시스템 재시도가 사용자 재생성 횟수에 영향을 주지 않음
- 생성된 PNG를 저장하고 React 화면에서 조회·다운로드
- React → FastAPI → AI Model 전체 V1/V2 연동 테스트 완료

## 기술 스택

| 구분 | 기술 |
| --- | --- |
| Backend | Python 3.12, FastAPI, Uvicorn, Pydantic |
| Database | PostgreSQL, SQLAlchemy, Alembic, Psycopg |
| Model communication | gRPC, Protobuf, grpcio-status |
| Image | Pillow |
| Infrastructure | Docker, Docker Compose, Nginx, GCP VM |

## 프로젝트 구조

```text
snapshot-backend-project/
├── alembic/                 # DB 마이그레이션
├── app/
│   ├── api/routes/          # REST API
│   ├── clients/             # AI Model gRPC 클라이언트
│   ├── db/                  # DB 연결
│   ├── grpc_stubs/          # Protobuf 및 생성 코드
│   ├── models/              # SQLAlchemy 모델
│   ├── schemas/             # 요청·응답 검증
│   └── services/            # 비즈니스 로직
├── document/
│   ├── api-spec.html
│   └── contribution.md
├── compose.yaml
├── Dockerfile
└── requirements.txt
```

## 실행 방법

```bash
cp .env.example .env
cp .postgres.env.example .postgres.env
docker compose up --build -d
```

- Swagger UI: `http://127.0.0.1:9000/docs`
- Backend health: `http://127.0.0.1:9000/health`
- Model health: `http://127.0.0.1:9000/api/model/health`

이 저장소의 Compose는 백엔드와 PostgreSQL을 실행합니다. Frontend 및 AI Model을 포함한 전체 실행은 [SnapshotMain](https://github.com/tjgh8167/snapshot-ai-ad-service)의 Compose 구성을 사용합니다.

## 한계와 개선 방향

프로젝트 기간에는 핵심 생성 흐름과 서비스 연동을 우선해 아래 항목은 후속 과제로 남겼습니다.

- 사용자 인증과 세션 소유권 검증
- 외부 오브젝트 스토리지 적용
- 비동기 작업 큐와 재시도 정책 고도화
- 생성 결과 수정 기능과 목적별 출력 규격 확장

## 배운 점

백엔드는 화면과 모델 사이의 데이터를 전달하는 역할만 하는 것이 아니라 서비스의 상태와 규칙을 일관되게 유지해야 한다는 점을 배웠습니다. 입력 구조, 단계 전환, 재시도 정책처럼 사용자는 직접 보지 못하는 규칙이 결과의 정확성과 사용 경험을 결정했습니다.

## 관련 자료

- [팀 통합 저장소](https://github.com/tjgh8167/snapshot-ai-ad-service)
- [백엔드 API 명세](document/api-spec.html)
- [백엔드 기여 기록](document/contribution.md)
