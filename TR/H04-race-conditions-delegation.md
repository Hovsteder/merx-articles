# Enerji Delegasyonunda Yarış Koşulları: Nasıl Çözdük

Bu, bir işlem başına 1,43 TRX tasarruf etmek yerine 19 TRX maliyeti olan bir hata hakkında bir hikaye. Testlerde doğru görünen, testnette çalışan ve kendini yalnızca mainnet koşullarında ortaya koyan bir yarış koşulu hakkında bir hikaye. Ve düzeltme hakkında bir hikaye -- bir işlemin devam etmesine izin vermeden önce zincir üstü onay için bekleyen bir yoklama mekanizması.

## Kurulum

MERX, kullanıcılar adına sağlayıcılardan enerji satın alır. Akış, teoride, basittir:

1. Kullanıcı adresleri için enerji talep eder
2. MERX en ucuz sağlayıcıyı seçer
3. Sağlayıcı enerjiyi kullanıcının adresine devre dışı bırakır
4. Kullanıcı, devredilen enerji ile işlemini yürütür
5. İşlem, TRX yakması yerine kiralanmış enerjiyi tüketir

Kritik kelime "önce"dir. Kullanıcının işlemi, delegasyon zincir üstünde doğrulandıktan sonra yürütülmelidir. İşlem, delegasyon gelmeden önce yayınlanırsa, işlem yine de başarılı olur -- ancak protokol yakma oranında TRX yakacaktır ve devredilen enerjiyi tüketmeyecektir. Kullanıcı iki kez öder: bir kez kira için, bir kez TRX yakması aracılığıyla.

## Hata

İlk uygulama sıralı bir desen izledi:

```typescript
async function executeWithEnergy(
  targetAddress: string,
  energyAmount: number,
  transactionFn: () => Promise<string>
): Promise<string> {
  // Adım 1: Enerji satın al
  const order = await merx.createOrder({
    energy_amount: energyAmount,
    target_address: targetAddress,
    duration: '1h'
  });

  // Adım 2: MERX'ten sipariş onayı için bekle
  await waitForOrderStatus(order.id, 'FILLED');

  // Adım 3: İşlemi yürüt
  const txHash = await transactionFn();

  return txHash;
}
```

Bu doğru görünüyor. Enerji satın al, siparişin doldurulmasını bekle, sonra işlemi yürüt. `waitForOrderStatus` işlevi, sipariş durumu `FILLED` olana kadar MERX API'sini yokladı.

Sorun "FILLED"in ne anlama geldiğidir. MERX sisteminde, sağlayıcı delegasyona başladığını onayladığında bir sipariş `FILLED` olarak işaretlenir. Sağlayıcı başarılı bir yanıt döndürür: "Delegasyon işlemini TRON ağına gönderdim."

Ancak "gönderilen" "onaylanan" değildir. Delegasyon, bir bloka dahil edilmesi gereken ve ağ tarafından işlenmesi gereken bir TRON işlemidir. Gönderim ile onay arasında bir pencere vardır -- tipik olarak 3 ila 6 saniye, ancak ağ tıkanıklığı sırasında bazen daha uzun.

Bu pencere boyunca, hedef adres henüz devredelen enerjiye sahip değildir.

### Mainnet'te Ne Oldu

Hata, yarış koşulunun tahmin ettiği gibi ortaya çıktı:

```
Zaman Çizelgesi:

  T+0.0s    Sipariş oluşturuldu, sağlayıcıya gönderildi
  T+0.3s    Sağlayıcı yanıtı: delegasyon TX gönderildi
  T+0.4s    Sipariş durumu -> FILLED
  T+0.5s    Kullanıcı işlemi yayınlandı (USDT transferi)
  T+0.6s    Kullanıcı TX blok N'ye dahil edildi
  T+3.2s    Delegasyon TX blok N+1'e dahil edildi

  Sonuç:
    - Blok N'deki Kullanıcı TX: delegasyon henüz yok
    - Tüketilen enerji: 0 (delegasyon aktif değil)
    - Yakılan TRX: 65.000 * 420 SUN = 27.300.000 SUN = 27,3 TRX
    - Enerji kira maliyeti: 1.820.000 SUN = 1,82 TRX
    - Toplam maliyet: 29,12 TRX (kira + yakma)
    - Beklenen maliyet: 1,82 TRX (yalnızca kira)
```

Kullanıcı 1,82 TRX yerine 29,12 TRX ödedi. Enerji kiralama, delegasyon bir blok geç geldiği için boşa harcandı.

### Sayılar

TRON mainnet'inde, her blok yaklaşık 3 saniye sürer. Sağlayıcı tarafından gönderilen bir delegasyon işlemi, diğer herhangi bir işlemle aynı blok dahil etme işleminden geçer. Kullanıcının işlemi ve delegasyon işlemi birkaç saniye içinde gönderilirse, farklı bloklarda sonuçlanabilirler -- sıralama garantisi olmadan.

Maliyet farkı açıktır:

```
Delegasyon olmadan (TRX yakma):
  65.000 enerji * 420 SUN/enerji = 27.300.000 SUN = 27,3 TRX

Delegasyon ile (kira):
  65.000 enerji * 28 SUN/enerji = 1.820.000 SUN = 1,82 TRX

Yarış koşulu başına harcanan para:
  27,3 + 1,82 = 29,12 TRX toplam maliyet
  vs.
  1,82 TRX beklenen maliyet

  Fazla ödeme: olay başına 27,3 TRX
```

Kötü bir günde, bu yarış koşulu, istemcinin FILLED durumunu aldıktan sonra hemen yürüttüğü siparişlerin %10-20'sinde tetiklenebilirdi.

## Neden Test Bunu Yakalamadı

### Testnet Davranışı

Shasta testneti'nde, blok süreleri mainnet'e benzer, ancak ağ tıkanıklığı minimumdur. Delegasyon işlemleri tipik olarak hemen sonraki bloka dahil edilir. "Gönderilen" ile "onaylanan" arasındaki pencere, testnette tutarlı bir şekilde 3 saniyenin altındaydı ve test sistemimiz, yarış koşulunu maskelleyen adımlar arasında yerleşik 2 saniyelik bir gecikmeye sahipti.

### Sıralı Test

Entegrasyon testlerimiz sıralıydı. Bir test enerji satın alacak, bekleyecek, bir işlem yürütecek ve doğrulayacaktı. Hiç eşzamanlı yük yoktu, delegasyon ve yürütme arasında asla yarış yoktu, mainnet'in ürettiği zamanlama basıncı hiç yoktu.

### Sipariş Durumu Tuzağı

En sinsi yön: sipariş durumu teknik olarak doğruydu. Sipariş doldurulmuştu -- sağlayıcı delegasyonu kabul etmiş ve başlatmıştı. Hata durum izlemede değildi. Bu, FILLED'in "enerji zincir üstünde kullanılabilir" anlamına geldiği varsayımındaydı.

## Düzeltme

Düzeltmenin iki bölümü vardır: hedef adresin zincir üstü kaynaklarını kontrol eden bir doğrulama adımı ve delegasyonun fiilen onaylanana kadar bekleyen bir yoklama döngüsü.

### Bölüm 1: check_address_resources

TRON, herhangi bir adres için şu anda mevcut kaynakları (enerji ve bandwidth) kontrol etmek için bir API sağlar:

```
GET https://api.trongrid.io/wallet/getaccountresource
```

Bu, bir adres için mevcut enerji limiti, kullanılan enerji, bandwidth limiti ve kullanılan bandwidth'i döndürür. Kritik olarak, bu zincir üstü durumu yansıtır -- delegasyon onaylanmışsa, enerji limiti bunu yansıtacaktır. Delegasyon hala bekliyorsa, enerji limiti bunu içermeyecektir.

### Bölüm 2: Onay Alıncaya Kadar Yokla

Düzeltme, tek "FILLED için bekle" kontrolünü, zincir üstü kaynakları doğrulayan bir yoklama döngüsü ile değiştirir:

```typescript
async function executeWithEnergy(
  targetAddress: string,
  energyAmount: number,
  transactionFn: () => Promise<string>
): Promise<string> {
  // Adım 1: Temel kaynakları kontrol et
  const baseline = await checkAddressResources(targetAddress);
  const baselineEnergy = baseline.energy_limit - baseline.energy_used;

  // Adım 2: Enerji satın al
  const order = await merx.createOrder({
    energy_amount: energyAmount,
    target_address: targetAddress,
    duration: '1h'
  });

  // Adım 3: Sağlayıcı tarafından doldurulması için siparişi bekle
  await waitForOrderStatus(order.id, 'FILLED');

  // Adım 4: Delegasyon onaylanana kadar zincir üstü kaynakları yokla
  const confirmed = await pollUntilDelegationConfirmed(
    targetAddress,
    baselineEnergy,
    energyAmount,
    { maxAttempts: 15, intervalMs: 2000 }
  );

  if (!confirmed) {
    throw new Error(
      'Delegasyon zaman aşımı içinde zincir üstünde onaylanmadı. ' +
      'İşlemi yürütme -- enerji kullanılamayabilir.'
    );
  }

  // Adım 5: İşlemi yürüt (delegasyon zincir üstünde onaylandı)
  const txHash = await transactionFn();

  return txHash;
}

async function pollUntilDelegationConfirmed(
  address: string,
  baselineEnergy: number,
  expectedIncrease: number,
  options: { maxAttempts: number; intervalMs: number }
): Promise<boolean> {
  for (let attempt = 0; attempt < options.maxAttempts; attempt++) {
    const resources = await checkAddressResources(address);
    const currentEnergy = resources.energy_limit - resources.energy_used;
    const increase = currentEnergy - baselineEnergy;

    if (increase >= expectedIncrease * 0.95) {
      // Yuvarlama için %5 tolerans ver
      return true;
    }

    await sleep(options.intervalMs);
  }

  return false;
}

async function checkAddressResources(
  address: string
): Promise<{ energy_limit: number; energy_used: number }> {
  const response = await fetch(
    'https://api.trongrid.io/wallet/getaccountresource',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        address: address,
        visible: true
      })
    }
  );

  const data = await response.json();
  return {
    energy_limit: data.EnergyLimit || 0,
    energy_used: data.EnergyUsed || 0
  };
}
```

### Neden 2 Saniyelik Aralıklar

Yoklama aralığı 2 saniyedir. Bu, TRON'un 3 saniyelik blok süresine dayanarak seçilmiştir:

- Her 1 saniyede yoklama, bloklar arasında birçok gereksiz kontrolle sonuçlanır
- Her 3 saniyede yoklama, bazı blokları kaçırır (saat kayması, ağ gecikmesi)
- Her 2 saniyede yoklama, her blok döngüsü içinde kontrol etmemizi sağlar

### Neden 15 Deneme

15 deneme 2 saniyelik aralıklarla = 30 saniyelik maksimum bekleme süresi. Pratikte, delegasyon 1-2 yoklama içinde onaylanır (3-6 saniye). 30 saniyelik zaman aşımı aşırı durumları ele alır:

- Blok dahil etmesini geciktiren ağ tıkanıklığı
- Sağlayıcı delegasyon işlemini geç gönderme
- Geçici RPC düğümü gecikmesi

Delegasyon 30 saniye sonra onaylanmamışsa, gerçekten yanlış bir şey vardır ve TRX yakması riskine girmek yerine sessizce başarısız olmak daha güvenlidir.

## Gerçek Mainnet Verileri

Düzeltmeyi dağıttıktan sonra, bir haftalık üretim trafiğinde yoklama davranışını ölçtük:

```
Delegasyon onay zamanlaması:

  İlk yoklamada onaylandı (0-2s):     %12
  İkinci yoklamada onaylandı (2-4s):   %61
  Üçüncü yoklamada onaylandı (4-6s):   %22
  Dördüncü yoklamada onaylandı (6-8s): %4
  Beşinci+ yoklamada onaylandı (8s+): %1
  Zaman aşımı (30s'de onaylanmadı):   %0,04

Sipariş FILLED'den zincir üstü onaya ortalama süre: 3,1 saniye
Medyan: 2,8 saniye
P99: 8,2 saniye
```

Veriler, yarış koşulu penceresini doğrular: sağlayıcının "FILLED" yanıtı ile zincir üstü onay arasında ortalama 3,1 saniye. Yoklama düzeltmesi olmadan, bu pencere içinde yürütülen herhangi bir işlem TRX yakardı.

### Maliyet Etkisi

```
Düzeltmeden önce (30 gün):
  Yarış koşulundan etkilenen siparişler: ~180
  Olay başına ortalama fazla ödeme: ~19 TRX
  Yarış koşullarının toplam maliyeti: ~3.420 TRX

Düzeltmeden sonra (30 gün):
  Yarış koşulu olayları: 0
  Zaman aşımı başarısızlıkları (delegasyon asla onaylanmadı): 2
    Zaman aşımı tarafından yakalandı, işlem yürütülmedi
    Siparişler otomatik olarak geri ödendi
```

Düzeltme, yarış koşulunu tamamen ortadan kaldırdı. İki zaman aşımı başarısızlığı, delegasyon işleminin hiç gönderilmediği gerçek sağlayıcı tarafı sorunlarıydı -- ilerlemek yerine başarısız olmak istediğiniz tam durumlar.

## Öğrendiklerimiz

### "Gönderilen" "Onaylanan" Değildir

Bu merkezi dersidir. Bir blockchain ile etkileşime giren herhangi bir sistemde, bir işlemin gönderilmesi ile bu işlemin zincir üstünde onaylanması arasında her zaman bir boşluk vardır. Gönderimi onay olarak değerlendiren herhangi bir mantık sonunda başarısız olacaktır.

### Hizmete Değil Zinciri Kontrol Et

Sağlayıcı delegasyonun yapıldığını söylüyor. MERX, siparişin doldurulduğunu söylüyor. Ancak bu ifadelerin hiçbiri delegasyonun hemen şimdi zincir üstünde var olduğu anlamına gelmez. Tek yetkili kaynak blockchain'in kendisidir. Zinciri kontrol et.

### Testnet Zamanlama Hatalarını Gizler

Testnette düşük yük, daha hızlı dahil etme ve daha öngörülebilir davranış vardır. Testnette hiç tetiklenmeyen zamanlamaya duyarlı hatalar mainnet'te görünecektir. Mantığınız iki zincir üstü olay arasındaki zamanlama bağımlıysa, bunu gerçekçi yük koşulları altında test et.

### Sessizce Başarısız Ol

Yoklama döngüsü zaman aşımına uğradığında, düzeltme bir hata atar ve işlemin yürütülmesini önler. Bu doğru davranıştır. Alternatif -- yine de yürütmek ve delegasyonun geldiğini ummak -- başarısızlık başına 19 TRX maliyetine neden olur. Bir hata mesajı hiçbir şeye mal olmaz.

## MERX Uygulaması

MERX MCP sunucusundaki `ensure_resources` aracı bu deseni uygular. Bir yapay zeka aracı, bir sözleşme çağrısını yürütmeden önce `ensure_resources` öğesini çağırdığında, araç:

1. Mevcut zincir üstü kaynakları kontrol eder
2. Açığı hesaplar
3. Tam olarak gerekli enerjiyi satın alır
4. Delegasyon zincir üstünde onaylanana kadar yoklar
5. Kaynaklar doğrulandığında yalnızca başarı döndürür

Aracı, yoklama mantığını kendisi uygulamak zorunda değildir. Yarış koşulu, platform düzeyinde işlenir.

```
Araç: ensure_resources
Girdi: {
  "address": "TYourAddress...",
  "energy_needed": 65000
}

Yanıt: {
  "status": "confirmed",
  "energy_available": 65000,
  "confirmation_time_ms": 4200,
  "order_id": "ord_abc123"
}
```

`confirmation_time_ms` alanı, aracıya yoklamanın ne kadar sürdüğünü söyler. `status: "confirmed"` enerjiyi zincir üstünde ve kullanıma güvenlidir.

Platform: [https://merx.exchange](https://merx.exchange)
Dokümantasyon: [https://merx.exchange/docs](https://merx.exchange/docs)
MCP sunucusu: [https://github.com/Hovsteder/merx-mcp](https://github.com/Hovsteder/merx-mcp)


## Şimdi Yapay Zeka ile Dene

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekle -- yükleme yok, salt okunur araçlar için API anahtarı gerekli değildir:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Yapay zeka aracınıza sor: "Şu an en ucuz TRON enerji nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatları alın.

Tam MCP dokümantasyonu: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)