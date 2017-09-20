---
layout: post
title:  "Localhost’a Diğer Cihazlardan Erişim"
date:   2017-08-15 22:00:00 -0600
image: /images/post/localhost.jpg
tags: web php
excerpt_separator: <!--more-->
---

Merhabalar. Geçen gün dedim ki “Bu bu localhost’a aynı ağda bulunan diğer cihazlardan neden erişemiyorum?”. Bu soruyu sormamın sebebi  geliştirdiğim PHP tabanlı projeleri diğer cihazlarda (özellikle mobil) <!--more--> nasıl göründünğünden emin olmak. İlk başta geçici olarak kişisel sayfama yüklüyordum ve diğer cihazlardan erişip ön izlemeyi o şekilde yapıyordum. Bir an durup “Bu na ne gerek var? Sonuçta evdeki cihazlarda ön izleme yapıyorum. Bu böyle olmamalı” dedim ve başladım araştırmaya. Olay çok basitmiş. Sadece bir kaç ayarı değiştirerek aynı ağdaki diğer tüm cihazlardan localhost’a erişmek mümkün.

**Dikkat:** Bu yöntem sadece aynı ağdaki cihazlara erişim verir fakat herhangi bir ortak ağ kullanıyorsanız aynı ağa bağlı herkese erişim izni vermiş olursunuz. Eğer projeleriniz hassas bilgiler veya dosya yazma gibi scriptler içeriyorsa bu yöntemi dikkatli kullanmanız gerekir.

## Nasıl?
**1)** WAMP Server kullanıyorsanız WAMP sembolüne sol tıklayıp **"Apache"** kısmından **"httpd-vhosts.conf"** dosyasını açmanız gerekiyor.

![Wamp arayüzü]({{ site.baseurl }}/images/post/localhost-wamp1.jpg "Wamp arayüzü")

**Not:** Eğer WAMP Server kullanmıyorsanız bu dosyayı Apache’nin yüklü olduğu klasörün içinde **"conf/extra"** klasörünün içinde de bulabilirsiniz.

**2)** httpd-vhosts.conf dosyasını açtığınız muhtemelen alttaki gibi bir şey çıkacaktır karşınıza. Burada <code>Require local</code> kısmını <code>Require all granted</code> olarak değiştirip dosyayı kaydediyoruz.

httpd-vhosts.conf
```apache
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot c:/wamp64/www
    <Directory "c:/wamp64/www/">
        Options +Indexes +Includes +FollowSymLinks +MultiViews
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

**3)** WAMP Server’i kapatıp açıyoruz veya menüden “Bütün Servisleri Yeniden Başlat” seçeneğine tıklıyoruz

Her şey hazır. Peki diğer cihazlardan nasıl bağlanacağız? Öncelikle geliştirme ortamınızın kurulu olduğu cihazın ağdaki IP’sini (IPv4) bulmanız gerekiyor. Bunu da cmd.exe’ye  (komut istemi) <code>ipconfig</code> komutunu yazarak öğrenebilirsiniz. Mesela benimki “IPv4 Address. . . . . . . . . . . : 192.168.0.16”.  Web sitenizi ön izlemek istediğiniz cihazınızdan 192.168.0.16 adresine bağlandığınızda geliştirme ortamınızdaki localhost’a başarılı bir şekilde bağlanabileceksiniz.