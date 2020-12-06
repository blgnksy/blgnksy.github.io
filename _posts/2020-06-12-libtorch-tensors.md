---
layout: post
comments: true
title: C++ Derin Öğrenme-2 Pytorch C++ API LibTorch Tensör İşlemleri
header:
  teaser: "https://pytorch.org/assets/images/logo-dark.svg"
tags: [Deep Learning, Derin Öğrenme, PyTorch, LibTorch, PyTorch C++ API, Machine Learning C++, Deep Learning C++, Machine Learning, Makine Öğrenmesi, Tensors, Tensör İşlemleri]
---
Yazı dizisinin bu yazısında *LibTorch*'da tensörlerin nasıl oluşturulduğunu, erişildiğini ve değiştirildiğini açıklayacağım. *LibTorch*'un ne olduğunu, neler yapılabildiğini açıkladığım giriş niteliğindeki yazıyı okumadıysanız [o yazıdan](https://blgnksy.github.io/2020/12/03/libtorch-config.html) başlamanızı tavsiye ederim. Bu noktada bu yazı çok daha uzun olabilirdi ama olabilen tüm sadeliği ama yeterli bilgiyi sağlayacak şekilde süzüldüğünü ve ana referansın dokümantasyonun kendisi olduğunu unutmayın. 

ATen tensör kütüphanesi *PyTorch*'un tensör işlemleri için  arka planda kullandığı kütüphanedir ve C++14 standartlarına uygun olarak yazılmıştır. Tensör tipleri dinamik olarak çözümlenmektedir. Sonuç olarak içinde tuttuğu veri tipi ne olursa olsun ya da CPU/GPU tensörü olursa olsun tek bir tensör arayüzü bizi karşılamaktadır. Tensör sınıfının arayüzünü [dokümantasyondan](https://pytorch.org/cppdocs/api/classat_1_1_tensor.html#exhale-class-classat-1-1-tensor) inceleyebilirsiniz.

Tensörler üzerinde işlem yapan yüzlerce fonksiyon bulunmaktadır. Bu [fonksiyonların listesine](https://pytorch.org/cppdocs/api/namespace_at.html#functions) bağlantıyı kullanarak ulaşabilirsiniz. Fonksiyon isimlendirmeleriyle ilgili olarak dikkatinizi çekmek istediğim bir nokta `_` karakteriyle biten fonksiyonlar tensör üzerinde değişiklik yapmaktadırlar yani bu fonksiyon çağrılırken tensör C++'da sol taraf referansı (lvalue reference) olarak aktarılmaktadır. Şimdi bu sınıfı kullanmaya başlayalım:



# 1. Tensör Oluşturma:

## 1.1 Fabrika Fonksiyonlarını Kullanma

Bu fonksiyonlar *Fabrika Tasarım Desenleri*nde olduğu gibi çalışıp sonuçta `torch::Tensor` geri döndüren fonksiyonlardır. Aslında bunlardan bir tanesini ilk [yazıda](https://blgnksy.github.io/2020/12/03/libtorch-config.html) kullanmıştım: `torch::rand()` fonksiyonu argüman olarak aldığı şekile göre bize tensör geri döndürmektedir. Bu fonksiyonlar:

- [arange](https://pytorch.org/docs/stable/torch.html#torch.arange): Sıralı tamsayılardan oluşan tensör geri döndürür,
- [empty](https://pytorch.org/docs/stable/torch.html#torch.empty): İlk değer verilmemiş ,
- [eye](https://pytorch.org/docs/stable/torch.html#torch.eye): Returns an identity matrix,
- [full](https://pytorch.org/docs/stable/torch.html#torch.full): Returns a tensor filled with a single value,
- [linspace](https://pytorch.org/docs/stable/torch.html#torch.linspace): Returns a tensor with values linearly spaced in some interval,
- [logspace](https://pytorch.org/docs/stable/torch.html#torch.logspace): Returns a tensor with values logarithmically spaced in some interval,
- [ones](https://pytorch.org/docs/stable/torch.html#torch.ones): Returns a tensor filled with all ones,
- [rand](https://pytorch.org/docs/stable/torch.html#torch.rand): Returns a tensor filled with values drawn from a uniform distribution on `[0, 1)`.
- [randint](https://pytorch.org/docs/stable/torch.html#torch.randint): Returns a tensor with integers randomly drawn from an interval,
- [randn](https://pytorch.org/docs/stable/torch.html#torch.randn): Returns a tensor filled with values drawn from a unit normal distribution,
- [randperm](https://pytorch.org/docs/stable/torch.html#torch.randperm): Returns a tensor filled with a random permutation of integers in some interval,
- [zeros](https://pytorch.org/docs/stable/torch.html#torch.zeros): Returns a tensor filled with all zeros.

Linkler *Python* dokümantasyonuna bağlantı vermektedir. C++ API'da ki işlevleri, parametreleri ve isimli argümanları aynıdır. İsimli argümanların `torch::TensorOptions` nesnesi vasıtasıyla tanımlanabildiğine, ulaşılabildiğine ve değiştirilebildiğine dikkat edin. Bunu birazdan `torch:rand()` fonksiyonunda ele alacağım ve diğer fonksiyonlarda da geçerli olacaklar. 

Şimdi kullanışlı fabrika fonksiyonlarına yakından bakalım (Not: İlk fonksiyonu detaylı inceleyeceğim, diğerleri için dokümantasyonu kullanmanızı önereceğim. Çünkü API'lar hızlı değişime uğruyor ve senkron tutmak zahmetli olacaktır.):

### 1.1.1. `torch::rand()` 

Bu fonksiyon `[0,1)` aralığında rastgele kayan noktalı sayılar üretir. Fonksiyonun protipine bakalım: `torch::rand(*size, *, out=None, dtype=None, layout=torch.strided, device=None, requires_grad=False) → Tensor`. 

#### Parametreleri

- **size** (int): Tensörün şeklini belirlediğimiz parametredir. Tamsayı değerler alır.

#### İsimli Argümanları

- **out** ([*Tensor*](https://pytorch.org/docs/stable/tensors.html#torch.Tensor)*,* *optional*) – Çıktı tensörü.
- **dtype** ([`torch.dtype`](https://pytorch.org/docs/stable/tensor_attributes.html#torch.torch.dtype), optional) – Tensör veri tipi.
- **layout** ([`torch.layout`](https://pytorch.org/docs/stable/tensor_attributes.html#torch.torch.layout), optional) – Tensörün bellekte tutulma yöntemi (`dense`/`strided`). Bu seçeneğin ileride ki versiyonlarda kaldırılması planlanmaktadır. 
- **device** ([`torch.device`](https://pytorch.org/docs/stable/tensor_attributes.html#torch.torch.device), optional) – Tensörün hangi aygıtta saklanacağı (CPU/GPU).
- **requires_grad** ([*bool*](https://docs.python.org/3/library/functions.html#bool)*,* *optional*) – Geri döndürülen tensörün otomatik gradyan işlemine tabii olup olmayacağı.

#### Kullanımı:

En basit kullanımı şekli verip tensörü kullanmaktır. 

```c++
torch::Tensor randTensor = torch::rand(/*size:*/{2, 3});
```

Bu kullanım ilk yazıdan hatırlarsanız şekli `2, 3` olan bir CPU tensörü geri döndürmüştü. Hatırlayacak olursak bu tensörü standart çıkış akımına yazdırırsak aşağıdaki çıktıyı alırız (çıktılardaki kayan noktalı sayıların rastgele olacağını unutmayın):

```shell
0.6147  0.6752  0.8963
0.5627  0.4836  0.5589
[ CPUFloatType{2,3} ]
```

İsimli argümanların kullanımı için önce bu seçenekleri bir `torch::TensorOptions` nesnesi içerisinde tanımlamamız gerekiyor. **Bu seçeneklerin diğer fabrika fonksiyonlarında da aynı şekilde kullanıldığını hatırlatıp** alabilecekleri değerlere göz atalım:

- `dtype`: `kUInt8`, `kInt8`, `kInt16`, `kInt32`, `kInt64`, `kFloat32` ve `kFloat64`,
-  `layout`: `kStrided` ve `kSparse`,
-  `device`: `kCPU` ya da `kCUDA` (birden fazla GPU'nuz varsa aygıt indeksi de alır),
-  `requires_grad`: `true` ya da `false`.

Şimdi seçenekleri kullanarak bir tensör oluşturalım. 32 bitlik kayan noktalı sayıları `strided` bellek yerleşiminde, 0 numaralı GPU'da otomatik gradyana dahil olacak bir tensör elde etmek için aşağıdaki kod öbeğini kullanabiliriz:

```c++
auto options = torch::TensorOptions()
            .dtype(torch::kFloat32)
            .layout(torch::kStrided)
            .device(torch::kCUDA, 0)
            .requires_grad(true);

torch::Tensor randTensor = torch::rand(/*size:*/{2, 3}, options);
```

Bu seçeneklerden bir veya birkaçını doğrudan fonksiyon olarak da kullanabilirsiniz. Bu fonksiyonlar bekleneceği üzere `torch::TensorOptions` nesnesi geri döndürür : 

```c++
torch::Tensor randTensor = torch::rand(/*size:*/{2, 3}, torch::TensorOptions().dtype(torch::kFloat32));
/* veya 
torch::Tensor randTensor = torch::rand({2, 3}, torch::dtype(torch::kFloat32));
torch::Tensor randTensor = torch::rand({2, 3}, torch::dtype(torch::kFloat32).device(torch::kCUDA, 0));
*/
```

### 1.1.2. `torch::randint()`

Bu fonksiyon verilen iki değer arasında düzgün dağılıma uygun şekilde tamsayılardan oluşan bir tensör geri döndürür. Hemen kullanımına bakalım:

```c++
auto intTensor = torch::randint(/*low:*/1, /*high:*/9, /*size:*/{3});
```

Yukarıdaki tanımlamayla 1 ile 9 arasında 3 elemanı olan bir vektör olacak bir tensör oluşturmuş oluyoruz. Bu tensörün çıktısına bakarsak:

```shell
 6
 1
 1
[ CPUFloatType{3} ]
```

3B bir tensör (tipik olarak bir görüntü verisini taşıyan bir tensör olarak düşünebilirsiniz) oluşturalım:

```c++
auto intTensor = torch::randint(/*low:*/1, /*high:*/9, /*size:*/{1920, 1080, 3});
```

### 1.1.3. torch::ones()/torch::zeros()

Bu fonksiyonlar isimlerinden anlaşıldığı üzere birler veya sıfırlardan oluşan tensörler oluşturmanızı sağlarlar. Kullanımları yine diğer fonksiyonlarla benzerdir. Örneğin:

```c++
auto onesTensor = torch::ones(/*size:*/{5, 10});
auto zerosTensor = torch::zeros(/*size:*/{1, 5, 10});
```

### 1.1.4. `torch::from_blob()`

Çoğunlukla tensörü oluşturacak veriyi başka bir kaynaktan okuyupo bu tensöre aktarırız. Bunu yapabilmek için `void*` olarak veriyi alan ve tensör geri döndüren kullanışlı bir fonksiyon bulunmaktadır.:

```c++
float data[] = {1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0 8.0, 9.0, 10.0};
auto blobData = torch::from_blob(data, /*size:*/{2, 5});
/*
  1   2   3   4   5
  6   7   8   9  10
[ CPUFloatType{2,5} ]
*/
```

**Bu fonksiyonun argümanına gönderilen verinin sahipliğini almaz. Ancak tensör nesnesinin ömrü bittiğinde özgün nesneyi de bellekten siler.**  Yine isterseniz tensör seçeneklerini kullanabilirsiniz. 

```c++
auto blobDataD = torch::from_blob(data, {1, 10}, torch::requires_grad(False));
```

### 1.1.5. `torch::tensor` fonksiyonu

Sınıfın kurucu işlevlerini doğrudan kullanan bu fonksiyonla da tensör oluşturmak söz konusudur.

```C++
auto tensorInit = torch::tensor({1.0, 2.0, 4.0, 2.0, 3.0, 5.0});
```

Diğer fabrika fonksiyonlarına dokümantasyona bırakıp şimdi `torch::Tensor` sınıfının üye fonksiyonlarını inceleyelim.



# 2. `torch::Tensor` Sınıf Üye Fonksiyonları

Artık elimizde tensörümüz var ve onun hakkında bilgi almak/değişiklik yapmak istiyoruz. Burada `torch::Tensor` sınıfının bize sağladığı sınıf üye fonksiyonlarından en önemlilerine göz atalım:

```c++
// 2 boyutlu bir tensör oluşturalım
auto tensorInit = torch::tensor({{1.0, 2.0, 4.0, 2.0, 3.0, 5.0},{1.0, 2.0, 4.0, 2.0, 3.0, 5.0}});

// dim() sınıf üye fonksiyonu tensörün kaç boyutlu olduğunu geri döndürür. Örneğimizde 2 olarak:
auto tDims = tensorInit.dim();

// dtype() sınıf üye fonksiyonu tensörün veri tipini geri döndürür. Örneğimizde float olarak:
auto tDtype = tensorInit.dtype();

// sizes() sınıf üye fonksiyonu tensörün tuttuğu verinin şeklini geri döndürür. Örneğimizde  [2, 6] olarak:
auto f = tensorInit.sizes();
```



# 3. Tensör Elemanlarına Erişim, İndeksleme ve Değiştirme

Tensör elemanlarına erişim için birden fazla yöntem bulunmaktadır. Öncelikle `torch::Tensor` sınıf üye fonksiyonlarından bir tanesi `torch::Tensor.data_ptr()` fonksiyonunu kullanarak tüm veriye erişebiliriz. Alternatif olarak `data()`fonksiyonunu kullanıp *Python*'da olduğu gibi bir elemanına erişmek mümkündür.

```c++
auto tensorInit = torch::tensor({{1.0, 2.0, 4.0, 2.0, 3.0, 5.0},{11.0, 12.0, 14.0, 12.0, 13.0, 15.0}});

// void* türünden bir gösterici (pointer) geri döndürür
auto pDataVoid  = tensorInit.data_ptr();
// veri tipinin float olduğu varsayımıyla dönüştürüp göstericiyi kullanarak elemanlara ulaşabiliriz. 
auto pDataFloat = static_cast<float*>(pDataVoid);
std::cout << pDataFloat[7] << "\n"; // İki boyutta [1][1]  tek boyutta 7. eleman yani 12

// Alternatif olarak 
// Doğrudan 1,0 indeksinde bulunan veriye data fonksiyonu ile de ulaşabiliriz.
std::cout << tensorInit.data()[1][1] <<"\n" ; // örnekte 12
```

*LibTorch* ile veriye erişim için `torch::Tensor` kütüphanesinin sunduğu bir diğer alternatif ve tavsiye edilen yöntemse `accessor` kullanımıdır. Burada CPU ve GPU için ayrı `accessor` kullanmak gerekmektedir. Önce bir CPU tensörü için ardından da GPU tensörü için bu işlemi kullanalım:

 ```c++
auto tensorInit = torch::tensor({{1.0, 2.0, 4.0, 2.0, 3.0, 5.0},{11.0, 12.0, 14.0, 12.0, 13.0, 15.0}});

auto tensorInitAccessor = tensorInit.accessor<float, 2>();
for (int i = 0; i < tensorInitAccessor.size(0); i++)
    for (int j = 0; j < tensorInitAccessor.size(1); j++) {
        std::cout << "Data in position " << i << "-" << j << ": " << tensorInitAccessor[i][j] << "\n";
    }
 ```

Bu kullanımda dikkat edilecek husus `accessor` nesnenin şablon parametrelerine veri tipi olan `float` ve boyutu gösteren `2` göndermemiz gerektiğidir. Yani farklı veri tipi ve boyutlar için bir özelleşme gerekecektir. Yani işleri çokta kısalttığını söyleyemem. Dokümantasyonun iddiası daha hızlı erişim sağladığı yönünde ama içimden bir ses gösterici kullanımının yavaş olmasını pek mümkün görmediğimi için büyük bir tensörde tüm elemanlara ulaşmayı test etmek istedim: 

```c++
#include <iostream>
#include <torch/torch.h>
#include <chrono>

int main() {
    using namespace std::chrono;
    
    const int HEIGHT = 1920, WIDTH = 1080, CH = 300;
    long double sum = 0;
    
    auto randBigTensor = torch::rand({HEIGHT, WIDTH, CH});
    
    auto start = high_resolution_clock::now();
    auto tensorInitAccessor = randBigTensor.accessor<float, 3>();
    for (int i = 0; i < tensorInitAccessor.size(0); i++)
        for (int j = 0; j < tensorInitAccessor.size(1); j++)
            for (int k = 0; k < tensorInitAccessor.size(2); k++) {
                sum += (tensorInitAccessor[i][j][k]) / 1000;
            }
    auto end = high_resolution_clock::now();
    duration<double> time_span = duration_cast<duration<double>>(end - start);
    std::cout << "It took me " << time_span.count() << " seconds (Using accesscor). Sum = " << sum << "\n";

    sum = 0;
    start = high_resolution_clock::now();
    auto pDataVoid = randBigTensor.data_ptr();
    auto pDataFloat = static_cast<float *>(pDataVoid);
    for (int i = 0; i < (HEIGHT * WIDTH * CH); i++)
        sum += (pDataFloat[i]) / 1000;
    end = high_resolution_clock::now();
    time_span = duration_cast<duration<double>>(end - start);
    std::cout << "It took me " << time_span.count() << " seconds (Using data_ptr). Sum = " << sum << "\n";
    
    return 0;
}
```

Sonuçlar aslında beklediğim gibi oldu. En hızlısı veri göstericiyi alıp veriye erişmek oldu. Ama bu testi farklı boyutlarda hatta ideali uygulamanızda kullanacağınız tensör boyutlarıyla test etmek daha mantıklı olabileceğini düşünüyorum.

```shell
It took me 18.573 seconds (Using accesscor). Sum = 311033
It took me 4.24126 seconds (Using data_ptr). Sum = 311033
```

Bazen bu verinin belirli elemanlarına erişmek isteyebiliriz. Bu durumda *Python*'dan alışık olduğumuz Indexing API kullanımı daha kolay olacaktır. Bu API'de hem okuma hem de yazma işlemi yapmak mümkündür:

```c++
auto randTensor = torch::rand({100, 100, 3});

//1,0,5 konumundaki eleman
std::cout << randTensor.index({1,0,5}); 

// Slice fonksiyonuyla 0.boyuttaki verileri 1.indeksten başlayıp 10'a kadar 2'şer artan şekilde alır ve diğer iki eksende 0. indekse denk gelen elemanları gösterir. Burada `torch::indexing` isim alanının eklendiğine dikkat edin.  
using namespace torch::indexing;
std::cout << randTensor.index({Slice(/*start_idx:*/1, /*stop_idx:*/10, /*step:*/2), 0, 0})}); 

//1,0,5 konumundaki elemana değer atama
randTensor.index({1,0,5}) = 0.05
```

Indexing API'nin *Python* ve *C++* için kullanımının kıyaslaması için [bağlantıya](https://pytorch.org/cppdocs/notes/tensor_indexing.html) tıklayın. 

# 4. Dönüşüm İşlemleri

Bu yazıyı bitirmeden önce son olarak tensörlerin dönüşüm işlemlerine bakalım.  Burada ilk oluşturma esnasında belirlediğimiz tensör seçeneklerini dönüştürmek mümkündür.

```c++
auto sourceTensor = torch::randn({2, 3}, torch::kFloat16);

// Veri tipini dönüştürme
auto floatTensor32 = sourceTensor.to(torch::kFloat32);

// Aygıt tipine göre dönüştürme
auto gpuTensor = floatTensor32.to(torch::kCUDA);
```

Evet yazının sonuna geldik. Yazı dizisinin bir sonraki yazısında artık modellere doğru geçiş yapacağımız yazı hazır olduğunda bu yazının altına bağlantıyı ekleyeceğim. 





