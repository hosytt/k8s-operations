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
  --kubernetes-version=v1.34.3
# --apiserver-advertise-address=10.10.1.1
# --control-plane-endpoint=10.10.1.250:16443
```

## 4) Worker 노드 조인

`kubeadm init` 결과로 출력된 `join` 명령을 각 워커 노드에서 실행합니다.

```bash
sudo kubeadm join 10.10.1.250:16443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
# --apiserver-advertise-address <NODE_IP>
```
