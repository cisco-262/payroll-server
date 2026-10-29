# Payroll Server

**Self-hosted payroll delivery for organizations. Official iOS client. Your infrastructure, your data.**

[English](#english) · [简体中文](#简体中文)

> **Release preparation — not yet a verified public release.** This repository is intended for official deployment files, documentation and signed-off release artifacts. Image names, download locations, and the App Store link must be published and verified before the first public release. Do not deploy placeholder image names.

---

<a id="english"></a>

## English

### What is Payroll Server?

Payroll Server is the self-hosted backend for **OpsHome Payroll**. It helps an organization prepare, review, publish and deliver employee payslips while keeping its primary payroll database on infrastructure it operates or controls.

The **Community Edition** is intended to be distributed as an official compiled container image and release binaries, together with deployment documentation. The proprietary Go backend source code is **not** included in this public distribution repository.

**The employee mobile client is the official OpsHome Payroll iOS app distributed through the Apple App Store.** The license does not authorize third-party, repackaged, or independently developed mobile clients to use employee mobile APIs. Administrators can use the authorized browser-based management console.

### Capabilities and scope

The R25-based release is intended to preserve the verified Payroll workflows, including:

| Area | Intended functionality |
| --- | --- |
| Payroll delivery | Import, review and publish payslips; review published history and corrections |
| Approval workflow | HR- or finance-led review paths, with configurable approval/release steps |
| Access management | Separate administrator, HR, finance and employee permissions |
| Employee enrollment | In-person, time-limited QR pairing and an additional temporary verification code |
| Employee access | Payslip viewing through a paired official iOS app |
| Deployment | Self-hosted Docker deployment, persistent data and HTTPS access |
| Initial geographic templates | Mainland China and Hong Kong |

**Release gate:** The public R25 Go build and the matching official iOS build must be regression-tested together before these capabilities are represented as generally available. Specific workflows, export controls, offline behavior and security mechanisms must follow the tested release notes, not assumptions from earlier Python builds.

Payroll Server is a **payslip distribution and workflow product**, not a substitute for a payroll calculation engine, statutory tax calculation, MPF processing or filing system. Organizations remain responsible for the correctness of their payroll figures and local legal compliance.

### Editions and licensing

- **Community Edition:** No server software license fee for use permitted by the accompanying license; self-hosted by the organization.
- **Source model:** Proprietary, source-closed. Public GitHub visibility does **not** make this an open-source project.
- **Mobile client:** The official App Store iOS application is separately distributed under its applicable terms. Its price and availability, if any, are determined by the official listing.
- **Other offerings:** Commercial editions, database engines, support or integrations may be offered separately; they are not promised as part of this release.

Read the complete [Payroll Server Community Edition License](LICENSE.md) before using or distributing the Software. Third-party dependencies retain their own applicable licenses.

> The `LICENSE.md` copyright holder, contact address and jurisdiction fields must be finalized before an official release.

### Official iOS client

**App Store link:** Not published in this README until the genuine listing and developer identity are verified.

For employee mobile pairing and salary-slip retrieval, use only the genuine official App Store app. The client must connect to the organization's configured **public HTTPS origin** using a certificate trusted by iOS.

Client authenticity, pairing, device binding and server-side authorization are separate security controls. **Do not assume that App Attest or another attestation mechanism is enforced in a particular build until that build passes the published release verification.** An app name, Bundle ID string or User-Agent alone is not proof of authenticity.

**Administrative browser access is not subject to the employee-mobile-client requirement.**

### Deployment model

```text
Employee iPhone (official App Store app)
                 |
         Trusted public HTTPS
                 |
        Organization's reverse proxy
                 |
          Protected backend HTTP
                 |
      Payroll Server (Go container)
                 |
      Persistent storage (/data)
```

The server can also expose an **internal HTTPS administration endpoint** when the released build supports and enables that feature. An environment variable or exposed port alone does not create a TLS listener.

### Prerequisites

- Linux server with Docker Engine and Docker Compose
- Organization-controlled persistent storage and a separate backup destination
- A DNS name reachable by authorized employee devices, with a publicly trusted TLS certificate for iOS
- A properly configured reverse proxy, or a supported verified TLS deployment mode
- Firewall rules limiting backend HTTP access to designated trusted reverse proxies
- The **official** Compose file, configuration template, and image tag from the same published release

### Quick start (after the official release is published)

The commands below assume that the public repository **already contains** the verified `docker-compose.yml` and `.env.example` files and that the Compose file references an **actual published official image**. They cannot be used to install a release that has not been uploaded.

```bash
git clone <OFFICIAL_REPOSITORY_URL>
cd payroll-server

cp .env.example .env
chmod 600 .env

# Edit .env: public HTTPS URL, trusted hosts/proxy sources,
# ports, stable secrets and the intended persistent data volume.
nano .env

docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Replace `<OFFICIAL_REPOSITORY_URL>` with the verified URL after publication. The initial organization and administrator setup procedures will be documented for the **actual R25 Go release**; do not reuse Python/Flask initialization commands.

**Important:** Never put real employee records, salary files, private keys, `.env`, database files or backups in this public repository.

### Key configuration concepts

The following names correspond to the current Go deployment configuration discussed for R25. Refer to the released `.env.example` and Compose file for exact defaults and support.

| Setting | Purpose |
| --- | --- |
| `PAYROLL_PUBLIC_URL` | Employee-facing trusted HTTPS origin; may include a non-default port |
| `PAYROLL_LAN_IP` | Host LAN IP used for the designated backend port mapping |
| `PAYROLL_HTTP_PORT` | Host HTTP port for a **restricted reverse-proxy upstream**, not a public login endpoint |
| `PAYROLL_TRUSTED_HOSTS` | Explicitly accepted HTTP hosts |
| `PAYROLL_TRUSTED_PROXY_CIDRS` | Actual trusted proxy source CIDRs; do not trust entire LANs by convenience |
| `PAYROLL_INTERNAL_TLS_IP` | Optional host bind address for a supported internal TLS port mapping |
| `PAYROLL_INTERNAL_TLS_PORT` | Internal HTTPS port when a genuine TLS listener/proxy is configured |
| `PAYROLL_DATA_VOLUME` | Exact Docker volume name to preserve across upgrades |

Keep `PAYROLL_PUBLIC_URL` on HTTPS. In production, a direct HTTP request can be rejected with `HTTPS_REQUIRED`; a reverse proxy must pass correct HTTPS forwarding information and be explicitly trusted. The iOS app must **not** accept a self-signed certificate for the employee-facing public origin merely because internal administrator browsers may support a separate self-signed IP certificate.

### Existing installations and upgrades

**Preserve the existing server identity and data.** Before any upgrade:

1. Record the actual Compose project name, environment file, image version and Docker volume name.
2. Create a **restorable, encrypted backup** of the payroll database, server keys and necessary configuration. For SQLite, use a database-consistent backup procedure; do not assume that copying a live database file is sufficient.
3. Test restoration separately and confirm that the backup is accessible outside the container host.
4. Follow the migration notes for the specific release. Reuse the existing named volume and stable cryptographic keys.
5. Pull the verified release image and recreate the service using the **same** Compose project and storage configuration; check logs, administrator login, employee accounts, roles and existing payslips.
6. Confirm official iOS pairing and access before treating the upgrade as complete.

**Never use `docker compose down -v` for a normal upgrade.** Do not reset or reinitialize the organization/admin just because the application reports a migration or data-readiness error. A new Compose project name can silently select a different volume and make existing data appear missing.

### Security guidance

- Publish the browser and mobile endpoints only through appropriately configured HTTPS.
- Keep the backend HTTP upstream inaccessible from the public Internet.
- Forward the original Host and HTTPS scheme through your **trusted** reverse proxy; verify the actual proxy source IP or container-network source.
- Apply least privilege for administrators, HR, finance and employee accounts, and protect privileged sessions with the supported MFA controls.
- Retain audit records, backups and encryption keys according to organizational policy and applicable law.
- Restrict access to payroll data, secrets and server logs; do not publish diagnostic logs containing personal data.
- Treat package checksums as **integrity checks**, not proof of code identity or an independent security audit. Verify signed artifacts where official signatures are available.

### Releases, support and security reports

- **Release artifacts:** Will appear in this repository's Releases section after publication and verification.
- **Container images:** The official registry path and immutable image digest will be announced with the release. No unverified image path is supplied here.
- **Supported architectures:** The planned R25 release targets Linux AMD64 and ARM64, subject to successful build and runtime validation.
- **Security contact:** To be published by the licensor before public distribution; do not post confidential employee data in public issues.
- **Support:** Community Edition does not include a guaranteed support SLA.

### License

Copyright © 2026 **[LEGAL NAME OF RIGHTS HOLDER]**. All rights reserved.

See [`LICENSE.md`](LICENSE.md). Proprietary, no-charge self-hosted distribution; **not open source**. Apple and App Store are trademarks of Apple Inc.; Apple does not sponsor or endorse this software.

---

<a id="简体中文"></a>

## 简体中文

### 项目介绍

**Payroll Server** 是 **OpsHome Payroll** 的企业私有部署工资条后端。企业将服务器和主要薪资数据库部署在自己控制的基础设施上，通过管理后台完成人员管理、薪资条导入、审核与发布，并由员工在官方 iOS 客户端查阅工资条。

**Community Edition（社区版）免费提供符合许可证条款的服务端使用授权，但并非开源软件。** 本公开仓库计划发布部署文件、说明文档和官方编译发行物，不公开 Go 核心业务源代码。

### 主要业务范围

- 人事专员导入 → 人事经理审核 → 可选财务发布。
- 财务专员导入 → 财务经理审核 → 发布。
- 支持自定义审批节点，最多三个节点；具体权限以发布版回归测试为准。
- 系统管理员、人事、财务与普通员工账号分离，并按权限展示管理菜单。
- 员工现场通过限时二维码及动态识别码完成设备配对。
- 员工通过官方 iOS App 查阅已发布工资条及相应历史记录。
- 首批部署目标为中国大陆与香港，默认采用自托管 SQLite 持久化方案。

**状态声明：** 上述为目标发行功能范围，不能取代 R25 Go 后端与配套 iOS 正式版本的逐项测试结论。离线查看、导出权限、离职员工 PDF 权限等行为应以实际验证的版本说明为准。本项目不替代税务、MPF 或工资计算系统。

### 仅使用官方 App Store 客户端

企业员工移动端使用必须通过开发者在 **Apple App Store** 官方发布的 OpsHome Payroll App；未经授权的第三方客户端、修改版或重签名版不属于本许可证的授权范围。

但**管理员的官方 Web 管理后台仍可正常使用浏览器访问**，不因上述要求受到限制。

客户端真实性需要由服务端结合设备绑定、请求证明、身份与权限检查等机制验证。仅靠 App 名称、请求头或 Bundle ID 文本不能证明客户端合法。**Apple App Attest 是否已接入并生效，必须由具体发行版本的测试结果确认，不能在发布前宣称已实现。**

### 部署及网络原则

- **公网：** iOS 客户端必须使用组织配置的 HTTPS 域名与 iOS 信任的有效证书。
- **反向代理：** 后端 HTTP 端口只能向明确允许的代理提供回源，不能直接暴露公网。
- **内网：** 管理员可通过支持的内网 HTTPS 方案访问；IP 自签名证书仅用于单独配置的内网浏览器场景，不用于 iOS 公网配对。
- **数据库：** SQLite 数据库与服务端密钥存放在持久化卷中；升级必须保留原卷。
- **首次初始化：** 使用正式 Go 发行版本提供的流程，不沿用旧 Python/Flask 的管理命令。

正式镜像发布后，再按上方 English / Quick start 部分的步骤下载**真实存在且已验证**的 Docker Compose 配置和镜像。

### 升级时特别注意

备份数据库、密钥和 `.env`，同时确认 Compose 项目名以及 `PAYROLL_DATA_VOLUME` 指向的真实 Docker 卷。备份必须可恢复；SQLite 应采用一致性备份。升级后检查管理员、员工、权限、工资记录和 iOS 客户端配对。

**禁止把 `docker compose down -v` 当成日常升级命令。** 不要因配置错误而删除数据卷、重建组织、重设管理员或清空 MFA。

### 许可协议

本项目采用 [Payroll Server Community Edition License v1.0](LICENSE.md)，属于**免费使用的专有许可证**，不是 MIT、Apache、GPL、AGPL 等开源许可证。个人与企业可在协议允许范围内免费自托管；未经授权不得修改、再分发、转售服务端或使用第三方员工移动客户端。

软件许可证不代表承诺未来全部功能永久免费，也不包含默认的托管服务或 SLA。企业自负主机维护、HTTPS、备份、数据隐私和当地合规责任。

**首次公开发布前待完成：** 补全许可证版权主体与联系信息，核实官方 iOS App Store 链接、镜像地址、签名与校验材料，以及 Go R25 的真实功能回归测试。
