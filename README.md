# ServerSetting

**EC2 伺服器自動化設定工具(Ansible)**

把常用的 EC2 初始化 / 維運步驟寫成 Ansible playbook,讓開新機器、換 SSL、加使用者等工作從「半小時手動操作」變成「一個指令」。換電腦也只要 `git clone` 就能繼續用。

---

## 目錄

- [需求](#需求)
- [初次安裝](#初次安裝)
- [專案結構](#專案結構)
- [使用情境](#使用情境)
  - [情境 1:設定一台全新 EC2](#情境-1設定一台全新-ec2)
  - [情境 2:對既有機器追加設定](#情境-2對既有機器追加設定)
  - [情境 3:替換 / 部署 SSL 憑證(GoDaddy)](#情境-3替換--部署-ssl-憑證godaddy)
  - [情境 4:新增 / 移除團隊成員帳號](#情境-4新增--移除團隊成員帳號)
  - [情境 5:新增 SFTP 使用者](#情境-5新增-sftp-使用者)
  - [情境 6:同時管理多台機器](#情境-6同時管理多台機器)
  - [情境 7:臨時對機器下指令](#情境-7臨時對機器下指令)
  - [情境 8:查稽核 log](#情境-8查稽核-log)
- [Playbook 對照表](#playbook-對照表)
- [Role 對照表](#role-對照表)
- [變數速查](#變數速查)
- [安全規範](#安全規範)
- [疑難排解](#疑難排解)

---

## 需求

### 目標機器(EC2)

**只支援 Ubuntu / Debian 系列**,因為 role 裡用了 `apt`、`ufw` 等 Debian 專屬工具。

| 系統 | 支援 | 備註 |
|---|---|---|
| **Ubuntu 24.04 LTS** | ✅ | 推薦(最新,支援到 2029) |
| **Ubuntu 22.04 LTS** | ✅ | 主要測試對象,支援到 2027 |
| **Ubuntu 20.04 LTS** | ✅ | 仍可用,支援到 2025 |
| **Debian 11 / 12** | ✅ | 同生態 |
| **Amazon Linux 2 / 2023** | ❌ | 用 dnf,不相容 |
| **CentOS / RHEL / Rocky** | ❌ | 用 dnf + firewalld |
| **Alpine** | ❌ | 用 apk |

> ⚠️ **開 EC2 時請選 Ubuntu AMI**,AWS 預設是 Amazon Linux,跑不起來。
> 推薦選 **Ubuntu Server 24.04 LTS**(最新 LTS,支援到 2029 年)。

### 本機(control node,你下指令的這台電腦)

- macOS / Linux(Windows 建議用 WSL)
- Python 3.8+
- Ansible 2.14+
- 能 SSH 進目標 EC2 的金鑰

```bash
# macOS
brew install ansible

# 或用 pip
pip3 install ansible
```

> 💡 **EC2 本身不用裝 Ansible**。Ansible 只要 SSH 連得上就能管,這是它的最大優勢之一。

---

## 初次安裝

```bash
# 1. clone 專案
git clone <你的 repo URL>
cd ServerSetting

# 2. 安裝第三方依賴 collection
ansible-galaxy collection install -r requirements.yml

# 3. 建立 inventory(機器清單)
cp inventory/hosts.ini.example inventory/hosts.ini

# 4. 編輯 inventory/hosts.ini,填入你的 EC2 IP / 金鑰路徑

# 5. 編輯 inventory/group_vars/all.yml,填入你的 SSH 公鑰(deploy_authorized_keys)

# 6. 測試連線
ansible all -m ping
# 看到 pong 就 OK
```

---

## 專案結構

```
ServerSetting/
├── README.md                      # 本檔案
├── ansible.cfg                    # Ansible 全域設定
├── requirements.yml               # 第三方 collection 依賴
├── .gitignore                     # 排除 hosts.ini、certs/、私鑰等
│
├── inventory/
│   ├── hosts.ini.example          # 機器清單範本(進 git)
│   ├── hosts.ini                  # 實際清單(不進 git,自己建立)
│   └── group_vars/
│       └── all.yml                # 全域變數(時區、使用者、套件版本...)
│
├── certs/                         # SSL 憑證暫存區(不進 git)
│   ├── README.md
│   └── <domain>/
│       ├── certificate.crt
│       ├── ca-bundle.crt
│       └── private.key
│
├── playbooks/
│   ├── site.yml                   # 主流程:初始化機器
│   └── deploy-ssl.yml             # 獨立流程:部署 SSL 憑證
│
└── roles/
    ├── common/                    # 系統更新、時區、deploy 使用者、swap
    ├── security/                  # UFW + fail2ban
    ├── audit/                     # auditd 稽核 + bash history 強化
    ├── docker/                    # Docker CE + Compose plugin
    ├── nodejs/                    # Node.js (NodeSource) + 全域 npm 套件
    ├── nginx/                     # Nginx 基本設定(HTTP)
    ├── nginx_ssl/                 # Nginx HTTPS 設定(獨立 role)
    └── sftp/                      # OpenSSH SFTP + chroot jail
```

---

## 使用情境

### 情境 1:設定一台全新 EC2

最常見的情境 — 剛開好 EC2,要把它從零裝起來。

```bash
# 1. 把新機器加到 inventory/hosts.ini
#    例如:
#    [web]
#    new-server ansible_host=54.123.45.67 ansible_user=ubuntu \
#               ansible_ssh_private_key_file=~/.ssh/aws-key.pem

# 2. 確認 inventory/group_vars/all.yml 的設定符合需求
#    (時區、Node 版本、要開的 port、SSH 公鑰...)

# 3. Dry run:看會改什麼,不實際執行
ansible-playbook playbooks/site.yml --limit new-server --check

# 4. 正式執行
ansible-playbook playbooks/site.yml --limit new-server
```

**會做的事(依序):**
1. ✅ `apt update && apt upgrade`
2. ✅ 設時區、建 swap、建 deploy 使用者(免密碼 sudo + SSH key 登入)
3. ✅ 開 UFW 防火牆 + 啟用 fail2ban
4. ✅ 啟用 auditd 稽核,強化 bash history
5. ✅ 安裝 Docker + Compose plugin
6. ✅ 安裝 Node.js 20 + pm2 + yarn
7. ✅ 安裝 Nginx(HTTP only,SSL 另外設)
8. ✅ 設定 SFTP(若有設定 `sftp_users`)

跑完整台機器就準備好了。

---

### 情境 2:對既有機器追加設定

**只想跑某些 role**,例如已經有機器,只想加 fail2ban + auditd:

```bash
# 用 --tags 限定要跑的部分
ansible-playbook playbooks/site.yml --tags security,audit
```

**Ansible 是冪等的**:已經設好的東西會跳過,不會重複安裝或破壞現有設定。

可用的 tag(每個 role 都有):

| Tag | 對應 role |
|---|---|
| `common` | 系統基礎 |
| `security` (`firewall`) | UFW + fail2ban |
| `audit` | 稽核設定 |
| `docker` | Docker |
| `nodejs` (`node`) | Node.js |
| `nginx` (`web`) | Nginx |
| `sftp` | SFTP |

```bash
# 只升級 Node 版本(改完 inventory/group_vars/all.yml 的 nodejs_version 後跑)
ansible-playbook playbooks/site.yml --tags nodejs

# 只重新套用 nginx 設定
ansible-playbook playbooks/site.yml --tags nginx

# 只對某一台機器跑某些 role
ansible-playbook playbooks/site.yml --tags docker --limit prod-1
```

---

### 情境 3:替換 / 部署 SSL 憑證(GoDaddy)

**這是獨立的 playbook,不會被 `site.yml` 觸發。** 跑 site.yml 不會碰 SSL,放心。

#### 步驟

```bash
# 1. 從 GoDaddy 後台下載新憑證
#    My Products → SSL Certificates → 點選憑證 → Manage → Download
#    Server type 選 "Other"

# 2. 解壓 + 改檔名後丟進 certs/<domain>/
#    需要 3 個檔案:
#      certs/example.com/
#      ├── certificate.crt   ← GoDaddy 給的 .crt(主憑證)
#      ├── ca-bundle.crt     ← GoDaddy 給的 bundle(中繼憑證)
#      └── private.key       ← 你產 CSR 時自己留的私鑰
#    (詳見 certs/README.md)

# 3. Dry run
ansible-playbook playbooks/deploy-ssl.yml -e ssl_domain=example.com --check

# 4. 部署
ansible-playbook playbooks/deploy-ssl.yml -e ssl_domain=example.com

# 限定特定機器
ansible-playbook playbooks/deploy-ssl.yml -e ssl_domain=example.com --limit web
```

#### 自動會做的事

- ✅ 確認 3 個憑證檔都在
- ✅ openssl 比對憑證 + 私鑰是同一對(modulus hash 相同)
- ✅ 顯示憑證有效期(Subject / Issuer / NotBefore / NotAfter)
- ✅ 備份 EC2 舊憑證到 `.backup-<時間戳>/`(可隨時 rollback)
- ✅ 合併 cert + ca-bundle 成 fullchain
- ✅ 上傳並設正確權限(私鑰 600,憑證 644)
- ✅ 部署 nginx site config(含 HTTP→HTTPS redirect)
- ✅ `nginx -t` 驗證設定,失敗不 reload
- ✅ reload nginx

#### 多 domain / wildcard

```bash
ansible-playbook playbooks/deploy-ssl.yml \
  -e ssl_domain=example.com \
  -e '{"ssl_extra_server_names": ["www.example.com", "api.example.com"]}'
```

---

### 情境 4:新增 / 移除團隊成員帳號

每個團隊成員一個獨立 Linux 帳號(audit 友善、權限細緻、離職好處理)。

#### 新增成員

編輯 [inventory/group_vars/all.yml](inventory/group_vars/all.yml) 的 `team_members`:

```yaml
team_members:
  - name: alice
    sudo: true                       # 有 sudo
    groups: [docker]
    authorized_keys:
      - "ssh-ed25519 AAAA... alice@laptop"

  - name: bob
    sudo: false                      # 沒 sudo,只能執行有限操作
    groups: [docker]                 # 但能跑 docker 指令
    authorized_keys:
      - "ssh-ed25519 AAAA... bob@desktop"
```

套用:

```bash
ansible-playbook playbooks/site.yml --tags team
```

#### 移除成員(離職)

把該成員的 `state` 改成 `absent`:

```yaml
team_members:
  - name: charlie
    state: absent          # ← 帳號 + 家目錄都會被刪除
    authorized_keys: []
```

```bash
ansible-playbook playbooks/site.yml --tags team
```

#### 連線

```bash
ssh alice@<EC2_IP>          # 用 alice 自己的 SSH key
```

#### 三種帳號的角色

| 帳號 | 用途 | 怎麼來的 |
|---|---|---|
| `ubuntu` | AWS Ubuntu AMI 預設,Ansible 用來連線提權 | AWS 預設 |
| `deploy` | 部署 / 自動化用的服務帳號 | `common` role 建立 |
| `alice` / `bob` 等 | **真人團隊成員**的個人帳號 | `team` role 建立(本情境) |

audit log 都會分別記錄,可以用 `sudo ausearch -ua alice` 查特定成員行為。

---

### 情境 5:新增 SFTP 使用者

#### 步驟

1. 編輯 [inventory/group_vars/all.yml](inventory/group_vars/all.yml),在 `sftp_users` 加上新使用者:

   ```yaml
   sftp_users:
     - name: client1
       authorized_keys:
         - "ssh-ed25519 AAAA... client1@laptop"
     - name: designer
       authorized_keys:
         - "ssh-rsa AAAA... designer@mac"
   ```

2. 套用:

   ```bash
   ansible-playbook playbooks/site.yml --tags sftp
   ```

#### 對方怎麼連?

```bash
sftp -i ~/.ssh/client1_key client1@<EC2_IP>
sftp> cd upload
sftp> put myfile.zip
```

或用 GUI(Cyberduck / FileZilla / Transmit),都填:
- Host: EC2 IP
- User: `client1`
- Auth: 私鑰
- Port: 22

#### 設計重點

- 使用者**只能 SFTP,不能 SSH 開 shell**(shell = `/usr/sbin/nologin`)
- 被 chroot **鎖在自己的家目錄**,看不到系統其他檔案
- 檔案實際放 `~/upload/`(因為 chroot 根目錄不能讓使用者寫,這是 OpenSSH 規定)
- 強制只用 SSH key,**密碼登入禁用**

---

### 情境 6:同時管理多台機器

**不用對每台機器各跑一次。** 在 inventory 列上去就好:

```ini
# inventory/hosts.ini
[web]
prod-1 ansible_host=1.1.1.1 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/k.pem
prod-2 ansible_host=2.2.2.2 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/k.pem
prod-3 ansible_host=3.3.3.3 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/k.pem

[db]
db-1 ansible_host=4.4.4.4 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/k.pem
```

```bash
# 全部 4 台一起跑(預設並行 10 台)
ansible-playbook playbooks/site.yml

# 只 web 群組
ansible-playbook playbooks/site.yml --limit web

# 指定某幾台
ansible-playbook playbooks/site.yml --limit prod-1,prod-2

# 排除某台(冒號 + ! 開頭)
ansible-playbook playbooks/site.yml --limit 'web:!prod-3'
```

如果想針對不同群組套不同變數,可以建立 `group_vars/web.yml`、`group_vars/db.yml`(優先級高於 `all.yml`)。

---

### 情境 7:臨時對機器下指令

不寫 playbook,**像批次 SSH 一樣下單次指令**:

```bash
# 看所有機器 uptime
ansible all -m shell -a "uptime"

# 看磁碟空間
ansible all -m shell -a "df -h"

# 看 nginx 狀態
ansible web -m shell -a "systemctl status nginx" --become

# 對所有機器更新套件
ansible all -m apt -a "update_cache=yes upgrade=dist" --become

# 重啟某個服務
ansible web -m service -a "name=nginx state=restarted" --become

# 把本機檔案拷到所有機器
ansible all -m copy -a "src=./local-file dest=/tmp/file" --become
```

`-m` = module,`-a` = arguments,`--become` = 用 sudo。

---

### 情境 8:查稽核 log

`audit` role 會記錄系統行為。實際查詢都在 EC2 上:

```bash
# SSH 進去
ssh deploy@<EC2_IP>

# 誰用過 sudo
sudo ausearch -k privilege_use

# 誰改過 sshd_config
sudo ausearch -k sshd_config

# 最近一小時所有事件
sudo ausearch --start recent

# 整體統計
sudo aureport --summary
sudo aureport --auth          # 登入失敗 / 成功
sudo aureport --executable    # 執行過的程式

# 看 SSH 登入紀錄
last
sudo tail /var/log/auth.log

# 看誰下過什麼指令(每筆有時間戳)
history
sudo cat /home/deploy/.bash_history
```

---

## Playbook 對照表

| Playbook | 用途 | 觸發 SSL 部署? |
|---|---|---|
| `playbooks/site.yml` | 機器初始化 / 維護(common, security, audit, docker, nodejs, nginx, sftp) | ❌ **不會** |
| `playbooks/deploy-ssl.yml` | 部署 / 替換 SSL 憑證 | ✅(只有跑這個才會) |

> 💡 跑 `site.yml` 完全不會碰 SSL。SSL 是獨立 playbook,還必須帶 `-e ssl_domain=...` 參數,沒帶會報錯停下來。
> 也就是說:**沒有要 HTTPS 的機器**(內網工具站、純 SFTP server 等),完全不需要碰 `deploy-ssl.yml`,也不會被誤觸。

---

## Role 對照表

| Role | 功能 | 預設啟用? | 對應變數(`inventory/group_vars/all.yml`) |
|---|---|---|---|
| `common` | 系統更新、時區、deploy 使用者、swap | ✅ | `timezone` / `deploy_user` / `deploy_authorized_keys` / `swap_size_gb` |
| `security` | UFW + fail2ban | ✅ | `ufw_allowed_ports` |
| `audit` | auditd + 強化 bash history | ✅ | (無需設定) |
| `docker` | Docker CE + Compose plugin | ✅ | (無需設定) |
| `nodejs` | Node.js + 全域 npm 套件 | ✅ | `nodejs_version` / `nodejs_global_packages` |
| `nginx` | Nginx HTTP server | ✅ | `nginx_default_server_name` |
| `sftp` | OpenSSH SFTP + chroot jail | ✅(`sftp_users` 為空就略過) | `sftp_users` |
| `nginx_ssl` | Nginx HTTPS 部署 | ❌(獨立 playbook) | 用 `-e ssl_domain=` 指定 |

「預設啟用」= 跑 `site.yml` 時會執行(但 role 內部仍會檢查現狀,該不該動才動)。

---

## 變數速查

打開 [inventory/group_vars/all.yml](inventory/group_vars/all.yml) 編輯:

| 變數 | 預設值 | 說明 |
|---|---|---|
| `timezone` | `Asia/Taipei` | 系統時區(EC2 預設 UTC,**要自己設**) |
| `deploy_user` | `deploy` | 部署用帳號名稱 |
| `deploy_user_groups` | `[sudo, docker]` | 部署使用者所屬群組 |
| `deploy_authorized_keys` | `[]` | **必填** — 部署使用者的 SSH 公鑰清單 |
| `swap_size_gb` | `2` | Swap 大小(GB),設 0 就不建立 |
| `nodejs_version` | `"20"` | Node.js 主版號(NodeSource:18 / 20 / 22) |
| `nodejs_global_packages` | `[pm2, yarn]` | 全域要裝的 npm 套件 |
| `nginx_default_server_name` | `_` | Nginx 預設 server_name |
| `ufw_allowed_ports` | `[22, 80, 443]` | UFW 開放的 TCP port |
| `sftp_users` | `[]` | SFTP 使用者清單(空則不建立) |

> ⚠️ **拿到自己的公鑰**:`cat ~/.ssh/id_ed25519.pub` 或 `cat ~/.ssh/id_rsa.pub`

---

## 安全規範

### 進 git 的檔案

✅ 設定範本(`.example`)、playbook、role、文件

### **絕對不能** 進 git 的檔案(已在 `.gitignore`)

❌ `inventory/hosts.ini`(含真實 IP / 金鑰路徑)
❌ `certs/<domain>/*.key`(SSL 私鑰)
❌ `certs/<domain>/*.crt`(雖然憑證本身不算機密,但跟私鑰放一起,整包排除)
❌ 任何 `*.pem` / `*.vault` / `.vault_pass`

### Port / 防火牆雙層概念

EC2 流量要通,**必須兩道都開**:

```
你的電腦 → AWS Security Group(雲端外層) → EC2 內 UFW → 服務
```

- **Security Group**:在 AWS 後台設,Ansible **不會幫你開**(本來就不該管 AWS 帳號)
- **UFW**:Ansible 透過 `ufw_allowed_ports` 設定

兩個都要開,流量才進得來。

### Secrets 管理

Production 用的密碼 / API key 不要直接寫在 `inventory/group_vars/all.yml`,用 [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html):

```bash
# 加密
ansible-vault encrypt group_vars/secrets.yml

# 編輯加密檔
ansible-vault edit group_vars/secrets.yml

# 跑時提供密碼
ansible-playbook playbooks/site.yml --ask-vault-pass
```

---

## 疑難排解

### 連不到機器

```bash
ansible all -m ping
# UNREACHABLE
```

排查:
1. inventory 裡 `ansible_host` IP 對嗎?
2. `ansible_user` 對嗎?(Ubuntu AMI = `ubuntu`,Amazon Linux = `ec2-user`)
3. 金鑰路徑對嗎?權限對嗎?(`chmod 400 ~/.ssh/xxx.pem`)
4. AWS Security Group 22 port 開了嗎?
5. EC2 instance 在跑嗎?Public IP 沒被換掉嗎?

### Playbook 跑到一半失敗

Ansible 預設**失敗的 task 之前的都已經執行**,後面的不會跑。修好原因再跑一次,**冪等性會讓已完成的步驟自動跳過**,不會重複動作。

### nginx -t 驗證失敗

通常是 SSL config 寫錯。`nginx_ssl` role 設計上**驗證失敗就不會 reload**,所以原本的服務不會掛掉,可以放心修。

### 不確定會改什麼

任何時候都可以加 `--check` dry run:

```bash
ansible-playbook playbooks/site.yml --check
ansible-playbook playbooks/deploy-ssl.yml -e ssl_domain=example.com --check
```

加 `--diff` 會額外顯示檔案內容的 diff:

```bash
ansible-playbook playbooks/site.yml --check --diff
```

### 想看更詳細的執行 log

```bash
ansible-playbook playbooks/site.yml -v       # 普通
ansible-playbook playbooks/site.yml -vv      # 詳細
ansible-playbook playbooks/site.yml -vvv     # 包含 SSH 連線細節
```

---

## 常用指令速查

| 動作 | 指令 |
|---|---|
| 測試連線 | `ansible all -m ping` |
| 設定全部機器 | `ansible-playbook playbooks/site.yml` |
| 設定特定機器 | `... --limit prod-1` |
| 只跑某 role | `... --tags nginx,docker` |
| Dry run | `... --check` |
| 部署 SSL | `ansible-playbook playbooks/deploy-ssl.yml -e ssl_domain=example.com` |
| 臨時下指令 | `ansible all -m shell -a "uptime"` |
| 列出所有機器 | `ansible all --list-hosts` |
| 列出機器變數 | `ansible <host> -m setup` |
