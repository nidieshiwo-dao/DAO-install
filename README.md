# DAO installation bootstrap

This public repository contains the standalone installation and diagnostic scripts, README and license. DAO application source code and release packages remain in the private `nidieshiwo-dao/DAO` repository.

## 主站

支持 Debian 12/13、Ubuntu 22.04/24.04，Linux amd64/arm64 和 systemd。无需域名或手填 IP。开放主站 TCP 80/443，脚本自动识别公网 IP：

```bash
curl -fsSL https://raw.githubusercontent.com/nidieshiwo-dao/DAO-install/main/install-main.sh | sudo bash -s -- --version v0.4.0-rc.12
```

也可用 `wget -qO-` 替代 `curl -fsSL`。私有安装包需要 GitHub 只读授权：脚本会隐藏输入 Token，或传入 `--github-token-file /root/dao-readonly-token`。已有且有效、能读取 DAO 私有仓库的 Token 也能使用，不强制另建。推荐专用 fine-grained PAT，仅选择 DAO 仓库，授予 Contents: Read-only。凭据只由主站保存，用于自身和节点的发行下载；不写入命令 URL，也不下发到 Worker。

不指定 `--version` 时只安装最新正式版。当前试部署版本为 `v0.4.0-rc.12`，请明确指定。安装器校验 SHA256，自动准备数据库、服务和管理员。首次启动前默认启用 HTTP Basic Auth，浏览器入口凭据在 `/etc/dao/basic-auth.json`（`sudo cat` 查看，root-only）；通过后再使用独立管理员密码登录。管理员初始密码在 `/etc/dao/admin-password`，运行后配置 LLM 与业务所需凭据。

新安装优先通过两个独立 HTTPS 查询源确认公网 IPv4，IPv4 不可用时尝试 IPv6；忽略代理环境变量，避免识别到代理 IP。查询冲突或失败会停止并提示加 `--public-ip 实际IP`，不回退 HTTP。多 IP 或入站/出口地址不同也可手动指定；重装默认沿用已保存地址，显式 `--public-ip auto` 才重新识别。查询只能确认出口地址，公网可达性仍由证书签发与节点接入验证。已有反向代理可使用 `--listen 127.0.0.1` 自行配置 HTTPS。

IP 证书由 Let’s Encrypt 签发，有效期约 6 天；安装器配置独立 `dao-https` 服务自动续期并加载新证书，无需手动重启主站。主站后端仅监听本机。请持续开放 TCP 80/443；证书日志：`journalctl -u dao-https`。有域名也可把 `--public-ip` 换成 `--domain dao.example.com`。IPv6 参数不加方括号，浏览器地址为 `https://[IPv6]`。

## Worker

在主站“远程 Worker → 安装与接入”添加节点，填写主站 HTTPS 地址和容量，在“安装 Worker 节点”生成一次性命令，复制到对应 VPS 执行。安装码 15 分钟有效，仅绑定该节点；不用手工上传 JSON，不需要 GitHub Token。

命令格式如下；节点 ID 和安装码必须使用主站生成的实际值，每台 VPS 各一份：

```bash
curl -fsSL https://raw.githubusercontent.com/nidieshiwo-dao/DAO-install/main/install-worker.sh | sudo bash -s -- --controller 'https://主站公网IP' --node '节点ID' --enroll '一次性安装码'
```

Worker 无需域名、IP 证书或入站端口，只需出站访问主站 HTTPS 和软件源。

节点直接从 GitHub 下载配套版本，准备本机 PostgreSQL 和执行环境，注册后由主站确认就绪。主站不转发安装包。默认执行环境包含基础命令、Python、Node、Chromium 和 Playwright MCP；其他工具的凭据、字典或特殊依赖按需要配置。

## 安装前检查与主机名修复

新引导脚本提前准备 PostgreSQL 管理依赖，避免首次安装的 `pg_lsclusters: not found`。这也覆盖旧 `rc.3` 主站生成的 Worker 安装命令，主站无需升级。

本机主机名无法解析时，仅在确认合法短名称或完整域名 FQDN 与静态配置一致且缺少 hosts 映射时，备份 `/etc/hosts` 后补齐映射并验证；保留已有条目、权限和特殊配置，复杂情况给出明确提示。完整域名只在短别名没有现有映射时才一同补齐，不覆盖服务商配置。已安装主站或 Worker 可单独执行：

```bash
curl -fsSL https://raw.githubusercontent.com/nidieshiwo-dao/DAO-install/main/install-main.sh | sudo bash -s -- --repair-hostname
```

该参数单独使用，不需要 GitHub Token，不安装应用或重启 DAO。若仍不能解析会返回非零；备份位于 `/etc/hosts.dao-backup-*`。第一次 sudo 启动时的旧解析警告可能仍会出现。

公开脚本的主机名/数据库准备修复覆盖后续旧版安装；已发布 ZIP 保持不变。使用旧本地包时，以最新公开入口加 `--bundle-dir` 调用即可获得这些引导修复。

`v0.4.0-rc.4` 的包内安装器另修正了参考 CLI 统计，并明确安全 tar 解压策略，消除 Python 3.13 弃用警告。CLI 按可导入配方去重统计，内置/待适配项不计入分母，缺失依赖写入工具清单；不表示所有第三方工具已安装。正在运行的 rc.3 可继续使用，无需为这些提示停机重装；其包内安装器和已有清单不会被公开脚本追溯替换。以后升级主站/Worker 时保持同版。

## 已安装 rc.3 / rc.4 的过渡升级

rc.10 增加默认入口 Basic Auth、DAO 标题、任务发现自动归属、菜单与实时连接优化；在线更新保留既有认证设置，已部署站点请在系统配置中确认启用。rc.9 修复旧审批入口的无节点请求、安装探测误报及数据库 journal 采集超时；rc.8 补齐登录限流、HTTP 审计、无节点页面门控及扩展诊断；此前 rc.7 包含七项能力、任务对话返回全局入口修复，以及 Worker 重启后配置不变时的在线更新健康确认修复。rc.5 增加管理对话、Telegram、二级菜单和主站/节点在线更新。旧版不能读取预览更新，也没有节点在线换装能力，首次需通过脚本升级。先完成或暂停任务、结束节点对话，主站升级并恢复访问后再升级各节点：

```bash
# 主站；沿用原配置、数据库和安装凭据
sudo systemctl stop dao && curl -fsSL https://raw.githubusercontent.com/nidieshiwo-dao/DAO-install/main/install-main.sh | sudo bash -s -- --version v0.4.0-rc.12
# 各 Worker；沿用原节点身份，无需重新生成安装码
sudo systemctl stop dao-worker && curl -fsSL https://raw.githubusercontent.com/nidieshiwo-dao/DAO-install/main/install-worker.sh | sudo bash -s -- --config /etc/dao-worker/config.json
```

以后在“系统配置 → 版本与更新”检查更新，预览版默认跟随预览通道。主站先更新，勾选同步的节点在空闲后分批更新（最多两台同时），直接从 GitHub 下载。任务运行中不强行升级；协议、数据库或运行依赖不兼容时明确要求安装器。

## 重装与排障

在主站和 Worker VPS 分别执行同一条命令，自动识别本机角色，收集程序版本、近期脱敏日志、服务状态、重启次数和资源用量：

```bash
curl -fsSL https://raw.githubusercontent.com/nidieshiwo-dao/DAO-install/main/diagnose.sh | sudo bash
```

支持已安装的 rc.3 及以后版本，无需先升级或提供 GitHub Token。最后打印 `/tmp/dao-check.XXXXXX/dao-diagnostics.tar.gz`，下载后提供给维护者；不会自动上传、不重启服务、不升级或安装软件。目录权限 0700，压缩包 0600。完整配置、密码文件、数据库与任务目录不会打包；常见日志凭据使用 DAO 已有脱敏，业务内容分享前仍需检查。某项无法读取时保留状态及错误记录，包内 `partial/issues` 表示缺失信息。离线日志不能代替数据库和实际任务的在线健康检查。

也可直接上传到私有 `nidieshiwo-dao/DAO-diagnostics`（通过 `--repo nidieshiwo-dao/DAO-diagnostics` 或环境变量 `DAO_DIAGNOSTICS_REPO` 指定），无需手动下载转交：

```bash
curl -fsSL https://raw.githubusercontent.com/nidieshiwo-dao/DAO-install/main/diagnose.sh | sudo bash -s -- --upload
```

终端会隐藏询问该诊断仓库的 GitHub 写入 Token，输入不保存。建议只授权 `DAO-diagnostics` 的 fine-grained Token，权限为 `Contents: Read and write`；也支持有效的单仓库 GitHub App 短期令牌。安装器的只读 Token 不足以上传，任务备份的单任务仓库令牌也不适用于诊断仓库；脚本不会读取或下发主站管理 Token。

重复使用可由管理员把专用授权保存为 root 所有、权限 0600 的 `/etc/dao-diagnostics/github-token`，或追加 `--token-file /root/dao-diagnostic-token`。无终端且未提供文件时明确失败，不回退明文输入。其他账户使用 `--repo owner/private-repo` 指定已创建的私有仓库。

若旧入口在交互式 SSH 终端误报“无法隐藏输入 Token”，重新运行上述 curl 命令获取修复后的脚本即可，无需升级 DAO。隐藏输入直接交由 Python `getpass` 打开控制终端，支持管道/内嵌脚本占用标准输入的场景；真正没有控制终端时仍需 `--token-file`。`sudo: unable to resolve host` 是独立的主机名解析问题，可使用上方 `--repair-hostname`，不通过重装服务处理。

各 VPS 直传 GitHub，每次最多 16 MiB，按 UTC 日期保存在私有 Release 附件中，不经过主站、不进入源码仓库或任务备份。核对仓库私有性、远端大小与 SHA-256 后才显示上传成功、诊断编号和地址；授权、网络或校验失败保留本地包，已有上传响应丢失时按唯一编号核查，不盲目重传。回执保存在包旁 `upload-receipt.json`，不含 Token。将编号告诉有仓库访问权限的维护者即可读取；上传不会自动启动检查，也没有定时上传或自动删除历史附件。

升级前先暂停/完成任务、停止接收新任务，再停止 `dao` 或 `dao-worker` 服务。安装器拒绝自动覆盖运行中的服务，保留配置、数据库与任务文件，不自动回滚数据库。

安装日志在 `/var/log/dao-install/`（0600，每角色最多 10 份、每份最多 8 MiB）。运行日志：`journalctl -u dao` 或 `journalctl -u dao-worker`。遇到下载中断可以重跑同一命令；已经启动但未就绪时先检查日志和主站节点状态。

The installation scripts are generated by `scripts/build-bootstrap.py`; `diagnose.sh` is generated by `scripts/build-diagnostics.py` from its shell template and opt-in uploader. No repository credentials, source history, or application binaries are included here. Licensed under AGPL-3.0; see LICENSE.

诊断脚本默认补充最近 24 小时系统日志、HTTP 审计和应用/代理/数据库 journal，兼容已经安装的 rc.3。追加 `--hours 72` 可看更早的保留记录；每来源最多 20000 行/16 MiB，包内 collection.json 标明条数、时间范围及截断，不再只依靠旧二进制最后 200 条。日志仍仅显式上传，不实时回传。
