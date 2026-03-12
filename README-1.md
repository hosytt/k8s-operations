# Kubernetes Node Setup (Manual)

수동으로 Kubernetes 노드를 구성할 때 사용하는 설치/초기화 가이드입니다.

## 1) 사전 준비

- OS: Ubuntu 계열
- root 권한(또는 `sudo`) 필요
- 노드 내부 IP를 환경변수로 지정

```bash
export NODE_IP="<노드 내부 IP>"
```

## 2) 노드 공통 설치 (Control Plane / Worker)

아래 스크립트를 각 노드에서 실행합니다.

```bash
#!/usr/bin/env bash

# 패키지 인덱스 최신화 및 보안 업데이트 반영
apt update
apt upgrade -y

# kubelet 요구사항: swap 비활성화(재부팅 후에도 유지)
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# crictl이 containerd 소켓을 사용하도록 지정
cat >/etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# Kubernetes 네트워킹에 필요한 커널 파라미터 설정
cat >/etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# 컨테이너/브릿지/iSCSI 관련 모듈을 부팅 시 자동 로드
cat >/etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
iscsi_tcp
EOF
modprobe overlay
modprobe br_netfilter
modprobe iscsi_tcp

# 컨테이너 런타임 + 저장소 연동 패키지 설치
apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  apt-transport-https \
  containerd \
  open-iscsi

# 기존 containerd 설정 백업 후 기본 설정 재생성
if [ -f /etc/containerd/config.toml ]; then
  mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
fi
containerd config default > /etc/containerd/config.toml

# kubelet과 cgroup 드라이버 일치(SystemdCgroup=true)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
# CNI 바이너리 경로를 기본값에서 /opt/cni/bin 으로 변경
sed -i 's|bin_dir = "/usr/lib/cni"|bin_dir = "/opt/cni/bin"|g' /etc/containerd/config.toml
systemctl restart containerd

# Kubernetes 공식 APT 저장소 키 등록
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Kubernetes v1.34 저장소 추가
cat >/etc/apt/sources.list.d/kubernetes.list <<EOF
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
EOF
chmod 644 /etc/apt/sources.list.d/kubernetes.list

# kubelet/kubeadm/kubectl 설치 및 버전 고정
apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# 핵심 서비스 부팅 시 자동 시작
systemctl enable --now kubelet
systemctl enable --now containerd
systemctl enable --now iscsid

# 기존 kubelet 추가 인자 백업/정리 (Bare metal server 인 경우)
if [ -f /etc/default/kubelet ]; then
  cp /etc/default/kubelet /etc/default/kubelet.bak.$(date +%Y%m%d_%H%M%S)
  sed -i '/^KUBELET_EXTRA_ARGS=/d' /etc/default/kubelet
fi

# 노드가 올바른 내부 IP로 등록되도록 고정
echo "KUBELET_EXTRA_ARGS=\"--node-ip=${NODE_IP}\"" >> /etc/default/kubelet

# kubelet 재시작으로 설정 반영
systemctl daemon-reload
systemctl restart kubelet
```

## 3) Control Plane 초기화 (master)

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --kubernetes-version=v1.34.5
# Bare metal 환경에서 각 마스터의 실제 내부 IP를 명시할 때 사용
# --apiserver-advertise-address=10.10.1.1
# keepalived + haproxy 구성으로 VIP를 운영하는 경우, VIP:PORT를 지정
# --control-plane-endpoint=10.10.1.250:16443
```

### 3-1) Helm 설치 (master)

```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## 4) Worker 노드 조인

`kubeadm init` 결과로 출력된 `join` 명령을 각 워커 노드에서 실행합니다.

```bash
sudo kubeadm join 10.10.1.250:16443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
# --apiserver-advertise-address <NODE_IP>
```

## 5) CNI 설정

Control Plane 초기화 후 CNI 플러그인을 설치해야 노드가 `Ready` 상태가 됩니다.  
`kubeadm init`에서 지정한 `--pod-network-cidr` 값과 CNI의 IP 대역이 일치해야 합니다.

Calico는 `infra/.manual-install/calico/values.yaml`을 기준으로 설치합니다.
해당 파일의 `cidr`가 `192.168.0.0/16`으로 맞춰져 있는지 확인합니다.

```bash
# control plane 노드에서 실행
cd infra/.manual-install/calico

helm repo add projectcalico https://projectcalico.docs.tigera.io/charts
helm repo update

helm upgrade --install calico projectcalico/tigera-operator \
  --version 3.31.2 \
  -n tigera-operator --create-namespace \
  -f values.yaml
```

## 6) Argo CD 설치

Calico 설치 완료 후 Argo CD를 설치합니다.  
설정은 `infra/.manual-install/argo-cd/values.yaml`을 사용합니다.

```bash
cd infra/.manual-install/argo-cd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --version 9.1.7 \
  -n argo-cd --create-namespace \
  -f values.yaml
```

초기 관리자 비밀번호 확인:

```bash
kubectl -n argo-cd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Argo CD에 Git 저장소를 추가할 때는 `infra/argo-cd/repository.yaml`을 적용합니다.

```bash
kubectl apply -f infra/argo-cd/repository.yaml
```

GitOps 부트스트랩은 `infra/app-project.yaml`부터 적용합니다.

```bash
kubectl apply -f infra/app-project.yaml
```

임시로 웹페이지 접근이 필요하면 포트 포워딩을 사용합니다.

```bash
kubectl port-forward service/argocd-server -n argo-cd --address 0.0.0.0 8080:80
```

## 7) ingress-nginx 설치

Argo CD가 준비된 뒤 `infra/ingress-nginx/application.yaml`을 적용해
ingress-nginx를 설치합니다.

```bash
kubectl apply -f infra/ingress-nginx/application.yaml
```

Argo CD Ingress 리소스는 `infra/ingress-nginx/resources/argocd.yaml`에 있으며,
필요 시 `kustomization.yaml`에 포함하거나 직접 적용합니다.

## 8) metrics-server 설치

HPA 등 메트릭 기반 기능을 사용하려면 metrics-server가 필요합니다.

```bash
kubectl apply -f infra/metrics-server/application.yaml
```

## 9) HPA 예제

metrics-server가 정상 동작하는 상태에서 아래 예제를 적용합니다.
HPA 리소스는 `infra/hpa-demo/`에 있습니다.

```bash
kubectl apply -f infra/hpa-demo/application.yaml
```

부하 테스트(스케일링 확인) 예시:

```bash
# 간단한 부하 생성용 Pod 실행 (hpa-demo 네임스페이스)
kubectl run -it --rm loadgen -n hpa-demo --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c 'while true; do wget -q -O- http://hpa-demo; done'
```
