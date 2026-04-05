# TRON'da Enerji Delegasyonu Nasıl Çalışır

Enerji delegasyonu, tüm TRON enerji kiralama pazarını mümkün kılan mekanizmadır. Bu olmadan, enerji transfer edilemez olurdu - stake sahipleri yalnızca kendi enerjilerini kullanabilir ve onu satmak veya paylaşmak için hiçbir yol olmaz. Delegasyonun protokol seviyesinde nasıl çalıştığını anlamak, TRON'da inşa eden veya enerji sağlayıcılarını değerlendiren herkes için gereklidir.

Bu makale Stake 2.0 delegasyon mekanizmasını detaylı olarak açıklar: sağlayıcıların alıcılara enerji nasıl delege ettikleri, delegasyon dönemi sırasında ve sonrasında ne olduğu ve MERX'in birden fazla sağlayıcı arasında tam delegasyon yaşam döngüsünü nasıl yönettiği.

---

## Stake 2.0 Temeli

TRON, orijinal kaynak dondurma mekanizmasının yerini almak için Stake 2.0'ı (ayrıca Stake v2 olarak da adlandırılır) sunmuştur. Stake 2.0'ın temel yeniliği, staking ile kaynak kullanımının ayrılmasıdır. Orijinal modelde (Stake 1.0), dondurulmuş TRX yalnızca dondurucunun kendi adresi tarafından kullanılabilen enerji üretirdi. Stake 2.0, delegasyonu tanıtarak bir adresin staked kaynaklarını ağdaki herhangi bir başka adrese yönlendirmesine izin verdi.

### Üç İşlem

Stake 2.0 delegasyonu üç zincir üstü işlemini içerir:

**1. Stake (freezeBalanceV2)**

Sağlayıcı TRX'i staking kontratında kilitler. Bu, stake edilen miktar ve toplam ağ stake'ine orantılı enerji üretir. Enerji 24 saatlik bir pencere içinde sürekli olarak yenilenir.

```
Sağlayıcı 36.000 TRX stake eder
Ağ, sağlayıcıya ~65.000 enerji/gün tahsis eder
```

**2. Delegate (delegateResource)**

Sağlayıcı enerjisinin bir kısmını hedef adrese yönlendirir. Bu gerçek delegasyon işlemidir. Bu işlem onaylandıktan sonra, hedef adres delege edilen enerjiyi kendi enerjisi gibi kullanabilir.

```json
{
  "owner_address": "TProviderAddress...",
  "receiver_address": "TBuyerAddress...",
  "resource": "ENERGY",
  "balance": 36000000000,
  "lock": true,
  "lock_period": 3
}
```

Temel parametreler:
- `owner_address`: sağlayıcının adresi (delegatör)
- `receiver_address`: alıcının adresi (alıcı)
- `resource`: ENERGY veya BANDWIDTH
- `balance`: bu delegasyonu destekleyen TRX miktarı (SUN cinsinden)
- `lock`: delegasyonun minimum bir dönem için kilitli olup olmadığı
- `lock_period`: kilit süresi gün cinsinden (eğer lock true ise)

**3. Undelegate (undelegateResource)**

Delegasyon dönemi sona erdiğinde, sağlayıcı delegasyonu geri alarak kaynağı geri talep eder. Bu işlem onaylandıktan sonra, enerji alıcının adresine akması durur ve sağlayıcının havuzuna geri döner.

```json
{
  "owner_address": "TProviderAddress...",
  "receiver_address": "TBuyerAddress...",
  "resource": "ENERGY",
  "balance": 36000000000
}
```

---

## Delegasyon Yaşam Döngüsü

Tam bir delegasyon birkaç aşamadan geçer. Her aşamayı anlamak, enerji kiralama etrafında güvenilir sistemler oluşturmak için kritiktir.

### Aşama 1: Sipariş Yerleştirme

Alıcı belirli bir adres ve süre için enerji talep eder. Bu zincir dışında gerçekleşir - alıcı sağlayıcıya (doğrudan veya MERX gibi bir toplayıcı aracılığıyla) ödeme yapar ve sağlayıcı delegasyonu kuyruğa alır.

### Aşama 2: Zincir Üstü Delegasyon

Sağlayıcı `delegateResource` işlemini yayınlar. Bu tipik olarak TRON'da 3-6 saniye içinde onaylanır (bir blok). Onaylandıktan sonra, enerji alıcının adresinde hemen kullanılabilir.

```
Alıcının etkili enerjisi = kendi_staked_enerjisi + tüm_delege_edilen_enerji
```

Önemli: alıcı delege edilen enerjiyi hemen kullanabilir. Isınma dönemi yoktur. Delegasyon işlemi onaylandıktan sonra, alıcının bir sonraki akıllı kontrat çağrısı delege edilen enerjiyi tüketecektir.

### Aşama 3: Aktif Delegasyon Dönemi

Delegasyon dönemi boyunca, alıcı enerjiyi işlemleri için kullanır. Bu aşamadaki temel davranışlar:

- **Yenileme**: delege edilen enerji, self-staked enerji ile aynı hızda yenilenir - 24 saat içinde sürekli olarak.
- **Tüketim önceliği**: bir alıcının hem self-staked hem de delege edilen enerjisi olduğunda, TRON bunlar arasında ayrım yapmaz. Tüm enerji havuzlanır.
- **Birden fazla delegasyon**: bir alıcı aynı anda birden fazla sağlayıcıdan delegasyon alabilir. Enerji miktarları toplamıdır.

### Aşama 4: Son Tarih ve Geri Talep

Kilit dönemi sona erdiğinde, sağlayıcı `undelegateResource` işlemini çalıştırarak kaynaklarını geri talep edebilir. Bu otomatik değildir - sağlayıcı bu işlevi aktif olarak çağırmalıdır.

```
Zaman çizelgesi:
  T+0:   delegateResource onaylandı (enerji aktif)
  T+3d:  Kilit dönemi sona erdi
  T+3d+: Sağlayıcı undelegateResource'ü çağırır (enerji kaldırıldı)
```

Sağlayıcı delegasyonu geri almaz ise, enerji alıcıya süresiz olarak akması devam eder. Sağlayıcılar derhal geri almaya teşvik edilirler çünkü delegasyonu destekleyen staked TRX, delegasyonu geri alınana kadar diğer müşteriler için kullanılamaz.

### Aşama 5: Delegasyon Sonrası

Delegasyon geri alındıktan sonra, sağlayıcının TRX'i "delegasyonu geri alınabilir" havuzuna geri döner. Daha sonra bunu bir sonraki müşteriye delege edebilirler. Alıcının mevcut enerjisi, delegasyonu geri alınan miktar kadar azalır.

---

## Kilit Dönemi Mekanikleri

Delegasyon işlemindeki `lock` parametresi ve `lock_period` özel dikkat gerektirir.

### Kilitli Delegasyonlar

`lock: true` olduğunda, sağlayıcı delegasyonu en az `lock_period` gün boyunca sürdürmeyi taahhüt eder. Bu süre boyunca, sağlayıcı delegasyonu geri alamaz. Bu, alıcıya kaynak kullanılabilirliğinin garantisini verir.

```
lock_period = 3 gün

Gün 0: Delegasyon oluşturuldu, kilit başladı
Gün 1: Sağlayıcı delegasyonu geri ALAMAZ
Gün 2: Sağlayıcı delegasyonu geri ALAMAZ
Gün 3: Kilit sona erdi, sağlayıcı delegasyonu geri ALABİLİR
```

### Kilitlenmemiş Delegasyonlar

`lock: false` olduğunda, sağlayıcı herhangi bir zamanda - delegasyondan hemen sonra bile delegasyonu geri alabilir. Bu alıcılar için risklidir çünkü sağlayıcı, alıcının enerjiyi kullanmadan önce enerjiyi geri talep edebilir.

Pratikte, saygın sağlayıcılar her zaman kilitli delegasyonları kullanır. MERX, tüm delegasyonların satın alınan süre için uygun şekilde kilitli olduğunu doğrular.

### Kilit Dönemi Granülarite

TRON'un kilit dönemi gün cinsinden belirtilir (teknik olarak bloklar, fakat protokol kabaca 24 saatlik dönemlere eşlenir). Minimum pratik kilit 1 gündür. Pazardaki yaygın süreler:

| Kilit Dönemi | Tipik Kullanım |
|-------------|-----------------|
| 0 (kilitlenmemiş) | Tavsiye edilmez |
| 1 gün | Kısa vadeli / test |
| 3 gün | Standart kiralama |
| 7 gün | Haftalık işlemler |
| 14 gün | İki haftalık kapsama |
| 30 gün | Aylık kontratlar |

---

## Enerji Teslimatı Doğrulaması

Delegasyonun gerçekten gerçekleştiğini nasıl bilirsiniz? Birkaç doğrulama yöntemi vardır.

### Zincir Üstü Doğrulama

TRON ağını doğrudan sorgulayın:

```typescript
// TronWeb kullanarak
const accountResources = await tronWeb.trx.getAccountResources(buyerAddress);
console.log('Toplam enerji limiti:', accountResources.EnergyLimit);
console.log('Kullanılan enerji:', accountResources.EnergyUsed);
console.log('Mevcut:', accountResources.EnergyLimit - accountResources.EnergyUsed);
```

`EnergyLimit` alanı tüm kaynaklardan toplam enerjiyi yansıtır: self-staked ve delege edilen. Delegasyon onaylandıktan sonra `EnergyLimit` tarafındaki artış teslimatı onaylar.

### Delegasyon Kaydı Arama

TRON, bir adres için delegasyon kayıtlarını sorgulamak üzere API'ler sağlar:

```typescript
// Bir adres tarafından alınan tüm delegasyonları al
const delegations = await tronWeb.trx.getDelegatedResourceV2(
  buyerAddress,
  providerAddress
);
```

Bu, belirli bir sağlayıcıdan alıcıya dönen spesifik delegasyon miktarlarını ve kilit dönemlerini döndürür.

### MERX Doğrulaması

MERX, her siparişten sonra otomatik doğrulama gerçekleştirir:

1. Sipariş verilir ve ödenir.
2. Sağlayıcı delegasyon işlemini çalıştırır.
3. MERX, delegasyon işlem onayı için blok zincirini izler.
4. MERX, enerji artışını doğrulamak için alıcının hesap kaynaklarını sorgular.
5. Doğrulama başarısız olursa (enerji zaman aşımı içinde alınmaz), sipariş işaretlenir ve alıcıya geri ödeme yapılır.

```typescript
import { MerxClient } from 'merx-sdk';

const client = new MerxClient({ apiKey: 'your-key' });

// Sipariş oluştur
const order = await client.createOrder({
  energy: 65000,
  targetAddress: 'TBuyerAddress...',
  duration: '1d'
});

// Sipariş durumunu kontrol et (delegasyon doğrulamasını içerir)
const status = await client.getOrder(order.id);
console.log(status.delegationStatus);
// 'pending' | 'confirmed' | 'verified' | 'failed'
```

---

## Çok Sağlayıcılı Delegasyon

Tek bir adres aynı anda birden fazla sağlayıcıdan delegasyon alabilir. MERX gibi toplayıcıların büyük siparişleri birden fazla sağlayıcı arasında bölme yöntemi budur.

### Nasıl Çalışır

```
Alıcı adresi: TBuyer123...

Delegasyon 1: Sağlayıcı A -> 200.000 enerji (65.000 x 3)
Delegasyon 2: Sağlayıcı B -> 130.000 enerji (65.000 x 2)
Delegasyon 3: Sağlayıcı C ->  65.000 enerji (65.000 x 1)

Toplam delege edilen enerji: 395.000/gün
Artı alıcının kendi staked enerjisi: 0
Toplam mevcut enerji: 395.000/gün
```

Tüm delegasyonlar bağımsızdır. Sağlayıcı A, Sağlayıcı B'nin veya C'nin delegasyonunu etkilemeden delegasyonu geri alabilir. Alıcının toplam enerjisi, tüm aktif delegasyonlar artı kendi staked enerjisinin toplamıdır.

### Toplayıcılık için Neden Önemli

Bir alıcı MERX aracılığıyla 500.000 enerji sipariş ettiğinde, hiçbir sağlayıcı bunun kadarına sahip olmayabilir. MERX siparişi bölebilir:

```
Sipariş: TBuyer123... için 500.000 enerji

Yönlendirme:
  Sağlayıcı A: 200.000 enerji 82 SUN/birim = 16.400.000 SUN
  Sağlayıcı B: 180.000 enerji 85 SUN/birim = 15.300.000 SUN
  Sağlayıcı C: 120.000 enerji 88 SUN/birim = 10.560.000 SUN

Toplam maliyet: 42.260.000 SUN
Etkili oran: 84,52 SUN/birim
```

Alıcı, karışık bir oranla tek bir siparişi görür. Arka planda, zincir üstünde üç ayrı delegasyon işlemi gerçekleşir.

---

## MERX Delegasyon Yaşam Döngüsünü Nasıl Yönetir

MERX, tüm entegre sağlayıcılar arasında her delegasyonun tam yaşam döngüsünü siparişten son tarihe kadar yönetir.

### Sipariş Yürütme

1. **Fiyat kontrolü**: Mevcut en iyi fiyat için tüm sağlayıcıları yoklayın.
2. **Yönlendirme**: Siparişi doldurabilecek en ucuz sağlayıcı(ları) seçin.
3. **Yürütme**: Siparişi seçilen sağlayıcı(ların) API'sine gönderin.
4. **İzleme**: Zincir üstü delegasyon işlemini izleyin.
5. **Doğrulama**: Enerjiyi hedef adrese taşındığını onaylayın.
6. **Bildirim**: Alıcıyı webhook veya WebSocket aracılığıyla bilgilendirin.

### Aktif Dönem Sırasında

- **Sistem durumu izleme**: Delegasyonların hala aktif olup olmadığını periyodik olarak doğrulayın.
- **Kaynak takibi**: Sorunları algılamak için alıcının enerji kullanımını izleyin.
- **Uyarılar**: Enerji azaldığında veya kullanım desenleri daha fazla ihtiyaç olduğunu öneriyorsa alıcıyı bilgilendirin.

### Son Tarihinde

- **Geri sayım bildirimi**: Delegasyon sona ermeden önce alıcıyı uyarın.
- **Yenileme seçeneği**: Mevcut en iyi fiyattan otomatik yenileme sunun.
- **Yenileme sonrası doğrulama**: Enerjiyi sağlayıcı tarafından uygun şekilde geri alındığını onaylayın.

### Hata Yönetimi

- **Delegasyon hatası**: Sağlayıcı beklenen zaman dilimi içinde delegasyonu yapmazsa, MERX otomatik olarak bir sonraki en ucuz sağlayıcıya yönlendirir.
- **Kısmi doldurma**: Bir sağlayıcı siparişin yalnızca bir kısmını doldurabildiyse, MERX kalanını diğer sağlayıcılardan doldurur.
- **Sağlayıcı kesintisi**: Bir sağlayıcıya ulaşılamıyorsa, fiyatları sipariş defterinden kaldırılır ve siparişler mevcut sağlayıcılara yönlendirilir.

---

## Yaygın Sınır Durumları

### Delegasyon Sona Ermeden Önce Enerji Kullanılır

Alıcı kullanılmayan enerjiyi "geri döndürmeye" yükümlü değildir. Delegasyon sona erdiğinde ve sağlayıcı delegasyonu geri aldığında, enerji basitçe kullanılabilir olmaktan çıkar. Geri dönecek bir şey yoktur.

### Sağlayıcı Satın Alınandan Daha Fazla Enerji Delege Eder

Bazen, bir sağlayıcı alıcının ödediğinden daha fazla enerji delege edebilir (dahili muhasebe farklarından dolayı). Alıcı, delegasyon dönemi boyunca fazladan enerjiden faydalanır. MERX, sipariş edilen ve teslimat edilen tam miktarları takip eder.

### Aynı Adrese Birden Fazla Sipariş

Bir alıcı farklı zamanlarda aynı hedef adrese birden fazla sipariş verirse, birikir. Her delegasyon bağımsızdır ve kendi kilit dönemine göre sona erer.

### Adres Mevcut Değil

TRON, hiç aktivenin olmamış bir adres de dahil olmak üzere herhangi bir geçerli adrese delegasyona izin verir. Enerji, adres aktivenin olduğunda kullanılabilir. Ancak, çoğu sağlayıcı ve MERX, hedef adresin siparişleri işlemeden önce var olduğunu ve aktivenin olduğunu doğrular.

---

## Sonuç

Enerji delegasyonu, TRON'un kaynak ekonomisinin temelini oluşturur. Stake 2.0 mekanizması, enerjiyi transfer edilemez bir kaynaktan ticareti yapılabilir bir emtiaya dönüştürerek tüm sağlayıcı ekosistemini etkinleştiren olayı sağlar. Delegasyonların nasıl çalıştığını - kilit mekanikleri, doğrulama süreci, çok sağlayıcılı bileşim - anlamak TRON'da üretim sistemleri oluşturan herkes için gereklidir.

MERX, birden fazla sağlayıcı arasında delegasyonları yönetmenin karmaşıklığını tek bir API çağrısına soyutlar, ancak başlık altında ne olduğunu bilmek, süre, hacim ve sağlayıcı seçimi hakkında daha iyi kararlar almanıza yardımcı olur.

[https://merx.exchange/docs](https://merx.exchange/docs) adresinde MERX API'sini keşfedin ve delegasyonları programlı olarak yönetmeye başlayın.

---

*Bu makale, TRON altyapısı hakkında MERX bilgi serisinin bir parçasıdır. MERX, ilk blockchain kaynak takasıdır. GitHub: [https://github.com/Hovsteder/merx-sdk-js](https://github.com/Hovsteder/merx-sdk-js).*


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- kurulum yok, salt okunur araçlar için API anahtarı gerekli değil:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI aracınıza şunları sorun: "Şu anda TRON enerjisinin en ucuzu nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)