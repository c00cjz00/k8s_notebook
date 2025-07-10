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


