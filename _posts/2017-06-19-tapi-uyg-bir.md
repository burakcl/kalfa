---
layout: post
title: Uygulama Bir
categories: [Uygulama]
tags: [twitter, bash, shell, termux, tweetle]
url: https://burakcl.github.io/kalfa/uygulama/2017/06/19/tapi-uyg-bir.html
---
Sıra geldi tweetlemeye. Kabuk programlama ve birkaç servis sayesinde oluşturduğumuz listeden sırayla tweet atan bir betik yazalım. İlk başta `curl` kullanırım diye düşünürken bunun yerine twitter için yazılmış bir aracı kullanmanın daha iyi olacağını düşündüm. Ve duckduckgo ile aradım sonuçta `t cli` adlı kabukta çalışacak bir twitter app geliştirme uygulaması. Metin tabanlı çalışması istediğim özellikte internet farklı serileri var ama basit kullanışlı olmasını istiyorum çünkü betiği bilgisayarımdan ziyade telefonumda çalıştırmak istiyorum.

Bu iş için malzemeleri listeleyelim. Kabuğu yazmıyorum o zaten çalışma masamız.
* [t cli](http://sferik.github.io/t/)
* [curl](https://curl.haxx.se/)
* [termux](https://termux.com/)

`t cli` kurmak için ruby gerekiyor ruby kurduktan sonra `t` aracını kuracağız. Bunu kendi adresindeki kurulumda belirtiyor. Hem bilgisayara hem de termux ile telefona nasıl kurarız bunları anlatacağım. Kurulum yapacağım işletim sistemi `Ubuntu 16.04`.
{% highlight ruby %}
sudo apt-get install ruby-dev
sudo gem install t
{% endhighlight %}

Kurulum tamam twitter hesabını bağlamak için `t authorize` komutunu çalıştırıyoruz.
Bize yapmamız gerekenleri söylüyor sırasıyla

* twitter geliştirici sayfasına gidip giriş yapmak
* ardından yeni bir app oluşturmak
* oluşturduğumuz uygulamanın anahtarlarını girmek
* uygulama yetkilerini belirlemek
* twitter a giriş yapmak
* yönlendirdiği sayfa da kendi uygulamamıza izin vermek
* verilen pin kodunu girmek eşleşme sağlanıp `t` bağlaması tamamlanıyor.

Bu kısmı otomatik olarak yönlendirdiği için herhangi bir ekleme yapmayacağım. Tweet atıp testimizi yapalım, `t update metin` şeklinde tweet atabiliyoruz.

<div style="text-align:center"><img src ="{{ site.BASE_PATH }}/assets/media/pimg/tw-robot/tweet.gif" /></div>

### Betiğimizin çalışma planı
Tweet metinlerini liste halinde oluşturalım. Betik metinleri buradan çeksin. Liste yapmamın sebebi her seferinde sitedeki değişikliği kontrol edip en son değişikliği paylaştırmak uzun ve bence uzun bir işlem bunun yerine oluşturduğumuz metinleri ve adresleri saate göre tweetlesin. Saat olmasının sebebi sayac tutmak yerine saati olan tweetler olsun her şeyin vaktinin ve yerinin olduğu bir düzen şeklinde saatin kendisi gibi çalışssın.

### Yüzlerce anlamsız tweete gerek yok az olsun zamanı gelince robot işini yapsın yeterli.

`tweetle` adında bir fonksiyon yazalım.

{% highlight bash %}
  function tweetle(){
    if [[ $(t accounts) ]]; then
    t update "$1"
    else
    echo "Aktif hesap bulunamadı. Lütfen hesap açınız. (t authorize)"
    fi
  }
{% endhighlight %}

`t accounts` aktif bulunan hesapları listeliyor ayrıntılı bilgisi için `--help`. Hesap varsa tweetleyecek
yoksa da hesap açınız diyerek yönlendirsin.

Listeyi çekip tweet metnini alacak bir fonksiyon yazalım. Metin listesi isterseniz yerelde yapabilirsiniz isterseniz burada olduğu gibi github deposunada yükleyip oradan ulaşabilirsiniz. Depoda yapmamın amacı adımlardan ya da uygulamalardan yazı paylaştığım zaman o listeyi de bir satır güncelleyerek robot telefondaki liste içinde uğraşmamak.
{% highlight bash %}
function metin_al() {
  liste=$(curl -s https://raw.githubusercontent.com/burakcl/kalfa/master/assets/bot/$2)
  metin=$(echo $liste | cut -d ',' -f $1)
  if [[ -z $metin ]]; then
    echo "metin bulunmuyor."
    exit
  fi
}
{% endhighlight %}
`curl` ile üç farklı listeyi çekmek istiyorum bunun için parametre olarak geçiyorum liste ismini. Alınan listedeki her bir metni virgül ile ayırmıştım metinleri ayıklamak için '$1' ile kaçıncı metni göndereceğimizi belirliyoruz. Çıkış durumu olarak metnin olmaması halinde `metin bulunmuyor.` uyarısı ile işlemi sonlandırıyoruz. Saatler ve metin göstergesi konusundan devam edelim.
### Saatler
Betiğin saat 8-10 arasında 30 dk arayla `adimlar` adında oluşturacağım listeye gidip aldığı metni tweetlemesini istiyorum. Yine öğlen vakti 12 de 20 dk arayla 'uygulamalar' adlı listeden metin tweetlesin, akşam içinde 16-19 arası her saat başı ve 45 dk bir `kalfarep` adlı listeden tweet atmasını istiyorum.

Bu saat hesaplarını seçiyorum çünkü bir zamanlanmış görev tanımlayacağı `crontab` servisini kullanarak.
Şimdi gelelim kaç tweet atacağız bunun metin göstergesini nasıl yapacağız.

Sabah için> 8-30-9-30-10-30 şeklinde 6 adet
Ögle için> 12-20-40 şeklinde 3 adet
Akşam için> 16-19,-45 lerden 8 adet tweet atacağız.

8-8.30--> 1 ve 2 --> 1 ve 1.5
9-9.30--> 3 ve 4 --> 2 ve 2.5
10-10.30--> 5 ve 6 --> 3 ve 3.5
Saat 9 olduğundan gösterge değeri 3 olmalı 3.metin için. Aynı saat göstergesindeki gibi 8.30 için 2 değeri hesaplayacak bir method oluşturalım.
Saati alalım. 8 -> değeri 1
Dakikayı alalım. 30 -> değeri 0.5
saat 8 için 1 inci 8.30 için 2 nci metni atmam gerekiyor. Birincide sorun yok İkincide bir sorun var elimizde 1.5 var bunu ikiye çevirmeliyiz. Neden case ya da if elif yazmıyorum çünkü zamanını sayısını değiştirmeyi planladığım liste uzandığında elimde kullanışlı bir method olsun istiyorum.

Biraz düşündükten sonra şimdiki(8) saatten başlangıç saatinin bir eksiğini çıkartarak bir puanı elde edelim. Yarım saat içinde şart bırakalım yarım puan olup olmayacağına karar versin. 0 sa 0 puan 30 ise 0.5 puan.

Bir kez daha yapalım hadi saat 9.30 ve 10 için,
* = 9 - 7
* = 2 Puan
* 30 dakika var mı? E +0.5 Puan >> 2.5 Puan
* = 10 - 7
* = 3 Puan
* 30 dakika var mı? H +0 Puan >> 3 Puan

Bir adım daha kaldı yukarıda sayılara uyarlamak gerekiyor yani 1, 2, 3, 4, 5, 6 şeklinde olmasını gerekiyor. Bunun için 2 ile çarpıp bir çıkardık ve istediğimiz sonucu elde ettik.
Yani saat 10 için 5 numaralı metni bulmamız gerekiyordu, 3 * 2 - 1 = 5 yaptık diğerleri içinde deneyebilirsin. Fonksiyonumuzu yazalım.
{% highlight bash %}
function sabahkusu() {
  [ -n $(date +%M) ] && c=0.5
  gosterge=$(echo "($simdi+$c-7)*2-1" | bc | cut -d '.' -f1)
  dosya=adimlar
}
{% endhighlight %}
Önce 0 farklı mı ona bakalım farklıysa `c` adındaki değişken yarım puan olsun. `date +%M` ile dakikayı sorguladık. Sonra `gosterge` puanını hesaplayalım. Fonksiyon adımızı `sabahkusu` yaptık bu sebepten başlangıç saatinin bir eksiği olan 7 yi kullanacağız. Şimdiden 7 yi çıkartıp `c` puanını ekliyoruz. Ardından 2 ile çarpıp bir ekliyoruz. Yukarıda bu işlemleri echo kullanarak `bc` ye gönderdik. Nedir bu bc dersek ondalık sayılarla kabukta doğrudan hesap yapamıyoruz. Bunun için ayrıca araç kullanmamız gerekiyor `awk`, `bc`, `python`, `perl`, `ruby` ... kullanabiliriz en ufağı bc olduğu için onu seçtim. `bc` ile puanı hesapladık ama çıkacak sayı tam sayıda olsa ondalık ifade şeklinde çıktı veriyor bunuda `cut` ile temizledik.
`dosya` adında da bir değişken oluşturduk saate özel tweetleyeceğiz ya hani. Şimdi gelin akşam ve öğle için fonksiyonları da oluşturalım ve tamamlayalım.
{% highlight bash %}
function ogle() {
  [ -n $(date +%M) ] && gosterge=2
  [ 40 -le $(date +%M) ] && gosterge=3
  dosya=uygulamalar
}
function aksamvakti() {
  [ -n $(date +%M) ] && c=0.5
  gosterge=$(echo "($simdi+$c-15)*2-1" | bc | cut -d '.' -f1)
  dosya=kalfarep
}
{% endhighlight %}
`ogle` fonksiyonunda gostergenin değerini belirlemek için 0 dan farklı mı kontrolü yaptık öyleyse göstergeyi 2 yaptık, daha sonra 40 veya 40 dan büyükse gostergeyi 3 yaptık böylece 3 metni seçebileceğiz. `dosya` değerini verdik.

`aksamvakti` fonksiyonu da yukarıda anlattığım `sabahkusu` fonksiyonunun aynısı.

Betiğin ana kısmını yazıp tamamlayalım. Şimdi yukarıda dikkatinizi çektiyse `c` ve `gosterge` puanının ilk değerlerini atamamıştık onları burada atayacağız. Daha sonra şimdi adında belirlediğimiz değişkene şimdiki zamanı sayı olarak atayacağız.

`case` ile `simdi` nin işaret ettiği fonksiyonu çağırıp, sonuçta gelen değerler ile metin alıp tweetleyeceğiz.

{% highlight bash %}
c=0;
gosterge=1;
simdi=$(($(date +%k)+0));
case $simdi in
  8|9|10) sabahkusu;;
  12) ogle;;
  16|17|18|19) aksamvakti;;
  *) exit;;
esac

metin_al $gosterge $dosya
tweetle "$metin"
{% endhighlight %}

Artık tweetleyebiliriz tabi uygun saatte.
### Crontab ayarlarımızı yapalım.
Crontab zamanlanmış görev servisidir, tanımladığım zaman dilimlerinde tanımladığım işi yap diyebileceğiniz bir servis. Bizde zamanları ayarlarak crontab servisini aktif yapalım.
{% highlight bash %}
*/30 8-10 * * 2,5 ~/Development/jack/shell-kalfa/tweetle-kalfa
*/20 12 * * 1,4 ~/Development/jack/shell-kalfa/tweetle-kalfa
*/45 16-19 * * 3,6 ~/Development/jack/shell-kalfa/tweetle-kalfa
{% endhighlight %}
Haftanın 2 ve 5'inci günleri saat 8-10 arası her yarım saatte betiği çalıştırmasını istedik `sabahkusu` fonksiyonu için, haftanın 1 ve 4'üncü günleri saat 12'de 20 dk ara ile yine betiği çalıştırsın `ogle` fonksiyonu için diğer kalan satır ise `aksamvakti` fonksiyonu için haftanın 3 ve 6'ncı 16-19 arası her saat başı ve her 45 dk da bir tweetlesin olarak yazdık.

`crontab -e` ile kullanıcı adımıza açılan dosyaya bu satırları yazdık. `cron`  servisini yeniden başlatıyoruz.
{% highlight bash %}
sudo service cron restart
{% endhighlight %}
Zamanı geldiğinde çalışacaktır betiğimizi tekrar bütün halinde yazacak olursak,
{% highlight bash %}
#!/bin/bash
function tweetle(){
  if [[ $(t accounts) ]]; then
  t update "$1"
  else
  echo -en "Aktif hesap bulunamadı. Lütfen hesap açınız. (t authorize)\n"
  fi
}
function metin_al() {
  liste=$(curl -s https://raw.githubusercontent.com/burakcl/kalfa/master/assets/bot/$2)
  metin=$(echo $liste | cut -d ',' -f $1)
  if [[ -z $metin ]]; then
    echo "metin bulunmuyor."
    exit
  fi
}
function sabahkusu() {
  [ -n $(date +%M) ] && c=0.5
  gosterge=$(echo "($simdi+$c-7)*2-1" | bc | cut -d '.' -f1)
  dosya=adimlar
}
function ogle() {
  [ -n $(date +%M) ] && gosterge=2
  [ 40 -lt $(date +%M) ] && gosterge=3
  dosya=uygulamalar
}
function aksamvakti() {
  [ -n $(date +%M) ] && c=0.5
  gosterge=$(echo "($simdi+$c-15)*2-1" | bc | cut -d '.' -f1)
  dosya=kalfarep
}
c=0;
gosterge=1;
simdi=$(($(date +%k)+0));
case $simdi in
  8|9|10) sabahkusu;;
  12) ogle;;
  16|17|18|19) aksamvakti;;
  *) exit;;
esac

metin_al $gosterge $dosya
tweetle "$metin"
{% endhighlight %}

### Termux a aktaralım ve telefonumuzda çalıştıralım.

Araç takımını kuralım.

* coreutils, curl ve bc
* ruby-dev
* gem install t
* vim

Kurulumu tamamladıktan sonra `t` yetkilendirme işlemlerini sağlıyorsunuz. Bu işlemleri yaparken tarayıcı ile masaüstü isterseniz daha sağlıklı istediği adımları gerçekleştirirsiniz.

> vim kullanırken esc tuşu lazım olacak hemen o paneli de aktif edelim de öyle kalmayalım. Ses artırma tuşuna basılı tutarken Q yapalım 6 tuş daha gelecektir.

`crontab` ayarı için,
{% highlight bash %}
cd ~/../usr/var/spool/cron/crontabs
vim $(whoami)
{% endhighlight %}

`crontab`a yazdığımız kuralları ekliyoruz sadece adresleri kontrol ediyoruz. Termux `~/bot/tweetle-kalfa` şeklinde. Kaydedip kapatıyoruz.

Burada sorunlardan biride  betikteki kabuk adresini değiştirmemiz gerekiyor.
{% highlight bash %}
#!/data/data/com.termux/files/usr/bin/bash
{% endhighlight %}
Telefona attığımız dosyayı termuxa aktarmak için dosyayı taşı sorulan adreste termux seçeneğini seç ya da birlikte aç seçeneği sağlayan bir dosya yöneticisi ile termuxu seçmek yeterli olacaktır.
Ek olarak dosyaya çalıştırma iznini `crontab`ında çalıştırabileceği şekilde (a+x) verelim.
### Bitti. Artık telefon bu işi istediğiniz saatlerde yapacaktır.
### Uygulama bir hatalar
Hiç test etmeden bıraktık kullandıkça çıkan sorun ve ürettiğimiz çözümleri buradan aşağıya not etmek istedim.

#### Hata 0: Tweet metin ayracı
Virgül kullanmıştık fakat cümle içerisindeki virgülü unuttuğumuzdan dolayı tweet metninin eksik oluşmasına yol açtı. Virgül yerine noktalı virgül ile değiştirdim.

#### Hata 1: Zaman ve Gösterge hatası
Bazen saat bir iki dk geçikme göstermekte en azından bu termux üzerinde karşılaştığım bir durum. Buna çözüm olarak tweetleri tam ayrım noktalarından yani tam ortada zamandan gösterge ayarı yapmak daha iyi bir çözüm oldu. Sonuç olarak fonksiyonların en son hali aşağıdadır.
Örnek, Sabah fonksiyon göstergesini dakikanın 0 dan farklı olup olmadığı kontrolünü 30 dan büyük mü diye değiştirdim. Bu sayede her tweeti atmak için 30 dk vaktim olmuş oldu.

{% highlight bash %}

function sabahkusu() {
  [ 30 -lt $(date +%M) ] && c=0.5
  gosterge=$(echo "($simdi+$c-7)*2-1" | bc | cut -d '.' -f1)
  dosya=adimlar
}
function ogle() {
  [ 20 -lt $(date +%M) ] && gosterge=2
  [ 40 -lt $(date +%M) ] && gosterge=3
  dosya=uygulamalar
}
function aksamvakti() {
  [ 45 -lt $(date +%M) ] && c=0.5
  gosterge=$(echo "($simdi+$c-15)*2-1" | bc | cut -d '.' -f1)
  dosya=kalfarep
}
{% endhighlight %}
