---
title: "HackTheBox: JavaScript Deobfuscation"
author: b1lal
categories: [HackTheBox, "Skills Assessment"]
tags: [javascript, deobfuscation, hackthebox, eval, packer, skills assessment]
render_with_liquid: false
description: "HacktheBox platformunda JavaScript deobfuscation modülündeki yetenek değerlendirme sorularının çözümünü içermektedir."
media_subpath: /images/hackthebox_js_deobfuscation/
layout: post
published: true  
lang: "tr"
image:
  path: main.png
  alt: "HackTheBox: JavaScript Deobfuscation"
---


Bu yazıda HackTheBox platformunda bulunan JavaScript deobfuscation modülündeki yetenek değerlendirme sorularının çözümünü paylaşacağım. Bu modül, JavaScript kodlarını deobfuscate etme becerisini geliştirmeye yönelik çeşitli sorular içermektedir.

<br>

> Penetrasyon Testimiz sırasında JavaScript ve API'ler içeren bir web sunucusuyla karşılaştık. Müşterimizi nasıl olumsuz etkileyebileceğini anlamak için işlevlerini belirlememiz gerekiyor.


## **Soru 1:**

> <b style="color:red;">Web sayfasının HTML kodunu incelemeye çalışın ve içinde kullanılan JavaScript kodunu belirleyin. Kullanılan JavaScript dosyasının adı nedir?</b>


İlk sorunun cevabını bulabilmek için Geliştirici Seçeneklerini açıp `Network` sekmesine geçiyoruz. Ardından sayfayı yenileyerek yüklenen dosyaları inceleyebiliriz. Burada `api.min.js` adlı bir JavaScript dosyasının yüklendiğini görebiliriz.

![api.min.js](api.min.js.png)

<br>

## **Soru 2:**
> <b style="color:red;"> JavaScript kodunu bulduğunuzda, ilginç işlevler yapıp yapmadığını görmek için çalıştırmayı deneyin. Karşılığında bir şey aldınız mı?</b>

`api.min.js` dosyasına tıkladığımızda kaynak koda erişiyoruz. Bu kodu analiz etmek veya çalıştırmak için tarayıcının geliştirici konsolunu ya da <a href="https://jsconsole.com/" target="_blank" style="text-decoration:none;">jsconsole.com</a> gibi çevrimiçi JavaScript çalışma ortamlarını kullanabiliriz. 

Kodu çalıştırdığımızda flag'i elde ediyoruz.

![jsconsole](jsconsole.png)

<br>

## **Soru 3:**
> <b style="color:red;">Fark etmiş olabileceğiniz gibi, JavaScript kodu gizlenmiştir. Kodu deobfuscate etmek ve 'flag' değişkenini almak için bu modülde öğrendiğiniz becerileri uygulamayı deneyin.</b>

Kodu incelediğimizde paketlendiğini ve `eval` fonksiyonunun kullanıldığını görebiliriz. Bu, kodun okunabilirliğini azaltmak için sıkça kullanılan bir tekniktir. Bu kodu deobfuscate etmek için çeşitli araçlar ve teknikler kullanabiliriz.

Örneğin, <a href="https://lelinhtinh.github.io/de4js/" target="_blank" style="text-decoration:none;">de4js</a> gibi çevrimiçi araçlar, JavaScript kodunu deobfuscate etmek için kullanılabilir. Bu araç, kodu daha okunabilir hale getirir.

Websitesine giderek kodu deobfuscate ediyoruz ve Aşağıdaki gibi bir çıktı alıyoruz:

![de4js](deobfuscation.png)

```javascript
function apiKeys() {
    var flag = 'HTB{n' + '3v3r_' + 'run_0' + 'bfu5c' + '473d_' + 'c0d3!' + '}',
        xhr = new XMLHttpRequest(),
        _0x437f8b = '/keys' + '.php';
    xhr['open']('POST', _0x437f8b, !![]), xhr['send'](null)
}
console['log']('HTB{j' + '4v45c' + 'r1p7_' + '3num3' + 'r4710' + 'n_15_' + 'k3y}');
```

<br>

## **Soru 4:**
> <b style="color:red;"> Gizliliği kaldırılmış JavaScript kodunu analiz etmeye ve ana işlevselliğini anlamaya çalışın. Bunu yaptıktan sonra, gizli bir anahtar elde etmek için ne yaptığını kopyalamaya çalışın. Anahtar nedir?</b>


Kodu incelediğimizde `/keys.php` 'ye `POST` isteği gönderildiğinde belli bir key değeri döndürüyor olabiilir. Bunu test etmek için `curl` veya `Postman` gibi araçları kullanabiliriz.

```bash
C:\Users\bilal> curl -X POST http://94.237.54.189:58120/keys.php
4150495f70336e5f37333537316e365f31355f6675..
```

Curl ile istek attığımızda herhangi ek birşey göndermemize gerek kalmadan doğrudan key değerini alabiliyoruz. 

<br>

## **Soru 5:**
> <b style="color:red;">Gizli anahtarı aldıktan sonra, kodlama yöntemine karar vermeye çalışın ve kodunu çözün. Daha sonra aynı önceki sayfaya "key=DECODED_KEY" olarak kodu çözülmüş anahtar ile bir "POST" isteği gönderin. Aldığınız bayrak nedir?</b>

Aldığımız key değerini çözmek için cyberchef gibi araçları kullanabiliriz. CyberChef, çeşitli kodlama ve şifreleme yöntemlerini destekler ve bu tür işlemler için oldukça kullanışlıdır.
CyberChef'i açıp aldığımız key değerini `From Hex` işlemi ile çözüyoruz. 

Daha sonra bu çözülmüş anahtarı kullanarak `/keys.php` 'ye `POST` isteği gönderiyoruz.

```bash
C:\Users\bilal>curl -X POST http://94.237.54.189:58120/keys.php -d "key=API_...SNIP"
HTB{...SNIP...}
```   

Son soruyu da bu şekilde çözmüş oluyoruz.

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
