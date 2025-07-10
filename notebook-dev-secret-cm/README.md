# k8s notebook

 Kubernetes/Jupyter 部署指令與 YAML 檔案的操作說明

---

## 沒有密碼
```bash
# 設定別名：將 kubectl 指令簡寫為 k
alias k=kubectl
# 自動補全功能支援 k 指令
complete -o default -F __start_kubectl k
# 將別名與補全設定永久寫入 bash 設定檔
echo "alias k=kubectl" >> ~/.bashrc # 永久加入別名設定
echo "complete -o default -F __start_kubectl k" >> ~/.bashrc # 永久加入補全設定
```

---

```bash
# 建立 jupyter 專屬 namespace
k apply -f jupyter-namespace.yaml

# 查看所有 namespace，確認是否成功建立
k get ns 
# 輸出示例：
# c00cjz00                Active   14m

# 將目前 kubectl 的預設 namespace 設定為 jupyter 專屬的 c00cjz00
k config set-context --current --namespace=c00cjz00

# 查看是否設定成功
k config view --minify | grep namespace
```

---

```bash
# 查看可用的儲存類別 (StorageClass)
k get sc

# 建立 PersistentVolumeClaim，為 Jupyter 提供持久化儲存空間
k apply -f pvc.yaml

# 確認 PVC 是否成功建立並綁定儲存卷
k get pvc
# 輸出示例：
# NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
# jupyter-pvc   Bound    pvc-xxxx...                                5Gi        RWO            csi-pgha       <unset>                 4m59s
```

---

```bash
# 部署 Jupyter 應用程式
k apply -f jupyter-deployment.yaml

# 確認 Deployment 是否成功執行
k get deployments.apps 
# 輸出示例：
# jupyter-deployment   1/1     1            1           15s

# 查看 Pod 狀態，確認是否執行中
k get po 
# 輸出示例：
# jupyter-deployment-xxxxxxx   1/1     Running   0          63s
```

---

```bash
# 建立對外服務（ClusterIP），讓內部集群可以存取 Jupyter
k apply -f jupyter-service.yaml

# 查看 service 狀態與 IP/port 設定
k get svc 
# 輸出示例：
# NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# jupyter-svc   ClusterIP   10.97.249.230   <none>        80/TCP    12s
```

---

```bash
# 建立 ingress 資源，透過網域名稱暴露 Jupyter 服務
k apply -f jupyter-ingress.yaml

# 查看 ingress 設定，確認對應的網址是否已建立
k get ingress 
# 輸出示例：
# NAME                        CLASS    HOSTS                                        ADDRESS   PORTS     AGE
# cm-acme-http-solver-xxxx    <none>   jupyter.c00cjz00.genai-staging.nchc.org.tw             80        17s
# jupyter-ingress             nginx    jupyter.c00cjz00.genai-staging.nchc.org.tw             80, 443   24s
```

---

這些步驟完成後，就可以透過 `https://jupyter.c00cjz00.genai-staging.nchc.org.tw`（或實際網域）從瀏覽器存取您的 Jupyter Notebook 服務。需要注意 ingress 是否正確配置 TLS 憑證與解析紀錄。若還需要設定 token、volume path 或 mount 位置，也可以補充提供 yaml 內容協助您檢查。



## 密碼補充
以下是你提供的指令，補上完整中文註解，整理成部署與密碼設定流程的說明：

---

```bash
# 套用 Secret 設定檔（目前內容為空的密碼）
k apply -f secret.yaml

# 檢查 Secret 是否成功建立，並確認名稱為 jupyter-secrets
k get secrets 
# 輸出範例：
# NAME              TYPE     DATA   AGE
# jupyter-secrets   Opaque   1      40s
```

---

```bash
# 套用 ConfigMap 設定檔，通常包含啟動腳本或環境設定檔
k apply -f configmap.yaml

# 檢查 ConfigMap 是否成功建立
k get configmaps 
# 輸出範例：
# NAME               DATA   AGE
# jupyter-scripts    1      11s         # 自己定義的腳本或設定檔
# kube-root-ca.crt   1      5m47s       # 系統預設的 CA 憑證 ConfigMap
```

---

```bash
# 部署或重新部署 Jupyter Deployment（含掛載 PVC、env、ConfigMap 等）
k apply -f jupyter-deployment.yaml
```

---

```bash
# 發現密碼是空的，需要重建 Secret
k delete secret jupyter-secrets
```

---

```bash
# 使用 CLI 直接建立新的 Secret，設定 Jupyter 登入密碼
kubectl create secret generic jupyter-secrets \
  --from-literal=JUPYTER_TOKEN=mysecuretoken \
  -n summerhill
```

---

```bash
# 查詢 Secret 中的 JUPYTER_TOKEN，解碼後可看到原始密碼
k get secret jupyter-secrets -o jsonpath="{.data.JUPYTER_TOKEN}" | base64 --decode

# 輸出：
# mysecuretoken
```

---

```bash
# 查看目前的 Pod 並檢查其 labels
k get pods -n summerhill --show-labels
# 找到對應的 app label 是 app=jupyter
```

```bash
# 根據 label 刪除 jupyter 的 Pod，Kubernetes 會自動依據 Deployment 重建新的 Pod
k delete pod -l app=jupyter -n summerhill
```

---

### ✅ 小結：

這樣做的目的是：

1. 設定並注入一個 Jupyter Notebook 的密碼。
2. 確保 Deployment 的容器可以正確讀取這個密碼（透過 Secret）。
3. 手動刪除舊 Pod，讓新的 Pod 在重建時載入最新的 Secret。

如果你有多個環境（如 staging、prod），這套流程也可以快速套用，只需換 namespace 與 token 值即可。需要我幫你整理成一個 shell script 流程也可以提出喔。
