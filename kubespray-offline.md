# Kubespray 離線安裝 Kubernetes 規劃書 v3
## kubespray v2.31.0 / Debian 13 / 供應鏈統一走 Nexus

本版與 v2 的差異:自架 nginx 檔案站與 registry 容器**全部移除**,二進位(raw)、映像(docker)、chart(helm)統一由《Nexus 部署規劃書 v2》的 `nexus.lab` / `registry.lab` 供應;系統套件改由既有離線 apt proxy 安裝(本文僅列清單);節點與 infra 的信任模型改為「DNS 指向 infra + 信任 Lab Root CA」,不再使用 insecure-registries。

**前提**:《Nexus 部署規劃書 v2》Phase B 已完成——`nexus.lab`(8081 via 443)、`registry.lab`(5000 via 443)、dnsmasq(`10.0.0.10`)、Lab Root CA 皆已就緒。

| 項目 | 內容 |
|---|---|
| kubespray | v2.31.0(容器映像執行,infra 不裝 venv) |
| Kubernetes / OS | v1.35.4 / Debian 13(cgroup v2) |
| 預設元件 | containerd 2.2.3、CNI plugins v1.9.1、etcd 3.6.10、Calico |
| 節點 | infra `10.0.0.10`(Nexus/nginx/dnsmasq/kubespray 執行器,**不入叢集**);node1–5 `10.0.0.11–15`(node1–3 control-plane+etcd,node1–5 worker) |
| Addons | helm、metrics-server、Gateway API CRD(1.5.1)、cert-manager、prometheus-operator CRDs、MetalLB(L2) |
| 叢集 UI / Ingress | Headlamp 0.43.0(LoadBalancer)/ NGF 2.6.5(相容 Gateway API 1.5.1) |
| 監控 | kube-prometheus-stack 81.6.9(operator v0.88.1,對齊 kubespray CRD;僅供 Headlamp 圖表,已關閉 Grafana/Alertmanager/exporters) |

**kubespray v2.31.0 關鍵變更**:ingress-nginx 與 Dashboard addon 已移除,addons.yml 殘留 `dashboard_enabled`/`ingress_nginx_*` 會使 playbook **啟動即中止**(整行刪除,不是設 false);Calico KDD CRD 改自 GitHub 下載,離線清單必須含它;cgroup v1 預設不支援(Debian 13 原生 v2,不受影響)。

---

# Part 0 — 部署前:節點網路設定(單一介面固定 IP)

**在 kubespray 部署之前,於每台節點完成網路設定。** kubespray 以節點當前的 IP 作為 node IP,先裝 k8s 再改網路等於重裝。

> 前提:Debian 13 以 **ifupdown / `/etc/network/interfaces`** 管理網路。
>
> ⚠️ **用 iLO / BMC Remote Console 操作**——套用網路設定瞬間會斷線。**一台一台做,做完立刻驗證再做下一台。**

## 0.1 確認介面名稱與速度【每台】

```bash
ip -br link show
ethtool ens1f0 | grep -E "Speed|Link detected"     # 確認 link up、速度符合預期
```

以下以 `ens1f0` 為例,替換為實查名稱。

## 0.2 設定 `/etc/network/interfaces`【每台】

```bash
cp /etc/network/interfaces /etc/network/interfaces.bak.$(date +%Y%m%d)
```

全檔只保留 loopback 與這一個介面(IP / gateway 換成該台實際值):

```
auto lo
iface lo inet loopback

auto ens1f0
iface ens1f0 inet static
    address <這台的IP>/24
    gateway <閘道IP>
    dns-nameservers 10.0.0.10
```

> **`dns-nameservers` 填 `10.0.0.10`**:與 Part 2 節點指向 infra(dnsmasq)一致,拉映像與下載二進位才解析得到 `registry.lab` / `nexus.lab`。
>
> **⚠️ 其他介面一律不要留同網段的 IP。** 機器上若還有 DHCP 介面(如 `eno1`)與本介面同網段,會同時觸發 ARP flux(對端時而拿到另一張網卡的 MAC)與預設路由互搶,症狀是「時通時斷」且極難排查。將該段改為 `iface eno1 inet manual`(不給 IP)、整段刪除,或實體拔線。

## 0.3 套用【iLO console】

```bash
reboot
```

**reboot 是最可靠的做法**:ifupdown 設計為開機時執行一次,手動改過介面狀態後再 `systemctl restart networking` 常因狀態脫節而失敗。

不便重開時,需先清乾淨殘留再套用:

```bash
ip addr flush dev <舊介面>       # ip link set down 不會移除 IP,殘留會造成 "address already assigned"
ip route flush dev <舊介面>
ifdown --force --all 2>/dev/null
rm -f /run/network/ifstate       # ifupdown 狀態檔與實際脫節時,restart 會失敗
systemctl restart networking
```

> 曾設定過 bonding 的機器,務必把四個 slave 的 `auto`/`iface` 段與整個 `bond0` 段從檔案刪除,並移除核心中的 bond0:
> `ip link set bond0 down; echo "-bond0" > /sys/class/net/bonding_masters`
> 留半套會使 `networking.service` 整體失敗——只要任一 `auto` 介面失敗,整個 service 即判定失敗。

## 0.4 驗證【每台】

```bash
ip -br addr                             # 僅該介面帶 IP,無其他同網段位址
ip route show                           # default 僅一條,且經由該介面
ping -c3 <閘道IP>
getent hosts registry.lab nexus.lab     # 皆回 10.0.0.10(Nexus 就緒後)
```

## 0.5 每台檢查清單(9 台各一次)

- [ ] iLO console 已開(救命通道)
- [ ] 介面名稱已查出、link up、速度符合預期
- [ ] `/etc/network/interfaces` 已備份,且只留 lo 與該介面(該台實際 IP/gateway,DNS=10.0.0.10)
- [ ] 無其他介面帶同網段 IP(`eno1` 等 DHCP 介面已停用)
- [ ] 重開後 `ip -br addr` / `ip route` 正確,ping 閘道通
- [ ] 記錄該台最終 IP,供 2.4 inventory 使用

**9 台全部完成後,才進入 Part 1 / Part 2。inventory 的 `ip`/`access_ip` 一律填此介面的 IP。**

> 日後若要改用 bonding 提升可用性:bond0 必須沿用**同一個 IP**,否則 node IP 改變等同重裝叢集。另需先確認交換器側四個 port 為獨立 port(非 port-channel / LAG)、A/B 同一 VLAN、已開 PortFast、無 port-security 綁定——否則會出現「切換後不通或斷斷續續」。

---

# Part 1 — 網路環境(WSL):蒐集素材

## 1.1 取得 kubespray 與 venv

```bash
git clone --branch v2.31.0 https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
python3 -m venv ~/ks-venv && source ~/ks-venv/bin/activate
pip install -r requirements.txt        # 僅 WSL 端為了跑 generate_list.sh
docker pull quay.io/kubespray/kubespray:v2.31.0
docker save quay.io/kubespray/kubespray:v2.31.0 -o kubespray-image.tar
```

## 1.2 設定 inventory

```bash
cp -rfp inventory/sample inventory/mycluster
```

`group_vars/k8s_cluster/k8s-cluster.yml`:

```yaml
kube_version: v1.35.4
kube_network_plugin: calico
container_manager: containerd
kube_proxy_mode: ipvs
kube_proxy_strict_arp: true              # MetalLB 在 ipvs 模式的硬性前提,角色第一步就檢查
```

`group_vars/k8s_cluster/addons.yml`(確認**無** `dashboard_enabled` 與任何 `ingress_nginx_*`):

```yaml
helm_enabled: true
metrics_server_enabled: true             # Headlamp 的 CPU/Memory 畫面即靠它,不需 Prometheus
gateway_api_enabled: true                # 未 pin,裝 Gateway API 1.5.1;NGF 須 >= 2.5(本文用 2.6.5)
cert_manager_enabled: true               # 三顆 image 皆在 kubespray 下載清單,自動收錄
prometheus_operator_crds_enabled: true   # 只裝 CRD(v0.88.1),Prometheus server 本身於 2.10 另裝

metallb_enabled: true
metallb_speaker_enabled: true
metallb_namespace: "metallb-system"
metallb_config:
  address_pools:
    primary:
      ip_range:
        - 10.0.0.200-10.0.0.220          # 與節點同網段、未被佔用
      auto_assign: true
  layer2:
    - primary
```

## 1.3 產生下載清單並下載

```bash
cd contrib/offline
./generate_list.sh -i ../../inventory/mycluster/hosts.yaml
grep -i calico temp/files.list           # 必須出現 Calico CRD 條目(v2.31.0 離線陷阱)

# 靜態檔(輸出 offline-files/,以原始網域為頂層目錄——上傳 Nexus 時保留此結構)
bash manage-offline-files.sh
tar czf offline-files.tar.gz offline-files/

# 叢集映像(輸出單一 container-images.tar.gz:內含各 image 的 .tar、images 清單、registry-latest.tar)
bash manage-offline-container-images.sh create -i temp/images.list
```

> 清單產生於 addons(1.2)設定**之後**,Gateway API CRD、cert-manager、MetalLB、metrics-server 的素材才會被收錄。

## 1.4 NGF 與 Headlamp 素材(非 addon,手動備)

```bash
mkdir -p ~/k8s-offline/charts ~/k8s-offline/extra-images && cd ~/k8s-offline

NGF_VER=2.6.5    # NGF 2.5.0 起對齊 Gateway API 1.5.1,2.6.x 維持
helm pull oci://ghcr.io/nginx/charts/nginx-gateway-fabric --version ${NGF_VER} -d charts
docker pull ghcr.io/nginx/nginx-gateway-fabric:${NGF_VER}
docker pull ghcr.io/nginx/nginx-gateway-fabric/nginx:${NGF_VER}

HL_VER=0.43.0
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/ && helm repo update
helm pull headlamp/headlamp --version ${HL_VER} -d charts
docker pull ghcr.io/headlamp-k8s/headlamp:v${HL_VER}

docker save \
  ghcr.io/nginx/nginx-gateway-fabric:${NGF_VER} \
  ghcr.io/nginx/nginx-gateway-fabric/nginx:${NGF_VER} \
  ghcr.io/headlamp-k8s/headlamp:v${HL_VER} | gzip > extra-images/addon-extra-images.tar.gz
```

### 1.4a Prometheus(kube-prometheus-stack)素材

Headlamp Pod detail 頁的 CPU/Memory/Network/Filesystem 圖表由 Prometheus plugin 提供,需叢集內有實際 Prometheus(1.2 的 `prometheus_operator_crds_enabled` 只裝 CRD,不含 server)。

```bash
cd ~/k8s-offline

# 版本必須對齊 kubespray 已裝的 prometheus-operator CRD v0.88.1
KPS_VER=81.6.9        # 此 chart 的 appVersion 即 prometheus-operator v0.88.1
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
helm pull prometheus-community/kube-prometheus-stack --version ${KPS_VER} -d charts

# 先寫好 2.10 的 values(內容見該節),用它枚舉「關閉元件後」實際需要的 image
helm template prom charts/kube-prometheus-stack-${KPS_VER}.tgz -n monitoring \
  -f prometheus-values.yaml \
  | grep -E '^\s+image:' | awk '{print $2}' | tr -d '"' | sort -u
```

以本規劃書的 values(關閉 Grafana / Alertmanager / node-exporter / kube-state-metrics / admission webhook)為例,應只剩三顆:

```bash
docker pull quay.io/prometheus-operator/prometheus-operator:v0.88.1
docker pull quay.io/prometheus-operator/prometheus-config-reloader:v0.88.1
docker pull quay.io/prometheus/prometheus:v3.9.1        # 以上一步枚舉結果為準

docker save \
  quay.io/prometheus-operator/prometheus-operator:v0.88.1 \
  quay.io/prometheus-operator/prometheus-config-reloader:v0.88.1 \
  quay.io/prometheus/prometheus:v3.9.1 | gzip > extra-images/prometheus-images.tar.gz
```

> **CRD 版本必須對齊**:kubespray v2.31.0 安裝 prometheus-operator CRD **v0.88.1**,而 chart 的 operator 版本需與之相符(operator 通常要求 CRD ≥ 自身版本)。選用其他 chart 版本時,先確認其 `appVersion`:`helm show chart prometheus-community/kube-prometheus-stack --version <ver> | grep appVersion`。

## 1.5 節點系統套件清單(由你的離線 apt proxy 安裝,本文不打包)

以下為對照 kubespray 原始碼 `roles/system_packages/vars/main.yml` 針對本配置(Debian 13、ipvs、k8s_cluster)算出的完整集合,節點於 2.3 一次裝齊:

```
conntrack socat ipset ipvsadm ebtables iptables nftables
python3 python3-apt python3-yaml
apparmor bash-completion iproute2 iputils-ping libseccomp2 openssl tar
apt-transport-https ca-certificates curl gnupg
unzip rsync e2fsprogs xfsprogs lvm2 chrony
```

要點:

- `python3-yaml`:`helm_enabled: true` 時 `kubernetes-apps/helm` 角色在**每個 control-plane 節點**無條件安裝它(相依 `libyaml-0-2`,apt proxy 會自動解決)——缺它就是常見的「libyaml 安裝失敗」。
- kubespray 的 system-packages 是**單一 apt 交易**:任何一顆解不出來,整批失敗。
- **不含** `software-properties-common`:Debian 13 已被上游移除,kubespray 亦排除 major_version 13。
- 套件預裝齊後,cluster.yml 的套件任務自然 no-op,不需 `--skip-tags`。

## 1.6 打包搬運

| 項目 | 檔案 |
|---|---|
| kubespray 專案(含 inventory) | `kubespray/` |
| kubespray 執行映像 | `kubespray-image.tar` |
| 靜態檔 | `offline-files.tar.gz` |
| 叢集映像 | `container-images.tar.gz` |
| NGF/Headlamp 映像 | `extra-images/addon-extra-images.tar.gz` |
| Prometheus 映像 | `extra-images/prometheus-images.tar.gz` |
| Charts | `charts/*.tgz`(NGF 2.6.5、Headlamp 0.43.0、kube-prometheus-stack 81.6.9) |
| Prometheus values | `prometheus-values.yaml`(2.10) |

```bash
tar czf k8s-offline-bundle.tar.gz kubespray/ kubespray-image.tar \
  offline-files.tar.gz container-images.tar.gz extra-images/ charts/ prometheus-values.yaml
```

---

# Part 2 — 離線環境:灌入 Nexus、部署、驗證

## 2.0 前置確認(缺一不可)

> 節點網路設定(Part 0)須已在 9 台全部完成:各台單一介面固定 IP 已確定、無其他介面帶同網段 IP——本階段起的 DNS、CA、時間同步都建立在該 IP 之上。

**infra(10.0.0.10)**:

```bash
tar xzf k8s-offline-bundle.tar.gz -C ~/k8s-offline && cd ~/k8s-offline
docker load -i kubespray-image.tar

# infra 自身信任 Lab CA(push 走 registry.lab TLS;取代舊版 insecure-registries)
sudo cp /etc/nginx/certs/ca.crt /usr/local/share/ca-certificates/lab-root-ca.crt
sudo update-ca-certificates && sudo systemctl restart docker
getent hosts registry.lab nexus.lab      # 皆應回 10.0.0.10
```

**五台節點(10.0.0.11–15)逐台**:

```bash
# DNS → infra
echo "nameserver 10.0.0.10" | sudo tee /etc/resolv.conf
# 信任 Lab CA(containerd 拉映像、kubespray 下載二進位皆走 TLS,吃系統 CA)
sudo cp ca.crt /usr/local/share/ca-certificates/lab-root-ca.crt && sudo update-ca-certificates
getent hosts registry.lab nexus.lab
```

**時間同步**(etcd 與憑證對時鐘偏移敏感;chrony 已列於 1.5 套件清單):

```bash
for n in 10.0.0.11 10.0.0.12 10.0.0.13 10.0.0.14 10.0.0.15; do
  ssh root@$n 'hostname; date; timedatectl show -p NTPSynchronized --value'
done
# 五台須一致到秒級。無內網 NTP 時以 chrony 一台為源、其餘為 client
```

**SSH**:infra 對五台 `ssh-copy-id` 已完成;金鑰若為 ed25519,2.5 的掛載路徑一併調整。

## 2.1 上傳靜態檔到 raw-hosted【infra】

```bash
tar xzf offline-files.tar.gz && cd offline-files
find . -type f | sed 's|^\./||' | while read -r f; do
  code=$(curl -s -o /dev/null -w '%{http_code}' \
    --upload-file "$f" "http://localhost:8081/repository/raw-hosted/k8s-files/$f")
  [ "$code" = "201" ] || echo "FAIL($code) $f"
done
cd ..
```

**驗證**(路徑保留原始網域頂層目錄,`*_download_url` 才對得上):

```bash
curl -sI https://nexus.lab/repository/raw-hosted/k8s-files/dl.k8s.io/release/v1.35.4/bin/linux/amd64/kubeadm | head -1   # HTTP/1.1 200
```

## 2.2 推送映像到 docker-hosted【infra】

infra 已信任 CA,直接經 `registry.lab`(443)推送,免 login:

```bash
push_tars() {   # docker load 後去原始 host 前綴,重標 registry.lab 推送
  for t in "$@"; do docker load -i "$t" | sed -n 's/^Loaded image: //p'; done |
  while read -r img; do
    first="${img%%/*}"
    if [[ "$first" == *.* || "$first" == *:* || "$first" == "localhost" ]]; then path="${img#*/}"; else path="$img"; fi
    docker tag "$img" "registry.lab/$path" && docker push "registry.lab/$path"
  done
}

mkdir -p container-images && tar xzf container-images.tar.gz -C container-images
# registry-latest.tar 為 kubespray 自架 registry 用,本架構(Nexus)不需要,排除之
push_tars $(find container-images -name '*.tar' ! -name 'registry-latest*')
push_tars extra-images/addon-extra-images.tar.gz     # docker load 可直接吃 .tar.gz
push_tars extra-images/prometheus-images.tar.gz      # 2.10 用
```

**驗證**:

```bash
curl -s 'https://nexus.lab/service/rest/v1/search?repository=docker-hosted' | grep -c '"name"'   # > 0
docker pull registry.lab/nginx/nginx-gateway-fabric:2.6.5 && echo PULL_OK                        # 抽測一顆
```

## 2.3 推送 Charts 到 helm-hosted【infra】+ 節點裝套件

```bash
for c in charts/*.tgz; do
  curl -s -o /dev/null -w "%{http_code} $c\n" http://localhost:8081/repository/helm-hosted/ --upload-file "$c"
done   # 皆應 200/201
```

**節點套件**(經你的 apt proxy,清單見 1.5;逐台或以迴圈執行):

```bash
for n in 10.0.0.11 10.0.0.12 10.0.0.13 10.0.0.14 10.0.0.15; do
  ssh root@$n 'DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y \
    conntrack socat ipset ipvsadm ebtables iptables nftables \
    python3 python3-apt python3-yaml \
    apparmor bash-completion iproute2 iputils-ping libseccomp2 openssl tar \
    apt-transport-https ca-certificates curl gnupg \
    unzip rsync e2fsprogs xfsprogs lvm2 chrony \
    && echo "PKG OK $(hostname)"'
done
```

## 2.4 offline.yml 與 hosts.yaml【infra,編輯 inventory】

`inventory/mycluster/group_vars/all/offline.yml`:

```yaml
registry_host: "registry.lab"                                  # 443,TLS,系統 CA 信任
files_repo: "https://nexus.lab/repository/raw-hosted/k8s-files"

kube_image_repo: "{{ registry_host }}"
gcr_image_repo: "{{ registry_host }}"
github_image_repo: "{{ registry_host }}"
docker_image_repo: "{{ registry_host }}"
quay_image_repo: "{{ registry_host }}"

# *_download_url:把 offline.yml 內被註解的整段原樣取消註解(勿手改路徑,
# 其相對結構與 2.1 上傳保留的網域頂層目錄一一對應)

containerd_registries_mirrors:
  - prefix: registry.lab
    mirrors:
      - host: https://registry.lab
        capabilities: ["pull", "resolve"]
  - prefix: docker.io
    mirrors:
      - host: https://registry.lab
        capabilities: ["pull", "resolve"]
  - prefix: registry.k8s.io
    mirrors:
      - host: https://registry.lab
        capabilities: ["pull", "resolve"]
  - prefix: quay.io
    mirrors:
      - host: https://registry.lab
        capabilities: ["pull", "resolve"]
  - prefix: ghcr.io
    mirrors:
      - host: https://registry.lab
        capabilities: ["pull", "resolve"]
# 全 TLS + 系統 CA:不需要 skip_verify,節點已於 2.0 信任 Lab CA
```

`inventory/mycluster/hosts.yaml`(node1–3 為 `kube_control_plane`+`etcd`,node1–5 為 `kube_node`;`ip`/`access_ip` 用 10.0.0.11–15)。

**驗證殘留變數**(應無輸出):

```bash
grep -RnE 'dashboard_enabled|ingress_nginx' inventory/mycluster/group_vars/ || echo CLEAN
```

## 2.5 執行部署【infra】

```bash
cd ~/k8s-offline/kubespray
docker run --rm -it \
  -e ANSIBLE_HOST_KEY_CHECKING=False \
  --mount type=bind,source="$(pwd)"/inventory/mycluster,dst=/kubespray/inventory/mycluster \
  --mount type=bind,source="${HOME}"/.ssh/id_rsa,dst=/root/.ssh/id_rsa,readonly \
  quay.io/kubespray/kubespray:v2.31.0 \
  ansible-playbook -i /kubespray/inventory/mycluster/hosts.yaml -b \
    --private-key /root/.ssh/id_rsa \
    cluster.yml
```

容器僅作為 SSH 執行器;二進位由節點向 `nexus.lab` 下載、映像由節點向 `registry.lab` 拉取,套件已預裝故套件任務 no-op。

## 2.6 驗證叢集【node1】

```bash
kubectl get nodes -o wide                    # 5 Ready
kubectl get pods -A | grep -vE 'Running|Completed' || echo ALL_RUNNING
kubectl get crd | grep gateway.networking.k8s.io   # Gateway API 1.5.1(kubespray 裝)
kubectl -n metallb-system get pods           # controller + 5 speaker Running
kubectl -n cert-manager get pods             # 3 pods Running
kubectl top nodes                            # metrics-server 生效
kubectl get ipaddresspool -n metallb-system  # primary 位址池存在
```

## 2.7 映像枚舉預檢(防 ImagePullBackOff)【node1】

NGF/Headlamp 非 addon,chart 可能含未預期 image(init/sidecar)。安裝前枚舉並比對:

```bash
helm repo add nexus https://nexus.lab/repository/helm-hosted/
helm repo update

helm template ngf nexus/nginx-gateway-fabric --version 2.6.5 -n nginx-gateway \
  --set image.repository=registry.lab/nginx/nginx-gateway-fabric --set image.tag=2.6.5 \
  --set nginx.image.repository=registry.lab/nginx/nginx-gateway-fabric/nginx --set nginx.image.tag=2.6.5 \
  | grep -E 'image:' | sort -u

helm template headlamp nexus/headlamp --version 0.43.0 -n kube-system \
  --set image.registry=registry.lab --set image.repository=headlamp-k8s/headlamp --set image.tag=v0.43.0 \
  | grep -E 'image:' | sort -u
```

列出的每個 image 都必須指向 `registry.lab/...`;若出現 `docker.io`/`ghcr.io` 等外部位址,回 infra 依 2.2 模式補推後再繼續。

## 2.8 安裝 NGF【node1】

```bash
helm install ngf nexus/nginx-gateway-fabric --version 2.6.5 \
  --create-namespace -n nginx-gateway \
  --set image.repository=registry.lab/nginx/nginx-gateway-fabric \
  --set image.tag=2.6.5 --set image.pullPolicy=IfNotPresent \
  --set nginx.image.repository=registry.lab/nginx/nginx-gateway-fabric/nginx \
  --set nginx.image.tag=2.6.5 --set nginx.image.pullPolicy=IfNotPresent \
  --set nginx.service.type=LoadBalancer
kubectl -n nginx-gateway wait deploy --all --for=condition=Available --timeout=5m
```

> tag 必須明確 pin:不指定時 chart 用 AppVersion,與推送 tag 不一致即 ImagePullBackOff。鍵名以 `helm show values nexus/nginx-gateway-fabric --version 2.6.5` 為準。

**NGF 2.x 資料面延後佈建**:上述 wait 只驗證控制面;nginx 資料面 Pod 於建立 Gateway 時才拉取映像。立即以最小 Gateway 驗證:

```bash
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: ngf-smoke, namespace: nginx-gateway }
spec:
  gatewayClassName: nginx
  listeners: [ { name: http, port: 80, protocol: HTTP } ]
EOF
kubectl -n nginx-gateway get pods -w    # 資料面 Pod Running(非 ImagePullBackOff)後:
kubectl -n nginx-gateway delete gateway ngf-smoke
```

## 2.9 安裝 Headlamp【node1】

Headlamp 的 Pod CPU/Memory 畫面由 **metrics-server** 供應(1.2 已啟用),不需 Prometheus。

```bash
cat > /root/headlamp-values.yaml <<'EOF'
image:
  registry: registry.lab
  repository: headlamp-k8s/headlamp
  tag: v0.43.0            # 必須 pin,理由同 2.8
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

helm install headlamp nexus/headlamp --version 0.43.0 -n kube-system -f /root/headlamp-values.yaml

kubectl create serviceaccount headlamp-admin -n kube-system
kubectl create clusterrolebinding headlamp-admin \
  --serviceaccount=kube-system:headlamp-admin --clusterrole=cluster-admin
```

**驗證與登入**:

```bash
kubectl -n kube-system get svc headlamp     # EXTERNAL-IP 為 MetalLB 位址池 IP(非 <pending>)
kubectl create token headlamp-admin -n kube-system          # 臨時 token(約 1h)
```

瀏覽器開 `http://<EXTERNAL-IP>`,貼 token 登入。此時 Pod 列表與 cluster overview 的用量數字來自 **metrics-server**(1.2 已啟用);Pod detail 頁上方的 CPU/Memory/Network/Filesystem **時序圖**另需 Prometheus,見 2.10。長期 token:

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

## 2.10 安裝 Prometheus(供 Headlamp Pod detail 圖表)【node1】

Headlamp Pod detail 頁的六張圖表查的**全部是 `container_*` 指標**(`container_cpu_usage_seconds_total`、`container_memory_working_set_bytes`、`container_network_receive/transmit_bytes_total`、`container_fs_reads/writes_bytes_total`),這些由 **kubelet 內建的 cAdvisor** 提供。因此 Grafana、Alertmanager、node-exporter、kube-state-metrics 一律不需要,只保留 operator + Prometheus server + kubelet scrape。

### values.yaml

```bash
ssh root@10.0.0.11
cat > /root/prometheus-values.yaml <<'EOF'
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
    # 儲存:預設 emptyDir(Pod 重啟即失去歷史)。要保留歷史請改用 storageSpec 掛 PVC,
    # 但勿使用 ES 專用的 local storageClass,避免與 ES 搶碟。
EOF
```

> **`headlamp-prometheus: "true"` 是 auto-detect 能成功的關鍵。** Headlamp plugin 的偵測順序是:①帶 `headlamp-prometheus=true` 的 **Pod** → ②帶該標籤的 **Service** → ③帶 `app.kubernetes.io/name=prometheus` 的 Pod → ④帶 `app.kubernetes.io/name=prometheus,app.kubernetes.io/component=server` 的 Service。kube-prometheus-stack 建立的 Service **不帶** `app.kubernetes.io/name=prometheus`(第④項不會命中),因此顯式加上自訂標籤,讓偵測在第②步就確定命中,不依賴 chart 內部標籤慣例。找到後 plugin 會逐一測試該資源的 port(以 `up` 查詢驗證可達)再採用。

### 安裝與驗證

```bash
helm install prom nexus/kube-prometheus-stack --version 81.6.9 \
  --create-namespace -n monitoring -f /root/prometheus-values.yaml

kubectl -n monitoring wait --for=condition=Ready pod -l app.kubernetes.io/name=prometheus --timeout=5m
kubectl -n monitoring get pods                    # 應僅 operator + prometheus,無 grafana/alertmanager
kubectl -n monitoring get svc -l headlamp-prometheus=true   # 標籤確實存在(auto-detect 靠它)

# 確認 kubelet/cAdvisor 已被 scrape 且 container_* 指標有資料
kubectl -n monitoring port-forward svc/prom-kube-prometheus-stack-prometheus 9090:9090 &
curl -s 'http://localhost:9090/api/v1/query?query=up{job="kubelet"}' | grep -o '"value"' | head -1
curl -s 'http://localhost:9090/api/v1/query?query=container_cpu_usage_seconds_total' | head -c 200
kill %1
```

`container_cpu_usage_seconds_total` 有回傳資料,才代表 Headlamp 圖表會有值。

### Headlamp 端啟用(auto-detect)

Headlamp UI → **Settings → Plugins → Prometheus**:

1. **Enable metrics** 開啟。
2. **Auto-detect** 保持開啟(不需填 Prometheus Service Address)。
3. 回到任一 Pod 的 detail 頁,上方應出現 CPU / Memory / Network / Filesystem 四張圖。

> 設定為每個瀏覽器各自保存,換瀏覽器或換人登入需重設一次。若 auto-detect 仍失敗,先用上面的 `kubectl get svc -l headlamp-prometheus=true` 確認標籤在,再確認登入的 token 有列出全叢集 Pod/Service 的權限(auto-detect 會呼叫 `/api/v1/services?labelSelector=...`;`headlamp-admin` 綁 cluster-admin 即滿足)。真的要手動指定時,關閉 auto-detect 並填 `monitoring/prom-kube-prometheus-stack-prometheus:9090`(格式為 `namespace/service-name:port`)。

---

## 附錄 A — 常見故障排除

| 症狀 | 原因 | 處理 |
|---|---|---|
| playbook 啟動即中止 `removed variable` | addons.yml 殘留 `dashboard_enabled`/`ingress_nginx_*` | 整行刪除(2.4 驗證步驟) |
| 節點下載二進位 404 | 2.1 上傳未保留網域頂層目錄,或 `*_download_url` 被手改 | 依 2.1 重傳;offline.yml 註解段原樣取消註解 |
| 節點拉映像 `x509 unknown authority` | 節點未信任 Lab CA | 2.0 節點步驟補做後重跑 |
| 節點拉映像 404 / manifest unknown | 該 image 未推入 docker-hosted,或 tag 不符 | infra 依 2.2 補推;`curl -s https://nexus.lab/service/rest/v1/search?repository=docker-hosted&name=<name>` 查證 |
| infra push 憑證錯誤 | infra 未裝 CA 或未重啟 docker | 2.0 infra 步驟 |
| 名稱解析失敗 | 節點 DNS 未指向 10.0.0.10,或 dnsmasq 未啟動 | `getent hosts registry.lab`;infra `systemctl status dnsmasq` |
| system-packages 整段失敗但錯誤只提一顆套件 | 單一 apt 交易,一顆解不出全批失敗 | 對照 1.5 清單補齊(含 python3-yaml) |
| `Install PyYaml`/libyaml 失敗 | 節點缺 `python3-yaml`(helm 角色於每個 control-plane 強制安裝) | apt proxy 補裝後重跑 |
| etcd 起不來 / TLS 握手失敗 | 節點間時鐘偏移或 IP 不可達 | 2.0 時間同步;確認 `ip` 已掛網卡 |
| 容器跑 cluster.yml 卡 SSH | 私鑰未掛載 / 金鑰名不符(ed25519) | 調整 2.5 掛載與 `--private-key` |
| MetalLB task `fail` 提到 strictARP | `kube_proxy_strict_arp` 非 true | 全新部署:1.2 已設;既有叢集:附錄 C |
| LoadBalancer `EXTERNAL-IP: <pending>` | MetalLB 未就緒或位址池耗盡/衝突 | `kubectl -n metallb-system get pods,ipaddresspool` |
| NGF 找不到 Gateway CRD | `gateway_api_enabled` 未啟用 | 1.2 確認後重跑 cluster.yml;`kubectl get crd \| grep gateway` |
| Helm/Headlamp `ImagePullBackOff` | tag 未 pin 或 chart 含未推送的 image | 2.7 枚舉補推;2.8/2.9 pin tag |
| `helm repo add nexus` 憑證錯誤 | node1 未信任 Lab CA | 2.0 節點步驟 |
| Headlamp Pod detail 無圖表 / 顯示無法偵測 Prometheus | 叢集內沒有 Prometheus(CRD ≠ server) | 依 2.10 安裝 kube-prometheus-stack |
| Prometheus 裝了但 auto-detect 仍失敗 | Service 缺 `headlamp-prometheus=true` 標籤,或登入 token 權限不足 | `kubectl -n monitoring get svc -l headlamp-prometheus=true` 確認標籤;token 需能列全叢集 svc/pod;必要時關 auto-detect 手動填 `namespace/service:port`(2.10) |
| 圖表有框但無資料 | kubelet/cAdvisor 未被 scrape | 確認 values 的 `kubelet.enabled: true`;`container_cpu_usage_seconds_total` 查詢須有回傳(2.10 驗證) |
| helm install kube-prometheus-stack CRD 衝突 | chart 又裝一次 CRD | values 設 `crds.enabled: false`(CRD 已由 kubespray 提供) |
| `Certificate` 一直 pending / 未產生 Secret | `cert_manager_enabled` 只裝 cert-manager,未建任何 Issuer | 依附錄 D 建立 CA 型 `ClusterIssuer`,再確認 `issuerRef` 的 name/kind 正確 |

## 附錄 B — 版本升級 / 重跑

- 升級 kubespray 前重讀 release notes 的 removed variables;etcd 版本順序:升級既有叢集需先 ≥3.5.26 再上 3.6.x(v2.31.0 對 K8s 1.35 預設 3.6.10)。
- 換 IP:`reset.yml` 後重跑 cluster.yml 即可(reset 會清 PKI/etcd,新 IP 重新簽憑證)。注意 reset 也會**清空 containerd image store 與所有二進位**,節點等同裸機——Nexus 供應鏈完整即可直接重裝;reset 後五台重開機以清淨網路殘留(CNI 介面、nftables 規則)。Debian 13 若遇 ansible 直譯器錯誤,加 `-e ansible_python_interpreter=/usr/bin/python3`。
- 「線上先裝一遍再 reset 搬離線」對離線重裝**沒有預載效益**(image store 會被清空),僅適合當連網排練。

## 附錄 C — 既有叢集事後啟用 MetalLB(不需重裝)

MetalLB 為純 `kubectl apply` 的 addon,不碰 etcd/kubelet/CNI;以 `--tags metallb` 局部套用。

```bash
# C1 既有叢集手動 flip strictARP(MetalLB 官方做法;動 kubeadm 層風險過高)【node1】
kubectl -n kube-system edit configmap kube-proxy      # strictARP: false → true
kubectl -n kube-system rollout restart daemonset kube-proxy
kubectl -n kube-system rollout status daemonset kube-proxy
# 同步把 1.2 的 kube_proxy_strict_arp 改 true,避免日後重裝重蹈覆轍

# C2 補映像【infra】(若當初清單產生時 metallb_enabled 為 false)
docker pull quay.io/metallb/speaker:v0.13.9 && docker pull quay.io/metallb/controller:v0.13.9
docker tag quay.io/metallb/speaker:v0.13.9    registry.lab/metallb/speaker:v0.13.9
docker tag quay.io/metallb/controller:v0.13.9 registry.lab/metallb/controller:v0.13.9
docker push registry.lab/metallb/speaker:v0.13.9 && docker push registry.lab/metallb/controller:v0.13.9

# C3 局部套用【infra,同 2.5 容器,附加 --tags metallb】
#    ... cluster.yml --tags metallb

# C4 驗證【node1】
kubectl -n metallb-system get pods,ipaddresspool,l2advertisement
```

同一 `--tags` 模式適用於事後補開 `cert-manager`(先推三顆 image)與 `prometheus_operator_crds`(純 YAML,無 image)。

---

---

## 附錄 D — 使用 cert-manager 簽發叢集內憑證(issuerRef)

1.2 的 `cert_manager_enabled: true` **只安裝 cert-manager 本體,不會建立任何 Issuer**。沒有 Issuer 之前,任何帶 `issuerRef` 的 `Certificate` 都會停在 pending。離線環境無法使用 ACME / Let's Encrypt(需對外驗證),因此正確做法是**以既有的 Lab Root CA 建立 CA 型 ClusterIssuer**——如此叢集內簽出的憑證與《Nexus 部署規劃書 v2》客戶端已信任的是同一條信任鏈,不需要再讓客戶端多裝一張 CA。

### E.1 建立 CA ClusterIssuer【node1】

把 infra 上的 Lab Root CA(`/etc/nginx/certs/ca.crt` 與 `ca.key`)存成 Secret,再建立 ClusterIssuer:

```bash
# 從 infra 取得 ca.crt / ca.key 後(ca.key 屬機密,傳輸與保存請比照私鑰處理)
kubectl -n cert-manager create secret tls lab-root-ca \
  --cert=ca.crt --key=ca.key
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: lab-ca-issuer
spec:
  ca:
    secretName: lab-root-ca      # 須位於 cert-manager 的部署 namespace
```

驗證:

```bash
kubectl get clusterissuer lab-ca-issuer      # READY 應為 True
```

### E.2 一般用法:自行簽發憑證

任何需要 TLS 的服務,建立 `Certificate` 並以 `issuerRef` 指向上面的 ClusterIssuer;cert-manager 會把結果寫進 `secretName` 指定的 Secret,並在到期前自動輪替。

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: headlamp-tls
  namespace: headlamp
spec:
  secretName: headlamp-tls        # 產出的 Secret,供 Gateway / Ingress / Pod 掛載
  duration: 8760h                 # 1y
  renewBefore: 720h               # 到期前 30d 自動更新
  commonName: headlamp.lab
  dnsNames:
    - headlamp.lab
  issuerRef:
    name: lab-ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

搭配 Gateway API(2.8 的 NGF)時,在 Gateway 的 HTTPS listener 引用該 Secret:

```yaml
listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
        - kind: Secret
          name: headlamp-tls
```

### E.3 kube-prometheus-stack 內建的 issuerRef 位置

本規劃書 2.10 已關閉 admission webhook(`admissionWebhooks.enabled: false`),因此不需要憑證。**若你之後改為啟用**,chart 支援直接把 webhook 憑證交給 cert-manager 簽發,不必用內建的自簽 job:

```yaml
prometheusOperator:
  admissionWebhooks:
    enabled: true
    certManager:
      enabled: true             # 改用 cert-manager,取代內建 kube-webhook-certgen
      issuerRef:
        name: lab-ca-issuer
        kind: ClusterIssuer
      rootCert:
        duration: ""            # 留空用預設 5y
      admissionCert:
        duration: ""            # 留空用預設 1y
```

> 啟用 `certManager.enabled: true` 後,`patch` 那顆 `kube-webhook-certgen` image 就不再需要;但 `admission-webhook` image 仍需鏡像進 `registry.lab`。改動後務必重跑 2.7 的枚舉確認 image 清單。

### E.4 注意事項

- **CA 私鑰的保管**:`ca.key` 一旦進入叢集 Secret,任何能讀該 Secret 的人即可簽發被全網信任的憑證。正式環境建議用中介 CA(intermediate CA)給叢集使用,Root CA 私鑰離線保存。
- **Issuer vs ClusterIssuer**:`Issuer` 只在自身 namespace 生效,`ClusterIssuer` 全叢集可用。跨 namespace 使用(如上面 headlamp / monitoring 各自簽發)須用 `ClusterIssuer`。
- **信任鏈**:由 Lab Root CA 簽出的憑證,客戶端只要已完成《Nexus 部署規劃書 v2》Phase C 的 CA 安裝即自動信任,不需額外動作;JVM 型客戶端(sbt/Maven)仍須另行 keytool 匯入。

# Part 3 — 維運操作(更新 / 擴充 / 縮容)

**離線環境的鐵則:任何操作前先補素材,再動叢集。** 節點在裝套件、拉映像、下載二進位時都只會找 Nexus,新版本或新元件若沒先進 Nexus,操作必定中途失敗。每個小節的第一步都是「補素材」,第二步才是「執行」。所有 ansible 指令沿用 2.5 的 kubespray 容器(掛載 inventory 與私鑰,附加各自的參數)。

> 操作前先備份 etcd(見 3.6),control-plane 或 etcd 相關變更尤其必要。

## 3.1 新增 worker 節點(擴充)

新節點先完成 2.0 前置(DNS 指向 infra、信任 Lab CA、時間同步)與 2.3 套件安裝。

```bash
# 1) inventory 追加新節點(例 node6 / 10.0.0.16)到 kube_node
# 2) 先刷新全叢集 facts(用 --limit 前必做,否則新節點缺既有節點事實)
docker run ... cluster.yml 改為： playbooks/facts.yml
# 3) 用 scale.yml 只納入新節點,不驚動既有節點
docker run --rm -it -e ANSIBLE_HOST_KEY_CHECKING=False \
  --mount ...inventory... --mount ...id_rsa... \
  quay.io/kubespray/kubespray:v2.31.0 \
  ansible-playbook -i .../hosts.yaml -b --private-key /root/.ssh/id_rsa \
  scale.yml --limit=node6
```

驗證:`kubectl get nodes` 出現 node6 且 Ready;`kubectl get pods -A -o wide | grep node6` 有 kube-proxy/calico DaemonSet。

> `scale.yml` 只能加 worker;新增 control-plane 或 etcd 見 3.4 / 3.5。素材面:worker 用的映像(pause/calico/kube-proxy 等)首次部署已在 Nexus,通常不需補;版本一致即可。

## 3.2 移除節點(縮容)

```bash
# 節點仍在 inventory 中,指定 -e node= 限定執行範圍,會自動 drain + reset
docker run ... remove-node.yml -e node=node6
# 節點已離線無法連線時,追加:
#   -e node=node6 -e reset_nodes=false -e allow_ungraceful_removal=true
# 之後再從 inventory 移除該節點條目
```

驗證:`kubectl get nodes` 不再有 node6;`kubectl get pods -A | grep -v Running` 無殘留。移除 control-plane 或 etcd 型節點也用此 playbook,但務必先讀 3.4 / 3.5 的順序限制。

## 3.3 升級 Kubernetes 版本

**素材面(關鍵)**:升級 = 新版本的二進位與映像。必須回 Part 1 用**目標版本**重跑清單並補進 Nexus,否則節點抓不到新版而失敗。

```bash
# infra/WSL:改 inventory 的 kube_version 為目標版本(例 v1.36.x),重跑 1.3 / 1.4
#   generate_list.sh → manage-offline-files.sh → manage-offline-container-images.sh
# 再依 2.1 / 2.2 把「新版」二進位與映像補進 Nexus(raw-hosted / docker-hosted)
```

kubespray **只支援逐一 minor 版升級**(如 1.35 → 1.36 → 1.37,不可跳版)。素材補齊後:

```bash
# 滾動升級:預設每批 20% 節點;serial=1 為一次一台(最保守,建議正式環境用)
# 先刷新 facts(同 3.1)
ansible-playbook ... playbooks/facts.yml
ansible-playbook ... upgrade-cluster.yml -e kube_version=v1.36.x -e "serial=1"
```

可選的節點級暫停(便於逐台觀察):`-e upgrade_node_confirm=true`(每台升級前等人工輸入 yes)或 `-e upgrade_node_pause_seconds=60`。

驗證:`kubectl get nodes`(VERSION 欄逐台更新且維持 Ready);升級中另開視窗 `watch kubectl get pods -A` 觀察 workload 是否正常 reschedule。

> `upgrade-cluster.yml` 會 graceful 逐節點 cordon/drain/升級/uncordon,比 `cluster.yml` 安全。跨多版時每次只升一個 minor,重複「補素材 + 升級」流程。etcd 版本順序:升級既有叢集需先 ≥3.5.26 再上 3.6.x。

## 3.4 新增 / 移除 control-plane 節點

**新增**:新節點完成 2.0 前置與 2.3 套件後,inventory 的 `kube_control_plane`(及需要時 `etcd`)追加該節點,然後:

```bash
ansible-playbook ... cluster.yml --limit=kube_control_plane
```

> control-plane 新增**不能**用 `scale.yml`,要用 `cluster.yml --limit`。第一台 control-plane(清單首位)無法直接移除,若必須更換需先調整 inventory 順序,細節依 kubespray `docs/operations/nodes.md`。新增後確認你的 API 存取端點(kube-vip / 外部 LB,若有)已納入新節點。

**移除**:同 3.2 的 `remove-node.yml -e node=<name>`,但先確認移除後 control-plane 仍為奇數且 ≥1 可用。

## 3.5 新增 / 移除 etcd 節點

etcd 必須維持**奇數**台,故 etcd 變更本質上是「替換」或「成對增減」。

```bash
# 新增(inventory 的 etcd 群組追加新節點後):
ansible-playbook ... cluster.yml --limit=etcd,kube_control_plane -e ignore_assert_errors=yes
ansible-playbook ... upgrade-cluster.yml --limit=etcd,kube_control_plane -e ignore_assert_errors=yes
#   一次加多台 etcd 時追加 -e etcd_retries=10,避免前一台 join 未完成就試下一台
# 之後在每台 control-plane 編輯 /etc/kubernetes/manifests/kube-apiserver.yaml,
# 確認 --etcd-servers=... 已含新 etcd 節點
```

移除 etcd:`remove-node.yml -e node=<name>`,再更新各 control-plane 的 `--etcd-servers=` 移除該節點。etcd 操作風險高,務必先做 3.6 備份。

## 3.6 etcd 備份與還原

```bash
# 備份(在任一 etcd 節點;endpoints/憑證路徑依實際)【etcd 節點】
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/node-$(hostname).pem \
  --key=/etc/ssl/etcd/ssl/node-$(hostname)-key.pem \
  snapshot save /var/backups/etcd-$(date +%F).db
```

還原屬災難復原情境,使用 kubespray 的 `recover-control-plane.yml`(需搭配可用的 etcd 快照與 inventory 中標記存活的節點),依官方 `docs/operations/recover-control-plane.md` 執行。日常維運前養成先 snapshot 的習慣即可。

## 3.7 只更新 addon / 元件設定(不動叢集版本)

改 addons.yml 或元件參數後,用 tag 局部套用,不必全量重跑:

```bash
# 先把新元件/新版本的 image 補進 registry.lab(依 2.2),再:
ansible-playbook ... cluster.yml --tags=<tag>
# 常用 tag:metallb / cert-manager / prometheus_operator_crds / metrics_server / gateway_api / apps
```

NGF、Headlamp 屬 helm 安裝(非 kubespray addon),更新走 `helm upgrade`:先把新版 chart 傳 `helm-hosted`、新版 image 推 `docker-hosted`(2.2 / 2.3),再 `helm repo update && helm upgrade <release> nexus/<chart> --version <new> -f <values>`,tag 仍須明確 pin。

---

*本規劃書結束。素材供應與客戶端信任模型見《Nexus 部署規劃書 v2》。*
