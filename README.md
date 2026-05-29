# SKALA TEAM5 개발/배포 운영 가이드

이 문서는 SKALA TEAM5 레포지토리의 공통 개발 방식, 브랜치 전략, 로컬 테스트, Kubernetes 접속, 배포 흐름을 정리합니다.

## 레포지토리 구성

```text
frontend  - Next.js 프론트엔드
backend   - Spring Boot 백엔드
fastapi   - AI Agent / FastAPI
db        - PostgreSQL, Flyway migration, MinIO
qdrant    - Qdrant Vector DB Kubernetes 배포
batch     - Qdrant/법령 데이터 적재 배치
deploy    - ArgoCD root application, Ingress
```

## 브랜치 전략

```text
main
  실제 운영 배포 브랜치입니다.
  main에 머지되면 GitHub Actions 또는 ArgoCD를 통해 Kubernetes에 반영됩니다.

develop
  팀 통합 테스트 브랜치입니다.
  기능 개발이 끝나면 먼저 develop으로 PR을 올립니다.

feature/*
  개인 작업 브랜치입니다.
  모든 기능/수정 작업은 feature 브랜치에서 시작합니다.
```

기본 흐름:

```text
feature/*
  -> develop PR
  -> 로컬/통합 테스트
  -> develop -> main PR
  -> 운영 배포
```

## 처음 한 번만: develop 브랜치 생성

각 레포에서 `develop` 브랜치가 없으면 생성합니다.

```bash
git checkout main
git pull origin main
git checkout -b develop
git push origin develop
```

이미 `develop`이 있으면 최신 상태로 받습니다.

```bash
git checkout develop
git pull origin develop
```

## 개인 작업 시작

항상 최신 `develop`에서 feature 브랜치를 만듭니다.

```bash
git checkout develop
git pull origin develop
git checkout -b feature/작업-이름
```

예시:

```bash
git checkout -b feature/login-fix
```

작업 후 커밋합니다.

```bash
git status
git add .
git commit -m "fix: 로그인 요청 오류 수정"
git push origin feature/login-fix
```

GitHub에서 PR을 생성합니다.

```text
base: develop
compare: feature/login-fix
```

## develop 통합 테스트

`develop`에 머지된 뒤 로컬에서 통합 테스트합니다.

```bash
git checkout develop
git pull origin develop
```

### Frontend

```bash
cd frontend
cp .env.example .env.local
npm install
npm run dev
```

`frontend/.env.local`:

```env
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

### Backend

```bash
cd backend
cp .env.example .env
```

공용 Kubernetes DB를 로컬에서 사용하려면 port-forward를 엽니다.

```bash
kubectl port-forward svc/team5-postgres 5433:5432 -n skala3-finalproj-class2-team5
```

MinIO가 필요하면 별도 터미널에서 엽니다.

```bash
kubectl port-forward svc/team5-minio 9000:9000 -n skala3-finalproj-class2-team5
```

Qdrant가 필요하면 별도 터미널에서 엽니다.

```bash
kubectl port-forward svc/team5-qdrant 6333:6333 -n skala3-finalproj-class2-team5
```

백엔드 실행:

```bash
set -a
source .env
set +a
./gradlew bootRun
```

로컬 통합 테스트 흐름:

```text
frontend localhost:3000
  -> backend localhost:8000
    -> postgres localhost:5433
    -> minio localhost:9000
    -> qdrant localhost:6333
```

## Kubernetes 접속

EKS kubeconfig를 설정합니다.

```bash
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name skala-2025
```

현재 context를 확인합니다.

```bash
kubectl config current-context
```

팀 namespace를 기본 namespace로 지정합니다.

```bash
kubectl config set-context --current --namespace=skala3-finalproj-class2-team5
```

리소스를 확인합니다.

```bash
kubectl get pods,svc,deploy,sts,pvc
```

## 로컬 포트포워딩

PostgreSQL:

```bash
kubectl port-forward svc/team5-postgres 5433:5432 -n skala3-finalproj-class2-team5
```

MinIO API:

```bash
kubectl port-forward svc/team5-minio 9000:9000 -n skala3-finalproj-class2-team5
```

MinIO Console:

```bash
kubectl port-forward svc/team5-minio 9001:9001 -n skala3-finalproj-class2-team5
```

브라우저 접속:

```text
http://localhost:9001
```

Qdrant:

```bash
kubectl port-forward svc/team5-qdrant 6333:6333 -n skala3-finalproj-class2-team5
```

Qdrant 컬렉션 확인:

```bash
curl http://localhost:6333/collections
```

`legal_documents` 확인:

```bash
curl http://localhost:6333/collections/legal_documents
```

데이터 일부 확인:

```bash
curl -X POST http://localhost:6333/collections/legal_documents/points/scroll \
  -H "Content-Type: application/json" \
  -d '{
    "limit": 5,
    "with_payload": true,
    "with_vector": false
  }'
```

## main 배포 규칙

`develop`에서 충분히 확인한 뒤에만 `main`으로 PR을 올립니다.

```text
base: main
compare: develop
```

`main`에 머지되면 아래 흐름으로 운영에 반영됩니다.

```text
frontend main merge
  -> GitHub Actions
  -> Docker image build/push
  -> Kubernetes frontend rollout

backend main merge
  -> GitHub Actions
  -> Docker image build/push
  -> Kubernetes backend rollout

db main merge
  -> migration 변경 시 Flyway Job 실행

deploy main merge
  -> ArgoCD root application / Ingress 반영

qdrant main merge
  -> ArgoCD가 Qdrant StatefulSet/Service 반영

batch main merge
  -> batch 이미지/Job 정의 업데이트
  -> 실제 적재 실행은 GitHub Actions에서 수동 실행
```

## 환경변수 관리

실제 `.env` 파일과 비밀번호는 커밋하지 않습니다.

커밋 가능한 파일:

```text
.env.example
README.md
k8s/*.yaml
.github/workflows/*.yml
```

커밋하면 안 되는 파일:

```text
.env
.env.local
비밀번호가 들어간 yaml
API key
개인 kubeconfig
```

로컬은 각자 `.env` 또는 `.env.local`을 사용합니다.

```text
frontend/.env.local
backend/.env
fastapi/.env
batch/.env
```

운영은 아래 방식으로 값을 주입합니다.

```text
Kubernetes ConfigMap
Kubernetes Secret
GitHub Actions Secrets
Docker build args
```

## DB migration 규칙

DB migration은 `db` 레포에서 관리합니다.

```text
develop
  migration 파일 작성, 리뷰, 로컬 검증

main
  실제 Flyway migration 실행
```

이미 운영 DB에 적용된 migration 파일은 수정하지 않습니다. 새 변경은 새 버전 파일로 추가합니다.

```text
V7__add_xxx.sql
V8__update_xxx.sql
```

## Batch / Qdrant 운영

Qdrant 서버는 `qdrant` 레포의 Kubernetes manifest를 ArgoCD가 관리합니다.

```text
qdrant/main
  -> team5-qdrant StatefulSet
  -> team5-qdrant Service
```

법령/RAG 데이터 적재는 `batch` 레포의 GitHub Actions에서 수동 실행합니다.

```text
SKALA-TEAM5/batch
  -> Actions
  -> Run Qdrant Batch
```

처음 전체 적재:

```text
mode: ingest
```

이미 데이터가 있고 변경분만 반영:

```text
mode: refresh
```

MinIO PDF 파일은 아래 위치에 올립니다.

```text
bucket: safety-files
prefix: data/
example: safety-files/data/example.pdf
```

`ingest`는 MinIO의 `safety-files/data/` 아래 PDF를 내려받아 처리한 뒤 Qdrant와 PostgreSQL에 적재합니다.

## Git 충돌 처리

원격 변경이 먼저 들어온 경우:

```bash
git fetch origin
git rebase origin/develop
```

`main` 기준 작업 중이면:

```bash
git fetch origin
git rebase origin/main
```

충돌 해결 후:

```bash
git add .
git rebase --continue
```

rebase를 취소하려면:

```bash
git rebase --abort
```

## 금지 사항

```text
main 직접 push
.env 커밋
운영 Secret 값 커밋
이미 적용된 migration 수정
kubectl delete pvc 무작정 실행
git push --force main
```

## PR 제목 예시

```text
feat(frontend): 로그인 화면 개선
fix(backend): 파일 업로드 오류 수정
chore(db): add V7 migration
chore(deploy): update ingress source
feat(batch): load PDFs from MinIO
chore(qdrant): update statefulset resources
```

## 최종 원칙

```text
1. 개인 작업은 feature/*
2. 팀 통합은 develop
3. 실제 배포는 main
4. main 머지는 신중하게
5. 환경값은 코드가 아니라 실행 환경에서 주입
```
