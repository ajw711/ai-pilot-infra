# AI-PILOT Infra Repository

본 저장소는 지식 검수 및 요약 위키 시스템(ai-pilot)의 단일 노드 k3s 클러스터 환경 구축, 공통 운영 플랫폼 구성, 그리고 GitOps 기반 어플리케이션 배포 자동화를 위해 설계된 인프라 코어 리포지토리입니다.

# ai-pilot-infra

GitOps Repository

## 목적
- k3s 클러스터 관리
- ArgoCD를 통한 GitOps
- Helm Chart 관리
- Platform 서비스 관리
- Application 배포 관리

## Architecture

GitHub
   ↓
ArgoCD
   ↓
k3s

---

## 📂 폴더 구조 및 설계 철학

저장소는 최초 셋업 단계부터 운영 서비스 배포까지의 생명주기(Lifecycle) 및 의존성 분리를 위해 다음과 같은 구조로 격리되어 관리됩니다.

```text
ai-pilot-infra/
├── bootstrap/          # 1회성 초기화 단계 (Ubuntu ➔ k3s ➔ Helm ➔ ArgoCD 설치 및 설정)
├── platform/           # 공통 운영 플랫폼 (Helm values.yaml 파일들만 독립 제어)
├── applications/       # ArgoCD 감시 및 배포 타겟 (ArgoCD Application 리소스 선언)
├── charts/             # 직접 제작한 커스텀 서비스 Helm Chart (개발 소스 배포용)
└── environments/       # 개발(dev) 및 운영(prod) 환경별 이미지 태그 및 환경 변수 격리
```

---

## 🎯 각 디렉토리의 역할 상세

### 1. `bootstrap/` (초기 1회성 셋업)
* **목적**: 인프라 가동을 위해 클러스터 생성부터 배포 엔진 가동까지 단 한 번만 실행하는 초기 구축 작업 스크립트와 가이드를 모아둡니다.
* **설계 이유**: 인프라 운영 과정에서 수시로 변경되는 파일들과 매번 실행할 필요가 없는 부트스트랩 스크립트를 분리하여 인프라의 복잡도를 낮춥니다.
* **주요 구성**:
  ```text
  bootstrap/
  ├── install-k3s.sh       # k3s 설치 및 kubeconfig 권한 구성 스크립트
  ├── install-helm.sh      # Helm CLI 설치 스크립트
  ├── install-argocd.sh    # ArgoCD 엔진 구성 및 외부 포트 노출 스크립트
  └── README.md            # 부트스트랩 상세 가이드 문서
  ```

### 2. `platform/` (공통 운영 플랫폼 - 선행 인프라)
* **목적**: 서비스 가동 전에 쿠버네티스 클러스터 위에 기본 탑재되어야 하는 공통 모듈 및 미들웨어를 관리합니다.
* **설계 이유**: 플랫폼 리소스들은 오픈소스 차트를 그대로 사용하므로, 복잡한 설치 템플릿을 직접 다루지 않고 오직 **Helm values** 설정 파일들만을 선언적으로 정의하여 가볍게 관리합니다.
* **주요 구성**:
  ```text
  platform/
  ├── ingress-nginx/       # NGINX 인그레스 컨트롤러 values.yaml
  ├── prometheus/          # 프로메테우스 수집기 values.yaml
  ├── grafana/             # 그라파나 대시보드 values.yaml
  ├── loki/                # 로키 로그 수집기 values.yaml
  ├── nats/                # NATS 메시지 브로커 values.yaml
  └── cert-manager/        # SSL 인증서 발급 모듈 values.yaml
  ```

### 3. `applications/` (ArgoCD 동기화 대상 정의)
* **목적**: ArgoCD 배포 엔진이 변경 사항을 감시하여 실제 쿠버네티스 클러스터 상태와 일치(Sync)시키는 타겟 리소스 파일들을 모아둡니다.
* **설계 이유**: 배포될 모든 컴포넌트들을 각각의 ArgoCD Application YAML 파일로 관리하여, 서비스 컴포넌트를 독립적으로 배포 및 모니터링할 수 있게 만듭니다.
* **주요 구성**:
  ```text
  applications/
  ├── ai-pilot.yaml        # ai-pilot 비즈니스 서비스 ArgoCD App
  ├── gateway.yaml         # 게이트웨이 서비스 ArgoCD App
  ├── grafana.yaml         # 그라파나 플랫폼 ArgoCD App
  └── nats.yaml            # NATS 플랫폼 ArgoCD App
  ```

### 4. `charts/` (자체 개발 커스텀 Helm Chart)
* **목적**: 우리가 직접 개발한 비즈니스 애플리케이션들을 쿠버네티스에 패키징하여 배포하기 위한 커스텀 Helm 차트 템플릿입니다.
* **설계 이유**: 다른 오픈소스 플랫폼과 달리, 사내 비즈니스 소스코드는 자주 변경되고 템플릿 구조를 직접 커스터마이징해야 하므로 독립된 차트 리포지토리 형태로 다룹니다.
* **주요 구성**:
  ```text
  charts/
  ├── ai-pilot/            # ai-pilot 웹 어플리케이션 용 Helm Chart
  └── gateway/             # 게이트웨이 서비스 용 Helm Chart
  ```

### 5. `environments/` (환경 격리 및 릴리즈 태그 관리 - 가장 중요 🌟)
* **목적**: 동일한 소스코드와 템플릿을 사용하되, 배포 타겟 환경(dev, prod 등)에 따라 주입될 고유 환경 변수와 이미지 태그 버전을 완전히 격리합니다.
* **설계 이유**: 빌드 릴리즈 단계를 환경별로 유연하게 제어하기 위함입니다.
  - **`environments/dev/`**: 신속한 피드백을 위해 수시로 빌드되는 `image.tag: latest`를 주로 바라봅니다.
  - **`environments/prod/`**: 검증 및 릴리즈가 완료된 안정적인 고유 버전 `image.tag: v1.0.3`으로 지정하여 운영 환경에 무분별한 배포가 되지 않도록 제어합니다.
* **주요 구성**:
  ```text
  environments/
  ├── dev/
  │   └── values.yaml      # 개발 환경 오버라이드 values (예: tag: latest)
  └── prod/
      └── values.yaml      # 운영 환경 오버라이드 values (예: tag: v1.0.3)
  ```

---

## 🚀 전체 설치 및 파이프라인 작동 흐름

1. **Bootstrap 단계**: [bootstrap/README.md](file:///C:/personal/ai-pilot-infra/bootstrap/README.md)에 따라 로컬 노드(k3s)를 구성하고 ArgoCD를 올립니다.
2. **Platform 단계**: 클러스터 위에 `platform/`에 명시된 Helm values를 적용하여 Ingress, NATS, Monitoring 인프라 서비스를 먼저 실행합니다.
3. **ArgoCD App 등록**: `applications/` 폴더 내에 있는 Application YAML 파일들을 적용하여 ArgoCD에게 배포 경로를 알려줍니다.
4. **Environments & GitOps 배포**: 
   - CI(GitHub Actions)에서 소스코드가 변경되면 이미지를 생성해 Docker Registry에 푸시합니다.
   - GitHub Actions가 `environments/{env}/values.yaml` 내의 이미지 태그 버전을 자동으로 변경(Commit)해 줍니다.
   - ArgoCD가 이 매니페스트 변경을 감지하고, 클러스터에 배포된 애플리케이션의 버전을 실시간으로 동기화(Pull Sync)합니다.
