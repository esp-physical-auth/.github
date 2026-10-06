# ESP Physical Auth 🦀

把 **ESP32-C5** 变成一把**私钥永不导出**的硬件认证器：TOTP / WebAuthn passkey /
BTC 冷钱包 / 设备身份。设备广播名 `ATRI-TOTP`。

## 子项目

| 目录 | 角色 | 语言 | 接口 |
|------|------|------|------|
| [`esp32c5-totp/`](../../../../physkey-firmware/) | 🧠 **硬件核心**：生成密钥 + 签名，私钥不出芯片 | ESP-IDF / C | BLE **NUS** 文本协议 |
| [`web/`](../../../../physkey-dashboard/) | 🌐 **浏览器直连**：TOTP / 冷钱包静态页 | HTML + JS | Web Bluetooth |
| [`passless/`](../../../../physkey-linux/) | 🐧 **Linux 桥**：把 ESP32 冒充成系统 FIDO2 密钥 | Rust | 虚拟 UHID + CTAP2 |
| [`intermediate-ca-worker/`](../../../../intermediate-ca-worker/) | 🔏 **证书签发器**：Cloudflare Worker，用根 CA 给中间 CA 签证书（含 Web UI） | JS | HTTPS |

## 架构：三条链路，一个核心

```
                        ┌─────────────────────────────┐
                        │   ESP32    固件（硬件核心）    │
                        │  私钥永不导出，只签名          │
                        │  WA_REG / WA_SIGNHASH / AUTH  │
                        └──────────────▲──────────────┘
                                       │ BLE NUS 文本协议
                                       │ （4hex长度前缀 + 20B分片 + OK/ERR）
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
      ┌───────▼────────┐      ┌────────▼────────┐      ┌────────▼────────┐
      │  web (浏览器)   │      │ passless (Linux) │      │ authnkey-esp32  │
      │ Web Bluetooth  │      │ 虚拟 UHID → 浏览器│      │ (Android 系统)   │
      │ TOTP/冷钱包     │      │ CTAP2 桥          │      │ CTAP2→NUS 桥     │
      └────────────────┘      └──────────────────┘      └─────────────────┘
```

三条访问路径汇聚到**同一个固件、同一套 CA 信任链、同一套 NUS 协议**。

## 固件 NUS 协议速查

- **帧** = `4位hex长度前缀（小写）+ 命令体 + \n`，如 `0017AUTHPASS <pw>\n`
- **响应** = 纯文本，`OK ...` / `ERR ...` 结尾；**无 0x04 EOT**
- **BLE**：NUS 服务 `6e400001-...`，RX `...0002`（写）、TX `...0003`（notify）
- **分片**：20B/片，不在多字节 UTF-8 中间切开
- **命令族**：`HASPASS`/`SETPASS`/`AUTHPASS`/`LOCK`、`WA_*`（WebAuthn）、
  `WALLET_*`（钱包）、`GETCERT`/`SETCERT`/`AUTH`/`WHOAMI`（设备身份/CA）
- **断连即上锁**（固件 disconnect 时 `esp_crypto_lock()`）

## 信任模型：SSL/TLS 式三级证书链

采用**类似 TLS 的证书链**：根 CA 签发中间 CA，中间 CA 签发设备证书。
客户端**只内置根 CA 公钥**，验签**全程离线**（不写死域名/IP、不联网）。

```
根 CA（root，私钥离线保管，绝不进任何服务端/客户端）
  └─签发→ 中间 CA 证书（0x10，内嵌 deployment_id）
            └── 中间 CA（部署者本地生成，私钥永不出本地）
                  └─签发→ ESP32 设备证书（0x02）
```

**证书格式**（自定义简单二进制，非 X.509）：

```
设备证书:    0x02 || id_len(1) || deployment_id || devicePub(65) || issue_time(8) || serial(8)
中间CA证书:  0x10 || id_len(1) || deployment_id || userCA_pub(65) || issue_time(8) || serial(8)
证书封装:    len(payload)(2B BE) || payload || sig_len(2B BE) || signature(64B raw r||s)
```

**客户端验签链**（全程离线，只用内置根 CA 公钥）：

```
根CA公钥 → 验中间CA证书 → 取中间CA公钥 → 验设备证书 → 取设备公钥
        → 验 AUTH 挑战响应 → 弹窗展示 deployment_id（用户确认设备归属）
```

**下发**：固件 `SETCERT <设备证书>|<中间CA证书>` 分段存储；`GETCERT` 一并返回两段。
单段（只有设备证书）时，客户端直接用内置根 CA 公钥验。

> **设计约束**：客户端代码**绝不写死域名/IP**、**绝不联网**。证书签发器（Worker）只在
> “签发中间 CA 证书”这一刻被使用一次，运行期与身份校验完全无关。

### 命名

- **`deployment_id`（部署标识）**：唯一，标识“谁的部署”。规范 `a-z0-9_-.`，3–64 位，
  小写、字母数字开头。内嵌进中间 CA 证书，客户端验签后展示给用户确认。

#### 🏷️ 部署标识（deployment_id）是什么？

`deployment_id` 是**你这次部署的唯一名字**，用来回答一个核心安全问题：
**“现在连上的这台设备，到底是不是我自己部署的？”**

- 它由你自定（如 `lianyu-tianhai`），在签发**中间 CA 证书**时被**嵌进证书内部**
  （见上“证书格式”里的 `deployment_id` 字段）。
- 它在**签发器端被 KV 锁定**：同一个 id 一旦登记了某把公钥，就**不能再换别的公钥**
  （否则 409）。这防止别人盗用/抢注你的 id。
- 客户端验签通过后，会把证书里**内嵌的 deployment_id 弹窗展示**给用户，让你肉眼确认，
  语义是 **TOFU（首次信任）**：确认“是我自己部署的”就继续，点否则断开。

> 一句话：**deployment_id = 设备归属的身份证 + 防抢注的锁。** 它不是登录账号，只是标示
> “这是谁的部署”，让用户在一个统一根 CA 下，仍能分辨出具体是哪台设备/哪个人部署的。

## CA 工具（`esp32c5-totp/tools/atri-ca.py`）

命令分两组：**根 CA**（`root-*`，离线权威）与**中间 CA**（`uca-*`，部署者本机）。

```bash
# —— 根 CA（只需一次，代表你的权威）——
python3 atri-ca.py root-init                                  # 生成根 CA 密钥对
python3 atri-ca.py root-pub                                   # 打印根 CA 公钥(base64)，嵌客户端
python3 atri-ca.py root-sign-uca <userpub_b64|hex> [deployment_id]  # 给中间 CA 公钥签“中间 CA 证书”
python3 atri-ca.py root-verify-uca <uca_cert_b64>             # 用根 CA 公钥验中间 CA 证书

# —— 中间 CA（部署者本机）——
python3 atri-ca.py uca-init                                   # 生成中间 CA 密钥对
python3 atri-ca.py uca-pub                                    # 打印中间 CA 公钥
python3 atri-ca.py uca-sign <device_pub_hex> [deployment_id] [uca_cert_b64]  # 给设备签证书（可拼两段链）
python3 atri-ca.py uca-verify <dev_cert_b64> [uca_cert_b64]   # 验设备证书（可选两段链）
```

### 📖 本地 CA 签发完整流程（不联网，纯离线）

适合自用、或不想把根 CA 私钥交给任何在线服务的情形。全程在本机完成。

> **前置**：`cd esp32c5-totp/tools`，以下命令都在此目录执行。
> `python3` 需装 `cryptography` 库（`pip install cryptography`）。

#### ① 生成根 CA（只做一次）

```bash
python3 atri-ca.py root-init
```

产出（都在 `ca/root/`）：
- `root_private.pem` —— **根 CA 私钥**。⚠️ 离线保管，绝不外传、不提交 git。
- `root_public.pem` / `root_meta.json` —— 公钥与元信息。

命令会打印一行 **根 CA 公钥(base64)**，形如 `BO+1Mjj...`——这串要**嵌进三端客户端**
（web/totp.html、passless/config.rs、authnkey/Esp32Config.kt 里的 `*_CA_PUBKEY_B64`）。

再次查看：

```bash
python3 atri-ca.py root-pub
```

#### ② 生成中间 CA 密钥对（部署者本机）

```bash
python3 atri-ca.py uca-init
```

产出 `ca/uca_private.pem`（本地私钥）、`ca/uca_public.pem`、`ca/uca_meta.json`。

#### ③ 用根 CA 给中间 CA 公钥签“中间 CA 证书”

```bash
UCAPUB=$(python3 atri-ca.py uca-pub)
python3 atri-ca.py root-sign-uca "$UCAPUB" <你的deployment_id>
```

- `<你的deployment_id>`：如 `lianyu-tianhai`（规范 `a-z0-9_-.`，3–64 位）。
- 产出 `ca/uca_cert.b64`（中间 CA 证书）。
- 自检：`python3 atri-ca.py root-verify-uca <uca_cert_b64>` 应打印“验签通过”。

#### ④ 取设备公钥

设备上电后，经 BLE 发送 `WHOAMI`（或串口），得到 **65 字节未压缩公钥**（hex，`04` 开头，共 130 个字符）。

#### ⑤ 用中间 CA 给设备签证书，并拼成两段链

```bash
UCACERT=$(cat ca/uca_cert.b64)
python3 atri-ca.py uca-sign <设备公钥hex> <你的deployment_id> "$UCACERT"
```

- 产出 `ca/device_cert.der` / `ca/device_cert.b64`（设备证书）。
- 同时产出 `ca/device_chain.b64`，并打印一条：

  ```
  SETCERT <设备证书>|<中间CA证书>
  ```

- 自检（两段链完整验签）：

  ```bash
  python3 atri-ca.py uca-verify "$(cat ca/device_cert.b64)" "$UCACERT"
  ```

#### ⑥ 把证书烧进设备

把上一步打印的整条 `SETCERT <设备证书>|<中间CA证书>` 经 **BLE**（web / passless / authnkey
的发送通道）或**串口**发给设备。设备分段存入 NVS。

#### ⑦ 验证

- 设备发 `GETCERT` → 应返回两段 `设备证书|中间CA证书`。
- 客户端连上后自动走三段链验签，并通过弹窗展示 `deployment_id` 让你确认设备归属。

> 💡 **提示**：`root-sign-uca` 也可换成把中间 CA 公钥提交给**在线证书签发器**
> （官方工具 [esp.oleanderchat.asia](https://esp.oleanderchat.asia/)，见下节），
> 由 Worker 用根 CA 私钥签发——适合把根 CA 私钥托管在 Cloudflare secret、不落地本地的场景。

## 证书签发器 Worker（`intermediate-ca-worker/`）

Cloudflare Worker，**只做一件事**——收到用户的**中间 CA 公钥** + 部署标识，用**根 CA 私钥**
签一张“中间 CA 证书”返回（用户的中间 CA 私钥永不经过 Worker）。自带图形化 Web UI。

> 💡 **为方便使用，建议直接用官方工具 [esp.oleanderchat.asia](https://esp.oleanderchat.asia/)**
> 签发中间 CA 证书，无需自己部署 Worker。
>
> 使用步骤：
> 1. 本地生成中间 CA 密钥对：`python3 atri-ca.py uca-init`，再 `python3 atri-ca.py uca-pub` 取公钥。
> 2. 浏览器打开 **https://esp.oleanderchat.asia/**（官方签发器）。
> 3. 填入你的 **`deployment_id`** 与 **中间 CA 公钥(base64)**，点击签发。
> 4. 复制返回的 **中间 CA 证书(base64)**，保存为 `ca/uca_cert.b64`。
> 5. 回到本地，走“本地 CA 签发流程”的第 ⑤ 步用中间 CA 给设备签证书。
>
> 官方签发器同样带 KV 唯一性保证：同一 `deployment_id` 不能换公钥重复签发。

**自建 Worker**（如果你想自己托管）：

**唯一性保证**：Worker 用 KV 记录 `deployment_id → 公钥`，保证二者一一对应；
同一个 id 换成别的公钥再签会被 **409 拒绝**（防抢注/劫持）。

```bash
# 1. 建 KV（记录 deployment_id ↔ 公钥），把返回的 id 填进 wrangler.toml
npx wrangler kv namespace create ATRI_ID_REGISTRY

# 2. 注入根 CA 私钥（PEM）
npx wrangler secret put ROOT_CA_PRIV_PEM < ../esp32c5-totp/tools/ca/root/root_private.pem

# 3.（可选）设 ALLOW_ORIGIN、自定义域（见 wrangler.toml 注释）
npx wrangler deploy
```

部署后用浏览器打开 Worker 根路径即可见 **Web UI**：填 `deployment_id` + 中间 CA 公钥，
一键签发并复制“中间 CA 证书”。API 也可直接调：

```bash
curl -X POST https://<worker>/sign-uca -H 'Content-Type: application/json' \
  -d "{\"deployment_id\":\"<你的id>\",\"uca_pub_b64\":\"$(python3 atri-ca.py uca-pub)\"}"
# → 保存 uca_cert_b64，再走本地流程的第 ⑤ 步
```

## 安全声明

- **私钥永不离开设备**：密钥生成、P-256 签名、凭证存储全在 ESP32 内完成
- **防中间人**：CA 证书链 + 挑战响应
- ESP32 不是安全芯片（无 Secure Element / TPM），能提供“私钥永不导出”边界，
  但无专用安全芯片的防物理克隆、侧信道防护。高价值账户建议 TPM 级硬件密钥。

## 许可

各子项目分别授权：`esp32c5-totp` / `web` / `passless` 为 GPLv3；
`authnkey-esp32` 保留上游 MIT。详见各目录 `LICENSE`。
