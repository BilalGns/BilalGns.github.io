---
title: "HackTheBox: Web Attacks"
author: b1lal
categories: [HackTheBox, "Skills Assessment"]
tags: [web attacks, ssti, hackthebox, skills assessment, idor, api, xxe, http verb tampering]
render_with_liquid: false
description: "HacktheBox platformunda Web Attacks modülündeki yetenek değerlendirme sorularının çözümünü içermektedir."
media_subpath: /images/hackthebox_web_attacks/
layout: post
published: false  
lang: "tr"
image:
  path: main.png
  alt: "HackTheBox: Web Attacks"
---

Bu yazımızda HackTheBox platformunda bulunan **Web Attacks** modülündeki yetenek değerlendirme sorularının çözümünü paylaşacağım. Bu modüldeki, yetenek değerlendirme sorusu web uygulamalarındaki zafiyetleri ne derece kavradığımızı ölçmeyi amaçlamaktadır.


> Bir yazılım geliştirme şirketi için bir web uygulaması sızma testi gerçekleştiriyorsunuz ve sizi sosyal ağ web uygulamalarının en son yapısını test etmekle görevlendiriyorlar. Web uygulamasında bulunan birden fazla güvenlik açığını tespit ederek kullanın.

> Giriş bilgileri aşağıdaki soruda verilmiştir.

> Ayrıcalıklarınızı yükseltmeye çalışın ve '`/flag.php`' adresindeki bayrağı okumak için farklı güvenlik açıklarından yararlanın. User "**htb-student**" and password "**Academy_student!**"


Sistemi başlattıktan sonra bize verilen IP adresine gidiyoruz. Giriş ekranı ile karşılaşıyoruz:

![Giriş Ekranı](login.png)

Daha sonra verilen kullanıcı adı ve şifre ile giriş yapıyoruz:

![Giriş](login2.png)

Şimdi F12 tuşuna basarak geliştirici araçlarını açıyoruz ve `Network` sekmesine geçiyoruz. Daha sonra sayfayı yenileyerek ağ trafiğini izliyoruz. Burada `api.php` adlı bir dosyaya yapılan istekleri görebiliriz. Bu dosya, web uygulamasının API'sine erişim sağlıyor.

![Network](network.png)

Daha sonra buradaki URL değerini kopyalayarak **curl** komutunu kullanarak bu API'ye istek gönderiyoruz:

```bash
C:\Users\bilal> curl http://94.237.48.12:33771/api.php/user/74
{"uid":"74","username":"htb-student","full_name":"Paolo Perrone","company":"Schaefer Inc"}
```

Buradaki **uid** değeriyle kullanıcı bilgilerini elde etmiş olduk. Şimdi bu API'ye farklı istekler göndererek daha fazla bilgi elde etmeye çalışacağız. Burada **0** dan başlayarak manuel istek atmaya çalıştığımda herhangi bir **admin** kullanıcı ismine yada role türüne rastlamadım bu yüzden işlerimizi hızlandırmak için basit bir Python betiği yazdım:

```python
import requests

url = "http://94.237.48.12:33771/api.php/user/"

for i in range(50, 101):
    print(f"Deneniyor : {url}{i}")

    response = requests.get(f"{url}{i}")

    if response.status_code == 200:
        data = response.json()

        text = str(data).lower()
        if "admin" in text or "administrator" in text:
            print("Bulunan kullanıcı:", data)
            break
```

Betiği çalıştırdığımda **company** alanında **Administrator** kelimesini içeren bir kullanıcıya ulaştım: 

```bash
Deneniyor : http://94.237.48.12:33771/api.php/user/X
Deneniyor : http://94.237.48.12:33771/api.php/user/X
Bulunan kullanıcı: {'uid': '...', 'username': 'a.corrales', 'full_name': 'Amor Corrales', 'company': 'Administrator'}
Deneniyor : http://94.237.48.12:33771/api.php/user/X
Deneniyor : http://94.237.48.12:33771/api.php/user/X
```

Şimdi **UID** değerini başarıyla elde ettiğimize göre bu kullanıcı ile giriş yapmayı deneyebiliriz. Bunun için de yine geliştirici araçlarını açarak **Çerezler** sekmesinde bulunan **uid** değerini bulduğumuz ID ile değiştiriyoruz:

![Çerezler](uid.png)

Ve hesap bu şekilde görünüyor: 

![Admin Hesabı](admin-dashboard.png)


Daha sonra sayfadaki Settings kısmına giderek şifre sıfırlama işlemini gerçekleştirmek için istek atıyoruz. `/api.php/token/X` uç noktasına istek atıldığını görüyoruz. Bu uç nokta, bize bir token döndürüyor. Bu token'ı kullanarak şifre sıfırlama işlemini gerçekleştirebiliriz. 

İsteği devam ettirip Burp Suite aracılığıyla `repeater` sekmesine gönderiyoruz:

![Burp Suite](burp-repeater.png)

**Access Denied** cevabını alıyoruz. Burada aklımıza istek methodunu değiştirmek geliyor. Eğer isteği `$_REQUEST` şeklinde alıyorsa ve doğrulamayı da `$_POST` ile yapıyorsa bu durumda `HTTP Verb Tampering` zafiyetini kullanarak isteğimizi kontrol olmadan gerçekleştirebiliriz.

Repeater üzerinde sağ tıklayarak **Change Request Method** seçeneğini kullanarak isteğin methodunu `GET` olarak değiştiriyoruz:

![HTTP Verb Tampering](request-method.png)

İsteği şimdi tekrar gönderdiğimizde başarılı bir şekilde şifre sıfırlama işlemini gerçekleştirebiliyoruz.

![Şifre Sıfırlama](reset-password.png)

Şimdi hesaba giriş yapabiliriz.

Giriş yaptığımızda farklı olarak takvim üzerinde etkinlik ekleme özelliğinin olduğunu görüyoruz: 

![Etkinlik Ekleme](add-event.png)

Bir etkinlik ekliyoruz ve daha sonra bu isteği yine Burp Suite aracılığıyla yakalıyoruz ve `repeater` sekmesine gönderiyoruz:

![Etkinlik Ekleme İsteği](xml.png)

İsteğin içeriğinde `XML` formatında bir veri olduğunu görüyoruz. Bu durumda `XXE` **(XML External Entity)** zafiyetini kullanarak sunucunun dosya sistemine erişim sağlamaya çalışabiliriz.

İlk olarak basit bir test yapıyoruz:

![XXE Test](xxe-test.png)

Başarılı bir şekilde ENTITY'yi çözebildiğini görüyoruz. Şimdi sunucunun dosya sistemine erişim sağlamaya çalışacağız ve bizden istenen `/flag.php` dosyasını okumayı deneyeceğiz: 

Denemeden önce `/flag.php` dosyasının özel karakter içerme olasılığı olduğunu düşünerek (PHP dosyaları genellikle `<?php` gibi özel karakterler içerir ve bunlar hatalara neden olabilir ) **base64** kodlaması ile encode ediyoruz:


![Base64 Encode](php-filter.png)

Şimdi son olarak base64 çıktısını seçerek `Decoder` sekmesine gönderiyoruz ve `Base64` seçeneğini seçerek decode ediyoruz:

![Flag](flag.png)

Başarılı bir şekilde `/flag.php` dosyasının içeriğini elde edebildik.

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

