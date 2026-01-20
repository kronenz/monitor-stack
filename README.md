# Kubernetes Monitoring & Logging Stack

GitOps (ArgoCD) 기반 모니터링 및 로깅 스택 배포 구성

## 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                               │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    Monitoring Stack (monitoring ns)               │   │
│  │  ┌────────────┐  ┌─────────────┐  ┌────────────────┐            │   │
│  │  │ Prometheus │  │ Alertmanager │  │ Grafana:30300  │            │   │
│  │  └────────────┘  └─────────────┘  └────────────────┘            │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Logging Stack (logging ns)                    │   │
│  │                                                                   │   │
│  │  ┌─────────────────┐                                             │   │
│  │  │  Fluent Bit     │ ─── K8s Logs ──┐                            │   │
│  │  │  (DaemonSet)    │ ─ HostPath ──┐ │     ┌──────────────────┐   │   │
│  │  └─────────────────┘              │ │     │   OpenSearch     │   │   │
│  │                                   ▼ ▼     │   (Single Node)  │   │   │
│  │                           ┌─────────────┐ │   Dashboards:    │   │   │
│  │                           │Data Prepper │→│   30601          │   │   │
│  │                           └─────────────┘ └──────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                        ArgoCD (argocd ns)                         │   │
│  │                    UI: 30080 (HTTP) / 30443 (HTTPS)               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## 구성 요소

| Component | Version | Description |
|-----------|---------|-------------|
| ArgoCD | stable | GitOps CD 도구 |
| kube-prometheus-stack | 81.0.1 | Prometheus + Grafana + Alertmanager |
| OpenSearch Operator | 2.8.0 | OpenSearch 클러스터 관리 |
| OpenSearch | 2.17.0 | 로그 저장소 |
| Fluent Operator | 3.5.0 | Fluent Bit CRD 컨트롤러 |
| Data Prepper | 0.3.1 | 로그 처리 파이프라인 |

## 빠른 시작

### 1. ArgoCD 설치

```bash
# monitoring 네임스페이스 정리 (기존 구성이 있는 경우)
kubectl delete namespace monitoring --ignore-not-found

# ArgoCD 설치
kubectl apply -k argocd/install/

# 초기 admin 비밀번호 확인
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2. ArgoCD UI 접속

- URL: `http://{NODE_IP}:30080`
- Username: `admin`
- Password: (위에서 확인한 비밀번호)

### 3. App of Apps 배포

```bash
kubectl apply -f argocd/applications/app-of-apps.yaml
```

이 명령으로 전체 스택이 자동 배포됩니다:
- monitoring-stack (kube-prometheus-stack)
- logging-opensearch (OpenSearch Operator + Cluster)
- logging-fluent-operator (Fluent Bit)
- logging-data-prepper (Data Prepper)
- log-pipelines (Fluent Bit CRD 파이프라인)

## 접속 정보

| Service | URL | Default Credentials |
|---------|-----|---------------------|
| ArgoCD | http://{NODE_IP}:30080 | admin / (secret) |
| Grafana | http://{NODE_IP}:30300 | admin / admin123! |
| OpenSearch Dashboards | http://{NODE_IP}:30601 | - |

## 디렉토리 구조

```
monitor-stack/
├── argocd/
│   ├── install/              # ArgoCD 설치 매니페스트
│   └── applications/         # ArgoCD Application 정의
├── manifests/
│   ├── monitoring/           # kube-prometheus-stack
│   ├── logging/
│   │   ├── opensearch/       # OpenSearch Operator + Cluster
│   │   ├── fluent-operator/  # Fluent Operator
│   │   └── data-prepper/     # Data Prepper
│   ├── log-pipelines/        # Fluent Bit CRD (Input/Parser/Filter/Output)
│   └── samples/              # 샘플 애플리케이션
└── docs/
    └── hostpath-log-collection-guide.md
```

## 로그 수집 파이프라인

### K8s Container Logs
```
/var/log/containers/*.log → Fluent Bit → Data Prepper → OpenSearch (k8s-logs-*)
```

### Custom HostPath Logs (Java)
```
/var/log/{namespace}/*.log → Fluent Bit → Data Prepper → OpenSearch (java-logs-*)
```

자세한 내용은 [HostPath 로그 수집 가이드](docs/hostpath-log-collection-guide.md)를 참조하세요.

## 커스터마이징

### Prometheus 설정 변경
`manifests/monitoring/overlays/production/values.yaml` 수정

### OpenSearch 클러스터 확장
`manifests/logging/opensearch/overlays/production/opensearch-cluster.yaml` 수정

### 새로운 로그 파이프라인 추가
`manifests/log-pipelines/base/cluster-inputs/` 디렉토리에 새 ClusterInput 추가

## 트러블슈팅

### ArgoCD Sync 실패

```bash
# Application 상태 확인
kubectl get applications -n argocd

# 상세 정보 확인
kubectl describe application monitoring-stack -n argocd
```

### Fluent Bit 로그 확인

```bash
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit -f
```

### OpenSearch 상태 확인

```bash
kubectl get opensearchcluster -n logging
kubectl logs -n logging -l app=opensearch
```
