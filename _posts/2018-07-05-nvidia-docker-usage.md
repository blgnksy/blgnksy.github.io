---
layout: post
comments: true
title: NVIDIA Docker Kullanımı
tags: [NVIDIA-Docker, Docker, Docker Temel Bilgiler, Docker Kurulumu, DockerFile, Deep Learning, Derin Öğrenme, Tensorflow]
---





   Derin Öğrenme (Deep Learning) ile uğraşmaya başlayan herkesin bir şekilde korkulu rüyası maalesef gerekli paketlerin, araçların kurulması ve birbirlerinin gereksinimleri ile uyumsuzluk yaratmadan çalışabilmesi olmuştur. Hele bir de aynı anda farklı projelerle çalışıyorsanız bunların hepsinin ayrı gereksinimleri varsa işler daha da sorunlu olmaktadır. Bunun için Python paket yönetimi ile sunulan virtualenv [virtualenv](https://virtualenv.pypa.io/en/stable/), [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/) kullanımı belli oranda işe yarasa bile bazen sorun, geliştirme ortamınızda mevcut ekran kartının(_NVIDIA Cuda_ çekirdeğine sahip olan) _CUDA_ sürücüleri, _cuDNN_ kütüphanesi kurmaya çalıştığınız paketler ile uyumlu olmaması/olamaması nedeniyle kurulum işlemleri gerekenden daha fazla zaman harcamamıza neden olabilmektedir. Bunların hepsi bir araya getirilse bile bu sefer de sisteminizde yaptığınız bir işletim sistemi/donanım sürücüsü güncellemesi bütün emeklerin çöpe gitmesi anlamına gelebilmektedir. 

   Bu noktada daha fazla izole/sanal ortama ihtiyaç ortaya çıkmaktadır. Sorunların ortadan kaldırılmasında [Docker](https://www.docker.com) etkin bir çözüm olarak karşımıza çıkmakta ve giderek daha çok geliştirici tarafından tercih edilmektedir. Dahası ekran kartının hesap gücünden faydalanmak isteyen kullanıcıların yardımına bir de [NVIDIA-Docker](https://github.com/NVIDIA/nvidia-docker) koşmaktadır. Bu yazımızda çok fazla teknik ayrıntısına boğulmadan gerekli kavramları öğrenerek bu çözümü derin/makina öğrenmesi geliştiricileri için nasıl faydalı bir şekilde kullanılabileceği üzerine odaklanacağız. Teknik ayrıntılar için [Gökhan Şengün](https://www.gokhansengun.com/docker-nedir-nasil-calisir-nerede-kullanilir/) tarafından kaleme alınan yazıya/yazılara başvurabilirsiniz.  Konuyu derin öğrenme özelinde anlatmaya çalışacağımı tekrar hatırlatarak özellikle İngilizce kaynak sayısı oldukça fazla olsa da Türkçe kaynak bulmakta sorun yaşanmakta olduğunu değerlendirdiğim için de bu yazıyı Türkçe olarak paylaşıyorum. 

![Nvidia-Docker](/assets/img/docker-usage/NVIDIA-GPU-Docker.png)

Ana Başlıklar:
1. Temel Kavramlar
2. Docker, NVIDIA Docker Kurulumu
3. Hazır görüntülerin(image) kullanımı
4. DockerFile ile özgün görüntülerin kullanılması
5. Komut satırı üzerinden Docker ile etkileşim

## 1. Temel Kavramlar:

### Docker nedir?

   Kendi başına çalışabilen, ihtiyaç duyduğu herşeyi (sistem araçları, sistem kütüphaneleri, gerekli paketler, donanım sürücüleri vb.) kendi içinde bulunduran, hafif bir yazılımdır. Dahası içerisinde barındırdığı tüm bileşenleri aynı makine üstünde (ana makine-host) çok farklı ayarlarla istediğiniz sayıda görüntüyü (image) farklı konteyner (container) içinde çalıştırmak mümkündür. Sonuç olarak; geliştirici için normal şartlarda ayrı ayrı sahip olmak veya ayarlamak çok maliyetli ve zahmetli olabilecek süreçler hem çok ekonomik hem de çok süratli olabilmektedir. 

#### Görüntü (Image) 

   İçerisinde işletim sistemi NVIDIA sürücülerini ve gerekli tüm araç, paket ve programları barındıran yapıdır. Docker kurulumunu anlattıktan sonra ana makinede mevcut görüntülerin (image) nasıl oluşturulacağını, görüntüleneceğini ve yapılabilecek işlemlere değineceğiz. 

#### Konteyner (Container)
   Docker görüntüsünün üzerinde koştuğu izole/sanal ortamdır. Konteyner üzerinde yapılabilecek işlemlere ileride değineceğiz. 

## 2. Docker, NVIDIA Docker Kurulumu:
   [Docker CE](https://docs.docker.com/install/) sürümünün kurulum yönergelerine bağlantı üzerinden ulaşabilirsiniz. Ben size Ubuntu bash terminal üzerinde kurulumunu göstereceğim.

  * İlk önce daha önce kurulan Docker CE sürümünü _apt_ ile kaldırıyoruz. 

```shell
$ sudo apt-get remove docker docker-engine docker.io
```

  * Daha sonra Docker CE kurulumuna geçiyoruz ve aşağıdaki komutları sıra ile terminalden uyguluyoruz.

   _apt_ paket endekslerini güncelliyoruz.
   
```shell
$ sudo apt-get update
```

   _apt_ ile gerekli paketleri kuruyoruz.
   
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

   stable olarak işaretlenmiş paketleri kurmak için depomuza ekliyoruz.

```shell
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

   _apt_ paket endekslerini tekrar güncelliyoruz.
   
```shell
$ sudo apt-get update
```

   Ve sonunda Docker CE'nin son sürümünü kuruyoruz.

```shell
$ sudo apt-get install docker-ce
```

   "_Merhaba_ _Dünya_"sız yapamazdık. Aşağıdaki komut ile henüz bilgisayarımızda olmayan hello-world isimli bir görüntüyü [DockerHub](https://hub.docker.com/) adı verilen geliştiricilerin ve resmi olarak kullanılan görüntülerin paylaşıldığı bir çeşit uygulama dükkanından indirip çalıştırıyoruz ve terminal standart çıktısında aşağıdaki çıktıyı görüyoruz.

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

   Docker kurma işlemimiz bitti. Şimdi de derin öğrenme (deep learning) modellerini eğitme işlemimizi kısaltacak önemli bir donanım olan ekran kartı üreticisi NVIDIA'nın hayatımıza soktuğu nimetlerden faydalanmak için bir de [NVIDIA-Docker](https://github.com/NVIDIA/nvidia-docker/blob/master/README.md) kurmaya başlayabiliriz. Bu noktada ana makinemizde (host) NVIDIA ekran kartı (CUDA yeteneğine sahip) ve ekran kartı sürücüsü kurulmuş olması gerekmektedir.

   Önce işletim sistemi dağıtımızı bir eçvre değişkenine yazıyoruz.
```shell
$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
```

   Sonra anahtar zincirimize resmi GPG anahtarını ekliyoruz.
```shell
$ curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
```

   Şimdide uygulama kaynaklarımıza nvidia dağıtımlarını ekliyoruz.
```shell
$ curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```

   NVIDIA-Docker kurulumunu yapıyoruz ve docker servisini yeniden başlatıyoruz.
```shell
$ sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
$ sudo systemctl restart docker
```

   Yine kurulumumuzu test etmek için bu sefer NVIDIA'ya ait son CUDA deposunu kendi bilgisayarımıza indirip herhangi bir sorun olmadığına emin oluyoruz. Bu noktada sizin ekran kartı modeli, sürücüsü ve özellikleri ile uyumlu olarak standart çıktıda aşağıdakine benzer bir sonuç alıyoruz. 
```shell
$ docker run --gpus all --rm nvidia/cuda nvidia-smi
Sun May 20 18:33:05 2018       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.111                Driver Version: 384.111                   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 1070    Off  | 00000000:0A:00.0  On |                  N/A |
|  0%   49C    P8    11W / 200W |    359MiB /  8110MiB |      7%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```

## 3. Hazır Görüntülerin(image) Kullanımı:
   [DockerHub](https://hub.docker.com/explore) üzerinden paylaşılmış hazır görüntülere (image) ulaşabilirsiniz. Kolaydan başlayarak zora doğru gideceğimiz için önce hazır depoları kullanacağız. Ben size Tensorflow'un resmi deposundan son sürümünü nasıl kuracağınızı göstereceğim.

   Terminal üzerinden aşağıdaki komutu verdiğimizde uzak depo alanından _tensorflow/tensorflow_ isimli deponun son sürümünü(_latest-gpu_) ana makinemize çekip (_pull_) etkileşimli modda çalıştırıp (-p) parametresi ile dış dünya ile 8888 nolu portdan haberleşmesini söylüyoruz. Daha sonra [localhost:8888](localhost:8888) üzerinden çalışan Jupyter Notebook karşımıza çıkıyor. Bundan sonra bu komutu her çalıştırdığımızda Docker, uzak depo yerel makinemize bulunduğu için indirmek yerine doğrudan çalıştırmaya başlayacaktır.

```shell
$ docker run -it --gpus all -p 8888:8888 tensorflow/tensorflow:latest-gpu
# NVIDIA ekran kartı olmayanlar için docker run -it -p 8888:8888 tensorflow/tensorflow ile kurulum yapılabilir.
```
   [DockerHub](https://hub.docker.com/) üzerinden ulaştığınız tüm depolarda farklı etiket (tag) varsa farklı sürümleri olduğunu düşünebilirsiniz. Gidip size uygun farklı sürümlerini de denemeniz mümkün olabilir. Görüldüğü gibi sadece parametreleri değiştirerek ana makinemiz (host) üzerinde bir çok farklı bilgisayar varmış gibi görüntüler (image) sayesinde istediğimiz özgürlüğe sahip oluyoruz. 
   
## 4. DockerFile ile Özgün Görüntülerin Kullanılması:

   Her zaman hazır bir depo kullanmak Docker'ın bize sunduğu esnekliği tam anlamıyla kullanmamıza imkan vermeyebilir. Bu noktada tamamıyla kendi istediğimiz bir görüntü oluşturmak gerekecektir. DockerFile bu eksikliği gidermek için kullanılan metin bazlı bir dosya olup içerisinde bulunan Docker'a özel sözdizim kuralları ile tam anlamıyla istediğimiz gibi bir görüntü oluşturmamıza yardım eder. Ben bu noktada DockerFile oluşturma konusunda yine [Gökhan Şengün](https://www.gokhansengun.com/docker-yeni-image-hazirlama/) tarafından kaleme alınan yazıya mutlaka göz atmanızı tavsiye edip derin öğrenme merkezli olarak nasıl bir DockerFile kullanabileceğimize değineceğim. 

   Öncelikle örnek bir DockerFile ele alalım. Burada çok kullanılan bir [Floyd Lab](https://github.com/floydhub/dl-docker) tarafından sağlanan hazır bir görüntünün DockerFile dosyasını kullanacağız. [Bağlantıdan](https://github.com/blgnksy/blgnksy.github.io/raw/master/assets/DockerFile.gpu) dosyayı indirebilirsiniz. 

   İndirdiğimiz DockerFile.gpu dosyasının olduğu dizine terminalden gelip aşağıdaki komut ile _floydhub/dl-docker_ adından ve _gpu_ etiketli bir görüntü oluşturuyoruz. 

```shell
$ docker build -t floydhub/dl-docker:gpu -f Dockerfile.gpu .
```

   Daha sonra yukarıda yaptığımız gibi bu görüntüyü bir konteyner içinde çalıştırabiliriz.

```shell
$ docker run -it --gpus all -p 8888:8888 floydhub/dl-docker:gpu
```

   Şimdi de DockerFile dosyasının içine bakarak neler yaptığını anlamaya ve sonra özelleştirmek için neler yapabileceğimeze bakalım. (Not: Dosya çok uzun olduğundan bazı bölümleri "..." ile belirterek kısalttığımı belirtmek isterim.) 

  * Öncelikle NVIDIA tarafından sağlanan CUDA'nın 8. sürümü ve cuDNN kütüphanesinin 5. sürümünü kullanan Ubuntu'nun 14.04 sürümünü görüntü içine kuruyor.
  * Daha sonra kullanacağı bazı parametreleri ARG ile belirliyor.
  * _apt_ ile gerekli kütüphaneleri kuruyor.
  * Python paket yöneticisini ve gerekli kütüphaneleri kuruyor.
  * Tensorflow, Caffe, Theano, Keras, Lasagne, Torch ve Lua derin öğrenme kütüphanelerini kuruyor ve ihtiyaç duyduğu bazı ortam değişkenlerini ayarlıyor.
  * Açık kaynak bilgisayarlı görü kütüphanelerinden OpenCV kurulumunu yapıyor.
  * Jupyter Notebook için ayar dosyasını ve Jupyter Notebook kullanımı root kullanıcı için sorunlu olduğundan küçük bir betik dosyası olan  dosyasını kopyalıyor. (Not: Kurulumdan önce [floydhub/dl-docker](https://github.com/floydhub/dl-docker) deposundan _jupyter_notebook_config.py_ dosyasını ve birazdan kullanacağımız _run_jupyter.sh_ dosyasını da DockerFile.gpu ile aynı dizine koymamız gerekiyor.
  * Daha sonra ana makine ile konuşmak üzere _6006_ ve _8888_ nolu portları açıyor. Genelde _6006_ nolu port Tensorboard, _8888_ nolu port ise Jupyter Notebook tarafından kullanılmaktadır.
  * Son olarak görüntü çalıştığında terminalden bash ile bizi karşılayacak komutu yazıyor.

```shell
$ cat DockerFile.gpu
FROM nvidia/cuda:8.0-cudnn5-devel-ubuntu14.04 

MAINTAINER Sai Soundararaj <saip@outlook.com>

ARG THEANO_VERSION=rel-0.8.2
ARG TENSORFLOW_VERSION=0.12.1
ARG TENSORFLOW_ARCH=gpu

...
...


# Install some dependencies
RUN apt-get update && apt-get install -y \
		bc \
		build-essential \
		cmake \
		curl \
		g++ \
		gfortran \
		git \
		libffi-dev \
		libfreetype6-dev \
		libhdf5-dev \
		libjpeg-dev \
		liblcms2-dev \
		libopenblas-dev \
		liblapack-dev \
		...
		python-dev \
		...
		&& \
	apt-get clean && \
	apt-get autoremove && \
	rm -rf /var/lib/apt/lists/* && \
# Link BLAS library to use OpenBLAS using the alternatives mechanism (https://www.scipy.org/scipylib/building/linux.html#debian-ubuntu)
	update-alternatives --set libblas.so.3 /usr/lib/openblas-base/libblas.so.3

# Install pip
RUN curl -O https://bootstrap.pypa.io/get-pip.py && \
	python get-pip.py && \
	rm get-pip.py

# Add SNI support to Python
RUN pip --no-cache-dir install \
		pyopenssl \
		ndg-httpsclient \
		pyasn1

# Install useful Python packages using apt-get to avoid version incompatibilities with Tensorflow binary
# especially numpy, scipy, skimage and sklearn (see https://github.com/tensorflow/tensorflow/issues/2034)
RUN apt-get update && apt-get install -y \
		python-numpy \
		python-scipy \
		python-nose \
		...
		&& \
	apt-get clean && \
	apt-get autoremove && \
	rm -rf /var/lib/apt/lists/*

# Install other useful Python packages using pip
RUN pip --no-cache-dir install --upgrade ipython && \
	pip --no-cache-dir install \
		Cython \
		ipykernel \
		jupyter \
		path.py \
		...
		&& \
	python -m ipykernel.kernelspec


# Install TensorFlow
RUN pip --no-cache-dir install \
	https://storage.googleapis.com/tensorflow/linux/${TENSORFLOW_ARCH}/tensorflow_${TENSORFLOW_ARCH}-${TENSORFLOW_VERSION}-cp27-none-linux_x86_64.whl


# Install dependencies for Caffe
RUN apt-get update && apt-get install -y \
		libboost-all-dev \
		libgflags-dev \
		libgoogle-glog-dev \
		libhdf5-serial-dev \
		...
		&& \
	apt-get clean && \
	apt-get autoremove && \
	rm -rf /var/lib/apt/lists/*

# Install Caffe
RUN git clone -b ${CAFFE_VERSION} --depth 1 https://github.com/BVLC/caffe.git /root/caffe && \
	cd /root/caffe && \
	cat python/requirements.txt | xargs -n1 pip install && \
	mkdir build && cd build && \
	cmake -DUSE_CUDNN=1 -DBLAS=Open .. && \
	make -j"$(nproc)" all && \
	make install

# Set up Caffe environment variables
ENV CAFFE_ROOT=/root/caffe
ENV PYCAFFE_ROOT=$CAFFE_ROOT/python
ENV PYTHONPATH=$PYCAFFE_ROOT:$PYTHONPATH \
	PATH=$CAFFE_ROOT/build/tools:$PYCAFFE_ROOT:$PATH

RUN echo "$CAFFE_ROOT/build/lib" >> /etc/ld.so.conf.d/caffe.conf && ldconfig


# Install Theano and set up Theano config (.theanorc) for CUDA and OpenBLAS
RUN pip --no-cache-dir install git+git://github.com/Theano/Theano.git@${THEANO_VERSION} && \
	\
	echo "[global]\ndevice=gpu\nfloatX=float32\noptimizer_including=cudnn\nmode=FAST_RUN \
		\n[lib]\ncnmem=0.95 \
		\n[nvcc]\nfastmath=True \
		\n[blas]\nldflag = -L/usr/lib/openblas-base -lopenblas \
		\n[DebugMode]\ncheck_finite=1" \
	> /root/.theanorc


# Install Keras
RUN pip --no-cache-dir install git+git://github.com/fchollet/keras.git@${KERAS_VERSION}


# Install Lasagne
RUN pip --no-cache-dir install git+git://github.com/Lasagne/Lasagne.git@${LASAGNE_VERSION}


# Install Torch
RUN git clone https://github.com/torch/distro.git /root/torch --recursive && \
	cd /root/torch && \
	bash install-deps && \
	yes no | ./install.sh

# Export the LUA evironment variables manually
ENV LUA_PATH='/root/.luarocks/share/lua/5.1/?.lua;/root/.luarocks/share/lua/5.1/?/init.lua;/root/torch/install/share/lua/5.1/?.lua;/root/torch/install/share/lua/5.1/?/init.lua;./?.lua;/root/torch/install/share/luajit-2.1.0-beta1/?.lua;/usr/local/share/lua/5.1/?.lua;/usr/local/share/lua/5.1/?/init.lua' \
	LUA_CPATH='/root/.luarocks/lib/lua/5.1/?.so;/root/torch/install/lib/lua/5.1/?.so;./?.so;/usr/local/lib/lua/5.1/?.so;/usr/local/lib/lua/5.1/loadall.so' \
	PATH=/root/torch/install/bin:$PATH \
	LD_LIBRARY_PATH=/root/torch/install/lib:$LD_LIBRARY_PATH \
	DYLD_LIBRARY_PATH=/root/torch/install/lib:$DYLD_LIBRARY_PATH
ENV LUA_CPATH='/root/torch/install/lib/?.so;'$LUA_CPATH

# Install the latest versions of nn, cutorch, cunn, cuDNN bindings and iTorch
RUN luarocks install nn && \
	luarocks install cutorch && \
	luarocks install cunn && \
    luarocks install loadcaffe && \
	\
	cd /root && git clone https://github.com/soumith/cudnn.torch.git && cd cudnn.torch && \
	git checkout R4 && \
	luarocks make && \
	\
	cd /root && git clone https://github.com/facebook/iTorch.git && \
	cd iTorch && \
	luarocks make

# Install OpenCV
RUN git clone --depth 1 https://github.com/opencv/opencv.git /root/opencv && \
	cd /root/opencv && \
	mkdir build && \
	cd build && \
	cmake -DWITH_QT=ON -DWITH_OPENGL=ON -DFORCE_VTK=ON -DWITH_TBB=ON -DWITH_GDAL=ON -DWITH_XINE=ON -DBUILD_EXAMPLES=ON .. && \
	make -j"$(nproc)"  && \
	make install && \
	ldconfig && \
	echo 'ln /dev/null /dev/raw1394' >> ~/.bashrc

# Set up notebook config
COPY jupyter_notebook_config.py /root/.jupyter/

# Jupyter has issues with being run directly: https://github.com/ipython/ipython/issues/7062
COPY run_jupyter.sh /root/

# Expose Ports for TensorBoard (6006), Ipython (8888)
EXPOSE 6006 8888

WORKDIR "/root"
CMD ["/bin/bash"]

```
   Bazılarınız hazırladığı özgün DockerFile dosyasını uzak depoya göndermek (_push_) isteyebilir. Yine ayrıntılar için [Gökhan Şengün](https://www.gokhansengun.com/docker-yeni-image-hazirlama/) bağlantısından _Basit Image Hazırlama ve DockerHub’a Push Etme_ bölümünde ulaşabilirsiniz.

## 5. Komut Satırı Üzerinden Docker ile Etkileşim
   -----
   Öncelikle var olan görüntülerin listesine terminal üzerinden erişelim:
```shell
#  Sadece docker images da kullanılabilir.
$ sudo docker images
REPOSITORY                     TAG                       IMAGE ID            CREATED             SIZE
hello-world                    latest                    e38bc07ac18e        5 weeks ago         1.85kB
gcr.io/tensorflow/tensorflow   latest-gpu_changed        f73dd685943c        5 weeks ago         14.8GB
gcr.io/tensorflow/tensorflow   1.7.0-rc0-devel-gpu-py3   a48c5d8684b3        2 months ago        3.1GB
```
   Görülebileceği gibi benim ana makinem üzerinde 3 adet görüntü var. Dikkat ederseniz aynı isimli ama farklı etikete sahip iki görüntü var. Altta ilk çektiğim (_pull_) hali üstte ise zaman içinde konteyner da yaptığım değişiklikleri aktardığım (_commit_) son halini verdiğim yeni etiketli olan bulunuyor.

### Konteyner'ı çalıştırmak (_run_)

   Görüntüyü oluşturduk. Şimdi görüntüyü bir konteyner da çalıştırmaya sıra geldi.

```shell
$ sudo docker run -it --gpus all -p 8888:8888  -p 6006:6006 gcr.io/tensorflow/tensorflow:latest-gpu_changed jupyter notebook --allow-root
```

   Yukarıdaki komut ile önce docker'a _run_ komutunu _it_ parametreleri ile çalıştırmasını söylüyoruz. _i_ etkileşimli modu ile konteyner çalışınca ona komutlar gönderebilmemiz için STDIN (standart girdiyi) açık tutuyor. _t_ ile konteyner için bir pseudo-TTY tahsis ediliyor. _p_ parametresi ile portların ana makine ile konteyner arasında nasıl yönlendirileceğini söylüyoruz. Burada docker konteynerinin 8888 nolu portu ile ana makinenin 8888. portu ve aynı şekilde 6666. portlarını birbirlerine yönlendirdik. Buna jupyter notebook kullanımında ihtiyaç duyacağız. Sonra hangi görüntünün çalıştırılmasını istediğimizi ve konteyner açılınca _jupyter notebook_ açılmasını istediğimizi docker'a söyledikten sonra işimiz bitiyor. Terminal ekranında aşağıdakine benzer bir çıktı görüyor olmalısınız.
```shell
$ sudo docker run -it --gpus all -p 8888:8888  -p 6006:6006 gcr.io/tensorflow/tensorflow:latest-gpu_changed jupyter notebook --allow-root
[I 14:52:56.460 NotebookApp] Serving notebooks from local directory: /notebooks
[I 14:52:56.460 NotebookApp] 0 active kernels
[I 14:52:56.460 NotebookApp] The Jupyter Notebook is running at:
[I 14:52:56.460 NotebookApp] https://[all ip addresses on your system]:8888/
[I 14:52:56.460 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
```

   Şimdi gidip Firefox'u açıp adres satırına "localhost:8888" yazıp sayfaya gittiğimizde aşağıdaki sayfa ile karşılaşıyoruz. Bu sayfada docker konteynerimizin _/notebooks_ klasörünün içeriğine ulaşıyoruz. Docker içerisinde _Jupyter Notebook_ kullanımı ve gerekli ayarların yapılmasını başka bir yazıda aktarmayı planlıyorum. Umarım en kısa zamanda onu da yayımlayacağım. Neyse şimdi konumuza devam edelim. 

![Jupyter Notebook](/assets/img/docker-usage/initial_jupyter_big.png)

   Şimdilik yukarıda sayfanın sağ üstünde bulunan _Logout_ tuşuna basıp bağlantımızı kestikten sonra terminal penceresinde Kontrol+C ile Jupyter sunucusunu ve çalışan docker konteynerimizi kapatıyoruz. Bazen konteyner çalıştığında terminal ekranına çıkmak isteyebilirsiniz. Bu durumda aşağıdaki gibi docker çalıştırma komutumuzun sonuna _bash_ eklemek yeterli olacaktır. 

```shell
$ sudo docker run -it --gpus all -p 8888:8888  -p 6006:6006 gcr.io/tensorflow/tensorflow:latest-gpu_changed bash
root@2fc479bed67f:/notebooks# 
```

   Görüldüğü gibi artık konterner içinde terminal ekranına bağlıyız ve artık özelleştirmek istersek şimdi güç bizim elimize geçti. Kullanıcı adımız _root_ ve konteyner anahtar adımız _2fc479bed67f_ (siz de bu alan farklı olacaktır ki bu rasgele verilen bir anahtar) ve aynı Jupyter'de olduğu gibi _/notebook_ klasöründe bulunuyoruz. Terminalde işimiz bittiğinde _exit_ komutu ile çıkıyoruz. 

### Konteyner'da değişiklik yapmak ve içe aktarmak (_commit_)

   Görüntüyü oluşturduk ve bir konteyner içinde çalıştırmaya başladık. Bu noktaya kadar sahip olduğumuz içe aktarılmayı bekleyen konteynerleri aşağıdaki komut ile listeyelebiliriz. Burada _CONTAINER ID_ ve _NAMES_ docker motoru tarafından verilen rasgele değerlerdir.

```shell
$ sudo docker ps -a 
CONTAINER ID        IMAGE                                             COMMAND             CREATED             STATUS                        PORTS               NAMES
2fc479bed67f        gcr.io/tensorflow/tensorflow:latest-gpu_changed   "bash"              5 minutes ago       Exited (130) 10 seconds ago                       musing_saha

```
   Eğer konteyner içerisinde oluşturulduktan sonra değişiklik yapılmışsa ve bu değişikliği görüntüye aktarmaz isek görüntü her çalıştırıldığında yeni bir konteyner oluşturacağından ve çalışan konteyner diğer konteynerde yapılan değişiklikten haberdar olmadığından bu değişiklikleri daimi olarak kullanmak istememiz halinde içe aktarmamız gerekmektedir. Bunun için aşağıdaki komutu çalıştırmamız yeterli olacaktır.

```shell
$ sudo docker commit 2fc479bed67f gcr.io/tensorflow/tensorflow:v1
sha256:0ddddaf2218987e2ed9f5cfa1976b635a7b811d68d986fef193af6e4c7cfcc30
```
   Bu komut ile _tag_ olarak v1 diye bir etiket tanımladığımız başlangıç görüntüsü üzerinde değişikliklerin eklenmiş olduğu yeni bir görüntüye sahip oluyoruz. İstersek etiketi olduğu gibi kullanıp iki ayrı görüntü yerine başlangıç görüntüsü üzerine de aktarım yapabilirdik. Bu noktada sürekli yeni etiketler vererek yola devam etmek bilgisayarınızda daha fazla depolama alanı gerektirecektir. 

### Kullanılmayan Konteynerları durdurmak ve silmek (_commit_)

   Çalışan konteynerlerden işimize yaramayanları veya içe aktarmayı tamamladığınız konteynerleri _CONTAINER ID_ parametresini kullanarak durdurmak ve silmek mümkündür. İlk olarak çalışan konteynerleri yukarıda gösterdiğimiz gibi listeleyelim. 

```shell
$ sudo docker ps -a
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS                     PORTS                                            NAMES
3a9777814a95        ndeep/dl-docker:cpu   "bash"                   2 days ago          Exited (255) 2 days ago    0.0.0.0:6006->6006/tcp, 0.0.0.0:8888->8888/tcp   festive_chatterjee
1a9add8b9905        ndeep/dl-docker:cpu   "bash"                   6 weeks ago         Exited (255) 6 weeks ago   0.0.0.0:6006->6006/tcp, 0.0.0.0:8888->8888/tcp   dazzling_ride
e1a552ab67bf        635015520b19          "bash"                   6 weeks ago         Exited (0) 6 weeks ago                                                      objective_northcutt
7ea8bd5094c6        635015520b19          "jupyter notebook --…"   2 months ago        Exited (0) 2 months ago                                                     priceless_khorana
```
   Bu noktada _3a9777814a95_ anahtar alanına sahip konteyneri durdurmak için:

```shell
$ sudo docker stop 3a9777814a95
3a9777814a95
```
komutunu kullanıyoruz. Durdurduğumuz konteyneri silmek için 

```shell
$ sudo docker rm 3a9777814a95
3a9777814a95
```
komutunu komut satırına yazmamız gerekiyor. Artık çalışan konteynerleri sıraladığımızda _3a9777814a95_ anahtar alanına sahip konteynerdan kurtulmuş oluyoruz. 

```shell
$ sudo docker ps -a
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS                     PORTS                                            NAMES
1a9add8b9905        ndeep/dl-docker:cpu   "bash"                   6 weeks ago         Exited (255) 6 weeks ago   0.0.0.0:6006->6006/tcp, 0.0.0.0:8888->8888/tcp   dazzling_ride
e1a552ab67bf        635015520b19          "bash"                   6 weeks ago         Exited (0) 6 weeks ago                                                      objective_northcutt
7ea8bd5094c6        635015520b19          "jupyter notebook --…"   2 months ago        Exited (0) 2 months ago                                                     priceless_khorana
```

### Görüntüleri silmek

   Son olarak; artık işimize yaramayacağını düşündüğümüz görüntüleri silme işlemine bakacağız. Bu noktada bir görüntüyü silmek için önce bu görüntünün çalışan tüm konteynerlerinin durdurulması ve silinmesi gerektiğini hatırlattıktan sonra görüntüyü silme işlemine geçelim. Öncelikle görüntüleri listeyelim:

```shell
$ sudo docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
ndeep/dl-docker      cpu2                8403972b7f68        14 minutes ago      11.6GB
ndeep/dl-docker      cpu1                0ddddaf22189        22 hours ago        11.6GB
ndeep/dl-docker      cpu                 e16010eb9c55        6 weeks ago         11.6GB
floydhub/dl-docker   cpu_changed         4a4e5cbd6476        5 months ago        11.8GB
ubuntu               14.04               67759a80360c        6 months ago        221MB
```

   Ben _8403972b7f68_ anahtar alanına sahip görüntüyü silmek istiyorum. Bunun için komut satırına:

```shell
$ docker rmi 8403972b7f68
Untagged: ndeep/dl-docker:cpu2
Deleted: sha256:8403972b7f68905eb2bb59efcbee05cefdd8ece92ab28a13d3326d07329437a8
```
komutunu yazdıktan sonra görüntümüzü silebiliyoruz.

   Sonuç olarak; _docker_/_nvidia-docker_ derin öğrenme alanında geliştirme/araştırma yapanların nasıl bu aracı kullanabileceğine dair temel bilgileri aktarmaya çalıştım. Elbette _docker_/_nvidia-docker_ kendilerine has birçok farklı özelliğe sahipler. Ama umarım kısa sürede çalışan bir geliştirme ortamı oluşturabileceksiniz. _Jupyter_ kurulumuna dair konuları başka bir yazıda aktarmaya çalışacağım. (Umarım en kısa zamanda) Eğer sorunuz olursa lütfen aşağıdaki bölümden bana yazınız. 

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
