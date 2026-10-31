# OpsHome Payslip Server 部署与后台还原指南

适用于 Self-Hosted Community Edition 的 Go 企业服务端，按 R27 交付配置整理。服务端保存企业员工、工资条和设备绑定，员工通过官方 iOS 客户端访问。本文覆盖 Docker Compose 首次部署、保留数据升级，以及通过 Web 后台上传加密备份还原。

首次部署必须按顺序执行 `db-upgrade`、`init-company`、`create-admin`，然后启动服务。当前版本没有网页初始化向导。后台还原也需要先建立一个可登录的临时环境，再上传原备份；成功后使用备份中的原管理员重新登录。

## 1 部署前准备

- Linux 服务器，以及已安装并可用的 Docker Engine 和 Docker Compose 插件。
- 官方发布的镜像地址与明确的版本号。镜像支持 Linux AMD64 和 ARM64。
- 企业公网域名及公网可信 HTTPS 证书；反向代理将 HTTPS 请求转发到服务端 HTTP 端口。
- 持久数据卷、可用磁盘空间、正常系统时间，以及管理员 TOTP 验证器。
- 如需灾备还原，准备已下载到服务器之外的 `.enc` 快照、独立备份口令和原管理员 MFA。

普通部署使用官方运行镜像，不需要 Go、Xcode、源码或 Garble seed。Self-Hosted Community Edition 不表示源码已开源；使用限制以随发行版提供的许可为准。

## 2 准备 Compose 与环境配置

创建专用目录，例如 `~/payroll-go`。以下是仅发布 HTTP 上游端口的部署示例；如已有官方 Compose 文件，可保留原文件并核对对应变量。

```bash
mkdir -p ~/payroll-go
cd ~/payroll-go
```

保存为 `compose.yml`：

```yaml
services:
  payroll:
    image: ${PAYROLL_IMAGE:?Set the official image and version}
    ports:
      - "${PAYROLL_LAN_IP:-127.0.0.1}:${PAYROLL_HTTP_PORT:-8098}:8000"
    environment:
      PAYROLL_LISTEN: "0.0.0.0:8000"
      PAYROLL_DATABASE_URL: "sqlite:////data/payroll.sqlite3"
      PAYROLL_SECRET_KEY: ${PAYROLL_SECRET_KEY:-}
      PAYROLL_PUBLIC_URL: ${PAYROLL_PUBLIC_URL:?Set the public HTTPS origin}
      PAYROLL_TRUSTED_HOSTS: ${PAYROLL_TRUSTED_HOSTS:?Set the allowed hosts}
      PAYROLL_PROXY_HOPS: "1"
      PAYROLL_TRUSTED_PROXY_CIDRS: ${PAYROLL_TRUSTED_PROXY_CIDRS:?Set the actual proxy source CIDRs}
      PAYROLL_DEV_MODE: "0"
      PAYROLL_CLIENT_LICENSE_ALLOW_SANDBOX: ${PAYROLL_CLIENT_LICENSE_ALLOW_SANDBOX:-0}
      PAYROLL_WORK_HOURS_ENABLED: ${PAYROLL_WORK_HOURS_ENABLED:-0}
      PAYROLL_CLIENT_CREDIT_ENABLED: ${PAYROLL_CLIENT_CREDIT_ENABLED:-1}
      PAYROLL_CLIENT_CREDIT_TEXT: ${PAYROLL_CLIENT_CREDIT_TEXT:-}
    volumes:
      - payroll-data:/data
    read_only: true
    tmpfs:
      - /tmp:size=320m,mode=1777
    cap_drop: [ALL]
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

volumes:
  payroll-data:
    name: ${PAYROLL_DATA_VOLUME:?Set a persistent volume name}
```

保存为 `.env`，并将示例值改为实际配置：

```dotenv
# 示例镜像地址与版本，替换为发布方实际提供的值。
PAYROLL_IMAGE=your-dockerhub-user/payroll-server:0.4.3
PAYROLL_DATA_VOLUME=payroll-go-data
PAYROLL_LAN_IP=127.0.0.1
PAYROLL_HTTP_PORT=8098
PAYROLL_PUBLIC_URL=https://payroll.example.com
PAYROLL_TRUSTED_HOSTS=payroll.example.com,localhost,127.0.0.1
# 填反向代理连接到容器时的实际来源 CIDR，不能直接照抄通用网段。
PAYROLL_TRUSTED_PROXY_CIDRS=127.0.0.1/32,::1/128
PAYROLL_CLIENT_LICENSE_ALLOW_SANDBOX=0
PAYROLL_WORK_HOURS_ENABLED=0
PAYROLL_SECRET_KEY=
```

```bash
chmod 600 .env
sudo docker compose config --quiet
sudo docker compose pull
```

`0.4.3` 仅为版本格式示例，不代表最新发布。镜像数字标签采用 `X.Y.Z`，不带 `v`；发布还会更新 `latest`，生产部署建议固定数字版本或 digest。私有源仓库的 Git 发布标签仍可为 `vX.Y.Z`，工作流会剥离 `v` 后生成镜像标签。

`PAYROLL_SECRET_KEY` 留空时，程序在 `/data` 生成并持续使用 `server.key`。日常升级不得删除或随意替换该密钥。`.env`、数据卷和证书配置不应提交到 GitHub。

## 3 配置 HTTPS 反向代理

同机反向代理可连接 `127.0.0.1:8098`；不同主机反代需将 `PAYROLL_LAN_IP` 改为明确的内网 IP，并用防火墙只允许反代主机访问上游端口。

反代应保留原始 Host，正确传递 HTTPS 协议和客户端地址。例如 Nginx 的 location 中可使用以下头部配置，`proxy_pass` 根据实际网络修改：

```nginx
proxy_pass http://127.0.0.1:8098;
proxy_set_header Host $http_host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

这段仅是转发示意，不包含完整的站点、证书和上传大小配置。可信代理 CIDR 必须匹配容器看到的实际 TCP 对端；Docker 网桥或多层反代可能使它不同于宿主机回环地址。不要以信任所有地址来解决 403。

App 只使用公网域名与公网可信证书。内网 IP 自签名 HTTPS 仅用于管理后台；本示例未启用该可选端口。官方 App API 和授权回调不能被仅浏览器可完成的质询拦截。

如上传备份出现 413 或超时，应检查反代及前置网关的请求大小、超时限制。服务端快照本身也有资源上限，不能仅通过调大反代参数实现无限容量。

## 4 首次部署

在 Compose 项目目录执行。每条命令成功后再继续下一条：

```bash
sudo docker compose run --rm payroll db-upgrade

# 中国大陆用 CN，香港用 HK；企业名称替换为实际名称。
sudo docker compose run --rm payroll init-company \
  --name '企业名称' --region CN

sudo docker compose run --rm payroll create-admin --username admin

sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 payroll
```

创建管理员时按提示输入并确认密码；程序只显示一次敏感的 TOTP 登记 URI。将它添加到管理员的验证器，不要公开到截图、工单或 CI 日志。随后通过企业 HTTPS 地址登录后台，使用密码和动态验证码完成认证。

系统管理员默认具备系统管理及设备管理能力。人事导入、审核、财务和发布等业务权限按实际分工授权，不应将创建系统管理员理解为自动获得全部业务角色。

正式企业默认 `PAYROLL_CLIENT_LICENSE_ALLOW_SANDBOX=0`。仅 TestFlight 测试环境按需设为 `1`；它不关闭官方 App 验证，App Attest 也不应因此改用 development。

## 5 创建后台加密备份

1. 使用系统管理员登录，打开「加密备份与灾备还原」，路径为 `/admin/system/backups`。
2. 设置至少 16 个字符的独立备份口令，并重复输入确认。
3. 下载全量加密快照 `.enc`，确认浏览器下载完成。
4. 将快照保存在服务器之外，并与口令分开保管。记录备份时间、运行镜像版本和企业公开 HTTPS 地址。

快照包含一致性数据库、服务端身份密钥和用于还原校验的信任配置。数据库中包含员工、工资条、业务配置、设备绑定及相应认证状态。快照不会自动重建宿主机、Docker、DNS、反代、证书或 `.env`，这些部署配置需另行安全保存。

备份口令不等于后台密码或 MFA，丢失后无法找回。备份之后新增的工资条、审批和设备变更不会出现在旧快照中。不要将运行中的 SQLite 主文件直接复制后当成全量一致性备份。

## 6 保留数据升级

先在后台下载备份，保留原数据卷、`.env` 和企业公开 origin。在 `.env` 中将 `PAYROLL_IMAGE` 改为已发布的新数字版本，然后执行：

```bash
sudo docker compose pull
sudo docker compose stop payroll
sudo docker compose run --rm payroll db-upgrade
sudo docker compose up -d
sudo docker compose logs --tail=100 payroll
```

已有数据库升级不执行 `init-company` 或 `create-admin`，不需要重新创建企业、管理员或员工。普通用户拉取镜像即可；私有源码构建是维护者流程，不能把 `--build` 当作清空数据库的方法。

`sudo docker compose down` 删除容器和项目网络，保留普通命名数据卷；重新启动后原账户仍可用属于预期行为。`down -v` 或 `docker volume rm` 可能删除持久数据，不用于日常升级。

修改 `.env` 后通过 `docker compose up -d` 重新创建容器以应用新配置。仅 `docker restart` 不会更新既有容器的环境变量。回滚旧镜像前确认数据库结构兼容；不兼容时应使用升级前备份与对应版本恢复。

## 7 空服务器通过后台还原

### 7.1 准备恢复目标

在新主机安装 Docker 并准备 Compose、`.env` 和反代。建议先使用与备份时相同的官方镜像版本，完成恢复后再按升级流程升级。

必须将 `PAYROLL_PUBLIC_URL` 配置为备份中的原企业 HTTPS origin，包括原有的非默认端口。公网可信证书、DNS 或反代入口须正常工作；官方信任参数与 App 身份也要匹配。后台会拒绝不匹配的信任链或公开地址，不应通过删除校验来绕过。

使用新数据卷进行恢复。异机部署可以保留相同卷名；同一主机演练则使用不同的新卷名，并停掉原容器以避免端口冲突，防止误覆盖原系统。卷名只表示存储位置，不是企业身份。

### 7.2 建立可登录的临时后台

在恢复目标的 Compose 目录执行：

```bash
sudo docker compose pull
sudo docker compose run --rm payroll db-upgrade

# 地区按原企业选择 CN 或 HK。
sudo docker compose run --rm payroll init-company \
  --name '灾备还原临时环境' --region CN

sudo docker compose run --rm payroll create-admin \
  --username recovery-admin

sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 payroll
```

给临时管理员设置密码并登记此次生成的 TOTP。空卷没有原管理员，因此此时必须使用临时账号登录；不能假设原账号已经存在。不要导入正式工资或给员工重新配对。

### 7.3 在网页上传备份

1. 使用 `recovery-admin` 的临时密码与临时 MFA 登录新后台。
2. 打开 `/admin/system/backups`，选择原 `.enc` 文件。
3. 输入创建该快照时设置的独立备份口令。
4. 输入大写 `RESTORE`，点击「校验并还原系统」。该操作替换当前系统，而非合并数据。
5. 等待「灾备还原完成」页面，按提示重新登录。

服务端先验证口令、格式、数据结构、信任参数及公开地址，成功后保存恢复前快照并切换数据库与身份密钥。新后台的临时企业、临时账号及其 MFA 会被原快照替换，当前后台登录会话会失效。

### 7.4 使用原账号验证

使用快照中的原管理员、原密码和原 MFA 重新登录。核对企业名称及 Logo、员工、工资批次、审批、回执与异议、后台角色以及设备绑定。数据应对应备份时点，不应期待恢复备份后的变更。

iPhone 保持原企业绑定，不卸载、不主动解绑，解锁后刷新工资条。恢复保留原身份和设备记录；当会话令牌发生变化时，配套客户端可通过设备签名恢复通信。设备已被撤销、硬件密钥丢失或绑定晚于备份时点时，不能保证自动恢复，应按实际设备状态处理。

还原成功后再次重启容器，复查原账号和 App 连接，确认恢复身份可持续使用。恢复前自动备份保存在数据卷的 `recovery-snapshots`，使用本次还原口令；它不能替代服务器外的独立备份。

## 8 在原主机演练

推荐停掉旧容器后，把 `.env` 中的 `PAYROLL_DATA_VOLUME` 改为一个新的、尚未使用的测试卷名，例如 `payroll-restore-test-20261031`。保留原卷不动，按照第 7 节建立临时后台并上传备份。

这能模拟空数据库恢复，同时保留退回原系统的条件。若需退回，停掉测试容器，将 `.env` 恢复为原卷名及其兼容镜像版本，再启动。切换过程中不要让两个环境同时对外写入或重新绑定同一批设备。

若明确需要删除旧卷测试，先确认服务器外备份可用、口令已保存，再核对实际卷名。以下只演示本次现场确认的 `payroll-go-data`，其他部署不能直接照抄：

```bash
sudo docker compose down
sudo docker volume rm payroll-go-data
sudo docker volume create payroll-go-data
```

删除后直接进入第 7.2 节。仅执行 `up -d` 不会替你初始化数据库和管理员；还原操作仍通过 Web 后台完成，不使用手工解压数据库代替。

## 9 常见问题

| 现象 | 检查与处理 |
| --- | --- |
| 删除容器后原账户还能登录 | 数据卷被保留，属于正常重建。验证空系统应使用新卷或明确删除旧卷。 |
| 空卷启动失败或旧账号不能登录 | 先 `db-upgrade`，再 `init-company`、`create-admin`；登录临时管理员。 |
| 临时管理员也不能登录 | 使用本次生成的 TOTP，核对系统时间、HTTPS 与 Host；不要使用原账号的验证码。 |
| 提示口令错误或快照损坏 | 核对备份口令和文件完整性，不以登录密码代替；验证失败时不应改动现有业务数据。 |
| `SNAPSHOT_TRUST_MISMATCH` | 使用对应官方发行镜像，核对 App 身份与官方验签参数；不要关闭校验。 |
| 提示使用原企业 HTTPS 域名 | 还原目标的 `PAYROLL_PUBLIC_URL` 必须与快照一致，包括端口。 |
| 还原完成后临时账号失效 | 属于预期；改用备份中的原账号、原密码和原 MFA。 |
| 上传返回 413 或超时 | 检查反代及前置网关上传限制、超时、可用磁盘与容器临时空间。 |
| 提示 `SNAPSHOT_INCOMPATIBLE` | 核对备份与镜像版本，先使用原版本恢复；不要直接替换运行中的数据库文件。 |
| 提示恢复切换中断 | 先保留数据卷与恢复前快照，按错误页提示重启完成恢复日志；不要再次清空卷。 |
| 设备恢复后不能刷新 | 检查原 HTTPS origin、可信证书、设备状态、备份时点与配套客户端版本；保留错误码排查。 |

## 10 验收记录

2026 年 10 月 31 日，维护者在实际环境完成本指南对应的临时初始化及 Web 后台还原流程，并反馈完成。该记录表示该次流程已走通，不替代各企业对具体备份、版本、网络和设备的验收。

交付或演练时记录镜像版本及 digest、备份时间、还原时间、企业 origin、后台数据核对结果、原设备连接结果和重启后结果。不要在 GitHub Issue 中提交快照、口令、MFA URI、私钥或真实工资数据。
