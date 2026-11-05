# API 명세서 (API Specification)

본 문서는 FastAPI 기반 콩잎 질병 분류 시스템의 전체 HTTP API 엔드포인트 명세, 요청/응답 규격 및 예외 처리 계약을 정의합니다.

---

## 1. API 목록 요약표

| API ID | Method | URL | 요청 형태 | 성공 HTTP | 주요 반환 필드 | 실패 HTTP | 설명 |
|:---:|:---:|---|---|---:|---|:---:|---|
| **API-01** | `GET` | `/health` | 없음 | 200 | `status` | 서버 다운 | 서비스 생존 헬스체크 |
| **API-02** | `POST` | `/api/sync/classify/` | `multipart/form-data` | 200 | `model_id`, `top1`, `all`, `inference_ms`, `cache_hit` | 400, 413 | 동기 즉시 추론 (캐시 연동) |
| **API-03** | `POST` | `/api/async/classify/` | `multipart/form-data` | **202** | `job_id`, `status`, `check_url` | 400, 413, 503 | 비동기 작업 큐 접수 |
| **API-04** | `GET` | `/api/async/jobs/{job_id}/`| Path 파라미터 | 200 | `job_id`, `status`, `top1`, `all`, `inference_ms` | 404 | 비동기 작업 상태 및 결과 조회 |
| **WEB-01** | `GET` | `/vision/sync/` | 없음 | 200 | HTML | — | 동기 추론 웹 인터페이스 |
| **WEB-02** | `GET` | `/vision/async/` | 없음 | 200 | HTML | — | 비동기 작업 큐 웹 인터페이스 |

---

## 2. 상세 API 명세

### API-02: 동기식 이미지 분류 (`POST /api/sync/classify/`)
- **설명:** 업로드된 콩잎 이미지를 즉시 추론하여 결과를 반환합니다. Redis 캐시를 조회하여 이전 동일 이미지가 있으면 모델 추론 없이 즉시 응답합니다.
- **요청 Body:** `multipart/form-data`
  - `image` (binary, 필수): JPEG / PNG / WEBP 파일 (최대 5MB 이하)

**성공 응답 JSON [샘플] (HTTP 200 OK — 캐시 MISS):**
```json
{
  "model_id": "artifacts/beans_resnet18",
  "top1": {"label": "bean_rust", "score": 0.8542},
  "all": [
    {"label": "bean_rust", "score": 0.8542},
    {"label": "angular_leaf_spot", "score": 0.1230},
    {"label": "healthy", "score": 0.0228}
  ],
  "inference_ms": 22.15,
  "cache_hit": false
}
```

**성공 응답 JSON [샘플] (HTTP 200 OK — 캐시 HIT):**
```json
{
  "model_id": "artifacts/beans_resnet18",
  "top1": {"label": "bean_rust", "score": 0.8542},
  "all": [
    {"label": "bean_rust", "score": 0.8542},
    {"label": "angular_leaf_spot", "score": 0.1230},
    {"label": "healthy", "score": 0.0228}
  ],
  "inference_ms": null,
  "cache_hit": true
}
```

---

### API-03: 비동기 작업 등록 (`POST /api/async/classify/`)
- **설명:** 이미지를 저장하고 Celery 큐 및 DB에 작업을 등록한 뒤, 대기 없이 즉시 접수증(`job_id`)을 발급합니다.
- **요청 Body:** `multipart/form-data` (`image` 파일, 최대 5MB)

**API-03 접수 응답 JSON [샘플] (HTTP 202 Accepted):**
```json
{
  "job_id": 1,
  "status": "queued",
  "check_url": "/api/async/jobs/1/"
}
```

---

### API-04: 비동기 작업 결과 조회 (`GET /api/async/jobs/{job_id}/`)
- **설명:** 발급받은 `job_id`를 통해 PostgreSQL에 기록된 판독 진행 상태 및 완료 결과를 조회합니다.

**API-04 완료 응답 JSON [샘플] (HTTP 200 OK — 판독 완료 `done`):**
```json
{
  "job_id": 1,
  "status": "done",
  "model_id": "artifacts/beans_resnet18",
  "top1": {"label": "angular_leaf_spot", "score": 0.4822},
  "all": [
    {"label": "angular_leaf_spot", "score": 0.4822},
    {"label": "bean_rust", "score": 0.3541},
    {"label": "healthy", "score": 0.1637}
  ],
  "inference_ms": 2564.31,
  "cache_hit": false,
  "error_message": null
}
```

---

## 3. 오류 응답 규격

**API-02 오류 응답 JSON [샘플] (HTTP 400 Bad Request):**
```json
{"detail": "올바른 이미지 파일이 아닙니다."}
```

**API-04 오류 응답 JSON [샘플] (HTTP 404 Not Found):**
```json
{"detail": "해당 작업이 없습니다."}
```

**기타 오류 규격:**
- **413 Payload Too Large:** `{"detail": "5MB 이하 파일만 허용합니다."}`
- **503 Service Unavailable:** `{"detail": "작업 큐 등록 실패: Redis 연결을 확인하세요."}`

---

## 4. 실행 근거표 (Verification Evidence)

| 검증 ID | 테스트 입력 및 시나리오 | 실제 상태코드 | 판정 | 첨부 증빙 |
|:---:|---|:---:|:---:|---|
| **API-02-T01** | `bean_rust` 정상 이미지 업로드 | **200** | PASS | `evidence/sync_async_compare.jpg` |
| **API-03-T01** | 콩잎 이미지 비동기 작업 접수 | **202** | PASS | `evidence/sync_async_compare.jpg` |
| **API-04-T01** | 접수된 정상 작업 번호(`job_id=1`) 조회 | **200** (`done`) | PASS | `evidence/db_rows.png` |
| **API-04-T02** | 없는 작업 번호(`job_id=999999`) 조회 | **404** | PASS | `evidence/pytest_result.png` |
| **API-02-T02** | 텍스트 파일(`x.txt`) 업로드 | **400** | PASS | `evidence/pytest_result.png` |
| **API-02-T03** | 5MB 초과 파일(`big.jpg`) 업로드 | **413** | PASS | `evidence/pytest_result.png` |
| **API-03-T02** | Redis 브로커 장애 모의 | **503** | PASS | `evidence/pytest_result.png` |

---

## 5. 제출 점검 사항 준수 확인
- [x] **`HTTP 202`와 `done`의 구분:** `202 Accepted`는 단순 작업 접수 완료를 뜻하며, 실제 분석 완료는 `status: "done"` 상태로 엄격히 구분하여 명세함.
- [x] **필드명 무결성:** DB 물리 컬럼명은 `all_scores`이지만, 02단계 제공 코드의 실제 API 반환 JSON 필드명인 **`all`**을 그대로 유지하여 계약 불일치 감점을 원천 방지함.
