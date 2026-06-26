# Kubernetes Bootstrap Guide (1회성 설치 단계)

본 디렉토리는 클러스터 최초 구축 시 **단 한 번만 실행**하는 인프라 초기화 작업용 스크립트와 가이드를 보관합니다. 

Ubuntu 혹은 로컬 가상머신 환경 위에서 **k3s ➔ Helm ➔ ArgoCD** 순서로 순차적으로 설치하여 쿠버네티스 기틀을 완성합니다.

---

## 📂 bootstrap 폴더 구성 및 설명

초기 부트스트랩 단계는 세부 스크립트 파일들로 조각화되어 독립적으로 관리됩니다.

```text
bootstrap/
├── install-k3s.sh       # Step 1: k3s 클러스터 가동 및 kubeconfig 권한 구성 스크립트
├── install-helm.sh      # Step 2: 로컬 패키지 관리를 위한 Helm CLI 설치 스크립트
├── install-argocd.sh    # Step 3: GitOps 배포 엔진 ArgoCD 구동 및 설정 스크립트
└── README.md            # 본 가이드 문서
```

---

## 🛠️ Step-by-Step 초기 셋업 스크립트 상세

아래 스크립트 예시들을 참조하여 각 스크립트 파일을 작성하고 순서대로 가동합니다.

### 1️⃣ Step 1: k3s 설치 스크립트 (`install-k3s.sh`)
이 스크립트는 가벼운 단일 노드 쿠버네티스를 로컬에 설치하고, 개발자가 `sudo` 없이 `kubectl` 명령어를 바로 실행할 수 있도록 보안 그룹 권한 설정을 자동화합니다.

```bash
#!/usr/bin/env bash
# install-k3s.sh
set -euo pipefail

echo "=== 1. k3s (Single-Node Kubernetes) 설치를 시작합니다 ==="
curl -sfL https://get.k3s.io | sh -

echo "=== 2. Kubeconfig 권한 설정을 진행합니다 ==="
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config

# KUBECONFIG 환경 변수가 등록되지 않은 경우 추가
if ! grep -q "KUBECONFIG" ~/.bashrc; then
  echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
  echo "[완료] ~/.bashrc 에 KUBECONFIG가 추가되었습니다."
fi

echo "=== 3. k3s 노드 기동 확인 ==="
kubectl get nodes
```

---

### 2️⃣ Step 2: Helm CLI 설치 스크립트 (`install-helm.sh`)
플랫폼 차트들을 다루기 위해 필요한 Helm 패키지 매니저 CLI 도구를 로컬 터미널에 구성합니다.

```bash
#!/usr/bin/env bash
# install-helm.sh
set -euo pipefail

echo "=== 1. Helm CLI 설치를 진행합니다 ==="
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

echo "=== 2. 설치 버전을 검증합니다 ==="
helm version
```

---

### 3️⃣ Step 3: ArgoCD 설치 및 환경 구성 스크립트 (`install-argocd.sh`)
GitOps의 코어인 ArgoCD 배포 엔진을 배포하고, 로컬 포트로 대시보드를 즉시 노출하며, 최초 로그인 admin 패스워드를 해독하여 안전하게 출력합니다.

```bash
#!/usr/bin/env bash
# install-argocd.sh
set -euo pipefail

echo "=== 1. ArgoCD Namespace 생성 및 리소스 적용 ==="
kubectl create namespace argocd || true
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

echo "=== 2. ArgoCD Server 대시보드 노출 (LoadBalancer 타입으로 패치) ==="
# k3s가 내장한 Klipper LoadBalancer를 활용해 localhost에서 바로 접근하도록 구성합니다.
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

echo "=== 3. 컨트롤러가 안정화될 때까지 대기합니다 ==="
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s

echo "=== 4. 최초 로그인용 admin 패스워드 비밀번호를 디코딩합니다 ==="
PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode)

echo "========================================================"
echo "  [ArgoCD 셋업 완료]"
echo "  - URL: https://localhost"
echo "  - ID: admin"
echo "  - PW: ${PASSWORD}"
echo "========================================================"
```

---

## 🏃‍♂️ 실행 순서

최초 1회 구축 시 터미널에서 다음 순서대로 실행해 주시면 설치가 마무리됩니다.

```bash
# 1. k3s 설치 실행
chmod +x install-k3s.sh && ./install-k3s.sh

# 2. 터미널 리프레시 (Kubeconfig 활성화)
source ~/.bashrc

# 3. Helm 설치 실행
chmod +x install-helm.sh && ./install-helm.sh

# 4. ArgoCD 배포 및 로그인 비밀번호 획득
chmod +x install-argocd.sh && ./install-argocd.sh
```
