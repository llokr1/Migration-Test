# Loventure GitOps Repository

이 프로젝트는 ArgoCD와 GitHub Actions를 통한 GitOps 기반 CI/CD 파이프라인을 지원합니다.

## 🏗️ 아키텍처

### 서비스 구성
- **auth-service**: 인증 서비스 (Spring Boot + PostgreSQL)
- **content-service**: 콘텐츠 관리 서비스 (Spring Boot + PostgreSQL)  
- **course-service**: 코스 관리 서비스 (Spring Boot + PostgreSQL)
- **ai-service**: AI 서비스 (FastAPI)

### 환경 구성
- **develop 브랜치(Staging)**: 카카오 클라우드 쿠버네티스 서비스
- **main 브랜치(Deployment)**: GCP 쿠버네티스 서비스

## 📁 프로젝트 구조

```
PitterPatter-Manifest/
├── Loventure-Manifest/                    # 메인 프로젝트 디렉토리
│   ├── apps/                              # ArgoCD Application 매니페스트
│   │   └── applications/
│   │       ├── argocd.yaml               # ArgoCD 자기 관리 Application
│   │       ├── loventure-dev.yaml         # Development 환경 루트 Application
│   │       └── loventure-prod.yaml        # Production 환경 루트 Application
│   └── charts/                            # Helm 차트
│       ├── argo-cd/                      # ArgoCD 설치용 Helm 차트
│       │   ├── Chart.yaml                # ArgoCD 공식 차트 의존성
│       │   └── values.yaml               # ArgoCD 설정값
│       └── loventure/                    # 부모 차트
│           ├── Chart.yaml
│           ├── values.yaml               # Production 설정
│           ├── values-dev.yaml           # Development 설정
│           ├── charts/                   # 서브차트들
│           │   ├── auth-service/         # 인증 서비스 차트
│           │   ├── content-service/      # 콘텐츠 서비스 차트
│           │   ├── course-service/       # 코스 서비스 차트
│           │   └── ai-service/           # AI 서비스 차트
│           └── templates/
│               └── _helpers.tpl
├── BOOTSTRAP_GUIDE.md                    # ArgoCD 부트스트랩 가이드
└── README.md                             # 이 파일
```

## 🔄 CI/CD 파이프라인

### 브랜치 전략
- **`develop` 브랜치** → **카카오 클라우드** (Development/Staging 환경)
- **`main` 브랜치** → **GCP** (Production 환경)

### 배포 흐름

#### 1. Development 환경 배포
```bash
# develop 브랜치에 코드 푸시
git push origin develop
```
→ ArgoCD가 자동으로 감지하여 **카카오 클라우드**에 배포

#### 2. Production 환경 배포
```bash
# main 브랜치에 머지
git merge develop
git push origin main
```
→ ArgoCD가 자동으로 감지하여 **GCP**에 배포

## 🚀 ArgoCD 설치 및 애플리케이션 생성 과정

### GCP에서 ArgoCD 설치하기

이 프로젝트는 **"ArgoCD가 자기 자신을 관리하도록 하는"** GitOps + Helm 방식을 사용합니다.

#### 📋 사전 준비

**1. GCP 쿠버네티스 클러스터 준비**
```bash
# GCP CLI 설치 및 인증
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# GKE 클러스터 생성 (아직 없다면)
gcloud container clusters create loventure-cluster \
  --zone=asia-northeast3-a \
  --num-nodes=3 \
  --machine-type=e2-medium \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5

# 클러스터 자격 증명 가져오기
gcloud container clusters get-credentials loventure-cluster \
  --zone=asia-northeast3-a \
  --project=YOUR_PROJECT_ID
```

**2. Helm 설치**
```bash
# Helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### 🔧 3단계 부트스트랩 과정

**1단계: ArgoCD 네임스페이스 생성**
```bash
kubectl create namespace argocd
```

**2단계: ArgoCD Helm 차트 최초 설치**
```bash
# 의존성 업데이트
cd Loventure-Manifest/charts/argo-cd
helm dependency update

# ArgoCD 설치
helm upgrade --install argo-cd . --namespace argocd

# ArgoCD 서버가 준비될 때까지 대기
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

**3단계: ArgoCD 자기 관리 Application 적용**
```bash
# ArgoCD가 자기 자신을 관리하도록 Application Manifest 적용
kubectl apply -f Loventure-Manifest/apps/applications/argocd.yaml
```

#### 📁 파일별 역할 및 실행 순서

**1. `Loventure-Manifest/charts/argo-cd/Chart.yaml`**
```yaml
# ArgoCD 공식 Helm 차트를 의존성으로 추가
dependencies:
  - name: argo-cd
    repository: "https://argoproj.github.io/argo-helm"
    version: "5.52.2"  # 버전 고정으로 불변성 확보
```

**2. `Loventure-Manifest/charts/argo-cd/values.yaml`**
```yaml
# ArgoCD 설정값 정의
argo-cd:
  server:
    service:
      type: ClusterIP
      port: 80
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
  # ... 기타 설정들
```

**3. `Loventure-Manifest/apps/applications/argocd.yaml`**
```yaml
# ArgoCD가 자기 자신을 관리하도록 하는 Application
spec:
  source:
    repoURL: 'https://github.com/PitterPetter/PitterPatter-Manifest.git'
    path: 'Loventure-Manifest/charts/argo-cd'  # ArgoCD Helm 차트 경로
    targetRevision: main
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
```

#### 🔄 애플리케이션 생성 과정

**1. Loventure 애플리케이션 배포**
```bash
# Development 환경
kubectl apply -f Loventure-Manifest/apps/applications/loventure-dev.yaml

# Production 환경
kubectl apply -f Loventure-Manifest/apps/applications/loventure-prod.yaml
```

**2. 자동 동기화 설정**
```bash
# ArgoCD CLI 설치
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# ArgoCD 로그인
kubectl port-forward svc/argocd-server -n argocd 8080:80
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
argocd login localhost:8080 --username admin --password "$ARGOCD_PASSWORD" --insecure

# 자동 동기화 설정
argocd app set loventure-prod --sync-policy automated
argocd app set loventure-prod --auto-prune
argocd app set loventure-prod --self-heal
```

#### ✅ 설치 완료 확인

**ArgoCD UI 접근**
```bash
# ArgoCD 서버 포트포워딩
kubectl port-forward svc/argocd-server -n argocd 8080:80

# 브라우저에서 http://localhost:8080 접속
# 사용자명: admin
# 비밀번호: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**애플리케이션 상태 확인**
```bash
# Pod 상태 확인
kubectl get pods -n loventure-app
kubectl get pods -n loventure-dev

# ArgoCD 애플리케이션 상태 확인
kubectl get applications -n argocd
```

#### 🔄 이후 운영 방법

**ArgoCD 설정 변경**
1. `Loventure-Manifest/charts/argo-cd/values.yaml` 파일 수정
2. `git add . && git commit -m "Update ArgoCD configuration" && git push`
3. ArgoCD가 자동으로 변경사항을 감지하고 자기 자신을 업데이트

**Loventure 애플리케이션 배포**
1. `Loventure-Manifest/charts/loventure/values.yaml` 파일 수정
2. `git push`
3. ArgoCD가 자동으로 변경사항을 감지하고 애플리케이션을 배포

### ArgoCD Application 설정

#### Development 환경 (`loventure-dev.yaml`)
```yaml
spec:
  source:
    repoURL: https://github.com/PitterPetter/PitterPatter-Manifest
    targetRevision: develop  # develop 브랜치 감시
    path: charts/loventure
    helm:
      valueFiles:
        - values-dev.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: loventure-dev
```

#### Production 환경 (`loventure-prod.yaml`)
```yaml
spec:
  source:
    repoURL: https://github.com/PitterPetter/PitterPatter-Manifest
    targetRevision: main  # main 브랜치 감시
    path: charts/loventure
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: loventure-app
```

## 📚 추가 문서

- [BOOTSTRAP_GUIDE.md](BOOTSTRAP_GUIDE.md) - ArgoCD 부트스트랩 상세 가이드
- [DEPLOYMENT_GUIDE.md](Loventure-Manifest/DEPLOYMENT_GUIDE.md) - 상세 배포 가이드
- [PROJECT_STRUCTURE.md](Loventure-Manifest/PROJECT_STRUCTURE.md) - 프로젝트 구조 상세 설명
- [Loventure-Manifest/README.md](Loventure-Manifest/README.md) - 메인 프로젝트 문서

## 🎯 핵심 특징

- **완전한 GitOps**: ArgoCD가 자기 자신을 Git에서 관리
- **Helm 기반**: 모든 애플리케이션을 Helm 차트로 관리
- **멀티 클라우드**: 카카오 클라우드(개발) + GCP(프로덕션)
- **자동 배포**: Git push만으로 자동 배포
- **일관성**: ArgoCD와 애플리케이션을 동일한 방식으로 관리


