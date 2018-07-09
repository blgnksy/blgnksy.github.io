---
layout: post
comments: true
title: NVIDIA Docker Üzerinde Jupyter Ayarları
tags: [NVIDIA-Docker, Docker, Deep Learning, Derin Öğrenme, Tensorflow, Jupyter, Jupyter Config, Jupyter Notebook]
---

    Python derin/makine öğrenmesi alanında geliştirme yapanların %57'si tarafından tercih edilen bir programlama dili olarak karşımıza çıkmaktadır. (linke ekle) Programlama dili seçiminden sonraki aşama geliştirme ortamının oluşturulması ve geliştirme arayüzünün(IDE-Integrated Developmnent Envirment) seçilmesidir. Geliştirme ortamının oluşturulması hakkında NVIDIA Docker Kullanımı (link ekle) isimli yazı da ayrıntılı bilgiye ulaşabilirsiniz. Alanda bir çok insan özellikle de akademisyenler ve öğrenciler özellikle de sunum imkanlarını da değerlendirerek Jupyter kullanmaktadır. NVIDIA Docker Kullanımı (link ekle) isimli yazıda da görebileceğiniz gibi Docker görüntüsünü çalıştırdığımızda bizi tarayıcı üzerinden Jupyter Notebook sayfasına ulaşabiliyoruz. Ben de kişisel tercih olarak hem geliştirme hem de sunum maksatlı olarak Jupyter Notebook ile çalışıyorum ve oldukça da keyif alıyorum. Bu yazıda Docker/NVIDIA Docker üzerinde koşacak Jupyter Notebook ile ilgili ayarlama (config) işlemlerini aktarmaya çalışacağım. 

Ana Başlıklar
1. Ayar Dosyasının Oluşturulması
2. Şifre Oluşturulması
3. Güvenlik, SSL Bağlantısı Oluşturulması

# 1. Ayar Dosyasının Oluşturulması
   Terminal ekranından Docker görüntüsünü _bash_ açılacak şekilde çalıştırmak için aşağıdaki komutu kullanıyoruz:

```shell
$ sudo nvidia-docker run -it  -p 8888:8888  -p 6006:6006 gcr.io/tensorflow/tensorflow:latest-gpu_changed bash
root@2fc479bed67f:/notebooks#
```
   Konteyner içinde _notebooks_ dizinin içindeyiz. _Jupyter_ her çalıştırıldığında tüm ayarları üzerinde barındıran <span style="color:red">_~/.jupyter_</span> dizinin altında bulunan <span style="color:red"> _jupyter_notebook_config.py_</span> isimli bir dosyayı okur. Bu dosya mevcut değil ise ilk olarak terminal ekranından:

```shell
$ jupyter notebook --generate-config
```
komutu ile ayar dosyasını kolaylıkla oluşturabiliriz. Şimdi bu dosyanın içeriğini standart çıktıya bastıralım:
   
```shell
$ cat jupyter_notebook_config.py
```

# 2. Şifre Oluşturulması
   _Jupyter Notebook_'a erişimi şifrelemek doğru bir tercih olacaktır. Bu işlem oldukça basittir. _sha1_ (Secure Hash Algorithm 1) ile şifrelenmiş bir şifreyi <span style="color:red"> _jupyter_notebook_config.py_</span> dosyasının içeriğine eklememiz yeterli olacaktır. Bunun için ilk önce terminal ekranından _ipython_ (etkileşimli olarak Python kodları yazıp çalıştırabildiğimiz bir program) çalıştırılır:
   
```shell
$ ipython
```

```ipython
iPythonPrompt> from IPython.lib import passwd 
iPythonPrompt> passwd() 
```
