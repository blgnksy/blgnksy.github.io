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
'''shell
$ sudo apt-get remove docker docker-engine docker.io
'''


<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
