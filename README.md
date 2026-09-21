# Headlamp + Prometheus Ansible 部署(airgap)

用 Ansible 在**已經跑起來的 airgap Kubernetes 叢集**上安裝 Headlamp 與 kube-prometheus-stack(供 Headlamp Pod detail 頁 CPU/Memory/Network/Filesystem 時序圖 auto-detect 用)。只做「從 Nexus 消費素材、跑 helm/kubectl」這段,**不會**幫你抓素材或推上 Nexus。

完整手動版流程與每個決策的理由見 `headlamp-offline.md`;叢集本身的建置見 `kubespray-offline.md`。

---

## Airgap 前置需求(執行前必須已經成立)

### 叢集側(依 kubespray-offline.md 應已完成)

- 5 節點 Ready,`helm_enabled: true`、`metrics_server_enabled: true`、`prometheus_operator_crds_enabled: true`(CRD 版本 **v0.88.1**)、MetalLB、cert-manager 皆已就緒。
- node1(inventory 預設 `10.0.0.11`)是 control-plane,存在 `/etc/kubernetes/admin.conf`,`helm`/`kubectl` 二進位已在 PATH 上。
- 節點 DNS 指向 infra(`10.0.0.10`),`getent hosts registry.lab nexus.lab` 解析得到;節點已信任 Lab Root CA。
- 你的 ansible 執行機能以 **root** SSH 免密碼登入 node1(`ssh-copy-id` 已做,同 kubespray-offline.md 2.0 的假設)。

### Nexus 側(你自己處理,這個 playbook 不會做)

以下東西必須**已經**在 Nexus 上,對應 `headlamp-offline.md` Part 1 + 2.1/2.2:

| repo | 內容 | 版本 |
|---|---|---|
| `helm-hosted` | `headlamp` chart | 0.43.0 |
| `helm-hosted` | `kube-prometheus-stack` chart | 81.6.9(operator v0.88.1,必須對齊叢集已裝的 CRD 版本) |
| `docker-hosted`(經 `registry.lab`) | `headlamp-k8s/headlamp` | `v0.43.0` |
| `docker-hosted` | `prometheus-operator/prometheus-operator` | `v0.88.1` |
| `docker-hosted` | `prometheus-operator/prometheus-config-reloader` | `v0.88.1` |
| `docker-hosted` | `prometheus/prometheus` | `v3.9.1`(或依 `helm template` 枚舉結果) |

版本沒對齊(尤其 tag 沒 pin 到已推送的版本、chart 版本沒對齊已裝 CRD)是這個環境最常見的失敗原因——playbook 的 `headlamp`/`prometheus` tag 裡各有一段「映像枚舉預檢」,任何 image 沒指向 `registry.lab` 會直接 fail,但**不會**幫你檢查版本是否真的存在,那個要靠你在 2.1/2.2 補推。

### 執行機本身

- 有 `ansible-playbook`(不需要裝 `kubernetes.core` 或任何額外 collection——這個 role 刻意只用 `command` 呼叫 kubectl/helm)。
- 能 SSH 到 node1。

---

## 換環境時大概率要改的變數

這份 role 的變數集中在三個地方,越下面越「這台特定叢集才有的值」:

### `inventory/hosts` ——連線資訊

```ini
node1 ansible_host=10.0.0.11 ansible_user=root
```

- `ansible_host`:node1 的實際 IP。換叢集、換 control-plane 節點、node1 改用其他 IP 都要改這裡。
- `ansible_user`:預設假設 root 直連(比照 kubespray-offline.md 的做法)。若之後改成一般帳號 + sudo,`ansible.cfg` 已開 `become = True`,不用改 site.yml,只要改這行的 user。

### `inventory/group_vars/all.yml` ——Nexus/registry 環境事實

```yaml
nexus_helm_repo_url: "https://nexus.lab/repository/helm-hosted/"
registry_host: "registry.lab"
```

- 這兩個值來自《Nexus 部署規劃書 v2》。如果你的 Nexus 主機名稱、repo 名稱、或走的 port/協定不一樣(例如不是 `helm-hosted`,或沒有走 443 TLS 而是別的 port),兩個都要改。
- `registry_host` 同時決定映像枚舉預檢會拿什麼字串去比對(`item.startswith(registry_host ~ '/')`),改錯會導致預檢誤判成失敗。

### `roles/kubectl/headlamp/defaults/main.yml` ——版本與部署細節

最容易因為「跟這次環境不一樣」而要改的幾個:

| 變數 | 何時要改 |
|---|---|
| `headlamp_kubeconfig`(預設 `/etc/kubernetes/admin.conf`) | 不是用 kubeadm/kubespray 佈署(例如 RKE2、k3s)時路徑不同;或 node1 上有另外準備給非 root 用的 kubeconfig。 |
| `headlamp_chart_version` / `headlamp_image_tag` | 這兩個**必須**跟你實際推上 Nexus 的版本一致,不然直接 `ImagePullBackOff` 或映像枚舉預檢 fail。升版時要同步改。 |
| `prometheus_chart_version` | **必須**對齊叢集內已裝的 prometheus-operator CRD 版本(目前 v0.88.1 對應 81.6.9)。CRD 版本變了(例如之後升級 kubespray),這裡也要跟著換,換之前先跑 `helm show chart <chart> --version <ver> \| grep appVersion` 確認對得上。 |
| `headlamp_service_type`(預設 `LoadBalancer`) | 沒有 MetalLB 或不想用 LoadBalancer 時,改成 `NodePort`/`ClusterIP`,但要自己另外處理曝露方式(這個 role 沒有處理 Ingress/Gateway)。 |
| `headlamp_admin_clusterrole`(預設 `cluster-admin`) | 想要限縮 Headlamp 權限而非給全叢集管理權時,先建好對應的 ClusterRole 再改這裡指過去。 |
| `prometheus_resources_requests_*` / `prometheus_resources_limits_*` | 節點資源比這次規劃(100m/1Gi request、2Gi limit)更緊或更寬裕時調整,避免 Pending 或浪費。 |
| `headlamp_namespace` / `prometheus_namespace` | 跟其他叢集慣例(namespace 命名規則)衝突時改。 |
| `headlamp_show_token` / `headlamp_print_temp_token` | 預設都關閉(避免 cluster-admin 憑證印進 ansible log)。要在跑完直接拿到明文 token 才打開,注意這樣 token 會出現在終端機輸出與 ansible log 裡。 |
| `headlamp_kubectl_bin` / `headlamp_helm_bin`(預設 `kubectl`/`helm`,吃 PATH) | role 用 `command` 呼叫,不走 login shell,PATH 可能跟你互動 SSH 看到的不同。**第一次跑前**先 `ssh root@<node1_ip> 'command -v kubectl helm'` 確認兩個都有回傳路徑,沒有就改這兩個變數成絕對路徑。 |
| `nexus_helm_repo_name`(預設 `nexus`) | 只有當同一台 ansible 執行機**未來要對接多個不同 Nexus 環境**時才需要改名,避免本地 helm repo 紀錄撞名(換 `nexus_helm_repo_url` 前記得先 `helm repo remove nexus` 或改名)。 |
| `headlamp_admin_sa`(預設 `headlamp-admin`) | 跟環境既有 ServiceAccount 命名慣例衝突時改。 |
| `prometheus_label_key`(預設 `headlamp-prometheus`) | **不要改。** Headlamp plugin 的 auto-detect 是寫死比對這個字面標籤(headlamp-offline.md 1.4 有說明偵測順序),改了 Prometheus 照樣裝得起來,但圖表會永遠偵測不到,而且不會有任何錯誤訊息。 |

---

## 執行方式

```bash
cd k8s-headlamp
ansible-playbook -i inventory/hosts site.yml --tags preflight   # 先確認連得到 node1、helm repo add 成功
ansible-playbook -i inventory/hosts site.yml                    # 全量部署(headlamp + prometheus + verify)
```

常用 tags:`preflight`(連線與 repo)、`headlamp`、`prometheus`、`verify`(等待就緒 + 印出存取資訊)。

---

## 這個 role 不會做的事(仍要手動)

- 抓素材、推 chart/image 到 Nexus(`headlamp-offline.md` Part 1 + 2.1/2.2)。
- Headlamp UI 內開 Prometheus auto-detect(Settings → Plugins → Prometheus,瀏覽器端設定,無 API 可打)。
- `headlamp-offline.md` 2.8 的 port-forward + PromQL 即時查詢複驗(`verify` tag 結束時會印出對應指令,自己手動跑)。
