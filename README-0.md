# Kubernetes 입문 가이드

쿠버네티스를 처음 접하는 사람을 위한 기초 문서입니다.  
이 문서의 목표는 아래 4가지를 한 번에 이해하는 것입니다.

1. Kubernetes가 왜 필요한지
2. 핵심 개념과 자주 쓰는 용어
3. 영어 용어를 어떻게 해석하면 되는지
4. 실무에서 가장 많이 쓰는 `kubectl` 명령어

---

## 1) Kubernetes란?

Kubernetes(쿠버네티스, 줄여서 K8s)는 **컨테이너 애플리케이션을 자동으로 배포/확장/복구**해 주는 오케스트레이션 플랫폼입니다.

쉽게 말해:

- 컨테이너를 수동으로 하나씩 띄우는 대신
- "원하는 상태"를 선언(YAML)하면
- Kubernetes가 그 상태를 계속 맞춰줍니다.

예:

- "웹 서버 3개를 항상 실행해 둬"라고 선언하면
- 하나가 죽어도 자동으로 다시 띄워 3개를 유지합니다.

---

## 2) 왜 Kubernetes를 쓰나?

### 장점

- **자동 복구(Self-healing)**: 장애 난 Pod를 자동 재생성
- **자동 확장(Scaling)**: 부하에 따라 Pod 수를 늘리거나 줄임
- **롤링 업데이트**: 무중단에 가까운 배포 가능
- **서비스 디스커버리**: 서비스 이름으로 내부 통신 가능
- **선언형 관리**: "어떻게"보다 "어떤 상태"인지 정의

### 단점/주의점

- 초기 학습 비용이 큼
- 네트워크/스토리지/보안까지 고려해야 함
- 클러스터 운영 자체도 하나의 시스템 운영 업무

---

## 3) 기본 구조(아키텍처) 한눈에 보기

- **클러스터(Cluster)**: Kubernetes 전체 환경
- **컨트롤 플레인(Control Plane)**: 클러스터 제어 두뇌
- **워커 노드(Worker Node)**: 실제 앱(Pod)이 실행되는 서버

### Control Plane 주요 구성요소

- **kube-apiserver**: 모든 요청의 진입점 (명령을 받는 API 서버)
- **etcd**: 클러스터 상태를 저장하는 Key-Value DB
- **kube-scheduler**: 어떤 노드에 Pod를 배치할지 결정
- **kube-controller-manager**: 상태를 목표 상태로 맞추는 컨트롤러 실행

### Worker Node 주요 구성요소

- **kubelet**: 노드에서 Pod 상태를 관리
- **container runtime**: 실제 컨테이너 실행(containerd 등)
- **kube-proxy**: 서비스 네트워크 라우팅 처리

---

## 4) 꼭 알아야 할 핵심 개념

### Pod

쿠버네티스에서 배포되는 가장 작은 단위입니다.  
보통 컨테이너 1개를 담고, 필요하면 여러 컨테이너를 한 Pod에 넣을 수도 있습니다.

### Deployment

Pod를 안정적으로 관리하는 리소스입니다.

- 원하는 Pod 개수 유지
- 업데이트 전략(롤링 업데이트) 관리
- 장애 시 자동 재생성

### Service

Pod 앞단의 고정된 접근 지점입니다.  
Pod IP는 바뀌기 쉬우므로, 서비스 이름으로 접근하게 만듭니다.

### 클러스터 내부 DNS (Service DNS)

쿠버네티스는 Service에 대해 내부 DNS 이름을 자동으로 만들어 줍니다.  
같은 클러스터 안에서는 IP 대신 DNS 이름으로 안정적으로 통신하는 것이 일반적입니다.

- 같은 네임스페이스에서 접근: `<service-name>`
- 다른 네임스페이스에서 접근: `<service-name>.<namespace>.svc.cluster.local`

예:

- `web-svc.default.svc.cluster.local`
- `prometheus-community-kube-prom-prometheus.monitoring.svc.cluster.local`

특히 Pod IP는 자주 바뀌므로, 내부 통신은 Service DNS를 기준으로 설계하는 것이 좋습니다.

### Namespace

클러스터 내부의 논리적 작업 공간입니다.  
팀/환경(dev, staging, prod) 분리에 자주 사용합니다.

### ConfigMap / Secret

- **ConfigMap**: 일반 설정값 저장
- **Secret**: 비밀번호/토큰 같은 민감정보 저장 (Base64 인코딩 기반)

### Ingress

외부 HTTP/HTTPS 요청을 클러스터 내부 서비스로 라우팅하는 규칙입니다.  
보통 Ingress Controller(예: ingress-nginx)와 함께 사용합니다.

### Resources (requests / limits)

Pod(정확히는 컨테이너)의 CPU/메모리 사용 기준을 정의하는 설정입니다.

- **requests**: 스케줄링 시 "최소 이 정도 자원은 필요"하다는 기준
- **limits**: 컨테이너가 사용할 수 있는 최대 자원 한도
- `requests`를 실제 사용량에 가깝게 설정하면 Pod 배치가 효율적이 되어 서버 리소스를 더 촘촘하게 활용할 수 있습니다.

---

## 5) 용어 해석 사전 (영어 -> 한국어)

- **Desired State**: 목표 상태 (원하는 상태)
- **Actual State**: 현재 상태
- **Reconcile**: 목표 상태와 현재 상태를 맞추는 과정
- **Manifest**: 리소스 정의 파일(YAML)
- **Rollout**: 배포 진행 과정
- **Rollback**: 이전 버전으로 되돌리기
- **Scale**: Pod 수 확장/축소
- **Drift**: 선언 상태와 실제 상태가 어긋난 상황
- **Taint / Toleration**: 특정 노드 스케줄 제한/허용 규칙
- **Affinity / Anti-affinity**: Pod 배치 선호/회피 규칙
- **Service Discovery**: 서비스 이름(DNS)으로 서로를 찾는 방식

---

## 6) YAML을 볼 때 읽는 순서

쿠버네티스 YAML은 처음엔 복잡해 보이지만 아래 순서로 보면 쉽습니다.

1. `apiVersion`: 어떤 API 그룹/버전인지
2. `kind`: 어떤 리소스인지(Deployment, Service 등)
3. `metadata`: 이름, 네임스페이스, 라벨
4. `spec`: 원하는 상태(핵심 설정)

예시:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 300m
              memory: 256Mi
```

해석:

- `web`라는 Deployment를 만들고
- `nginx:1.27` Pod를
- 항상 3개 유지한다는 의미입니다.

### Deployment + Service + Ingress(ingress-nginx) 연결 예제

아래 예제는 `web` 애플리케이션을 외부에서 `demo.local` 도메인으로 접근하게 만드는 가장 기본적인 흐름입니다.

사전 조건:

- 클러스터에 `ingress-nginx` 컨트롤러가 설치되어 있어야 합니다.
- 로컬 테스트 시 `/etc/hosts`에 Ingress 진입 IP를 매핑해야 할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: default
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

적용/검증:

```bash
# 1) 예제 파일 적용
kubectl apply -f web-example.yaml

# 2) 리소스 생성 확인
kubectl get deploy,svc,ing -n default

# 3) Ingress 주소 확인
kubectl get ingress web-ingress -n default

# 4) 앱 응답 확인
curl -H "Host: demo.local" http://<INGRESS_IP>/
```

해석:

- `Deployment`가 `web` Pod를 2개 유지합니다.
- `Service(web-svc)`는 **Ready 상태인 Pod**로만 트래픽을 전달합니다.
- `Ingress(web-ingress)`가 `demo.local` 요청을 `web-svc:80`으로 라우팅합니다.
- `readinessProbe`는 서비스 트래픽 수신 가능 여부를 판단하고, `livenessProbe`는 비정상 Pod를 재시작합니다.
- `resources.requests/limits`는 스케줄링 기준과 최대 사용량을 정의해 자원 과사용을 방지합니다.

---

## 7) 많이 사용하는 `kubectl` 명령어

아래 명령어만 익혀도 초반 운영/디버깅이 훨씬 쉬워집니다.

### 7-1. 현재 상태 조회

```bash
# 노드 목록
kubectl get nodes

# 네임스페이스 목록
kubectl get ns

# 특정 네임스페이스의 Pod 목록
kubectl get pods -n <namespace>

# 서비스 목록
kubectl get svc -n <namespace>

# 배포(Deployment) 목록
kubectl get deploy -n <namespace>
```

### 7-2. 상세 정보 확인(문제 분석)

```bash
# Pod 상세 상태 (이벤트 포함)
kubectl describe pod <pod-name> -n <namespace>

# Deployment 상세 상태
kubectl describe deploy <deploy-name> -n <namespace>

# 최근 이벤트 확인
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

### 7-3. 로그 확인

```bash
# Pod 로그 보기
kubectl logs <pod-name> -n <namespace>

# 실시간 로그 팔로우
kubectl logs -f <pod-name> -n <namespace>

# 다중 컨테이너 Pod일 때 컨테이너 지정
kubectl logs <pod-name> -c <container-name> -n <namespace>
```

### 7-4. 리소스 적용/삭제

```bash
# YAML 적용 (생성/수정)
kubectl apply -f <file-or-dir>

# 리소스 삭제
kubectl delete -f <file-or-dir>
```

### 7-5. 스케일/재시작/롤아웃

```bash
# Deployment Pod 수 조정
kubectl scale deploy <deploy-name> --replicas=3 -n <namespace>

# 롤아웃 상태 확인
kubectl rollout status deploy/<deploy-name> -n <namespace>

# 롤아웃 히스토리 확인
kubectl rollout history deploy/<deploy-name> -n <namespace>

# 롤아웃 재시작
kubectl rollout restart deploy/<deploy-name> -n <namespace>

# 이전 버전으로 롤백
kubectl rollout undo deploy/<deploy-name> -n <namespace>
```

### 7-6. 클러스터 내부 접속/테스트

```bash
# Pod 내부 셸 접속
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# 로컬 포트를 Pod/Service에 연결
kubectl port-forward pod/<pod-name> 8080:80 -n <namespace>
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
```

### 7-7. 파일/설정 관련

```bash
# 현재 컨텍스트 확인
kubectl config current-context

# 컨텍스트 목록 확인
kubectl config get-contexts

# 사용 가능한 API 리소스 목록
kubectl api-resources

# 리소스 YAML 출력(실제 적용 상태)
kubectl get deploy <deploy-name> -n <namespace> -o yaml
```

---

## 8) 초보자가 자주 겪는 문제와 보는 포인트

### Pod가 `CrashLoopBackOff`

- 먼저 로그 확인: `kubectl logs ...`
- 환경변수/시크릿/DB 연결정보 확인
- 이미지 태그 오타 여부 확인

### Pod가 `Pending`

- 노드 자원 부족(CPU/Memory) 여부
- PVC 바인딩 문제 여부
- 스케줄링 제약(taint, affinity) 여부

### 서비스가 안 붙음

- Service selector와 Pod label 일치 여부
- `targetPort`와 컨테이너 포트 일치 여부
- Ingress 규칙/호스트명/경로 확인

---

## 9) 처음 공부할 때 추천 순서

1. Pod -> Deployment -> Service
2. ConfigMap/Secret
3. Ingress
4. Volume/PVC
5. Helm
6. GitOps(Argo CD, Flux)

이 순서대로 가면 "왜 이런 도구가 필요한지"가 자연스럽게 연결됩니다.

---

## 10) 한 줄 정리

Kubernetes는 "컨테이너를 실행하는 도구"가 아니라,  
**원하는 상태를 선언하면 시스템이 그 상태를 자동으로 유지해 주는 운영 플랫폼**입니다.

