# xy：VLESS + WebSocket + TLS 管理脚本

基于 [brian952700/xray-xhttp-manager](https://github.com/brian952700/xray-xhttp-manager) 的源码快照（提交 `891ad80f8f1287c33ebcb1c572c7a80479b916a2`）复制并改造，使用 **VLESS + WS + TLS**，默认启用 **VLESS Encryption**，Flow 留空。

## 为什么启用 VLESS Encryption

仅将 xHTTP 换成 WS 不能解决以下 VLESS 弃用警告：

> The feature VLESS (with no Flow, etc.) is deprecated

本项目通过 `xray vlessenc` 生成配对参数：服务端使用 `decryption`，分享链接携带客户端 `encryption`，不使用 `encryption=none`。WebSocket 不应填入 `xtls-rprx-vision`；Flow 保持为空。TLS 的 ALPN 明确为 `http/1.1`。

客户端及其实际运行的 Xray 核心必须支持 VLESS Encryption，且导入器必须完整保留链接中的 `encryption` 参数。安装后重新导入新链接；若客户端仍显示该警告，检查它实际生成的配置中 `encryption` 是否被丢弃或改为 `none`。不能承诺不支持该功能的旧客户端不出现警告。

## 环境与安装

主要目标：**Debian 13 + systemd 的 Linux VPS**。需要 root 或 sudo，以及解析到服务器的域名。公网 TCP 80 用于 ACME 申请及续期，节点端口默认 TCP 443；脚本不自动修改防火墙。有 AAAA 记录时 IPv6 也必须可达。

```bash
bash -o pipefail -c 'curl -fsSL https://raw.githubusercontent.com/brian952700/xray-ws-manager/main/install.sh | bash'
```

安装时输入域名、端口和 WebSocket 路径（默认 `/`），将生成的 `vless://` 链接导入支持的客户端。以后运行 `sudo xy`（root 直接运行 `xy`）。

## 管理菜单

1. 安装/更新环境，保留已有配置
2. 添加用户
3. 删除用户（至少保留一个）
4. 查看用户及分享链接
5. 彻底卸载
6. 退出
7. 重启 Xray 服务
8. 更新域名和证书

保留原项目的候选配置校验、实际服务身份权限检查、操作锁、证书续期、配置及证书失败回滚。增删用户、普通更新和证书续期不会轮换加密参数。私有参数和客户端参数保存在 `/usr/local/etc/xray/config.json`，请保留 `_xy` 元数据，不要公开完整配置。

此版本仅管理启用 VLESS Encryption 的单入口 WS+TLS 配置，拒绝覆盖旧 xHTTP、多入口和其他不兼容配置。它与原项目共用 `xy` 命令、Xray 服务和配置目录；建议部署到新的 VPS，不能在同一服务器上作为两个独立实例并行安装。若需替换原节点，应先备份；没有自动迁移菜单。

## 验证与限制

[GitHub Actions](https://github.com/brian952700/xray-ws-manager/actions/workflows/debian13.yml) 使用 Debian 13 容器和真实 systemd，运行真实 Xray 服务与客户端，验证：

- 安装入口、Bash 语法、ShellCheck，以及公开下载内容与提交一致。
- 加密参数生成、WS 配置及分享链接；客户端从分享链接读取参数后完成 WS+TLS 代理传输。
- 客户端和服务端日志中无上述旧式 VLESS 警告。
- 用户增删、更新保留配置、服务重启、域名切换、证书部署及失败回滚、卸载清理。

测试证书为本地自签名证书，客户端固定其证书 SHA-256；不会关闭证书验证。CI 使用 ACME 测试替身，**不覆盖真实公网证书签发、DNS、防火墙和真实定时续期**。是否通过请查看对应提交的 Actions，不沿用原项目的测试结果。也不保证消除未来核心的所有其他警告。

回滚仅覆盖本脚本的配置和证书，不包括软件包、Xray 二进制或 acme.sh 升级；重启会短暂断线，断电或 SIGKILL 无法保证多文件恢复。

```bash
sudo xray run -test -config /usr/local/etc/xray/config.json
sudo systemctl status xray --no-pager
sudo journalctl -u xray -n 50 --no-pager
```

续期日志：`/var/log/xray_cron_error.log` 和 `/var/log/xy-acme.log`。

## 终端交互

安装过程按五个阶段展示，管理菜单按节点与用户、服务维护分组，保留原编号。
青色表示标题、绿色表示成功、黄色表示提示、红色表示错误；非终端输出、`TERM=dumb` 或设置 `NO_COLOR` 时自动使用纯文本。
域名、端口、路径和用户名格式错误时可原地重试；默认值支持直接回车。
节点信息与导入链接分开展示，链接独占一行，不插入颜色或缩进。

软件包、核心和证书操作成功时收起详细输出；失败会显示最后 25 行及完整日志路径。日志以 0600 权限保留在 `/tmp/xy-operation.*`，排查完成后可自行删除；成功日志自动清理。
若需要纯文本界面，可运行 `sudo env NO_COLOR=1 xy`。
