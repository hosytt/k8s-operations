# Helm -> YAML 예제

이 문서는 "Helm이 결국 YAML을 만들어 적용한다"는 개념을 빠르게 확인하기 위한 최소 예제입니다.

## 1) values.yaml (입력값)

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"

service:
  port: 80
```

## 2) templates/deployment.yaml (Helm 템플릿)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-web
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-web
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-web
    spec:
      containers:
        - name: web
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          ports:
            - containerPort: {{ .Values.service.port }}
```

## 3) helm template 결과 (렌더링된 YAML)

`helm template demo ./mychart -f values.yaml` 실행 시 아래처럼 변수 치환된 YAML이 생성됩니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-web
  template:
    metadata:
      labels:
        app: demo-web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
```

## 4) 핵심 요약

- `helm template`: YAML 렌더링만 수행 (클러스터 적용 안 함)
- `helm install/upgrade`: 렌더링 + 클러스터 적용까지 수행
- 따라서 Helm은 "YAML을 직접 다 쓰는 부담을 줄여주는 템플릿/패키징 도구"입니다.
