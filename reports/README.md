# FastAPI Beans ResNet18 동기·비동기 서빙 프로젝트

## 1. 환경
- Windows + WSL2 Ubuntu, Python 가상환경 (`.venv`)
- Docker Desktop / Docker Compose
- Docker: Redis 6379, PostgreSQL 5432
- Python 프로세스: FastAPI, Celery Worker

## 2. 데이터 준비
- 제공 원본 `validation/`의 각 클래스 이미지를 코드 실행 전 `dataset/val/`에 배치
- `dataset/train/`, `dataset/val/`, `dataset/test/` 모두 동일 3클래스 (`angular_leaf_spot`, `bean_rust`, `healthy`)
- 데이터셋의 공개·배포 권한 확인 후 공유

## 3. 모델 준비
- 제공 `training/finetune.py` 실행 (Head-only 파인튜닝)
- 모델 저장: `artifacts/beans_resnet18/`
- `.env`: `MODEL_PATH=artifacts/beans_resnet18`

## 4. 구동 순서
1. `source .venv/bin/activate`
2. `docker compose up -d`
3. `docker compose exec redis redis-cli ping` 결과 `PONG` 확인
4. 별도 터미널: `celery -A ai.celery_app:celery_app worker --loglevel=info --pool=solo --concurrency=1`
5. 별도 터미널: `uvicorn main:app --reload`
6. 동기 화면: http://127.0.0.1:8000/vision/sync/
7. 비동기 화면: http://127.0.0.1:8000/vision/async/
8. API 문서: http://127.0.0.1:8000/docs

## 5. 확인된 응답
- `/health`: `{"status":"ok"}`
- 동기 POST: `top1`, `all`, `cache_hit`, `inference_ms`
- 비동기 POST: `202`, `job_id`; GET: `status=done`
- PostgreSQL: `image_jobs`에 비동기 작업 기록

## 6. 결과물 위치
- 개별 보고서: `reports/01_*.md` ~ `reports/08_*.md`
- 측정 표본: `reports/data/timings.csv`
- 실행 증빙: `reports/evidence/`
- 학습 결과: `artifacts/beans_resnet18/training_report.json`

## 7. 제출 및 보안
- `.env`, 비밀번호, API 키는 GitHub에 업로드하지 않음
- 데이터셋·가중치 대용량 파일은 허용된 제출 경로로 별도 전달
- GitHub에 제외된 파일의 확보·배치 절차를 명시
