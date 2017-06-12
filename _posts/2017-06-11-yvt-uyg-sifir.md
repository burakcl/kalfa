---
layout: post
title: Uygulama Sıfır
categories: [Uygulama]
tags: [youtube, bash, shell, resim]
url: https://burakcl.github.io/kalfa/uygulama/2017/06/11/yvt-uyg-sifir.html
---
Geçen ay ödevlerimden bir tanesi html dosyalarını işleyebilen bir java programlama dili kütüphanesi olan [Jsoup](https://jsoup.org/) üzerine çalışmıştım. Verilen ödev [beyazperde.com](http://www.beyazperde.com/) üzerindeki barınan film yorumlarını işlemekti. Bütün filmlerin olduğu sayfadan sırasıyla film adı, film adresi, film hakkındaki eleştirilere erişmemiz gerekiyordu ve jsoup kütüphanesini kullanarak bu işlemleri yapmak hatta verileri html etiketlerine göre ayıklama özelliği hoşuma gitmişti. Buraya kadar sorun yok olay basit.

Site üzerinde bulunan film sayısı yaklaşık 16.000 civarı ve her filmin sabit olmayan sayıda eleştirisi mevcuttu. Her sayfada belirli sayıda yorum veya film vardı. Örneğin, B filminin eleştiri sayısı 300 diyelim her sayfada 20 yorum görüntülendiğini düşünürsek 15 sayfadan oluşturan bir eleştiri sayfa dizisi mevcut ve bunları işlemek için gerekli araç seti jsoup sayesinde baya baya rahattı. Ama kabukta bunu yapsak nasıl olurdu diye aklıma gelen parçalara bakınca `curl`, `grep`, `cut` üçlüsü bu işi görürdü dedim orada kalmaması için yazdım not defterime.

Devam edelim, [youtube-dl](http://rg3.github.io/youtube-dl/) aracını artık çoğumuz biliyoruz yeni başlayan arkadaşlara tavsiye edebileceğim oradan buradan video, müzik ve daha türlü iş için alet çantası gibi bir konsol uygulaması. Uygulama yer alan video ile birlikte tırnak resmini indirebildiğimiz bir parametresi mevcut( Parametre ne hacı? `man youtube-dl /thumbnail`). Ben sadece resmi istiyorum ama o videoyu bonusa ekliyordu. Sonra dedim bu işi yapan bir betik yazalım.

### Düşünceden analize
Youtube'a girip hemen `f12` ile html kodumuza bir göz gezdirelim. Neyi istiyorduk videonun tırnak fotoğrafını. Hemen istediğimiz elemanı seçip resim nereden geliyormuş bakalım.

<div style="text-align:center"><img src ="{{site.BASE_PATH}}/assets/media/pimg/yvk-sh/iytthumb.png" /></div>

Evet tırnak resimler `https://i.ytimg.com/vi/CC9zCAmkGvI/hqdefault.jpg` video koduna göre geliyor. İki kaynak adreste de değişen tek bilgi 11 haneli video kodu. O zaman resim için yazacağımız betik video kodunu vermemiz yeterli olacaktır.

Adresi kontrol edeyim diye adrese girdiğimde ne göreyim bizim resim 480*360 ama ben daha büyük çözünürlükte istiyorum. youtube-dl aracında video adresi verdiğimize göre video adresinde bu bilginin mevcut olduğunu düşünüyorum. O zaman adreste f12 ile bir inceleme yapalım thumbnail etiketi olduğunu görüyorum. Bu yüzden thumbnail ifadesini aratıp bakacağım.

<div style="text-align:center"><img src ="{{site.BASE_PATH}}/assets/media/pimg/yvk-sh/iytthres.png" /></div>

Buradaki adresi diğeri ile karşılaştırıp farkları görelim.

{% highlight ruby %}
  `https://i.ytimg.com/vi/CC9zCAmkGvI/hqdefault.jpg`
  `https://i.ytimg.com/vi/CC9zCAmkGvI/maxresdefault.jpg`
{% endhighlight %}

Farklılığın sadece resim adında olduğunu görüyoruz. Demekki çözünürlük ayarı bu şekilde belirlenmiş. İki resmin sonundaki "default" ifadesi sanki bir resme daha işaret eder gibi diyerek istek gönderdiğimde küçük resmi de tespit etmiş olduk.

{% highlight ruby %}
 `https://i.ytimg.com/vi/CC9zCAmkGvI/default.jpg`
{% endhighlight %}

Nasıl çalışacağı hakkında artık bir sıralama yapabiliriz.

* Parametre olarak dışarıdan video kodu ve büyüklük alınır.
* Resmin video kodu ve resim büyüklüğü ile adresi oluşturulur.
* Resim adresten indirilir.

### Yazmaya başlayalım
İki parametre ve bir adet kullanımı açıklayan yardım bilgisi olsun.
* Değişkenlere değer atarken eşittirin sağında ya da solunda boşluk bırakılmaz.
Kuralımızı hatırlayarak yardım bilgisini oluşturalım.

{% highlight sh %}
#!/bin/bash
yardim="resim-cek [youtube video kodu] [ kucuk | orta | buyuk | k | o | b ]\n"
{% endhighlight %}

Boyut bilgisi için çoktan seçmeli bir yola gitmemiz gerekiyor. Bir kaç farklı deyimle bunu yapabiliriz ama if-elif yapısını kullanmak istiyorum ve aynı zamanda yazdığımız bu bloğu bir fonksiyon haline getirmek istiyorum geliştirme yaparken kolaylık olması açısından bu tarz bir seçime gidiyorum. Herhangi bir duruma karşı else ifadesi ile seçeneklerimi destekliyorum.

{% highlight sh %}
function fboyut(){
    if [[ $1 = "k" || $1 = "kucuk" ]]; then
      boyut="";
    elif [[ $1 = "o" || $1 = "orta" ]]; then
      boyut="hq";
    elif [[ $1 = "b" || $1 = "buyuk" ]]; then
      boyut="maxres";
    else
      echo -en "Resim boyutu giriniz. Yardim için -y veya --yardim\n";
      exit;
    fi
}
{% endhighlight %}

* Fonksiyonları betik içerisinde bir betik gibi düşünebilirsiniz. Kullanımına baktığımızda aynı komut satırından yazdığımız komutlar gibi çağırıp parametre gönderebiliyoruz. İlk gönderilen parametre ana betikte olduğu gibi bu fonksiyon bloğunda da $1 ile alınır. Bu aklımızın bir kenarında dursun lazım olacak.

{% highlight sh %}
if [[ $1 = -y || $1 = --yardim ]]; then
    echo -en "$yardim";
    exit;
fi
{% endhighlight %}

Yardım almak için gereken parametre ve kontrol yapısını oluşturduk. Yardım bilgisini ekrana bastıktan sonra sürecin çıkış işlemini gerçekleştirdik.

{% highlight sh %}
if [[ $1 ]]; then
  fboyut $2;
  wget -b -q $(echo "https://i.ytimg.com/vi/$1/$boyut")default.jpg;
  echo -en "Başarıyla indirildi.\n";
else
  echo -en "Video kodunu giriniz. Yardim için -y veya --yardim \n";
fi
{% endhighlight %}

Betiğin ana kısmına geldik. Öncelikle parametre kontrolu yapıyoruz $1 değeri mevcut ise fboyut fonksiyonuna boyut bilgisi için boyut parametresini gönderiyoruz. Boyut kontrolu olumlu sonuçlanır ise wget aracına oluşturacağımız adresi indirmesini isteyeceğim. $1 mevcut değilse else durumu ile video kodu istenerek tavsiye bilgisi ekrana basılacak.

### Betiğimizi toparlayalım

{% highlight sh %}
#!/bin/bash

yardim="resim-cek [youtube video kodu] [ kucuk | orta | buyuk | k | o | b ]\n"

function fboyut(){
    if [[ $1 = "k" || $1 = "kucuk" ]]; then
      boyut="";
    elif [[ $1 = "o" || $1 = "orta" ]]; then
      boyut="hq";
    elif [[ $1 = "b" || $1 = "buyuk" ]]; then
      boyut="maxres";
    else
      echo -en "Resim boyutu giriniz. Yardim için -y veya --yardim\n";
      exit;
    fi
}

if [[ $1 = -y || $1 = --yardim ]]; then
    echo -en "$yardim";
    exit;
elif [[ $1 ]]; then
  fboyut $2;
  wget -b -q $(echo "https://i.ytimg.com/vi/$1/$boyut")default.jpg;
  echo -en "Başarıyla indirildi.\n";
else
  echo -en "Video kodunu giriniz. Yardim için -y veya --yardim \n";
fi
{% endhighlight %}

Zamanı geldi, betiğimize çalıştırma yetkisi atayalım ve kabuğa gönderelim. Sırasıyla yazdıklarımızı test edelim.

<div style="text-align:center"><img src ="{{site.BASE_PATH}}/assets/media/pimg/yvk-sh/res-cek-sh.png" /></div>

İndirilen resimlerin isimleri dikkatimi çekmekte. `default`, `hqdefault` ve  `maxresdefault` bu isimlerin bu şekilde kalmasını istemiyorum video ismi neyse onlara kendi ismini verelim bunun için yine bir fonksiyon yazalım.
Bu sefer yapacağımız iş sırası

* Video kodu ile youtube adresini oluşturmak
* Oluşan video adresine bağlantı sağlamak
* Bu adresten başlık veya video ismini ayıklama işlemini gerçekleştirmek
* Daha sonra bunu bir değişken ile tutmak gerekecek

{% highlight sh %}
function fyeniad(){
  yeniad=$(curl -s $(echo -n  "https://www.youtube.com/watch?v=$1") | grep "eow-title"| cut -d '"' -f8 ).jpg
}
{% endhighlight %}

Fonksiyon adını `fyeniad` olarak belirledim. Yukarıda yazdığımız sıra ile `adresi oluşturduk(echo)`, arkasından oluşan `adrese bağlantı sağladık(curl)`, bağlantı sonucu elde ettiğimiz veri içerisinden başlığı ya da `video ismini bulmak için grep` kullandık arama işleminde kullanılan `eow-title` ifadesi video adının bulunduğu html etiketinin id değeri neden başlığı çekmedin diye bir soru oluşabilir `title` araması yaptığımda video ismi çıktı elbette ama ayıklama işleminde iki kere kesmem gerekiyordu bende daha rahat ayıklayabileceğim video isminin bulunduğu etiketi seçtim çünkü kendi sayfasında eşsiz etiketlerden ikincisi de oydu. `Ayıklama işlemi için cut` ile tırnaklardan ayırma işlemi gerçekleştirdim 8'inci parçaya denk gelen bölüm video ismiydi tamamdır bu iş diyerek paranteze alıp değişken değeri haline getirdim. `yeniad` isimli değişkene atadım. Şimdi bu parçayı ana blokta yerine koyalım.

{% highlight sh %}
if [[ $1 ]]; then
  fboyut $2;
  fyeniad $1;
  wget -b -q -O $yeniad $(echo "https://i.ytimg.com/vi/$1/$boyut")default.jpg;
  echo -en "$yeniad Başarıyla indirildi.\n";
else
  echo -en "Video kodunu giriniz. Yardim için -y veya --yardim \n";
fi
{% endhighlight %}

İşlem tamam çalıştıralım dediğimde sonuç bir sorun olduğunu gösteriyordu. Çıktıya baktığımda video isminin parça parça gönderildiğini görünce kabukta boşluk karakterlerinin dans ettiğini anladım. Sorun video ismindeki boşluklar, kabuğun bunu yorumlamasını bekleyemeyiz zaten o boşluklara göre hareket ediyor. Deyimlere, parametre göndermelerine bakınca bu kolaylıkla anlaşılıyor aslında. Çözümü tr ile sağladım, `tr ' ' '-'` boşluk yerine tire atalım şimdi silsek pek okunaklı olmaz.

İsim ekleme işleminide başarılı bir şekilde tamamladık en son haline baktığımızda. Şunu farkettim resimleri indiriyorum iyi güzel ama bir büyük bir küçük indirmek istediğimde aynı isim verildiğini ve birbiri üzerine yazıldığını farkettim hemen buna bir çözüm getirelim diyerek `fyeniad` fonksiyonu üzerinde değişikliğe başlayalım.

* Resim büyüklüğüne göre sonuna k, o ve b bırakmak istiyorum.
* .jpg ifadesin hemen öncesinde olmasını istiyorum.

{% highlight sh %}
# fyeniad fonksiyonumuzun son hali

function fyeniad(){
  yeniad=$(curl -s $(echo -n  "https://www.youtube.com/watch?v=$1") | grep "eow-title"| cut -d '"' -f8 | tr ' ' '-')
  if [[ $boyut = "" ]]; then
    yeniad=$yeniad-k.jpg
  elif [[ $boyut = "hq"  ]]; then
    yeniad=$yeniad-o.jpg
  elif [[ $boyut = "maxres"  ]]; then
    yeniad=$yeniad-b.jpg
  fi
}
{% endhighlight %}

Artık istediğimiz gibi çalışıyor.

<div style="text-align:center"><img src ="{{site.BASE_PATH}}/assets/media/pimg/yvk-sh/res-cek-snc.png" /></div>

### Şimdi tek tek kim indirecek bunları
Buna çözüm olarak veya bir kere de birden fazla indirmek istenen durumlar için youtube listelerini lehimize kullanabiliriz. O zaman hemen liste yapısını inceleyelim neler yapabiliriz görelim.

Örnek liste adresim : `https://www.youtube.com/playlist?list=PL5l85sJPWx22bzIbzPNBQ7ew8IK9B-nli`
Liste kodu : `PL5l85sJPWx22bzIbzPNBQ7ew8IK9B-nli`

Listedeki videoların adreslerini buradan almamız gerekli.

<div style="text-align:center"><img src ="{{site.BASE_PATH}}/assets/media/pimg/yvk-sh/yliste-yapisi.png" /></div>

Bunun için öncelikle yazdığımız betiğimizi bu tarz bir geliştirme için hazırlayalım.

* `-l` veya `--liste` adında parametremiz olsun.
* videolar için parametremiz `-c` olsun.
* daha sonra liste kodunu alıp listeyi çekelim.
* listedeki video sayısını ve video kodlarını bulalım.
* daha sonra bunları istenen büyüklüğe göre liste adında oluşturduğumuz dizine indirelim.

### Başlayalım
Sıradan gidersek yardım bilgisini düzenleyelim.
{% highlight sh %}
yardim="resim-cek [-c youtube video kodu] [ kucuk | orta | buyuk | k | o | b ]\n Liste resimlerini indirmek için: [-l veya --list (youtube video liste kodu)]\n\n"
{% endhighlight %}
İsim ve boyut fonksiyonlarında herhangi bir değişikliğe ihtiyaç görünmüyor. Aslında bir noktaya değinelim fonksiyon olarak hazırlamak bize fayda sağlıyor betiğin okunulabilirliğinden tutunda herhangi bir yeni seçenek için rahat işlem kolaylılığı sağlıyor.

{% highlight sh %}
if [[ $1 = -y || $1 = --yardim ]]; then
    echo -en "$yardim";
    exit;
elif [[ $1 = -l || $1 = --liste ]]; then
   # Buraya liste işlemlerimizi yazacağız
elif [[ $1 = -c ]]; then
  fboyut $3;
  fyeniad $2
  wget -b -q -O $yeniad $(echo "https://i.ytimg.com/vi/$2/$boyut")default.jpg;
  echo -en "$yeniad başarıyla indirildi.\n";
else
  echo -en "Video kodunu giriniz. Yardim için -y veya --yardim \n";
fi
{% endhighlight %}

İlk iki adım tamam sıra liste kodunu kullanarak liste uzunluğunu ve adını alalım. `liste_uz` adında bir değişken tanımladık ve `echo` ile liste adresini oluşturup bunu `curl` aracına verelim ve listeyi çekelim. Çekilen bu listede etiket id'si veya class'ı aramalıyım çünkü videolar düzenli halde ve belirli bir formatta. İncelememe göre `class="pl-video-title-link"` adında bir linkleme etiketine sahip ve video sayısı kadar aradağımızı bulduk bu parçayı `grep` ile arayacağız ve `wc -l` ile satır sayısını baz alarak video sayısını çıkartmış olacağız.
{% highlight sh %}
liste_uz=$(curl -s $(echo -n  "https://www.youtube.com/playlist?list=$2") | grep "pl-video-title-link"| wc -l);
{% endhighlight %}
Liste adına gelince hemen bakalım etiket adının `class="pl-header-title"` olduğunu görüyoruz. Bu kısmı incelediğim `curl` çıktısında 3 satır olarak çıktı ve buradan çıkartamadım `grep` ile bulamadığım için sayfa başlık bilgisine yöneldim oradan çıkartma işlemini sağladım. Bu çıkarma işlemine bakalım.

{% highlight sh %}
liste_ad=$(curl -s $(echo -n  "https://www.youtube.com/playlist?list=$2") | grep "<title>" | cut -d '-' -f1 | sed 's/<title>//g' | sed 's/ //g');
{% endhighlight %}

`<title>` ifadesini arattığımda çıktı tek satır ve işlem yapmaya müsaitti.
{% highlight sh %}
<title>Stolk Seçme Haberler - YouTube</title>
{% endhighlight %}
`cut` ile ikiye böldüm ilk parçayı aldım. Kalan parça olarak `<title>` kalmıştı o zaman ara bul değiştir üçlüsünü bize sağlayan `sed` den yardım alarak çıkartma işlemi tamamlıyoruz ve yine kalan boşluklar için iki seçenek mevcut ya `\` işaretini kullanacağız ya da yoket diyerek boşluk sorununu ortadan kaldırıyoruz. Liste adında bir dizin oluşturup içerisine girelim. Ardından fboyut fonksiyonumuzu gönderelim.

{% highlight sh %}
mkdir $liste_ad;
cd $liste_ad;
fboyut $3;
{% endhighlight %}

Şimdi sıra geldi listeden kodları bulup indirme işlemine. Öncelikle bir döngü ayarlayalım liste uzunluğu kadar dönsün hatırlarsanız liste uzunluğunu video etiketlerini bulmuştuk şimdi onun içindeki kodları bulup eğlenelim.

{% highlight sh %}
j=1
while [ $j -le $liste_uz ]; do
  # yvkodlarını üretip indirme işlemine yönlendireceğiz.
  j=$((j+1));
done
{% endhighlight %}

`cut` ile sadece `yvkodunu` barındıran listeyi elde ettik. `cut` ile kesme olayında önemli noktalardan biri sürekli tekrar eden en yakın noktayı bulmak diyebilirim orasını bir sınır düşünebilirsiniz elimizde bölecek bir taş kalmadığında içerisinde erişmek istediğimiz ifadenin şekli şemalini [düzenli ifadeler](http://www.regexr.com/) ile belirtip aradığınızı bulabilirsiniz.

{% highlight sh %}
yvkodu="$( curl -s $(echo -en  "https://www.youtube.com/playlist?list=$2") | grep "pl-video-title-link" | cut -d '=' -f 5 | cut -d '&' -f1 | head -n $j | tail -n 1)";
{% endhighlight %}

`head` ve `tail` ikisini kullanmak herhalde yazdığımız betik içerisinde en çok hoşuma giden kısımlardan bir tanesi galiba. Yukarıda yazdığımız `head` listenin her seferinde `j`'ye kadar olan bölümünü listelerken bunu `|` ile `tail -n 1` ifadesine yönlendiriyoruz. Yani her head çıktısının son satırını alarak ilerliyoruz. Kodları elde ettik hadi indirme kısmını düzenleyelim.

{% highlight sh %}
fonksiyon findir(){
  fyeniad $1;
  wget -b -q -O $yeniad $(echo "https://i.ytimg.com/vi/$1/$boyut""default.jpg");
  echo -e " $yeniad başarıyla indirildi.\n";
}
{% endhighlight %}
Şeklinde düzenleme işlemini sağladık. Burada `$1` tekrar hatırlarsak fonksiyona gelen ilk parametreydi.

### Son Dokunuşlar
Son halinde betiğimizi toparlarsak

{% highlight sh %}
#!/bin/bash
yardim="resim-cek [-c youtube video kodu] [ kucuk | orta | buyuk | k | o | b ]\n Liste resimlerini indirmek için: [-l veya --list (youtube video liste kodu)] [ kucuk | orta | buyuk | k | o | b ]\n\n";

function fboyut(){
    if [[ $1 = "k" || $1 = "kucuk" ]]; then
      boyut="";
    elif [[ $1 = "o" || $1 = "orta" ]]; then
      boyut="hq";
    elif [[ $1 = "b" || $1 = "buyuk" ]]; then
      boyut="maxres";
    else
      echo -en "Resim boyutu giriniz. Yardim için -y veya --yardim\n";
      exit;
    fi
}

function fyeniad(){
  yeniad=$(curl -s $(echo -n  "https://www.youtube.com/watch?v=$1") | grep "eow-title"| cut -d '"' -f8 | tr ' ' '-');
  if [[ $boyut = "" ]]; then
    yeniad=$yeniad-k.jpg
  elif [[ $boyut = "hq"  ]]; then
    yeniad=$yeniad-o.jpg
  elif [[ $boyut = "maxres"  ]]; then
    yeniad=$yeniad-b.jpg
  fi
}

function findir(){
  fyeniad $1;
  wget -b -q -O $yeniad $(echo "https://i.ytimg.com/vi/$1/$boyut""default.jpg");
  echo -e " $yeniad başarıyla indirildi.\n";
}

if [[ $1 = -y || $1 = --yardim ]]; then
    echo -en "$yardim";
    exit;
elif [[ $1 = -l || $1 = --liste ]]; then
    liste_uz=$(curl -s $(echo -n  "https://www.youtube.com/playlist?list=$2") | grep "pl-video-title-link"| wc -l);
    liste_ad=$(curl -s $(echo -n  "https://www.youtube.com/playlist?list=$2") | grep "<title>" | cut -d '-' -f1 | sed 's/<title>//g' | sed 's/ //g');
    mkdir $liste_ad;
    cd $liste_ad;
    fboyut $3;
    j=1
    while [ $j -le $liste_uz ]; do
    yvkodu="$( curl -s $(echo -en  "https://www.youtube.com/playlist?list=$2") | grep "pl-video-title-link" | cut -d '=' -f 5 | cut -d '&' -f1 | head -n $j | tail -n 1)";
    findir $yvkodu;
    j=$((j+1));
    done
elif [[ $1 = -c ]]; then
  fboyut $3;
  findir $2;
else
  echo -en "Video kodunu giriniz. Yardim için -y veya --yardim \n";
fi

{% endhighlight %}

#### Çalıştıralım.
<div style="text-align:center"><img src ="{{site.BASE_PATH}}/assets/media/pimg/yvk-sh/sonuc.png" /></div>

## Uygulama 0 bitmedi olarak buraya not düşelim eksik bıraktığımız noktalar mevcut.
Buraya o eksikleri yazalım.
* Oluşturulacak dizin kontrolü aynı isimde dizin var mı ya da hedef dizin var mı?
* Bağlantı kontrolu
* Video kodu ve liste kodu doğrulama
* Yeni bir özellik eklersek onu da buraya ekleriz.

### Suyu verin ordan.
<center><iframe width="560" height="315" src="https://www.youtube.com/embed/bqaEfyQy6UQ" frameborder="0" allowfullscreen></iframe></center>
