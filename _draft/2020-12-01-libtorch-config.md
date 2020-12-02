---
layout: post
comments: true
title: Pytorch C++ API LibTorch'a Giriş
header:
  teaser: "https://pytorch.org/assets/images/logo-dark.svg"
tags: [Deep Learning, Derin Öğrenme, PyTorch, LibTorch, C++]
---

Çoğumuz farklı tartışma ortamlarında Makine Öğrenmesi için en iyi dilin (genelde buna bir de en iyi ML kütüphanesi tartışmaları eklenmektedir) hangisi olduğu konusunda çeşitli tartışmalara denk gelmişizdir. Henüz gelmemişseniz çok heveslenmeyin yakında denk geleceksinizdir. Bu soruya verilecek en doğru cevap bence *"duruma göre değişir"* olabilir. Bu noktada bu konuda daha fazla laf kalabalığı yapmak yerine (zaten farklı platformlarda en alası yapılıyor) bu yazıda neler bulabileceğinize geçeceğim. Yazının bölümleri:

[TOC]

# 1. Giriş

Özellikle üretim aşamasında düşük gecikme süreli/yakın gerçek zamanlı modeller kullanmak istediğinizde karşınıza bir kaç sorun çıkmaktadır. Benim için karşılaştığım en kritik sorunların başında *Python*'da yaşanan (yanlış anlaşılması dilin daha farklı bir iddiası yok) hız sorunları ve çoklu işlem/iş parçacığı kullanımı (nesnelere erişimde Global Interpreter Lock tarafından kısıtlanan hususları kastediyorum) kaynaklı sorunlar gelmektedir. Bu noktada *C++* gibi bir dilin getireceği bazı kolaylıkları kullanmak isteyebilirsiniz. İleride bu noktada *ONNX*, *TensorRT* gibi kütüphanelerinde nasıl işe dahil edilebileceğini açıklayan yazılar yazmayı düşünüyorum. Ama bu yazıda *PyTorch*tarafından geliştiricilere sağlanan ve aslında Pytorch'un da arka planda kullandığı bence oldukça başarılı ve öğrenme eğrisi çok dik olmayan *LibTorch* kütüphanesinde bahsedeceğim. Giriş niteliğindeki bu yazıyı tensör işlemleri, çıkarım yapma ve bir model sıfırdan C++'da oluşturmayı anlatan ayrı yazılar takip edecek.

*LibTorch* kütüphanesi *PyTorch 1.0* versiyonu ile hayatımıza giren *PyTorch*'un *C++ API*'sidir. Facebook tarafından hem araştıma hem de üretimde kullanılmaktadır. *Python* tarafında kullanılan tüm özellikler aşağı yukarı *C++ API*'sinde de bulunmaktadır. Ancak biraz geriden de gelse genelde 1 versiyon farkla *Pytorch* da kullanılan özelikler *C++ API*'sine eklendiğini gözlemliyorum. Kendi dokümantasyonunda *"beta"* olarak değerlendirmemiz gerektiği ve *PyTorch*'un *Python* arayüzünün daha stabil olduğu konuları vurgulanmaktadır. Ancak ben uzun bir süredir kullanıyorum ve önemli hiçbir sorun yaşamadığımı belirtmek isterim. 

Peki *LibTorch* bize neler sunuyor:

- Makine öğrenmesi modelleri tanımlamak için bir arayüz (Python'da `torch.nn.Module` karşılığı),
- En yaygın modüller (evrişim, yinelemeli ağlar, yığın normalleştirme vb.) için standart bir kütüphane,
- Eniyileme API (SGD, Adam vb.),
- Veri setlerini ve veri iletim hatlarınızı temsil eden ve CPU çekirdeklerinde paralel işleme kabiliyeti,
-  Modellerin otomatik olarak GPU'da paralelleştirme kabiliyeti (`torch.nn.parallel.DataParallel`),
- *C++* modellerini kolayca Python'da kullanabilme,
- TorchScript JIT derleyici kullanımı,
- ATen (temel tensör arayüzü) ve Autograd API (hesaplamalı çizge üzerinde otomatik olarak gradyanları hesaplayan API) sunmaktadır.

Sunduğu bileşenlerin listesine ve tanımlarına [dokümantasyonundan](https://pytorch.org/cppdocs/frontend.html ) ulaşabilirsiniz. 

# 2. Kullanım Senaryoları

Peki *C++'da LibTorch* kullanım senaryoları neler olabilir?

- Bence öncelikle üretim aşamasında kaydedilen model dosyası (jit scripted/traced) okunup doğrudan çıkarım işlemi yapılabilir. Bu arada çoklu iş parçacığı kullanımı, düşük gecikme süresi gibi avantajları kullanabilirsiniz. Bu yazı dizisinin 3. yazısında doğrudan çıkarım için kullanımı ele alacağım. Elbette o yazıda farklı karşılaştırmalar bulabileceksiniz.
- Model sıfırdan *C++* da oluşturulabilir/eğitilebilir.  Yazı dizisinin 4. yazısında modeli *C++*'da modeli sıfırdan yazıp eğiteceğim.

# 3. [Kurulum](#kurulum)

Kurulum konusunda iki seçenek söz konusudur:

 1. Derlenmiş kütüphaneyi indirmek:

     	1. [Bağlantıya](https://pytorch.org/get-started/locally/) tıklayın.
          	2. Önce Stable linkine, daha sonra kullandığınız işletim sistemi linkine, daha sonra LibTorch linkine, ardından C++/Java linkine tıklayıp son olarak GPU için CUDA versiyon linkine eğer sadece CPU kullanacaksanız None linkine tıklayıp gelen dosya bağlantısı indirip dilediğiniz bir dizine içeriğini kopyalayın.

    ![Derlenmiş kütüphaneyi indirmek](/mnt/9fb7d2f0-a307-40ff-8656-de4d502a984b/blgnksy.github.io/assets/img/libtorch-intro/start_locally.png)

 2. Kaynak [kodundan](https://github.com/pytorch/pytorch#from-source) derlemek: Biraz zahmetli olabilir (aslında epey zahmetli oluyor). Ama kişisel olarak benim tercihim bu seçenektir. GitHub reposunu klonlayıp gerekli adımları uygulayın (Bu bölüm çok uzamaması için ben atlıyorum ama ileride bir yazıyı buna ayırabilirim).

# 4. `Hello LibTorch`

Benim bu yazıda anlatacak olduklarım Linux işletim sisteminde geliştirme editörü olarak [Clion](https://www.jetbrains.com/clion/) ve `cmake` aracını kullanmayı içerecek ama kendi işletim sisteminiz ve derleme aracınıza aynı işlem maddelerini aktarabilirsiniz. Öncelik bir `CMakeLists.txt` isimli bir dosya oluşturup aşağıdaki içeriği ekleyin:

```cmake
cmake_minimum_required(VERSION 3.17) #cmake versiyonu en düşük 3.0 olduğu sürece mevcut cmake kurulumunuzu kullanabilirsiniz
project(libtorchHelloWorld)

set(CMAKE_CXX_STANDARD 14)

find_package(Torch 1.7.0 REQUIRED) # Şu anda mevcut sürüm 1.7.0 farklı bir versiyon kullanırsanız burayı düzeltmelisiniz
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")

add_executable(libtorchHelloWorld main.cpp) 
target_link_libraries(libtorchHelloWorld ${TORCH_LIBRARIES})
```

Artık `main.cpp` içerisinde kaynak kodlarımızı ekleyelim:

```c++
#include <iostream>
#include <torch/torch.h>

int main() {
    torch::Tensor randTensor = torch::rand({2, 3});
    
    std::cout << "Hello LibTorch\n" << "Torch Tensor: " << randTensor << "\n";
}
```

Ve kodumuzu içerisine derlemek üzere bir dizin oluşturup içerisinde derlemeyi başlatalım (`-DCMAKE_PREFIX_PATH=/absolute/path/libtorch` ile belirttiğiniz yol [kurulum](#kurulum) bölümündeki içeriği kopyaladığınız dizin olmalı): 

```shell
$ mkdir build
$ cd build
$ cmake -DCMAKE_PREFIX_PATH=/absolute/path/libtorch ..
$ cmake --build . --config Debug
```

Artık derlenen dosyamızı çalıştırıp Hello World klasiğini yerine getirelim:

```shell
$ ./libtorchHelloWorld
Hello Libtorch
Torch Tensor:  0.6147  0.6752  0.8963
 0.5627  0.4836  0.5589
[ CPUFloatType{2,3} ]
```

Şimdi şu kısacık uygulamaya bakarak API hakkında yorum yapalım. İlk olarak `torch/torch.h` başlık dosyası *LibTorch*'un tüm diğer başlık dosyalarını içeren `torch/all.h` başlık dosyasını içerir. 

```c++
#pragma once

#include <torch/all.h>

#ifdef TORCH_API_INCLUDE_EXTENSION_H
#include <torch/extension.h>

#endif // defined(TORCH_API_INCLUDE_EXTENSION_H)
```

`torch/all.h` başlık dosyasını incelersek gerekli tüm başlık dosyalarına sahip olduğumuzu görürüz. Bu nokta da `torch/torch.h` başlık dosyasından başka bir başlık dosyasına ihtiyacımız olmadığını görmüş olduk:

```c++
#pragma once

#include <torch/cuda.h>
#include <torch/data.h>
#include <torch/enum.h>
#include <torch/jit.h>
#include <torch/nn.h>
#include <torch/optim.h>
#include <torch/serialize.h>
#include <torch/types.h>
#include <torch/utils.h>
#include <torch/autograd.h>
```

Aslında bir ipucu olarak *PyTorch* kullanırken `.` operatörünü `::` ile değiştirdiğinizde çoğunlukla *C++ API* için geçerli bir kod yazmış olursunuz dersem çok da yanıltıcı olmaz diye düşünüyorum. Dolayısıyla `main` fonksiyonu içerisinde tanımladığımız `randTensor` isimli değişken *Python API*'den alışık olduğumuz `torch.tensor` sınıfının bir örneğidir.  Daha sonra bu sınıfı daha ayrıntılı inceleyeceğim. 

```c++
	torch::Tensor randTensor = torch::rand({2, 3});
```

Yukarıdaki kod ile `torch` isim alanı içerisinde tanımlı `rand` fonksiyonun farklı yüklemelerinden birisini kullanarak `2, 3` şekline sahip rastgele değerlerle doldurulmuş bir `tensor` elde etmiş oluyoruz. 

```c++
	std::cout << "Hello LibTorch\n" << "Torch Tensor: " << randTensor << "\n";
```

Yukarıdaki kod satırındaysa `randTensor` `<<` operatörünün sağ operandı olduğunda  `at` isim alanı (bu isim alanı LibTorch ve PyTorch tarafından kullanılan temel tensör kütüphanesi olan ATen kütüphanesini içermektedir) içerisindeki `print` fonksiyonunu çağırır ve standart çıktıya tensör verisini, veri tipini ve şeklini yazdırır. `torch::Tensor`  sınıfının aynı *Python API* da olduğu sağladığı çok fazla şey var. Ama onlar bir sonra ki yazıda yer alacak. Yazının sonuna ulaştık. Bu yazıyı giriş bölümünde de belirttiğim gibi bir dizi yazı takip edecek. Yeni bölümleri yazdıktan sonra bu yazının altına bağlantılarını ekleyeceğim. 



