# OpenWrt / procd 部署说明（fork 增补）

上游面板假定系统里有 systemd 和 journalctl。OpenWrt 没有这两样，所以本 fork 增加了
一层可替换的服务后端，并用 procd init 脚本 + 环境变量把三条路径换掉：

| 上游行为 | OpenWrt 上的替代 | 开关 |
| --- | --- | --- |
| `systemctl show --property=ActiveState,SubState,MainPID` 读服务状态 | 读 pidfile + 判断 `/proc/<pid>` 是否存在 | `--service-pidfile` |
| `journalctl -u kixdns -o json` 读日志 | 直接读内核日志文件（按行解析，游标=字节偏移） | `--service-log` |
| helper 用 `systemctl start/stop/restart` 启停 | 执行 `/etc/init.d/kixdns start|stop|restart` | `--init-script` |

未设置这三个参数时行为与上游完全一致（仍然走 systemd），所以这个 fork 也能正常装在有
systemd 的机器上。

## 目录布局

```
/usr/bin/kixdns                         内核
/usr/bin/kixdns-panel-server            面板
/usr/libexec/kixdns-panel-helper        root helper（唯一能启停服务的东西）
/usr/share/kixdns-panel/web             前端静态资源
/usr/share/kixdns-panel/geo             GeoSite 种子
/etc/kixdns/pipeline.json               内核配置（面板接管后以数据库版本为准）
/etc/kixdns-panel/panel.db              面板数据库（必须在 overlay 上，不能放 /var）
/var/log/kixdns.log                     内核日志（面板日志页读它）
/var/run/kixdns.pid                     procd 写的 pid 文件
/run/kixdns/admin.sock                  内核管理 Socket
/run/kixdns-panel/control.sock          helper Socket
```

注意 OpenWrt 的 `/var` 是 tmpfs：数据库若放 `/var/lib/...`（上游默认）重启就丢，所以这里
统一改到 `/etc/kixdns-panel/`。

## 安装

```sh
sh install.sh              # 只装面板，内核不启动
START_DNS=1 sh install.sh  # 装完顺手启动内核（占用 53）
```

## 与 systemd 版的行为差异

- 进程以 root 运行，没有 `ProtectSystem`/独立 `kixdns` 用户。helper 的 `--allowed-uid` 因此是 0。
- 面板日志页的级别过滤是按行文本猜的（含 `ERROR`→err、`WARN`→warn、其余 info），
  不像 journald 有结构化 priority。
- 在线更新（面板自己替换内核二进制）在 OpenWrt 上直接 spawn 更新器，不走 `systemd-run`。
