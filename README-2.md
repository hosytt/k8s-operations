# Kubernetes 클러스터 GitOps 저장소

이 저장소는 **GitOps** 방식으로 Kubernetes 클러스터 인프라와 애플리케이션을 관리하기 위한 설정 파일을 포함합니다.

## 🚀 시작하기

클러스터를 처음 구성할 때는 필수 구성요소(`Calico`, `Argo CD`)를 아래 순서대로 수동 설치해야 합니다.  
이후 인프라는 Argo CD를 통해 **순차적으로** 배포됩니다.

### 사전 준비

- Kubernetes 클러스터 (Master/Worker 노드 준비 완료)
- `kubectl` (클러스터 접근 설정 완료)
- `helm` (v3 이상)

---

## 1. 필수 네트워크 및 도구 설치 (수동 설치)

초기 부트스트랩은 `infra/.manual-install/` 디렉토리의 스크립트와 설정 파일을 사용합니다.

### 1-1. Calico(CNI) 설치

Pod 간 통신을 위해 네트워크 플러그인을 먼저 설치해야 합니다.

```bash
# 디렉토리 이동
cd infra/.manual-install/calico

# Helm 저장소 추가
helm repo add projectcalico https://projectcalico.docs.tigera.io/charts
helm repo update

# Calico 설치
helm upgrade --install calico projectcalico/tigera-operator \
  --version 3.31.0 \
  -n tigera-operator --create-namespace \
  -f values.yaml

# 설치 확인
kubectl get pods -n calico-system
```

### 1-2. Argo CD 설치

GitOps의 핵심 도구인 Argo CD를 설치합니다.

```bash
# 디렉토리 이동
cd ../argo-cd

# Helm 저장소 추가
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Argo CD 설치
helm upgrade --install argocd argo/argo-cd \
  --version 8.5.8 \
  -n argo-cd --create-namespace \
  -f values.yaml

# 설치 확인 (모든 Pod가 Running 상태가 될 때까지 대기)
kubectl get pods -n argo-cd
```

> **참고:** 초기 관리자 비밀번호 확인
>
> ```bash
> kubectl -n argo-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
> ```

---

## 2. GitOps 부트스트랩 (순차 적용)

Argo CD 설치 후 의존성이 맞도록 아래 순서대로 애플리케이션을 적용합니다.

### 2-1. AppProject 적용

먼저 `infra-structure` 프로젝트를 적용합니다.

```bash
kubectl apply -f infra/app-project.yaml
```

### 2-2. 기본 인프라 적용

다른 애플리케이션 동작에 필요한 기본 구성요소(시크릿 관리, 인그레스 관련 기능)입니다.

> **⚠️ 중요:** 마이그레이션 또는 재설치 환경이라면 Sealed Secrets 적용 전에 **개인 키(Private Key)** 를 반드시 복구해야 합니다.
> 키를 복구하지 않으면 기존 암호화 시크릿을 복호화할 수 없습니다.
>
> ```bash
> # 예시: 마스터 키 복구
> kubectl apply -f master-key.yaml
> ```

```bash
# 1. Sealed Secrets (암호화)
kubectl apply -f infra/sealed-secrets/application.yaml

# 2. Reflector (시크릿 복제)
kubectl apply -f infra/reflector/application.yaml

# 3. Ingress NGINX (인그레스 컨트롤러)
kubectl apply -f infra/ingress-nginx/application.yaml

# 4. Reloader (Config/Secret 변경 시 Pod 자동 재시작)
kubectl apply -f infra/reloader/application.yaml
```

> **대기:** 다음 단계로 진행하기 전에 `ingress-nginx`가 완전히 실행 중인지 확인하세요.

### 2-3. 스토리지 및 모니터링 적용

```bash
# 5. Longhorn (분산 스토리지)
kubectl apply -f infra/longhorn/application.yaml

# 6. Prometheus & Grafana (모니터링)
kubectl apply -f infra/prometheus-community/application.yaml

# 7. Metrics Server
kubectl apply -f infra/metrics-server/application.yaml
```

### 2-4. 애플리케이션 적용

```bash
# 8. Sealed Secrets Web (UI)
kubectl apply -f infra/sealed-secrets-web/application.yaml
```

---

## 3. 검증 및 접근

### 3-1. Argo CD 애플리케이션 확인

```bash
# Argo CD가 관리하는 전체 애플리케이션 확인
kubectl get applications.argoproj.io -n argo-cd

# 동기화/헬스 상태 상세 확인 (예시: sealed-secrets)
kubectl describe application sealed-secrets -n argo-cd
```

### 3-2. Prometheus & Grafana 확인

- **네임스페이스**: `monitoring`
- **Grafana 관리자 비밀번호 확인**:
  ```bash
  kubectl get secret -n monitoring prometheus-community-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
  ```
- **Grafana 접속 (Port Forward)**:
  ```bash
  kubectl port-forward -n monitoring svc/prometheus-community-grafana 3000:9091
  ```
- Prometheus 데이터소스가 자동 생성되지 않으면 Grafana에서 수동 추가:
  - URL: `http://prometheus-community-kube-prom-prometheus.monitoring.svc.cluster.local:9090`
  - Access: `Server`

### 3-3. Argo CD 초기 관리자 비밀번호 확인

```bash
kubectl -n argo-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

---

## 📂 디렉토리 구조

```
.
├── infra/                    # Argo CD(GitOps)로 관리하는 인프라
│   ├── README.md             # 인프라 부트스트랩 및 운영 가이드
│   ├── app-project.yaml      # Argo CD AppProject 정의
│   ├── .manual-install/      # 초기 클러스터 구성용 수동 설치 파일
│   │   ├── README.md
│   │   ├── argo-cd/          # 초기 Argo CD 설치용 Helm values
│   │   └── calico/           # 초기 CNI 설치용 Helm values
│   ├── ingress-nginx/        # 인그레스 컨트롤러
│   ├── longhorn/             # 분산 스토리지
│   ├── metrics-server/       # 메트릭 수집 서버
│   ├── prometheus-community/ # 모니터링 (Prometheus & Grafana)
│   ├── reflector/            # 시크릿 복제
│   ├── reloader/             # Config 변경 시 Pod 자동 재시작
│   ├── sealed-secrets/       # 시크릿 암호화
│   └── sealed-secrets-web/   # Sealed Secrets 웹 UI
```
