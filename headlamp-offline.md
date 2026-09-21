# Headlamp + Prometheus 離線安裝規劃書(airgap)

**前提**:《Kubespray 離線安裝 Kubernetes 規劃書 v3》(`kubespray-offline.md`)已完成——5 節點 Ready,`helm_enabled`/`metrics_server_enabled`/`prometheus_operator_crds_enabled`(CRD v0.88.1)/MetalLB/cert-manager 皆已就緒;《Nexus 部署規劃書 v2》的 `nexus.lab`(8081 via 443)、`registry.lab`(5000 via 443)已就緒,節點皆信任 Lab Root CA、DNS 指向 infra。

本文件只涵蓋 Headlamp 與其 Prometheus 依賴,**不含 NGF**(NGF 是給其他服務用的 Gateway,與 Headlamp 曝露方式無關)。Headlamp 以 MetalLB LoadBalancer 曝露純 HTTP,不做 TLS。

| 項目 | 內容 |
|---|---|
| Headlamp | 0.43.0(LoadBalancer,namespace `kube-system`) |
| kube-prometheus-stack | 81.6.9(operator v0.88.1,對齊 kubespray 已裝的 CRD;namespace `monitoring`) |
| Prometheus server image | quay.io/prometheus/prometheus:v3.9.1(以 1.5 枚舉結果為準) |
| CPU/Memory(列表、總覽) | metrics-server 提供,已由 kubespray 啟用,不需 Prometheus |
| CPU/Memory/Network/Filesystem(Pod detail 時序圖) | 需 Prometheus scrape kubelet 內建 cAdvisor 的 `container_*` 指標 |
| Headlamp auto-detect 關鍵 | Prometheus Service 加標籤 `headlamp-prometheus: "true"` |

---

# Part 1 — 網路環境(有網路的機器,例如 WSL)

## 1.1 前置

```bash
mkdir -p ~/headlamp-offline/{charts,images} && cd ~/headlamp-offline

helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## 1.2 抓 Headlamp chart + image

```bash
HL_VER=0.43.0
helm pull headlamp/headlamp --version ${HL_VER} -d charts
docker pull ghcr.io/headlamp-k8s/headlamp:v${HL_VER}
docker save ghcr.io/headlamp-k8s/headlamp:v${HL_VER} | gzip > images/headlamp-image.tar.gz
```

## 1.3 抓 kube-prometheus-stack chart

版本**必須**對齊叢集內已裝的 prometheus-operator CRD v0.88.1(kubespray 的 `prometheus_operator_crds_enabled` 只裝 CRD),否則 operator 起不來或行為不符預期。

```bash
KPS_VER=81.6.9
helm pull prometheus-community/kube-prometheus-stack --version ${KPS_VER} -d charts

helm show chart charts/kube-prometheus-stack-${KPS_VER}.tgz | grep appVersion   # 應為 0.88.1
```

> 若改用其他 chart 版本,務必先跑上面這行確認 `appVersion` 對得上叢集已裝的 CRD 版本,對不上就換版本或先升級 CRD(見附錄 B)。

## 1.4 落地 `prometheus-values.yaml`

只保留 Headlamp Pod detail 圖表需要的 kubelet/cAdvisor scrape,Grafana / Alertmanager / node-exporter / kube-state-metrics / 控制面 scrape / admission webhook 全部關閉。CRD 已由 kubespray 裝過,`crds.enabled` 必須設 `false`,否則 `helm install` 會因 CRD 已存在而衝突。

```bash
cat > ~/headlamp-offline/prometheus-values.yaml <<'EOF'
# CRD 已由 kubespray(prometheus_operator_crds_enabled)安裝 v0.88.1,不可重複安裝
crds:
  enabled: false

# ── 關閉 Pod detail 圖表用不到的元件 ──
grafana:
  enabled: false            # Headlamp 自己畫圖
alertmanager:
  enabled: false            # 無告警需求
nodeExporter:
  enabled: false            # 節點級指標,container_* 不需要
kubeStateMetrics:
  enabled: false            # 物件狀態指標,container_* 不需要
defaultRules:
  create: false             # 告警規則(依賴上面兩者,一併關閉)
windowsMonitoring:
  enabled: false

# 控制面 scrape:僅供監控 control-plane 本身,Pod 圖表不需要
kubeApiServer:
  enabled: false
kubeControllerManager:
  enabled: false
kubeScheduler:
  enabled: false
kubeProxy:
  enabled: false
kubeEtcd:
  enabled: false
coreDns:
  enabled: false

# ── 必須保留:cAdvisor 指標的唯一來源 ──
kubelet:
  enabled: true

prometheusOperator:
  admissionWebhooks:
    enabled: false          # 僅用於驗證 PrometheusRule,已關 defaultRules,省一顆 image
  image:
    registry: registry.lab
    repository: prometheus-operator/prometheus-operator
  prometheusConfigReloader:
    image:
      registry: registry.lab
      repository: prometheus-operator/prometheus-config-reloader

prometheus:
  service:
    labels:
      headlamp-prometheus: "true"     # ← Headlamp auto-detect 的第一順位比對標籤
  prometheusSpec:
    image:
      registry: registry.lab
      repository: prometheus/prometheus
    retention: 24h          # plugin 預設顯示範圍即 24h,再長也用不到
    resources:
      requests:
        cpu: 100m
        memory: 1Gi
      limits:
        memory: 2Gi
    # 儲存:預設 emptyDir(Pod 重啟即失去歷史)。要保留歷史請改用 storageSpec 掛 PVC。
EOF
```

> **`headlamp-prometheus: "true"` 是 auto-detect 能成功的關鍵。** Headlamp plugin 的偵測順序是:①帶 `headlamp-prometheus=true` 的 **Pod** → ②帶該標籤的 **Service** → ③帶 `app.kubernetes.io/name=prometheus` 的 Pod → ④帶 `app.kubernetes.io/name=prometheus,app.kubernetes.io/component=server` 的 Service。kube-prometheus-stack 建立的 Service 不帶第④項標籤,因此顯式加自訂標籤讓偵測在第②步就命中。

## 1.5 枚舉並抓 Prometheus 相關 image

```bash
cd ~/headlamp-offline
helm template prom charts/kube-prometheus-stack-${KPS_VER}.tgz -n monitoring \
  -f prometheus-values.yaml \
  | grep -E '^\s+image:' | awk '{print $2}' | tr -d '"' | sort -u
```

以上述 values 為例,應只剩三顆:

```bash
docker pull quay.io/prometheus-operator/prometheus-operator:v0.88.1
docker pull quay.io/prometheus-operator/prometheus-config-reloader:v0.88.1
docker pull quay.io/prometheus/prometheus:v3.9.1        # tag 以枚舉結果為準,不同版本可能不同

docker save \
  quay.io/prometheus-operator/prometheus-operator:v0.88.1 \
  quay.io/prometheus-operator/prometheus-config-reloader:v0.88.1 \
  quay.io/prometheus/prometheus:v3.9.1 | gzip > images/prometheus-images.tar.gz
```

## 1.6 落地 `headlamp-values.yaml`

```bash
cat > ~/headlamp-offline/headlamp-values.yaml <<'EOF'
image:
  registry: registry.lab
  repository: headlamp-k8s/headlamp
  tag: v0.43.0            # 必須 pin,不指定會用 chart AppVersion,與推送 tag 不一致即 ImagePullBackOff
  pullPolicy: IfNotPresent
service:
  type: LoadBalancer
  port: 80
serviceAccount:
  create: true
clusterRoleBinding:
  create: true
  clusterRoleName: cluster-admin
EOF
```

## 1.7 打包搬運

| 項目 | 檔案 |
|---|---|
| Headlamp chart | `charts/headlamp-0.43.0.tgz` |
| kube-prometheus-stack chart | `charts/kube-prometheus-stack-81.6.9.tgz` |
| Headlamp image | `images/headlamp-image.tar.gz` |
| Prometheus 相關 image(3 顆) | `images/prometheus-images.tar.gz` |
| Headlamp values | `headlamp-values.yaml` |
| Prometheus values | `prometheus-values.yaml` |

```bash
cd ~/headlamp-offline
tar czf headlamp-prometheus-bundle.tar.gz charts/ images/ headlamp-values.yaml prometheus-values.yaml
```

把 `headlamp-prometheus-bundle.tar.gz` 搬到 infra(`10.0.0.10`)。

---

# Part 2 — airgap 環境:灌入 Nexus、部署、驗證

## 2.0 前置確認

```bash
getent hosts registry.lab nexus.lab      # 皆回 10.0.0.10
kubectl get nodes -o wide                # 5 Ready
kubectl -n metallb-system get pods       # controller + speaker Running(LoadBalancer 依賴它)
kubectl get pods -n kube-system -l k8s-app=metrics-server   # Running(CPU/Memory 列表依賴它)
```

## 2.1 上傳 image 到 docker-hosted【infra】

```bash
tar xzf headlamp-prometheus-bundle.tar.gz -C ~/headlamp-offline && cd ~/headlamp-offline

push_tars() {   # docker load 後去原始 host 前綴,重標 registry.lab 推送
  for t in "$@"; do docker load -i "$t" | sed -n 's/^Loaded image: //p'; done |
  while read -r img; do
    first="${img%%/*}"
    if [[ "$first" == *.* || "$first" == *:* || "$first" == "localhost" ]]; then path="${img#*/}"; else path="$img"; fi
    docker tag "$img" "registry.lab/$path" && docker push "registry.lab/$path"
  done
}

push_tars images/headlamp-image.tar.gz
push_tars images/prometheus-images.tar.gz
```

驗證:

```bash
curl -s 'https://nexus.lab/service/rest/v1/search?repository=docker-hosted&name=headlamp' | grep -c '"name"'      # > 0
curl -s 'https://nexus.lab/service/rest/v1/search?repository=docker-hosted&name=prometheus' | grep -c '"name"'    # > 0
docker pull registry.lab/headlamp-k8s/headlamp:v0.43.0 && echo PULL_OK
```

## 2.2 上傳 chart 到 helm-hosted【infra】

```bash
for c in charts/*.tgz; do
  curl -s -o /dev/null -w "%{http_code} $c\n" http://localhost:8081/repository/helm-hosted/ --upload-file "$c"
done   # 皆應 200/201
```

## 2.3 加 Nexus helm repo【node1】

```bash
helm repo add nexus https://nexus.lab/repository/helm-hosted/
helm repo update
```

把 `headlamp-values.yaml`、`prometheus-values.yaml` 從 infra 傳到 node1(`scp root@10.0.0.10:~/headlamp-offline/*.yaml .`)。

## 2.4 映像枚舉預檢【node1】

安裝前先確認 chart 內所有 image 都指向 `registry.lab`,避免 `ImagePullBackOff`:

```bash
helm template headlamp nexus/headlamp --version 0.43.0 -n kube-system \
  --set image.registry=registry.lab --set image.repository=headlamp-k8s/headlamp --set image.tag=v0.43.0 \
  | grep -E 'image:' | sort -u

helm template prom nexus/kube-prometheus-stack --version 81.6.9 -n monitoring \
  -f prometheus-values.yaml \
  | grep -E 'image:' | sort -u
```

若出現 `docker.io`/`quay.io`/`ghcr.io` 等外部位址,回 infra 依 2.1 補推對應 image 後再繼續。

## 2.5 安裝 Headlamp【node1】

```bash
helm install headlamp nexus/headlamp --version 0.43.0 -n kube-system -f headlamp-values.yaml

kubectl create serviceaccount headlamp-admin -n kube-system
kubectl create clusterrolebinding headlamp-admin \
  --serviceaccount=kube-system:headlamp-admin --clusterrole=cluster-admin
```

## 2.6 驗證與登入【node1】

```bash
kubectl -n kube-system rollout status deploy/headlamp --timeout=3m
kubectl -n kube-system get svc headlamp     # EXTERNAL-IP 為 MetalLB 位址池 IP(非 <pending>)
kubectl create token headlamp-admin -n kube-system          # 臨時 token(約 1h)
```

瀏覽器開 `http://<EXTERNAL-IP>`,貼 token 登入。此時 Pod 列表與 cluster overview 的用量數字已由 **metrics-server** 提供;Pod detail 頁上方的 CPU/Memory/Network/Filesystem **時序圖**另需 Prometheus,見 2.7–2.9。

長期 token:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: headlamp-admin-token
  namespace: kube-system
  annotations: { kubernetes.io/service-account.name: headlamp-admin }
type: kubernetes.io/service-account-token
EOF
kubectl -n kube-system get secret headlamp-admin-token -o jsonpath='{.data.token}' | base64 -d; echo
```

## 2.7 安裝 Prometheus【node1】

```bash
helm install prom nexus/kube-prometheus-stack --version 81.6.9 \
  --create-namespace -n monitoring -f prometheus-values.yaml

kubectl -n monitoring wait --for=condition=Ready pod -l app.kubernetes.io/name=prometheus --timeout=5m
kubectl -n monitoring get pods                              # 應僅 operator + prometheus,無 grafana/alertmanager
kubectl -n monitoring get svc -l headlamp-prometheus=true   # 標籤確實存在(auto-detect 靠它)
```

## 2.8 驗證 cAdvisor 指標有資料【node1】

```bash
kubectl -n monitoring port-forward svc/prom-kube-prometheus-stack-prometheus 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query?query=up{job="kubelet"}' | grep -o '"value"' | head -1
curl -s 'http://localhost:9090/api/v1/query?query=container_cpu_usage_seconds_total' | head -c 200
kill %1
```

`container_cpu_usage_seconds_total` 有回傳資料,才代表 Headlamp 圖表會有值。

## 2.9 Headlamp 端啟用 auto-detect

瀏覽器 Headlamp UI → **Settings → Plugins → Prometheus**:

1. **Enable metrics** 開啟。
2. **Auto-detect** 保持開啟(不需填 Prometheus Service Address)。
3. 回到任一 Pod 的 detail 頁,上方應出現 CPU / Memory / Network / Filesystem 四張圖。

> 設定為每個瀏覽器各自保存,換瀏覽器或換人登入需重設一次。若 auto-detect 仍失敗,先用 `kubectl -n monitoring get svc -l headlamp-prometheus=true` 確認標籤在,再確認登入 token 有列出全叢集 Pod/Service 的權限(auto-detect 會呼叫 `/api/v1/services?labelSelector=...`;`headlamp-admin` 綁 cluster-admin 即滿足)。要手動指定時,關閉 auto-detect 並填 `monitoring/prom-kube-prometheus-stack-prometheus:9090`(格式為 `namespace/service-name:port`)。

---

## 附錄 A — 常見故障排除

| 症狀 | 原因 | 處理 |
|---|---|---|
| 節點拉映像 `x509 unknown authority` | 節點未信任 Lab CA | 依 kubespray-offline.md 2.0 補做後重跑 |
| 節點拉映像 404 / manifest unknown | image 未推入 docker-hosted,或 tag 不符 | infra 依 2.1 補推;`curl -s https://nexus.lab/service/rest/v1/search?repository=docker-hosted&name=<name>` 查證 |
| `helm repo add nexus` 憑證錯誤 | node1 未信任 Lab CA | 依 kubespray-offline.md 2.0 節點步驟 |
| Helm install/upgrade `ImagePullBackOff` | tag 未 pin 或 chart 含未推送的 image | 2.4 枚舉補推;values 明確 pin tag(1.6/1.4) |
| `helm install kube-prometheus-stack` CRD 衝突 | chart 又裝一次 CRD | values 設 `crds.enabled: false`(CRD 已由 kubespray 提供) |
| LoadBalancer `EXTERNAL-IP: <pending>` | MetalLB 未就緒或位址池耗盡/衝突 | `kubectl -n metallb-system get pods,ipaddresspool` |
| Headlamp Pod detail 無圖表 / 顯示無法偵測 Prometheus | 叢集內沒有 Prometheus server(CRD ≠ server) | 依 2.7 安裝 kube-prometheus-stack |
| Prometheus 裝了但 auto-detect 仍失敗 | Service 缺 `headlamp-prometheus=true` 標籤,或登入 token 權限不足 | `kubectl -n monitoring get svc -l headlamp-prometheus=true` 確認標籤;token 需能列全叢集 svc/pod;必要時關 auto-detect 手動填 `namespace/service:port`(2.9) |
| 圖表有框但無資料 | kubelet/cAdvisor 未被 scrape | 確認 values 的 `kubelet.enabled: true`;`container_cpu_usage_seconds_total` 查詢須有回傳(2.8) |
| `kubectl top nodes/pods` 無資料但 Headlamp 列表卻有數字 / 反之 | 兩者資料來源不同:列表數字靠 metrics-server,Pod detail 時序圖靠 Prometheus | 分開排查:metrics-server 見 kubespray-offline.md 2.6;Prometheus 見本文 2.8 |

## 附錄 B — 升級 / 更新

Headlamp、kube-prometheus-stack 皆為 helm 安裝(非 kubespray addon),更新流程固定為:

```bash
# 1) 網路環境:新版 chart helm pull,新版 image docker pull + save(同 Part 1,改版本號)
# 2) infra:新版 image 推 docker-hosted(2.1),新版 chart 傳 helm-hosted(2.2)
# 3) node1:
helm repo update
helm upgrade headlamp nexus/headlamp --version <new> -f headlamp-values.yaml   # tag 仍須明確 pin
helm upgrade prom nexus/kube-prometheus-stack --version <new> -f prometheus-values.yaml
```

> 若之後升級 kubespray 動到 prometheus-operator CRD 版本,kube-prometheus-stack 版本要重新核對 `appVersion`(1.3),不對齊會導致 operator 無法辨識新版 CRD 欄位。CRD 本身的升級走 kubespray `cluster.yml --tags=prometheus_operator_crds`,不透過 helm(values 已設 `crds.enabled: false`)。

---

*本規劃書結束。叢集建置見《Kubespray 離線安裝 Kubernetes 規劃書 v3》,Nexus 供應鏈見《Nexus 部署規劃書 v2》。*
