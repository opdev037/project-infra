# 詳細部署指南

本文檔提供完整的部署步驟和最佳實踐。

## 環境準備

### 1. 檢查 Kubernetes 集群

```bash
# 檢查節點狀態
kubectl get nodes

# 應該看到 3 個 Ready 狀態的節點
# lan-test1 (master)
# lan-test2 (worker)
# lan-test3 (worker)
```

### 2. 檢查 Helm 安裝

```bash
helm version
# 應該顯示 v3.18.4 或更高版本
```

### 3. 檢查 Ingress Controller

```bash
kubectl get pods -n ingress-nginx
# 應該看到 ingress-nginx-controller Pod 處於 Running 狀態
```

## 部署前準備

### Step 1: Clone 倉庫

在 K8s Master 節點 (lan-test1) 執行：

```bash
cd ~
git clone https://github.com/opdev037/project-infra.git
cd project-infra/helm
```

### Step 2: 生成 Laravel APP_KEY

你有兩種方式生成 APP_KEY：

#### 方式 1: 使用 Docker（推薦）

```bash
docker run --rm opdev037/project-api php artisan key:generate --show
```

會輸出類似：`base64:abcdefghijklmnopqrstuvwxyz1234567890ABCD=`

#### 方式 2: 在本地 Laravel 項目

```bash
cd ~/api  # 你的本地 Laravel 項目
php artisan key:generate --show
```

### Step 3: 配置 Secret

編輯 `templates/secret.yaml`：

```bash
vim templates/secret.yaml
```

替換以下值：

```yaml
stringData:
  # 替換成步驟 2 生成的 APP_KEY
  app-key: "base64:你生成的KEY"
  
  # 數據庫配置（如果使用外部 MySQL）
  db-username: "your_db_user"
  db-password: "your_db_password"
  
  # Google OAuth 配置
  google-client-id: "你的Google Client ID"
  google-client-secret: "你的Google Client Secret"
```

### Step 4: 配置數據庫（可選）

如果需要在 K8s 內運行 MySQL，修改 `values.yaml`：

```yaml
mysql:
  enabled: true  # 改為 true
  storageSize: 20Gi  # 根據需求調整
```

如果使用外部 MySQL，確保：
1. MySQL 可以從 K8s 集群訪問
2. 在 `values.yaml` 中配置正確的 `DB_HOST`

## 部署應用

### 完整部署命令

```bash
# 在 project-infra/helm 目錄下執行
helm install japanese-learning . \
  --namespace web-app \
  --create-namespace \
  --debug
```

### 檢查部署狀態

```bash
# 等待 Pods 啟動（可能需要 1-2 分鐘）
watch kubectl get pods -n web-app

# 預期輸出：
# NAME                        READY   STATUS    RESTARTS   AGE
# api-xxxxxxxxx-xxxxx         1/1     Running   0          2m
# api-xxxxxxxxx-xxxxx         1/1     Running   0          2m
# frontend-xxxxxxxxx-xxxxx    1/1     Running   0          2m
# frontend-xxxxxxxxx-xxxxx    1/1     Running   0          2m
```

### 驗證 Services

```bash
kubectl get svc -n web-app

# 應該看到：
# api         ClusterIP   10.x.x.x    <none>   80/TCP
# frontend    ClusterIP   10.x.x.x    <none>   80/TCP
```

### 驗證 Ingress

```bash
kubectl get ingress -n web-app

# 應該看到：
# NAME                 CLASS   HOSTS                         ADDRESS        PORTS
# japanese-learning    nginx   outer-jenkins.gbboss.com      10.203.100.x   80, 443
```

## 訪問應用

### 1. 配置 DNS（如果需要）

確保域名 `outer-jenkins.gbboss.com` 解析到你的 Ingress Controller IP。

你可以查看 Ingress IP：

```bash
kubectl get ingress japanese-learning -n web-app -o wide
```

### 2. 測試訪問

```bash
# 測試健康檢查
curl http://outer-jenkins.gbboss.com/health

# 測試 API
curl http://outer-jenkins.gbboss.com/api/health

# 在瀏覽器訪問
http://outer-jenkins.gbboss.com
```

## 數據庫遷移

首次部署後，需要運行 Laravel 遷移：

```bash
# 獲取一個 backend Pod 名稱
POD_NAME=$(kubectl get pods -n web-app -l app=api -o jsonpath='{.items[0].metadata.name}')

# 執行遷移
kubectl exec -it $POD_NAME -n web-app -- php artisan migrate --force

# 如果需要 seed 測試數據
kubectl exec -it $POD_NAME -n web-app -- php artisan db:seed --force
```

## 更新部署

### 1. 更新 Docker 映像

當你推送新代碼到 GitHub，GitHub Actions 會自動構建新的 Docker 映像。

### 2. 重啟 Pods 拉取新映像

```bash
# 重啟 Backend Pods
kubectl rollout restart deployment/api -n web-app

# 重啟 Frontend Pods
kubectl rollout restart deployment/frontend -n web-app

# 檢查更新狀態
kubectl rollout status deployment/api -n web-app
kubectl rollout status deployment/frontend -n web-app
```

### 3. 使用 Helm 升級

如果修改了 Helm Chart 配置：

```bash
# 更新 Git 倉庫
cd ~/project-infra
git pull origin main

# 升級 Helm Release
cd helm
helm upgrade japanese-learning . -n web-app
```

## 擴展和優化

### 水平擴展

```bash
# 擴展 Backend 到 3 個副本
kubectl scale deployment/api --replicas=3 -n web-app

# 擴展 Frontend 到 3 個副本
kubectl scale deployment/frontend --replicas=3 -n web-app
```

### 資源調整

修改 `values.yaml` 中的資源限制：

```yaml
backend:
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
```

然後升級：

```bash
helm upgrade japanese-learning . -n web-app
```

## 監控和日誌

### 查看 Pod 日誌

```bash
# Backend 日誌
kubectl logs -f deployment/api -n web-app

# Frontend 日誌
kubectl logs -f deployment/frontend -n web-app

# 查看最近 100 行日誌
kubectl logs --tail=100 deployment/api -n web-app
```

### 查看 Events

```bash
kubectl get events -n web-app --sort-by='.lastTimestamp'
```

### 進入容器調試

```bash
# 進入 Backend 容器
kubectl exec -it deployment/api -n web-app -- /bin/sh

# 進入 Frontend 容器
kubectl exec -it deployment/frontend -n web-app -- /bin/sh
```

## 備份和恢復

### 備份數據庫

```bash
# 如果使用 K8s 內的 MySQL
kubectl exec -it mysql-0 -n web-app -- mysqldump -u root -p laravel > backup.sql

# 恢復
kubectl exec -i mysql-0 -n web-app -- mysql -u root -p laravel < backup.sql
```

### 備份 Helm 配置

```bash
# 導出當前配置
helm get values japanese-learning -n web-app > current-values.yaml

# 導出完整 manifest
helm get manifest japanese-learning -n web-app > current-manifest.yaml
```

## 故障排除

### Pod CrashLoopBackOff

```bash
# 查看崩潰原因
kubectl logs <pod-name> -n web-app --previous

# 查看詳細信息
kubectl describe pod <pod-name> -n web-app
```

### ImagePullBackOff

```bash
# 檢查映像名稱是否正確
kubectl describe pod <pod-name> -n web-app

# 在節點上手動拉取測試
docker pull opdev037/project-api:latest
```

### Service 無法訪問

```bash
# 檢查 Service 端點
kubectl get endpoints -n web-app

# 測試從 Pod 內訪問
kubectl run -it --rm debug --image=busybox -n web-app -- sh
wget http://api/health
```

### Ingress 503 錯誤

```bash
# 檢查 Backend Service 是否正常
kubectl get svc api -n web-app

# 檢查 Backend Pods 是否 Ready
kubectl get pods -n web-app -l app=api

# 檢查 Nginx Ingress Controller 日誌
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

## 安全加固

### 啟用 HTTPS

安裝 cert-manager 並配置 Let's Encrypt：

```bash
# 安裝 cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# 創建 ClusterIssuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: operationtgfc037@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
