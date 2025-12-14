---
layout: post
comments: true
title: C++ Derin Öğrenme-3 Pytorch C++ API LibTorch Modelleri Çalıştırma
lang: tr
header:
  teaser: "https://blgnksy.github.io/assets/img/libtorch_infer/cpu_gpu.png"
tags: [Deep Learning, Derin Öğrenme, PyTorch, LibTorch, PyTorch C++ API, Machine Learning C++, Deep Learning C++, Machine Learning, Makine Öğrenmesi, Inference, Çıkarım]

---



Yazı dizisinin bu yazısında *LibTorch*'u kullanarak modelleri nasıl çalıştıracağımızı, çıkarım (inference) için nasıl kullanacağımızı göreceğiz. Yazı dizisinin ilk yazısında kullanım senaryolarını açıklarken *Python*'da eğitilen modeli *C++*'da çıkarım için kullanıp bazı darboğazları atlatabileceğinizi aktarmıştım. Şimdi *Python*'da bir modeli eğitip (bununla zaman kaybetmemek için öneğitimli bir modeli kullanacağım), kaydedip *C++*'da yükleyip çıkarım yapacağız. 

İlk önce *Python*'da bir model eğittimizi ve modelin `resnet152` nesnesinde tutulduğunu varsayalım. Dediğim gibi ben doğrudan öneğitimli bir model kullanacağım:

```python
import torchvision.models as models
import torch

resnet152 = models.resnet152(pretrained=True)

script = torch.jit.script(resnet152)
traced = torch.jit.trace(resnet152, torch.rand(1, 3, 224, 224)) #Traced model için örnek girdi sağlanmalıdır.

script.save("./model_zoo/resnet152_sc.pt")
traced.save("./model_zoo/resnet152_tr.pt")
```

Burada modeli `torch`'un `jit` modülünü kullanarak hem `script` modunda hem de `trace` modunda kaydedelim. *Python API*'da `jit` modülünün kullanımı hakkında bilgi sahibi değilseniz kısa bir ara verip [dokümantasyonu](https://pytorch.org/tutorials/beginner/Intro_to_TorchScript_tutorial.html) okumanızı tavsiye ederim. Her ikisini de kaydetme nedenim ileride karşılaştırma için kullanacak olmamdır. Şimdi kaydedilen modelleri kullanarak *C++*'da hızlıca çıkarım yapalım:

```c++
#include <torch/script.h>
#include <iostream>

int main(int argc, const char* argv[]) {
    if (argc != 2) {
        std::cerr << "Usage: infer <path-to-exported-script-module>\n";
        return -1;
    }

    at::globalContext().setBenchmarkCuDNN(1);
    torch::jit::script::Module module;
    try {
        // Deserialize the ScriptModule from a file.
        module = torch::jit::load(argv[1]);
        std::cout << "Module loaded successfully.\n";
        module.eval();
    }
    catch (const c10::Error& e) {
        std::cerr << "Error loading the model.\n";
        return -1;
    }
	
    //RESNET input shape (BATCH_SIZE, 3, 224, 224)
    const int BATCH_SIZE = 8, CHANNELS = 3, HEIGHT = 224, WIDTH = 224; 

    torch::NoGradGuard no_grad;
    // Create a vector of inputs.
    std::vector<torch::jit::IValue> inputs;
    inputs.emplace_back(torch::ones({BATCH_SIZE, CHANNELS, HEIGHT, WIDTH}));

    // Execute the model and turn its output into a tensor.
    auto output = module.forward(inputs).toTensor();
    for (int i = 0; i < BATCH_SIZE; ++i) {
        std::cout << i << "th element class: " << torch::argmax(output.data()[i]).item<long>() << "\n";
    }
	return 0;
}
```

Bu kodu derleyip çalıştıralım ve arkasından kodu inceleyelim:

```shell
$ ./infer ./model_zoo/resnet152_sc.pt
Module loaded successfully.
Class of 0th element: 600
Class of 1th element: 600
Class of 2th element: 600
Class of 3th element: 600
Class of 4th element: 600
```

Kodu en baştan inceleyecek olursak ilk olarak `torch/script.h` başlık dosyasını kodumuza dahil etmemiz yeterli olacaktır. Ardından `torch::jit::script::Module` sınıfının bir örneğini oluşturuyoruz. Bu aslında *Python* da kullandığımız `torch.nn.Module` sınıfıdır. Sonuç olarak model nesnesini taşır ve *Python* API'ın sağladığı özellikleri kullanmamızı sağlar:

```c++
torch::jit::script::Module module;
```

Ardından daha önce *Python*'da kaydettiğimiz modelleri dosyadan okuyup bu nesneye aktarıyoruz. Kaydedilen modeli komut satırından argüman olarak alıp modeli okuyoruz ve hata yakalama mekanizmasını kullanıyoruz:

```c++
    try {
        // Deserialize the ScriptModule from a file.
        module = torch::jit::load(argv[1]);
        std::cout << "Module loaded successfully.\n";
    }
    catch (const c10::Error& e) {
        std::cerr << "Error loading the model.\n";
        return -1;
    }
```

Burada model CPU üzerinde çalışacak şekilde kullanıyoruz. GPU kullanmak isterseniz aşağıdaki kodu modeli yüklediğiniz satırdan sonraki satıra eklemeniz yeterli olacaktır:

```c++
module.to(at::kCUDA);
```

Girdilerimizi modele beslemek için ihtiyacımız olan vektörü hazırlayacağız. `torch::jit::script::Module` sınıfının `forward` fonksiyonu bizden `std::vector<torch::jit::IValue>` türünden bir vektör bekliyor. Fonksiyon argümanı olan vektörü `std::move()` ile taşıma semantiğine uygun olarak kullandığından bellek kullanımı ve hız kaybını olası en düşük hale getirmeyi amaçladığını hatırlatmak isterim: 

```c++
std::vector<torch::jit::IValue> inputs;
inputs.push_back(torch::ones({BATCH_SIZE, CHANNELS, HEIGHT, WIDTH}));
//for GPU tensors:
//inputs.push_back(torch::ones({BATCH_SIZE, CHANNELS, HEIGHT, WIDTH}).to(at::kCUDA));
```

`torch::jit::IValue` bir sınıf olarak tanımlanmıştır. *Interpreter Value*'nun kısaltması olarak `IValue` sınıfı *TorchScript interpreter* tarafından desteklenen tüm temel türleri sarmalamaktadır. `IValue` sınıfı modellere girdi ve çıktılar için kullanılır. Bu sınıfın arayüzü oldukça geniş olmasına rağmen temel iki fonksiyonu için aşağıya bakabilirsiniz:

```c++
///   // Make the IValue
torch::IValue my_ivalue(26);
std::cout << my_ivalue << "\n";
///
///   // Unwrap the IValue
int64_t my_int = my_ivalue.toInt() //toX() instead of X use appropriate data type for wrapped data.
std::cout << my_int << "\n";
```

Bu noktada neden böyle bir sınıfa ihtiyaç duyulduğuna gelecek olursak *C++* tarafından sağlanan temel türlerden farklı olarak *LibTorch*'un sağladığı türler farklıdır. *Python* API ile *C++* API arasında uyumu sağlamaya yardımcı olduğundan öğrenme/kullanma/alışmayı kolaylaştırmaktadır.

Artık son olarak modele girdileri besleyip çıktıda yığındaki (batch) her bir eleman en olası sınıfı standart çıktıya yazdırıyoruz:

```c++
auto output = module.forward(inputs).toTensor();
for (int i = 0; i < BATCH_SIZE; ++i) {
    std::cout << "Class of " << i << "th element: " << torch::argmax(output.data()[i]).item<long>() << "\n";
}
```

Bu noktada bir karşılaştırma yapmakta fayda olacaktır. Bunun için Python'da `jit` modülünü kullanmadan, `script` ve `trace` modunda farklı yığın boyutları için CPU ve GPU'da kaydedilen modeli okuma ve çalışma süreleriyle bu işlemlerin C++'da yaptığımızda elde edeceğimiz süreleri (tüm testler 10 defa çalıştırılıp süre ortalama olarak hesaplanmıştır) karşılaştıracağız. İlk olarak kaydedilen model dosyalarının boyutlarına bakalım. Dosya boyutları arasında anlamlı bir fark bulunmadığını söyleyebiliriz: 

| Modül Tipi       | Dosya Boyutu |
| ---------------- | ------------ |
| torch.nn.Module  | 241.6 MB     |
| torch.jit.script | 242 MB       |
| torch.jit.trace  | 242.2 MB     |

Şimdi model dosyalarının diskten okunup CPU üzerinde modelin çalıştırılabilir hale getirilmesi için geçen süreye bakalım. Görüldüğü üzere öncelikle okumayı C++'da çok daha hızlı yapabiliyoruz:

| CPU Okuma zamanı (ms) | Python | C++   |
| --------------------- | ------ | ----- |
| Script                | 0.985  | 0.449 |
| Trace                 | 0.912  | 0.356 |

Şimdi de model dosyasının okunması ve GPU üzerinde çalıştırılabilir hale getirilmesi için geçen süreye bakalım. Hız farkı azalsa da *C++* hala daha hızlı okuyor ve modeli çalıştırılabilir hale getiriyor:

| GPU Okuma zamanı (ms) | Python | C++   |
| --------------------- | ------ | ----- |
| Script                | 2.286  | 2.137 |
| Trace                 | 2.214  | 2.025 |

Artık yığın işleme sürelerini inceleyelim. Soldaki çizgede CPU, sağdaki çizgedeyse GPU üzerinde farklı yığın boyutlarında bir örnek için modelin çıkarım süreleri görülmektedir. Test durumlarının %95'inde değişen oranlarda *C++* API'ı daha hızlı çalışmaktadır. CPU üzerinde genelde `script` modunda kaydedilen modeller, GPU tarafındaysa `trace` modunda kaydedilen modeller daha hızlı çalışmışlardır. Bu hesaplamalarda tek işlem/iş parçacığı kullanılmıştır:



![](/assets/img/libtorch_infer/cpu_gpu.png)

Sonuç olarak aslında *C++ API* kullanarak modelleri çalıştırmanın hiç de zahmetli olmadığını gördüğünüzü düşünüyorum. Aslında sıfırdan bir modeli *C++* üzerinde geliştirmekte çok zor değil. Sonraki yazılarda bunu da açıklamaya çalışacağım. Ama öncelikle verilerle nasıl başa çıkacağımızı incelemenin daha iyi olacağını düşünüyorum. Sonraki yazıda verileri (resim, video, csv, metin vb.) okumayı ve *LibTorch*'a çıkarım ve eğitim sırasında bu verileri nasıl sağlayacağımızı açıklayacağım. 

*Dizinin diğer yazıları:*

1. [C++ Derin Öğrenme-1 Pytorch C++ API LibTorch Giriş](https://blgnksy.github.io/tr/2020/12/03/libtorch-config.html)

2. [C++ Derin Öğrenme-2 Pytorch C++ API LibTorch Tensör İşlemleri](https://blgnksy.github.io/tr/2020/12/06/libtorch-tensors.html)



