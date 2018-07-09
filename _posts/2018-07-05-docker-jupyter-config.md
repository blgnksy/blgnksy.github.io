---
layout: post
comments: true
title: NVIDIA Docker Üzerinde Jupyter Ayarları
header:
  teaser: "/assets/img/jupy-config/jupyter_logo.png"
tags: [NVIDIA-Docker, Docker, Deep Learning, Derin Öğrenme, Tensorflow, Jupyter, Jupyter Config, Jupyter Notebook]
---

   Python derin/makine öğrenmesi alanında geliştirme yapanların [%57](https://towardsdatascience.com/what-is-the-best-programming-language-for-machine-learning-a745c156d6b7)'si tarafından tercih edilen bir programlama dili olarak karşımıza çıkmaktadır.  Buna elbette üssel artan yeni paketler, öğrenme kolaylığı vb. etkenler katkı sağlamaktadır. Programlama dili seçiminden sonra geliştirme ortamının oluşturulması ve geliştirme arayüzünün (IDE-Integrated Developmnent Envirment) kurulması gerekmetedir. Geliştirme ortamının oluşturulması hakkında [NVIDIA-Docker Kullanımı](https://blgnksy.github.io/2018/07/05/nvidia-docker-usage.html) isimli yazım da ayrıntılı bilgiye ulaşabilirsiniz. Özellikle de akademisyenler ve öğrenciler özellikle de sunum imkanlarını da değerlendirilerek geliştirme arayüzü olarak _Jupyter_ tercih edilmektedir. [NVIDIA-Docker Kullanımı](https://blgnksy.github.io/2018/07/05/nvidia-docker-usage.html) isimli yazımda da görebileceğiniz gibi Docker konteynerini çalıştırdığımızda bizi tarayıcı üzerinden _Jupyter Notebook_ sayfasına ulaşabiliyoruz. Ben de kişisel tercih olarak hem geliştirme hem de sunum maksatlı olarak _Jupyter Notebook_ ile çalışıyorum ve oldukça da keyif alıyorum. Bu yazıda Docker/NVIDIA-Docker üzerinde koşacak _Jupyter Notebook_ ile ilgili ayarlama (config) işlemlerini aktarmaya çalışacağım. 

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
komutu ile ayar dosyasını kolaylıkla oluşturabiliriz. 

# 2. Şifre Oluşturulması

   _Jupyter Notebook_'a erişimi şifrelemek doğru bir tercih olacaktır. Bu işlem oldukça basittir. _sha1_ (Secure Hash Algorithm 1) ile şifrelenmiş bir şifrenin doğrulama kodunu <span style="color:red"> _jupyter_notebook_config.py_</span> dosyasının içeriğine eklememiz yeterli olacaktır. Bunun için ilk önce terminal ekranından _ipython_ (etkileşimli olarak Python kodları yazıp çalıştırabildiğimiz bir program) çalıştırılır:
   
```shell
$ ipython
```
   Daha sonra bize _sha1_ ile şifrelenmiş şifremizi oluşturalacak modülü _import_ edip _passwd()_ metodunu çağıracağız. Dilediğimiz şifreyi girip hücreyi çalıştırdığımızda çıktı olarak _sha1_ ile şifrelenmiş doğrulama kodunu elde etmiş olacağız. Doğrulama kodunu kopyalayıp _ipython_'dan _exit_ metodu ile çıkabiliriz.
   
```python
iPythonPrompt> from IPython.lib import passwd 
iPythonPrompt> passwd() 
sha1:fc216:3a35a98ed980b9...
iPythonPrompt> exit 
```
   
   _vi_ ile <span style="color:red"> _jupyter_notebook_config.py_</span> dosyasını düzenlemek için açıyoruz:
   
```shell
$ vi ~/.jupyter/jupyter_notebook_config.py
```

   Ve aşağıdaki kodları bu dosyaya ekleyip kaydedip kapatıyoruz. Artık _Jupyter Notebook_ her açılışta bir şifre ekranı bizi karşılayacak ve _ipython_ üzerinden girdiğimiz şifre ile ürettiğimiz doğrulama kodunu kullanarak şifrenin doğru olması halinde _Jupyter Notebook_ ulaşmak mümkün olacak.

```
c = get_config()  # Eğer daha önceden yoksa bu satır eklenecek.
c.NotebookApp.password = 'sha1:fc216:3a35a98ed980b9...'  #Doğrulama kodunu buraya ekleyeceğiz. 
```
   Bundan sonra _Jupyter Notebook_ her açıldığında bizi aşağıdaki gibi bir ekran karşılayacak.
   
![Jupyter](/assets/img/jupy-config/jupyter_password.png)

# 3. Güvenlik, SSL Bağlantısı Oluşturulması

   Bazen _Jupyter Notebook_ ile farklı bilgisayarlardan erişerek çalışmamız gerekebilir. Ben şahsen evdeki bilgisayarımda çalışan NVIDIA-Docker konteynerine uzaktan bağlanmak suretiyle çalışma veya sunum esnasından ulaşarak  _Jupyter Notebook_ kullanıyorum. Şu ana kadar yaptığımız ayarlar şifre hariç bağlantının güvenliğine dair bir tedbir barındırmıyor. Bağlantı güvenliği sağlanmaz ise uzak bilgisardan yerel bilgisayara öntanımlı olarak _HTTP_ üzerinden konuşmak mümkün olabilir. Bu noktada _SSL_ ile şifrelenmiş bir bağlantı kullanarak _HTTPS_ üzerinden konuşmak tercih edilmesi tavisye edilmektedir. Son bölümde güvenli bağlantı için gerekli ayarları uygulayacağız.
   
   Öncelikle bağlantının güvenli olması (uçtan uca şifrelenecek olan paketler) için _ssl_ sertifikası üretmemiz gerekiyor. Bunun için:
 
```shell
$ cd
$ mkdir ssl
$ cd ssl
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout "cert.key" -out "cert.pem" -batch
```   
komutlarını kullanarak sırasıyla _ev_ dizinine gidip orada _ssl_ isimli bir dizin oluşturuyoruz. Bu dizinin içerisinde iken _openssl_ ile _365_ gün süreli _rsa:1024_ ile şifrelenmiş bağlantımız için gerekli olan iki dosyayı (_sertifika anahtarı_ ve _sertifika_) oluştuyoruz. Şimdi yine <span style="color:red"> _jupyter_notebook_config.py_</span> ayar dosyamızı düzenleyip aşağıdaki hale dönüştürüyoruz.

```
c = get_config()  # Eğer daha önceden yoksa bu satır eklenecek.
c.NotebookApp.certfile = u'~/ssl/cert.pem' # sertifika dosyasının yolu
c.NotebookApp.keyfile = u'~/ssl/cert.key' # sertifika anahtar dosyasının yolu
c.NotebookApp.password = 'sha1:fc216:3a35a98ed980b9...'  #Doğrulama kodunu buraya ekleyeceğiz. 
```
   Artık geliştirmeye başlayabiliriz.  İlk defa _Jupyter_ sunucusuna bağlandığınızda aşağıdaki ekran ile karşılaşabilirsiniz. Bu sizin ürettiğiniz sertikanın tanınmamasından kaynaklanıyor. Eğer tarayıcınızdan istisna eklerseniz bir daha sizi bu ekran karşılamayacaktır.
  
 ![Jupyter](/assets/img/jupy-config/jupyter-add-exception.png) 
   Bu arada tüm bu yaptıklarınızı çalışan konteynerdan çıktıktan sonra her seferinde çalışabilmesi için içe aktarmayı (commit) unutmayın. Aksi halde Docker görüntüsünü çalıştırdığınız zaman içe aktarılmamış değişikliklerin konteyner de geçerli olmayacaktır. 

NOT: _Jupyter_ sunucusunun çalıştığı ağa bağlanmak için bilgisayarınızda/modeminizde/yönlendiricinizde sabit IP, port yönlendirme vb. bir takım ayarlar yapmanız gerekebilir. Bunun için işletim sistemi/modem marka ve modeline uygun ayarlar için ilgili yönergeleri takip edebilirsiniz. Yine de bu konuda dahil bir sorunuz olursa bana ulaşabilirsiniz. 

