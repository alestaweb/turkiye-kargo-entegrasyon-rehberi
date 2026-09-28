# Kargo Entegrasyonu — E-Ticaret Yazılımı Tarafı Rehberi

Bir e-ticaret sitesinde "kargoya verildi" yazısı tek satırdır. Arkasında ise bir durum
makinesi, bir adres doğrulama problemi, bir etiket yazdırma zinciri, bir para akışı
(kapıda ödeme) ve bir iade süreci vardır. Kargo firmalarının kendi entegrasyon belgeleri
**kendi servislerini** anlatır. Bu belgelerin anlatmadığı şey, bunların sipariş sisteminin
içinde nasıl bir araya geleceğidir.

Bu depo o boşluğu kapatmak için yazıldı. Sipariş ile gönderi arasındaki ilişki, takip
numarasının yaşam döngüsü, durum eşleme, desi, çoklu paket, kapıda ödeme, iade ve
sessizce yanlış giden işler anlatılıyor.

**Kod yok, kütüphane yok, firma önerisi yok.** Veri modeli soruları, sıra kritik akışlar,
karar tabloları ve bir değerlendirme kontrol listesi var. Anlatılanlar belirli bir kargo
firmasına bağlı değildir; her entegrasyonda aynı sorular çıkar.

Hazırlayan: [Alesta WEB](https://alestaweb.com) — 2004'ten beri haber yazılımı ve
e-ticaret yazılımı geliştiriyor.

---

## İçindekiler

1. Entegrasyon neyi çözer, neyi çözmez
2. Sipariş ve gönderi ayrı kayıtlardır
3. Gönderi numarası, takip numarası, barkod
4. Gönderi yaşam döngüsü ve durum eşleme
5. Adres: en çok hata buradan çıkar
6. Desi ve ücret hesabı
7. Çoklu paket ve kısmi gönderim
8. Etiket yazdırma
9. Takip bilgisini almak: sorgulama mı, bildirim mi
10. Kapıda ödeme
11. İade ve değişim
12. Müşteriye bildirim
13. Kişisel veri: alıcı bilgisi kargo firmasına gider
14. Fatura, irsaliye ve kargo
15. Birden fazla kargo firması
16. Sessizce yanlış giden 12 şey
17. Test ortamı ve canlıya geçiş
18. 20 maddelik değerlendirme kontrol listesi
19. İlgili rehberler ve resmî kaynaklar

---

## 1. Entegrasyon neyi çözer, neyi çözmez

Kargo entegrasyonu dendiğinde genellikle üç iş kastedilir:

| İş | Ne olur | Entegrasyon olmadan |
|---|---|---|
| **Gönderi oluşturma** | Sipariş bilgisi kargo firmasına iletilir, gönderi kaydı ve barkod alınır | Şubeye gidilir, bilgiler elle yazılır |
| **Etiket** | Barkodlu gönderi etiketi yazdırılıp pakete yapıştırılır | Elle yazılan gönderi formu |
| **Takip** | Gönderinin hareketleri sipariş ekranına ve müşteriye yansır | Takip numarası müşteriye elle iletilir, müşteri kendisi bakar |

Entegrasyonun **çözmediği** şeyler de vardır ve bunları bilmek beklentiyi doğru kurar:

- Kargo firmasının kendi operasyonel gecikmeleri yazılımla düzelmez.
- Yanlış adres, entegrasyonla doğru adrese dönüşmez; sadece daha hızlı yanlış gider.
- Takip bilgisi kargo firmasının sistemine düştüğü anda değil, **sizin onu sorguladığınız
  veya size bildirildiği anda** sizde görünür. Arada her zaman bir gecikme vardır.

---

## 2. Sipariş ve gönderi ayrı kayıtlardır

En sık yapılan tasarım hatası, takip numarasını sipariş tablosuna tek bir alan olarak
eklemektir. Bu, ilk gün çalışır; ilk çoklu paket, ilk kısmi gönderim veya ilk değişim
talebinde çöker.

Doğru model: **bir siparişin sıfır, bir veya birden fazla gönderisi olabilir.** Her
gönderinin kendi kargo firması, kendi takip numarası, kendi durumu ve kendi hareket
geçmişi vardır.

| Kayıt | Sorumluluğu |
|---|---|
| **Sipariş** | Ne satıldı, kime, hangi fiyattan, ödeme durumu |
| **Gönderi** | Hangi ürünler, hangi pakette, hangi firmayla, hangi takip numarasıyla gitti |
| **Gönderi kalemi** | Gönderinin içindeki ürün ve adet (kısmi gönderim için şart) |
| **Gönderi hareketi** | Kargo firmasından gelen her durum değişikliği, zaman damgasıyla |

Sipariş durumu ("kargoda", "teslim edildi") bu gönderilerin durumundan **türetilir**,
elle ayrıca tutulmaz. Üç gönderiden ikisi teslim edildiyse sipariş "teslim edildi"
değildir, "kısmen teslim edildi"dir.

İade gönderisi de bir gönderidir; yönü tersine çevrilmiş olarak aynı yapıda tutulur.

---

## 3. Gönderi numarası, takip numarası, barkod

Bu üç kavram firmaya göre farklı adlarla çağrılır ve sıkça karıştırılır:

- **Sizin referansınız** — gönderiyi oluştururken kargo firmasına verdiğiniz, sizin
  sisteminizdeki benzersiz kimlik. Genellikle sipariş numarası veya ondan türetilen bir değer.
- **Kargo firmasının gönderi kimliği** — firmanın kendi sisteminde gönderiye verdiği numara.
- **Takip numarası** — müşterinin firma sitesinde sorgulayabileceği numara.
- **Barkod** — etikette basılı olan, şubede ve transfer merkezinde okutulan değer.

Bazı firmalarda bunların ikisi veya üçü aynı değerdir, bazılarında hepsi farklıdır.
Yazılımın her birini **ayrı alan** olarak tutması, firma değiştirildiğinde veya ikinci
bir firma eklendiğinde veri modelinin yeniden yazılmasını önler.

Kritik nokta: **sizin referansınız benzersiz olmalı ve gönderi oluşturma isteği tekrar
edildiğinde aynı gönderiyi döndürmelidir.** Ağ zaman aşımına uğrayan bir istek tekrar
gönderildiğinde iki ayrı gönderi ve iki ayrı barkod oluşuyorsa, depoda iki etiket basılır
ve biri boşuna faturalanır.

---

## 4. Gönderi yaşam döngüsü ve durum eşleme

Her kargo firmasının kendi durum kodları vardır; sayıları birkaç taneden birkaç düzineye
kadar değişir. Bunları olduğu gibi müşteriye göstermek hem tutarsızdır hem de firma
değiştiğinde her şeyi bozar.

Doğru yaklaşım, **kendi sabit durum kümenizi** tanımlayıp her firmanın kodlarını ona
eşlemektir:

| Sizin durumunuz | Anlamı |
|---|---|
| Hazırlanıyor | Gönderi kaydı oluştu, paket henüz kargoya teslim edilmedi |
| Kargoya verildi | Paket firma tarafından teslim alındı (ilk okutma) |
| Yolda | Transfer merkezleri arasında |
| Dağıtımda | Teslimat şubesinden kuryeye çıktı |
| Teslim edildi | Alıcıya teslim edildi |
| Teslim edilemedi | Adreste bulunamadı, adres hatalı, alıcı reddetti vb. |
| Şubede bekliyor | Alıcının şubeden alması bekleniyor |
| İade yolunda | Gönderici adresine geri dönüyor |
| İade teslim edildi | Göndericiye geri ulaştı |
| İptal | Kargoya teslim edilmeden iptal edildi |

Eşleme tablosunda bilinmeyen bir kod geldiğinde ne olacağı **önceden** karar verilmelidir.
Önerilen: son bilinen durum korunur, ham kod hareket geçmişine yazılır, yöneticiye uyarı
düşer. Bilinmeyen kodu sessizce "yolda" saymak, teslim edilemeyen bir paketin günlerce
fark edilmemesine yol açar.

### Durum geriye gidebilir

"Dağıtımda"dan sonra "teslim edilemedi", ardından tekrar "dağıtımda" gelmesi normaldir.
Durumun yalnızca ileri gidebileceğini varsayan kod bu senaryoda ya hata verir ya da
yanlış durumda takılı kalır. Durum, **en son zaman damgalı hareketten** hesaplanmalıdır;
hareketlerin geliş sırasından değil.

### Teslim edildi son durum değildir

Teslim edildi olarak işaretlenen bir gönderi için sonradan iade süreci başlayabilir.
"Teslim edildi" durumu siparişi kapatır ama gönderi kaydını kilitlememelidir.

---

## 5. Adres: en çok hata buradan çıkar

Teslim edilemeyen gönderilerin önemli bir kısmının nedeni adrestir ve bu hatanın maliyeti
iki yönlü kargo ücretidir.

Adres formunda dikkat edilecekler:

- **İl ve ilçe serbest metin olmamalı**, listeden seçilmelidir. Kargo firmaları gönderiyi
  il/ilçe bilgisine göre şubeye yönlendirir; "Kadiköy", "Kadıköy", "KADIKÖY" farklı
  yazımları eşleşmeyebilir.
- **Mahalle** mümkünse listeden seçilmelidir. Aynı ilçede benzer adlı mahalleler olabilir.
- **Açık adres** alanı serbesttir ama bir alt ve üst uzunluk sınırı olmalıdır. Kargo
  firmalarının adres alanına karakter sınırı koyduğunu unutmayın; uzun adres kesilirse
  kapı numarası kaybolabilir.
- **Telefon** tek bir biçimde saklanmalıdır (ör. başında ülke kodu veya 0 olmadan 10 hane).
  Kurye alıcıya ulaşamadığında telefon tek kurtarıcıdır.
- **Alıcı adı** sipariş verenden farklı olabilir; hediye gönderimleri için ayrı alan gerekir.
- Kurumsal teslimatta **firma adı** ayrı bir alan olmalıdır; bina içinde doğru kişiye
  ulaşmayı sağlar.

Gönderi oluşturulduktan **sonra** adres değişikliği talebi gelirse, çoğu durumda mevcut
gönderi iptal edilip yenisi oluşturulur. Bunun için gönderinin henüz kargoya teslim
edilmemiş olması gerekir. Yazılım, adres değişikliği düğmesini gönderi durumuna göre
açıp kapatmalıdır.

---

## 6. Desi ve ücret hesabı

Kargo ücreti ağırlık ile hacimsel ağırlığın (**desi**) büyük olanına göre hesaplanır.

Desi, paketin santimetre cinsinden **en × boy × yükseklik** çarpımının bir bölene
bölünmesiyle bulunur. Türkiye'de yurt içi taşımada yaygın bölen **3000**'dir; ancak
bölen ve yuvarlama kuralı (yukarı tam sayıya, yarıma vb.) **firma ve sözleşmeye göre
değişebilir**, kendi sözleşmenizden teyit edin.

Yazılım tarafında sorun şudur: **ürün başına desi toplanamaz.** İki küçük kutu tek bir
büyük kutuya konduğunda desi, iki ayrı kutunun desi toplamından farklıdır. Seçenekler:

| Yaklaşım | Artısı | Eksisi |
|---|---|---|
| Ürün desi toplamı | Basit | Birleştirilen paketlerde fazla ücret gösterir |
| Standart koli ölçüleri | Depo gerçeğine yakın | Hangi ürünün hangi koliye sığdığı tanımlanmalı |
| Paketleme anında ölçüm | En doğru | Sipariş anında kesin ücret gösterilemez |

Müşteriye sipariş anında gösterilen kargo ücreti ile firmanın size fatura ettiği ücret
arasındaki fark, zamanla görünmez bir zarar kalemine dönüşür. Aylık olarak **tahmini
ve gerçekleşen desi** karşılaştırması raporlanmalıdır.

### Ücretsiz kargo eşiği

"X TL üzeri kargo bedava" kuralında eşiğin hangi tutara uygulandığı net olmalıdır:
indirim öncesi mi, indirim sonrası mı, KDV dahil mi? İade sonrası tutar eşiğin altına
düşerse ne olacağı da önceden yazılmalıdır.

---

## 7. Çoklu paket ve kısmi gönderim

**Çoklu paket:** Tek sipariş fiziksel olarak birden fazla koliye bölünür ama aynı anda
gönderilir. Bazı firmalar bunu tek gönderi altında birden fazla parça olarak kabul eder
(her parçanın kendi barkodu olur), bazıları her koli için ayrı gönderi ister.

**Kısmi gönderim:** Siparişin bir kısmı stokta vardır ve önden gönderilir, kalanı sonra
gider. Bu, iki ayrı gönderidir.

Her iki durumda yazılımın cevaplaması gereken sorular:

- Hangi ürün hangi kolide / gönderide? (gönderi kalemi kaydı)
- Kargo ücreti ikinci gönderi için tekrar alınacak mı?
- Müşteriye iki ayrı takip numarası nasıl gösterilecek?
- Kapıda ödemeli siparişte tahsilat hangi pakette yapılacak? (bkz. bölüm 10)
- Bir paket teslim edilip diğeri edilemezse sipariş hangi durumda?

---

## 8. Etiket yazdırma

Gönderi oluşturulduğunda firma genellikle bir etiket verir: PDF, görüntü dosyası veya
termal yazıcı dili (ör. ZPL) biçiminde.

- **Termal etiket yazıcı** depo için en hızlı yoldur. Etiket boyutu (yaygın olarak
  100×100 mm veya 100×150 mm) firmanın verdiği biçimle ve yazıcıdaki rulo ile uyuşmalıdır.
  PDF'in A4'e basılıp kesilmesi, düşük hacimde işe yarar ama hızlanamaz.
- Etiket dosyası **saklanmalıdır.** Yeniden yazdırma ihtiyacı (etiket yırtıldı, yazıcı
  sıkıştı) için tekrar gönderi oluşturmak yerine saklanan etiket basılır.
- Toplu yazdırmada etiketlerin **sipariş sırasına göre** basılması, paketleme sırasında
  karışıklığı önler. Her etikette sizin referansınızın da okunabilir olması iyi olur.
- Barkod okunaklılığı, yazıcı ısı ve hız ayarına bağlıdır. İlk kurulumda birkaç etiket
  şubede okutularak test edilmelidir.

---

## 9. Takip bilgisini almak: sorgulama mı, bildirim mi

Gönderi hareketlerini almanın iki yolu vardır:

| Yöntem | Nasıl çalışır | Dikkat |
|---|---|---|
| **Sorgulama (polling)** | Yazılım belirli aralıklarla açık gönderilerin durumunu sorar | İstek sınırı, gereksiz yük, gecikme |
| **Bildirim (webhook)** | Firma, durum değiştiğinde sizin adresinize bildirim gönderir | Adres erişilebilir olmalı, bildirim doğrulanmalı, kayıp bildirim telafi edilmeli |

Pratikte ikisi birlikte kullanılır: bildirim birincil yoldur, sorgulama **kaçan
bildirimleri** yakalamak için yedek görev görür.

Sorgulama kurallarının örneği:

- Yalnızca **açık** gönderiler sorgulanır (teslim edildi / iade teslim edildi / iptal
  olanlar hariç).
- Sorgulama sıklığı gönderi yaşına göre azaltılır; yeni gönderi sık, eski gönderi seyrek.
- Belirli bir süreden uzun süredir hareket görmeyen gönderi **"takılı"** olarak işaretlenir
  ve yöneticiye listelenir. Kaybolan paketler en çok böyle fark edilir.

Bildirim alan uç noktada:

- Gelen bildirimin gerçekten ilgili firmadan geldiği doğrulanmalıdır (imza, gizli anahtar
  veya firmanın sunduğu yöntem).
- Aynı bildirim birden fazla kez gelebilir; işleme **tekrar güvenli** olmalıdır.
- Bildirimler sırası karışık gelebilir; durum hareket zaman damgasından hesaplanır
  (bkz. bölüm 4).
- Uç nokta hızlı yanıt vermeli, ağır işi (e-posta, SMS, stok) sonraya bırakmalıdır.

### Saat dilimi

Kargo firmasından gelen zaman damgasının hangi saat diliminde olduğu açıkça
belirtilmemişse, ilk entegrasyonda bir gönderinin gerçek teslim saati ile karşılaştırarak
teyit edin. Saat dilimi hatası, "teslim edildi" hareketinin "dağıtımda" hareketinden
**önce** görünmesine ve durum hesabının bozulmasına yol açar.

---

## 10. Kapıda ödeme

Kapıda ödemede ürün bedeli kurye tarafından alıcıdan tahsil edilir ve sözleşmedeki
süre içinde satıcıya aktarılır. Bu, kargo entegrasyonunu bir **para akışına** dönüştürür.

- Gönderi oluşturulurken **tahsil edilecek tutar** ve **tahsilat türü** (nakit, kart)
  firmaya doğru iletilmelidir. Tutarın sonradan değişmesi (ör. müşteri bir ürünü iptal
  etti) çoğu durumda gönderinin yeniden oluşturulmasını gerektirir.
- Çoklu paketli siparişte tahsilatın **tek pakette** yapılması gerekir; tutar paketlere
  bölünmüşse biri teslim edilip diğeri edilemediğinde tahsilat kısmi kalır.
- Kapıda ödeme için ek hizmet bedeli alınıyorsa, bu bedel sipariş öncesinde müşteriye
  açıkça gösterilmelidir.
- **Mutabakat**: Firmanın aktardığı tahsilat tutarları siparişlerle eşleştirilmelidir.
  Teslim edildi görünen ama tahsilatı aktarılmayan siparişler ayrı bir raporda
  listelenmelidir. Bu rapor yoksa kayıp aylar sonra fark edilir.
- Teslim edilemeyip geri dönen kapıda ödemeli gönderide para akışı yoktur ama **iki yönlü
  kargo masrafı** vardır. Aynı alıcıdan tekrarlanan geri dönüşler takip edilmelidir.

---

## 11. İade ve değişim

Mesafeli satışlarda tüketicinin **cayma hakkı** vardır ve iade gönderisi e-ticaret
yazılımının kargo tarafındaki en karmaşık akışıdır.

Mevzuatın ayrıntısı (cayma süresi, iade masrafının kime ait olduğu, geri ödeme süresi)
**Mesafeli Sözleşmeler Yönetmeliği**nde düzenlenir. Yazılım açısından kritik olan şudur:
**ön bilgilendirme metninde ve mesafeli satış sözleşmesinde iade için ne yazdıysanız,
iade akışı birebir onu uygulamalıdır.** Metinde anlaşmalı bir taşıyıcı ve iade yöntemi
belirttiyseniz, yazılım müşteriye o yöntemi sunmalıdır.

Yaygın iade yöntemleri:

| Yöntem | Akış |
|---|---|
| **İade kodu** | Yazılım firmadan bir iade kodu alır, müşteri paketi şubeye bu kodla bırakır |
| **Adresten alım** | Yazılım firmadan kurye talebi oluşturur, paket adresten alınır |
| **Müşterinin kendi gönderimi** | Müşteri kendisi gönderir, takip numarasını sisteme girer |

İade akışında yazılımın tutması gerekenler:

- İade talebi, hangi ürünler, hangi adet, gerekçe
- İade gönderisi (bölüm 2'deki yapı, ters yönlü)
- Paketin depoya ulaştığı tarih (geri ödeme süresi buna bağlı olabilir)
- Kontrol sonucu (sağlam / hasarlı / eksik)
- Geri ödeme kaydı

**Değişim** iki gönderidir: gelen iade ve giden yeni ürün. İkisi ayrı gönderi kaydı
olarak tutulur, aynı talebe bağlanır.

---

## 12. Müşteriye bildirim

Gönderi durumu değiştiğinde müşteriye e-posta veya SMS gönderilmesi yaygındır. Dikkat
edilecekler:

- **Her hareket bildirim gerektirmez.** Transfer merkezleri arası her okutmada SMS
  göndermek müşteriyi rahatsız eder ve maliyet yaratır. Önerilen: kargoya verildi,
  dağıtımda, teslim edildi, teslim edilemedi.
- Aynı durum için **tekrar bildirim gönderilmemelidir** (bkz. bölüm 9, tekrarlanan
  bildirimler).
- Sipariş ve teslimatla ilgili bilgilendirme mesajları, **ticari elektronik ileti
  değildir**; içine kampanya veya indirim eklendiği anda ticari iletiye dönüşür ve izin
  gerektirir. Ayrıntı için İYS rehberine bakın (bölüm 19).
- Bildirimdeki takip bağlantısı, müşteriyi mümkünse **sizin sitenizdeki** sipariş takip
  sayfasına götürmelidir; hem tutarlı bir deneyim sunar hem de firma değiştiğinde
  bağlantılar kırılmaz.

---

## 13. Kişisel veri: alıcı bilgisi kargo firmasına gider

Gönderi oluşturmak, alıcının adını, adresini ve telefonunu kargo firmasına **aktarmak**
demektir. Bu bir kişisel veri aktarımıdır.

- Aydınlatma metninde, teslimat amacıyla kargo firmalarına veri aktarıldığı belirtilmelidir.
- Firmaya **yalnızca teslimat için gereken** veri gönderilmelidir. E-posta adresi, doğum
  tarihi, sipariş geçmişi gibi alanların gönderi isteğine eklenmesi gereksizdir.
- Etiket dosyaları ve firmadan gelen yanıtlar kişisel veri içerir; saklama süresi ve
  erişim yetkisi buna göre belirlenmelidir.
- Entegrasyon için kullanılan firma API anahtarları, kaynak kodunun içine değil yapılandırma
  alanına veya gizli değişkenlere konmalıdır.

---

## 14. Fatura, irsaliye ve kargo

Kargo takip numarası mali bir belge değildir. Malın taşınmasına ilişkin belge düzeni
(fatura, sevk irsaliyesi, e-İrsaliye) **Vergi Usul Kanunu** ve ilgili tebliğlerle
belirlenir; takip numarası bu belgelerin yerini tutmaz.

Yazılım tarafında dikkat edilecek nokta, **kısmi gönderim** ile belge düzeninin uyumudur:
tek fatura kesilip iki ayrı gönderi yapıldığında veya iade sonrası kısmi iade belgesi
gerektiğinde, hangi belgenin hangi gönderiyle ilişkili olduğu kayıt altında olmalıdır.
Bu konunun ayrıntısı e-Fatura / e-Arşiv / e-İrsaliye rehberinde (bölüm 19) anlatılıyor.

---

## 15. Birden fazla kargo firması

Büyüyen mağazalar genellikle ikinci, bazen üçüncü bir kargo firmasıyla çalışmaya başlar:
bölgeye göre, ürün boyutuna göre veya maliyete göre.

Buna hazırlıklı veri modeli:

- Gönderi kaydında **firma** bir alandır; takip numarası tek başına benzersiz kabul edilmez,
  **firma + takip numarası** birlikte benzersizdir.
- Durum eşleme (bölüm 4) firma başına ayrı tablodur, sizin durum kümeniz ortaktır.
- Firma seçimi kuralları (bölge, desi, ürün kategorisi, ödeme türü) yazılımda
  tanımlanabilir olmalıdır; kod değişikliği gerektirmemelidir.
- Bir firmanın servisi çalışmadığında gönderi oluşturmanın **başka firmaya yönlendirilmesi**
  veya kuyruğa alınıp sonra denenmesi karar verilmiş bir davranış olmalıdır.

---

## 16. Sessizce yanlış giden 12 şey

1. **Zaman aşımında tekrarlanan istek iki gönderi oluşturur.** Referans benzersizliği ve
   tekrar güvenli istek (bölüm 3) yoksa depo iki etiket basar.
2. **Bilinmeyen durum kodu "yolda" sayılır.** Teslim edilemeyen paket günlerce fark edilmez.
3. **Durum sadece ileri gidebilir varsayımı.** "Teslim edilemedi"den sonra tekrar dağıtıma
   çıkan paket yanlış durumda takılır.
4. **Takip numarası sipariş tablosunda tek alan.** İlk çoklu paket veya değişimde veri kaybolur.
5. **İl/ilçe serbest metin.** Aynı yerin farklı yazımları firma tarafında eşleşmez.
6. **Adres alanı kesilir.** Firma karakter sınırını aşan adres sessizce kırpılır, kapı
   numarası gider.
7. **Desi ürün başına toplanır.** Gösterilen ücret ile faturalanan ücret sürekli farklıdır,
   kimse raporlamaz.
8. **Kapıda ödeme tutarı paketlere bölünür.** Bir paket geri dönünce tahsilat eksik kalır.
9. **Tahsilat mutabakatı yapılmaz.** Aktarılmayan tutarlar aylar sonra fark edilir.
10. **Aynı bildirim iki kez işlenir.** Müşteriye iki "teslim edildi" SMS'i gider.
11. **Saat dilimi karışır.** Hareket sırası bozulur, durum hesabı yanlış çıkar.
12. **Takılı gönderi raporu yoktur.** Kayıp paket müşteri şikâyet edene kadar görünmez.

---

## 17. Test ortamı ve canlıya geçiş

- Firmaların çoğu ayrı bir **test ortamı** ve test hesabı sunar. Canlı hesapla test
  gönderisi oluşturmak, iptal edilmezse faturalanabilir.
- Test ortamında durum hareketleri genellikle gerçekleşmez; durum eşleme ve bildirim
  işleme **örnek hareket verisiyle** ayrıca test edilmelidir.
- Canlıya geçişte ilk günlerde düşük hacimle başlayıp birkaç gönderiyi uçtan uca izlemek
  (etiket şubede okundu mu, hareketler geliyor mu, teslim durumu doğru mu) sürprizleri önler.
- Test ve canlı ortam anahtarları **ayrı** saklanmalı, yanlışlıkla karışmamalıdır.
- Entegrasyon hataları (reddedilen gönderi, geçersiz adres yanıtı) kayıt altına alınmalı
  ve yöneticinin göreceği bir listede toplanmalıdır. Sadece günlük dosyasına yazılan hata
  kimse tarafından okunmaz.

---

## 18. 20 maddelik değerlendirme kontrol listesi

Bir e-ticaret yazılımının kargo tarafını değerlendirirken sorulacak sorular:

**Veri modeli**
- [ ] Bir siparişin birden fazla gönderisi olabiliyor mu?
- [ ] Gönderi kalemi (hangi ürün hangi gönderide) tutuluyor mu?
- [ ] Gönderi hareketleri zaman damgasıyla ayrı kayıt olarak saklanıyor mu?
- [ ] Firma + takip numarası birlikte benzersiz mi?

**Gönderi oluşturma**
- [ ] Tekrarlanan istek ikinci bir gönderi oluşturmuyor mu?
- [ ] Adres formunda il, ilçe ve mahalle listeden mi seçiliyor?
- [ ] Adres uzunluğu firma sınırına göre denetleniyor mu?
- [ ] Gönderi, kargoya verilmeden önce iptal edilebiliyor mu?

**Etiket ve depo**
- [ ] Termal etiket biçimi destekleniyor mu?
- [ ] Etiket yeniden yazdırılabiliyor mu (gönderi yeniden oluşturmadan)?
- [ ] Toplu yazdırma sipariş sırasına göre yapılıyor mu?

**Takip**
- [ ] Firma durum kodları sabit bir durum kümesine eşleniyor mu?
- [ ] Bilinmeyen durum kodu yöneticiye bildiriliyor mu?
- [ ] Bildirim + yedek sorgulama birlikte çalışıyor mu?
- [ ] Takılı gönderi raporu var mı?

**Para ve iade**
- [ ] Kapıda ödeme tahsilat mutabakatı raporu var mı?
- [ ] Tahmini ve gerçekleşen desi karşılaştırılıyor mu?
- [ ] İade gönderisi ters yönlü gönderi olarak aynı yapıda tutuluyor mu?

**Bildirim ve veri**
- [ ] Müşteri bildirimleri tekrarlanmıyor, yalnızca anlamlı durumlarda gidiyor mu?
- [ ] Kargo firmasına yalnızca teslimat için gereken kişisel veri gönderiliyor mu?

---

## 19. İlgili rehberler ve resmî kaynaklar

Bu rehber özet ve yön göstericidir; **hukuki veya mali görüş değildir**. Bağlayıcı olan
mevzuat metni, kargo firmasıyla yapılan sözleşme ve firmanın güncel entegrasyon belgesidir.

**İlgili rehberler**

- [E-Ticaret Hukuki Uyum Rehberi](https://github.com/alestaweb/eticaret-hukuki-uyum-rehberi) — mesafeli satış, ön bilgilendirme, cayma
- [e-Fatura / e-Arşiv / e-İrsaliye Rehberi](https://github.com/alestaweb/efatura-earsiv-eirsaliye-rehberi) — belge düzeni, kısmi sevkiyat
- [İYS Rehberi](https://github.com/alestaweb/iys-ticari-elektronik-ileti-rehberi) — bilgilendirme mesajı ile ticari ileti farkı
- [Türkiye E-Ticaret POS Rehberi](https://github.com/alestaweb/turkiye-eticaret-pos-rehberi) — ödeme tarafı
- [KVKK için Web Geliştirici Rehberi](https://github.com/alestaweb/kvkk-icin-web-gelistirici-rehberi) — veri aktarımı, aydınlatma

**Resmî kaynaklar**

- **Mesafeli Sözleşmeler Yönetmeliği** — https://mevzuat.gov.tr
- **6502 sayılı Tüketicinin Korunması Hakkında Kanun** — https://mevzuat.gov.tr
- **T.C. Ticaret Bakanlığı** — https://ticaret.gov.tr
- **Bilgi Teknolojileri ve İletişim Kurumu** (posta ve kargo hizmetleri düzenleyicisi) — https://btk.gov.tr
- **Kişisel Verileri Koruma Kurumu** — https://kvkk.gov.tr
- **Gelir İdaresi Başkanlığı e-Belge** — https://ebelge.gib.gov.tr

---

## Katkı

Eksik gördüğünüz, güncelliğini yitirmiş veya yanlış bulduğunuz bir madde varsa issue açın.

## Lisans

MIT — serbestçe kullanın, çoğaltın, alıntılayın.

---

*Hazırlayan: [Alesta WEB](https://alestaweb.com) · haber yazılımı ve e-ticaret
yazılımı geliştiricisi · 2004'ten beri*
