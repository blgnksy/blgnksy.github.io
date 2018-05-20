---
layout: post
title: NVIDIA Docker Kullanımı
tags: [NVIDIA-Docker, Docker, Docker Temel Bilgiler, Docker Kurulumu, DockerFile]
---



Derin Öğrenme (Deep Learning) ile uğraşmaya başlayan herkesin bir şekilde korkulu rüyası maalesef gerekli paketlerin, araçların kurulması ve birbirlerinin gereksinimleri ile uyumsuzluk yaratmadan çalışabilmesi olmuştur. Bunun için Python Package Index ile sunulan virtualenv [virtualenv](https://virtualenv.pypa.io/en/stable/), [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/) kullanımı belli oranda işe yarasa bile bazen sorun geliştirme ortamınızda mevcut ekran kartının(NVIDIA Cuda çekirdeğine sahip olan) CUDA sürücüleri, cuDNN kütüphanesi kurmaya çalıştığınız paketler ile uyumlu olmaması/olamaması neticesinde kurulum işlemleri olması gerekenden daha fazla zaman harcamamıza neden olabilmektedir. Bazen bunların hepsi bir araya getirilse bile bu seferde sisteminizde yaptığınız bir işletim sistemi/donanım sürücüsü güncellemesi bütün emeklerin çöpe gitmesi anlamına gelebilmektedir. 

Bu noktada daha fazla izole/sanal ortama ihtiyaç ortaya çıkmaktadır. Sorunların ortadan kaldırılmasında [Docker](https://www.docker.com) etkin bir çözüm olarak karşımıza çıkmakta ve giderek daha çok geliştirici tarafıundan tercih edilmektedir. Dahası ekran kartının hesap gücünden faydalanmak isteyen kullanıcıların yardımına bir de [NVIDIA-Docker](https://github.com/NVIDIA/nvidia-docker) koşmaktadır. Çok fazla teknik ayrıntısına boğulmadan gerekli kavramları öğrenerek bu çözümü derin/makina öğrenmesi geliştiricileri için nasıl faydalı bir şekilde kullanılabileceği üzerine odaklanacağız. Teknik ayrıntılar için [Gökhan Şengün](https://www.gokhansengun.com/docker-nedir-nasil-calisir-nerede-kullanilir/) tarafından kaleme alınan yazıya başvurabilirsiniz.  Konuyu derin öğrenme özelinde anlatmaya çalışacağım. Özellikle İngilizce kaynak sayısı oldukça fazla olsa da Türkçe kaynak bulmakta sorun yaşanmakta olduğunu değerlendirdiğim için de bu yazıyı Türkçe olarak paylaşıyorum. 

Ana Başlıklar:
1. Temel Kavramlar
2. Docker, NVIDIA Docker Kurulumu
3. Hazır görüntülerin(iamge) kullanımı
4. DockerFile ile özgün görüntüleri oluşturulması
5. Komut satırı üzerinden Docker ile etkileşim
6. Jupyter kurulumu ve ayarlanması

## 1. Temel Kavramlar:

### Docker nedir?

Kendi başına çalışabilen, ihtiyaç duyduğu herşeyi (sistem araçları, sistem kütüphaneleri, gerekli paketler, donanım sürücüleri vb.) kendi içinde bulunduran, hafif bir yazılımdır. Dahası içerisinde barındırdığı tüm bileşenleri aynı makine üstünde (ana makine-host) çok farklı ayarlarla istediğiniz sayıda görüntüyü (image) farklı konteyner (container) çalıştırmak mümkün ki bu haliyle her geliştirici için normal şartlarda ayrı ayrı sahip olmak veya ayarlamak çok maliyetli ve zahmetli olabilecek süreçler hem çok ekonomik hem de çok süratli olabilmektedir. 

#### Görüntü (Image) 

İçerisinde işletim sistemi NVIDIA sürücülerini ve gerekli tüm araç, paket ve programları barındıran yapıdır. Docker kurulumunu anlattıktan sonra ana makine de mevcut görüntüleri (image) nasıl oluşturulacağını, görüntüleneceği ve yapılabilecek işlemlere değineceğiz. 

#### Konteyner (Container)
Docker görüntüsünün üzerinde koştuğu izole/sanal çalıştığı ortamdır. Yine çalışan, görüntü içine aktarmayı (commit) veya aktaılmamış olanlar üzerinde yapılan işlemlere ileride değineceğiz. 

## 2. Docker, NVIDIA Docker Kurulumu:
[Docker CE](https://docs.docker.com/install/) versiyonun kurulum yönergelerine bağlantı üzerinden ulaşabilirsiniz. Ben size Ubuntu bash terminal üzerinde kurulumunu göstereceğim.

  * İlk önce daha önce kurulan Docker CE versiyonunu kaldırıyoruz. 
```shell
$ sudo apt-get remove docker docker-engine docker.io
```

  * Daha sonra Docker CE kurulumuna geçiyoruz ve aşağıdaki komutları sıra ile terminalden uyguluyoruz.

   apt paket endekslerini güncelliyoruz.
   
```shell
$ sudo apt-get update
```

   apt ile gerekli paketleri kuruyoruz.
   
```shell
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

   Docker'ın resmi GPG anahtarını kendi anahtar zincirimize ekliyoruz.

```shell
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

   stable olarak işaretlenmiş paketleri kurmak için ekliyoruz.

```shell
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

   apt paket endekslerini tekrar güncelliyoruz.
   
```shell
$ sudo apt-get update
```

   Ve sonunda Docker CE'nin son versiyonunu kuruyoruz.

```shell
$ sudo apt-get install docker-ce
```

   "Merhaba Dünya"sız yapamazdık. Aşağıdaki komut ile henüz bilgisayarımızda olmayan hello-world isimli bir görüntüyü [DockerHub](https://hub.docker.com/) adı verilen geliştiricilerin ve resmi olarak kullanılan görüntülerin paylaşıldığı bir çeşit uygulama dükkanından görünütüyü indirip çalıştırıyoruz ve terminal standart çıktısında aşağıdaki çıktıyı görüyoruz.

```shell
$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
9bb5a5d4561a: Pull complete 
Digest: sha256:f5233545e43561214ca4891fd1157e1c3c563316ed8e237750d59bde73361e77
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```


<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
