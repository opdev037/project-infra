# Japanese Learning - Infrastructure

Kubernetes 部署配置和 Helm Chart。

## 技術棧

- **Kubernetes**: v1.29+
- **Helm**: v3.18+
- **Ingress Controller**: Nginx Ingress
- **Storage**: NFS Client Provisioner

## 倉庫結構

```
infra/
├── helm/
│   ├── Chart.yaml              # Helm Chart 元數據
│   ├── values.yaml             # 配置值
│   └── templates/              # Kubernetes 資源模板
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       ├── ingress.yaml
│       └── secret.yaml
├── README.md                    # 本文件
└── DEPLOYMENT.md                # 詳細部署指南
```

## 快速開始

### 前置要求

1. Kubernetes 集群已運行
2. Helm 3 已安裝
3. kubectl 已配置並可訪問集群
4. Nginx Ingress Controller 已安裝

### 部署步驟

#### 1. Clone 倉庫

```bash
git clone https://github.com/opdev037/project-infra.git
cd project-infra/helm
```

#### 2. 修改 Secret

編輯 `templates/secret.yaml`，替換以下值：

```bash
# 生成 Laravel APP_KEY
docker run --rm opdev037/project-api php artisan key:generate --show

# 然後在 secret.yaml 中替換：
# - app-key
# - db-username
# - db-password
# - google-client-id
# - google-client-secret
```

#### 3. 安裝應用

```bash
helm install japanese-learning . \
  --namespace web-app \
  --create-namespace
```

#### 4. 驗證部署

```bash
# 檢查 Pods
kubectl get pods -n web-app

# 檢查 Services
kubectl get svc -n web-app

# 檢查 Ingress
kubectl get ingress -n web-app
```

#### 5. 訪問應用

訪問: http://outer-jenkins.gbboss.com

## 配置說明

### values.yaml 主要配置

| 參數 | 說明 | 默認值 |
|------|------|--------|
| `global.domain` | 應用域名 | `outer-jenkins.gbboss.com` |
| `global.namespace` | K8s 命名空間 | `web-app` |
| `backend.replicaCount` | Backend 副本數 | `2` |
| `frontend.replicaCount` | Frontend 副本數 | `2` |
| `backend.image.repository` | Backend 映像 | `opdev037/project-api` |
| `frontend.image.repository` | Frontend 映像 | `opdev037/project-frontend` |

### 自定義配置

創建 `custom-values.yaml`：

```yaml
backend:
  replicaCount: 3
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
```

使用自定義配置安裝：

```bash
helm install japanese-learning . \
  --namespace web-app \
  -f custom-values.yaml
```

## 管理命令

### 升級部署

```bash
helm upgrade japanese-learning . \
  --namespace web-app
```

### 查看狀態

```bash
helm status japanese-learning -n web-app
```

### 查看配置值

```bash
helm get values japanese-learning -n web-app
```

### 卸載應用

```bash
helm uninstall japanese-learning -n web-app
```

## 故障排除

### Pods 無法啟動

```bash
# 查看 Pod 詳情
kubectl describe pod <pod-name> -n web-app

# 查看 Pod 日誌
kubectl logs <pod-name> -n web-app
```

### Ingress 無法訪問

```bash
# 檢查 Ingress 配置
kubectl describe ingress japanese-learning -n web-app

# 檢查 Nginx Ingress Controller
kubectl get pods -n ingress-nginx
```

### 映像拉取失敗

```bash
# 檢查映像是否存在
docker pull opdev037/project-api:latest
docker pull opdev037/project-frontend:latest
```

## 相關倉庫

- [Backend API (Laravel)](https://github.com/opdev037/project-api)
- [Frontend (Vue 3)](https://github.com/opdev037/project-frontend)

## 架構圖

```
Internet
    │
    ↓
Nginx Ingress Controller
    │
    ├─→ /api → Backend Service → Backend Pods (Laravel API)
    │
    └─→ /    → Frontend Service → Frontend Pods (Vue 3 + Nginx)
```

## License

MIT
