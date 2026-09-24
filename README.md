# Headlamp + Prometheus Ansible 部署(airgap)

用 Ansible 在**已經跑起來的 airgap Kubernetes 叢集**上安裝 Headlamp 與 kube-prometheus-stack(供 Headlamp Pod detail 頁 CPU/Memory/Network/Filesystem 時序圖 auto-detect 用)。只做「從 Nexus 消費素材、跑 helm/kubectl」這段,**不會**幫你抓素材或推上 Nexus。

完整手動版流程與每個決策的理由見 `headlamp-offline.md`。叢集本身的建置不在本案範圍,參考外部文件《Kubespray 離線安裝 Kubernetes 規劃書 v3》。

---

## 版本對應(本案驗證組合)

| 層 | 元件 | 版本 | 說明 |
|---|---|---|---|
| 叢集(前提) | Kubernetes | v1.35.4 | kubespray v2.31.0 / Debian 13 / containerd 2.2.3 / Calico(僅供參考,本案不安裝) |
| 叢集 addon(前提) | metrics-server | kubespray 內建 | Pod 列表與叢集總覽的 CPU/Memory 數字 |
| 叢集 addon(前提) | prometheus-operator CRD | **v0.88.1** | 決定下方 kube-prometheus-stack 的版本 |
| 叢集 addon(前提) | MetalLB | kubespray 內建 | 分配 Headlamp 的 LoadBalancer IP |
| 本案安裝 | Headlamp chart / image | 0.43.0 / `v0.43.0` | namespace `kube-system` |
| 本案安裝 | kube-prometheus-stack chart | 81.6.9 | appVersion 0.88.1,對齊 CRD;namespace `monitoring` |
| 本案安裝 | prometheus-operator / prometheus-config-reloader | `v0.88.1` | |
| 本案安裝 | prometheus | `v3.9.1` | 以 `helm template` 枚舉結果為準 |
| 執行機 | ansible-core | ≥ 2.15(已於 2.19 驗證) | 不需任何額外 collection |

> **版本連動規則**:叢集 CRD 版本 → kube-prometheus-stack chart 版本(其 appVersion 必須等於 CRD 版本)→ 3 顆 Prometheus 相關 image tag。任一層變了,後面都要重新核對(`helm show chart <chart> --version <ver> | grep appVersion`)。Headlamp 與此鏈無關,chart 與 image tag 兩者一致即可。

---

## 需要依環境調整的參數

### 必改(3 個)

| 檔案 | 變數 | 預設 | 說明 |
|---|---|---|---|
| `inventory/hosts` | `ansible_host` | `10.0.0.11` | 任一台有 `admin.conf` 的 control-plane 節點 IP |
| `inventory/group_vars/all.yml` | `nexus_helm_repo_url` | `https://nexus.lab/repository/helm-hosted/` | Nexus helm hosted repo URL |
| `inventory/group_vars/all.yml` | `registry_host` | `registry.lab` | 節點拉映像用的 registry 主機名(同時是映像預檢的比對字串,改錯預檢會誤判) |

### 視情況改(在 `roles/kubectl/headlamp/defaults/main.yml`,或用 `-e` 覆蓋)

| 變數 | 預設 | 何時要改 |
|---|---|---|
| `headlamp_service_type` | `LoadBalancer` | 叢集沒有 MetalLB → 改 `NodePort`(曝露方式自行處理,本 role 不做 Ingress/Gateway) |
| `prometheus_resources_*` / `prometheus_retention` | 100m/1Gi request、2Gi limit、24h | 節點資源較緊或較寬裕時調整,避免 Pending 或浪費 |
| `headlamp_show_token` | `false` | 想在跑完直接看到長期 token 明文時設 `true`(token 會進終端機與 ansible log) |

### 不要改

- **版本類變數**(`headlamp_chart_version`、`headlamp_image_tag`、`prometheus_chart_version`):只在升版時依上面的版本連動規則一起改,且新版素材必須已推上 Nexus。

### 固定值(直接寫在 role 裡,不開變數)

單一叢集、沿用 kubespray 預設,以下值直接寫死在 tasks/templates:

| 項目 | 值 |
|---|---|
| kubectl / helm | 直接呼叫 `kubectl`、`helm`(PATH),不帶 `--kubeconfig`,吃 root 預設的 `~/.kube/config` |
| helm values 檔 | `/etc/helm/headlamp-values.yaml`、`/etc/helm/prometheus-values.yaml` |
| k8s manifest | `/etc/k8s/headlamp-rbac.yaml` |
| helm repo 名稱 | `nexus` |
| namespace | Headlamp `kube-system`、Prometheus `monitoring` |
| Headlamp 登入帳號 | ServiceAccount `headlamp-admin` 綁 `cluster-admin`,長期 token 在 Secret `headlamp-admin-token` |
| Headlamp Service port | `80` |
| Prometheus auto-detect 標籤 | `headlamp-prometheus: "true"`(**不可改**:Headlamp 寫死比對,改了圖表永遠偵測不到且無錯誤訊息) |

---

## Airgap 前置需求(執行前必須已經成立)

### 叢集側

- 所有節點 Ready;上方版本表中的 metrics-server、prometheus-operator CRD v0.88.1、MetalLB(含可用的 IPAddressPool)皆已就緒。
- 目標節點(`ansible_host`)是 control-plane,以 root 登入後**不帶任何參數**即可執行 `kubectl get nodes` 與 `helm list -A`(kubespray 預設會把 admin kubeconfig 放到 `/root/.kube/config`)。
- 目標節點上 `/etc/k8s/`(k8s manifest)與 `/etc/helm/`(helm values)兩個目錄已建立。playbook 不會建立或改動這兩個目錄的權限,只會在裡面寫入檔案(權限 0600)。
- 所有節點 DNS 可解析 `registry.lab`、`nexus.lab`(`getent hosts registry.lab nexus.lab`),且已信任 Lab Root CA(否則拉映像 `x509 unknown authority`、`helm repo add` 憑證錯誤)。
- ansible 執行機能以 **root** SSH 免密碼登入目標節點(`ssh-copy-id` 已做)。

### Nexus 側(你自己處理,這個 playbook 不會做)

以下東西必須**已經**在 Nexus 上,對應 `headlamp-offline.md` Part 1 + 2.1/2.2:

| repo | 內容 | 版本 |
|---|---|---|
| `helm-hosted` | `headlamp` chart | 0.43.0 |
| `helm-hosted` | `kube-prometheus-stack` chart | 81.6.9 |
| `docker-hosted`(經 `registry.lab`) | `headlamp-k8s/headlamp` | `v0.43.0` |
| `docker-hosted` | `prometheus-operator/prometheus-operator` | `v0.88.1` |
| `docker-hosted` | `prometheus-operator/prometheus-config-reloader` | `v0.88.1` |
| `docker-hosted` | `prometheus/prometheus` | `v3.9.1`(或依 `helm template` 枚舉結果) |

playbook 的 `headlamp`/`prometheus` tag 各有一段「映像枚舉預檢」,任何 image 沒指向 `registry_host` 會直接 fail,但**不會**檢查該 tag 是否真的已推上 Nexus,那個要靠你在 2.1/2.2 確認。

### 執行機本身

- 有 `ansible-playbook`(ansible-core 即可,這個 role 刻意只用 `command` 呼叫 kubectl/helm)。

---

## 執行方式

```bash
cd k8s-headlamp/k8s-headlamp-playbook                            # 必須在這層執行,才會讀到 ansible.cfg
ansible-playbook site.yml --tags preflight   # 先確認連得到節點、kubectl 通、helm repo add 成功
ansible-playbook site.yml                    # 全量部署(headlamp + prometheus + verify)
```

常用 tags:`preflight`(連線與 repo)、`headlamp`、`prometheus`、`verify`(等待就緒 + 印出存取資訊)。

---

## 這個 role 不會做的事(仍要手動)

- 抓素材、推 chart/image 到 Nexus(`headlamp-offline.md` Part 1 + 2.1/2.2)。
- Headlamp UI 內開 Prometheus auto-detect(Settings → Plugins → Prometheus,瀏覽器端設定,無 API 可打)。
- `headlamp-offline.md` 2.8 的 port-forward + PromQL 即時查詢複驗(`verify` tag 結束時會印出對應指令,自己手動跑)。

