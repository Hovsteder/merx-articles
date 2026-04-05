# NFT Basımını Kaynak Farkında İşlemlerle Otomatikleştirme

TRON üzerinde NFT basımı bir akıllı kontrat işlemidir ve her akıllı kontrat işlemi energy tüketir. İster tek bir koleksiyonerlik basıyor ister 10.000 parçalık bir koleksiyonla başlatıyor olun, energy maliyetleri önemli ve sıklıkla eksik değerlendirilen bir gider kalemiydir. Tek bir NFT basımı, kontratın karmaşıklığına, metadata işlemesine ve zincir üstü depolama gereksinimlerine bağlı olarak 100.000 ila 300.000 energy tüketebilir.

Bu makale, TRON üzerinde NFT basımının energy ekonomisini incelemekte, kaynak farkında basım boru hatlarını nasıl oluşturacağınızı göstermekte ve MERX toplaması yüksek verimliliği korurken birim başına maliyetleri nasıl azalttığını göstermektedir.

## NFT Basımının Energy Maliyeti Anatomisi

TRC-721 kontratı üzerinde bir basım işlemi çağırdığınızda, EVM seviyesinde birkaç şey gerçekleşir:

1. **Depolama tahsisi**: Yeni bir jeton kimliği oluşturulur ve bir sahip adresine eşlenir. Bu, blokzincir depolamasına yazmanın hesaplamadan çok daha pahalı olması nedeniyle en pahalı energy işlemidir.
2. **Metadata ataması**: Kontrat bir jeton URI'sini zincir üstünde saklıyorsa, bu başka bir depolama yazısıdır.
3. **Sayaç artışı**: Toplam arz sayacı güncellenir.
4. **Olay yayını**: Bir Transfer olayı kaydedilir.
5. **Erişim kontrol kontrolleri**: Mülkiyet doğrulaması, basım sınırları, beyaz liste kontrolleri.

Basit basım işlevleri (tek depolama yazısı, sayaç artışı, olay) yaklaşık 100.000-120.000 energy tüketir. Zincir üstü metadata, royalti konfigürasyonu ve numaralandırılabilir takibi olan karmaşık basımlar 250.000-300.000 energy'ye ulaşabilir.

### Energy Olmadan Maliyet

| Basım Karmaşıklığı | Energy | TRX Yakılması | USD Maliyeti |
|---|---|---|---|
| Basit (sayaç + sahip) | ~100.000 | ~21 TRX | ~$2,50 |
| Standart (+ metadata URI) | ~150.000 | ~31 TRX | ~$3,70 |
| Karmaşık (+ royalties, enum) | ~250.000 | ~52 TRX | ~$6,20 |

Standart basım karmaşıklığına sahip 10.000 parçalık bir koleksiyonda, optimizasyon olmadan toplam energy maliyeti yaklaşık 310.000 TRX ($37.000)'dır. MERX aracılığıyla pazar oranlarında satın alınan energy ile bu, yaklaşık 42.000 TRX ($5.040)'e düşer - %86'lık bir azalma.

## Neden Sabit Energy Tahminleri NFT'ler için Başarısız Olur

NFT kontratları, birim başına maliyet sabit olmadığı için sabit energy tahminleri açısından özellikle problematiktir. Tüketilen energy şunlara göre değişebilir:

- **Jeton Kimliği**: Daha büyük jeton kimlikleri, depolanması için daha fazla bayt gerektirir ve energy'i marjinal olarak artırır
- **Bir adrese ilk kez basım**: Alıcı bu kontrattan hiç jeton tutmadıysa, bakiye eşlemesi oluşturmak var olan birini artırmaktan daha pahalıydır
- **Zincir üstü randomlığı**: Rastgele niteliklere sahip kontratlar ek hesaplama gerçekleştirir
- **Toplu iş boyutu**: N jetonu tek bir işlemde toplu olarak basmak N kez tek bir basım maliyetine mal olmaz

Bu varyasyonlar, birim başına 150.000 energy'lik sabit kodlanmış bir tahmin bazen fazla satın almak (para boşa harcamak) ve bazen eksik satın almak (kısmi TRX yakılmasına neden olmak) anlamına gelir.

## NFT Basımı için Tam Simülasyon

MERX'in energy tahmini, yürütmeden önce tam basım işlemini simüle etmek için `triggerConstantContract` kullanır:

```typescript
import { MerxClient } from 'merx-sdk';

const merx = new MerxClient({ apiKey: process.env.MERX_API_KEY });

// Tam basım çağrısını simüle edin
const estimate = await merx.estimateEnergy({
  contract_address: NFT_CONTRACT_ADDRESS,
  function_selector: 'mint(address,string)',
  parameter: [
    recipientAddress,
    metadataURI
  ],
  owner_address: minterAddress
});

console.log(`Bu basım için gerekli energy: ${estimate.energy_required}`);
// Çıktı şöyle olabilir: 143,287
```

Bu, mevcut kontrat durumunuyla birlikte özel basımınız için tam energy'yi döndürür. Tahmin yok.

## Kaynak Farkında Basım Boru Hattı Oluşturma

Koleksiyon başlatmaları veya devam eden basım işlemleri için, energy tedarikini otomatik olarak işleyen bir boru hattına ihtiyacınız var.

### Energy ile Tek Basım

```typescript
async function mintWithEnergy(
  recipient: string,
  metadataURI: string
): Promise<string> {
  // 1. Tam energy'yi tahmin edin
  const estimate = await merx.estimateEnergy({
    contract_address: NFT_CONTRACT,
    function_selector: 'mint(address,string)',
    parameter: [recipient, metadataURI],
    owner_address: MINTER_WALLET
  });

  // 2. Mevcut energy'yi kontrol edin
  const resources = await merx.checkResources(MINTER_WALLET);
  const deficit = estimate.energy_required - resources.energy.available;

  // 3. Gerekirse energy satın alın
  if (deficit > 0) {
    const order = await merx.createOrder({
      energy_amount: deficit,
      duration: '5m',
      target_address: MINTER_WALLET
    });
    await waitForOrderFill(order.id);
  }

  // 4. Sıfır TRX yakılması ile basımı yürütün
  const tx = await mintNFT(recipient, metadataURI);
  return tx;
}
```

### Toplu Basım Boru Hattı

Sırayla birçok NFT bastığınız koleksiyon başlatmaları için:

```typescript
async function batchMint(
  mintRequests: MintRequest[],
  batchSize: number = 10
): Promise<MintResult[]> {
  const results: MintResult[] = [];

  // Toplu işlerde işleyin
  for (let i = 0; i < mintRequests.length; i += batchSize) {
    const batch = mintRequests.slice(i, i + batchSize);

    // Toplu işte her basım için energy'yi tahmin edin
    let totalEnergy = 0;
    for (const req of batch) {
      const estimate = await merx.estimateEnergy({
        contract_address: NFT_CONTRACT,
        function_selector: 'mint(address,string)',
        parameter: [req.recipient, req.metadataURI],
        owner_address: MINTER_WALLET
      });
      totalEnergy += estimate.energy_required;
    }

    // Tahmin ve yürütme arasındaki durum değişiklikleri için
    // %5 arabellek ekleyin
    totalEnergy = Math.ceil(totalEnergy * 1.05);

    // Tüm toplu iş için energy satın alın
    const order = await merx.createOrder({
      energy_amount: totalEnergy,
      duration: '30m',
      target_address: MINTER_WALLET
    });
    await waitForOrderFill(order.id);

    // Toplu işte tüm basımları yürütün
    for (const req of batch) {
      try {
        const tx = await mintNFT(req.recipient, req.metadataURI);
        results.push({ success: true, txId: tx, request: req });
      } catch (error) {
        results.push({ success: false, error, request: req });
      }
    }

    console.log(
      `Toplu iş ${Math.floor(i / batchSize) + 1}: ` +
      `${batch.length} basım tamamlandı`
    );
  }

  return results;
}
```

Toplu iş yaklaşımı birçok avantaj sağlar:

- **Daha iyi fiyatlandırma**: Daha büyük energy satın alımları genellikle daha iyi birim başına oranlar alır
- **Daha az API çağrısı**: Birim başına yerine toplu iş başına bir energy satın alması
- **Öngörülebilir zamanlama**: Tüm toplu iş için energy mevcuttur
- **Durum farkındalığı**: %5 arabellek, tahmin ve yürütme arasındaki küçük energy varyasyonlarını hesaplar

## Koleksiyon Başlatma Stratejisi

Büyük bir NFT koleksiyonu başlatmak energy maliyetleri etrafında planlama gerektirir. İşte 10.000 parçalık bir koleksiyon için bir strateji:

### Ön Başlatma: Maliyet Tahmini

```typescript
async function estimateCollectionCost(
  totalMints: number,
  sampleSize: number = 20
): Promise<CostEstimate> {
  // Ortalama energy elde etmek için basımların bir örneğini simüle edin
  let totalEnergy = 0;

  for (let i = 0; i < sampleSize; i++) {
    const estimate = await merx.estimateEnergy({
      contract_address: NFT_CONTRACT,
      function_selector: 'mint(address,string)',
      parameter: [SAMPLE_ADDRESS, `ipfs://sample/${i}`],
      owner_address: MINTER_WALLET
    });
    totalEnergy += estimate.energy_required;
  }

  const avgEnergy = totalEnergy / sampleSize;

  // Mevcut en iyi energy fiyatını alın
  const prices = await merx.getPrices({
    energy_amount: Math.round(avgEnergy),
    duration: '5m'
  });

  const costPerMint =
    (avgEnergy * prices.best.price_sun) / 1e6; // TRX

  return {
    averageEnergy: avgEnergy,
    bestPriceSun: prices.best.price_sun,
    costPerMintTrx: costPerMint,
    totalCostTrx: costPerMint * totalMints,
    totalCostUsd: costPerMint * totalMints * 0.12
  };
}
```

### Başlatma Sırasında: Uyarlanabilir Toplu İş İşlemesi

```typescript
async function launchCollection(
  metadata: string[],
  recipients: string[]
): Promise<void> {
  const BATCH_SIZE = 50;
  const totalBatches = Math.ceil(metadata.length / BATCH_SIZE);

  console.log(
    `${metadata.length} NFT ${totalBatches} toplu işte başlatılıyor`
  );

  for (let batch = 0; batch < totalBatches; batch++) {
    const start = batch * BATCH_SIZE;
    const end = Math.min(start + BATCH_SIZE, metadata.length);
    const batchMeta = metadata.slice(start, end);
    const batchRecipients = recipients.slice(start, end);

    // Duran sipariş mantığını kullanın: fiyat eşikten yüksekse,
    // daha iyi bir oran için bekleyin
    const prices = await merx.getPrices({
      energy_amount: 150000 * batchMeta.length,
      duration: '1h'
    });

    if (prices.best.price_sun > 35) {
      console.log(
        `Fiyat ${prices.best.price_sun} SUN'da. ` +
        `Daha iyi oran için bekleniyor...`
      );
      // Bekleme mantığı uygulayın veya duran siparişi kullanın
    }

    // Toplu basımla devam edin
    await processBatch(batchMeta, batchRecipients);

    console.log(
      `Toplu iş ${batch + 1}/${totalBatches} tamamlandı. ` +
      `${end}/${metadata.length} basıldı.`
    );
  }
}
```

## Birim Başına Maliyet Karşılaştırması

| Yöntem | Birim Başına Energy | Birim Başına Maliyet (TRX) | Birim Başına Maliyet (USD) | 10K Koleksiyon (USD) |
|---|---|---|---|---|
| Optimizasyon yok (TRX yakılması) | 150.000 | 30,9 | $3,71 | $37.100 |
| Sabit tahmin + tek sağlayıcı | 200.000 (fazla satın alma) | 5,6 | $0,67 | $6.700 |
| Tam simülasyon + MERX (28 SUN) | 143.000 (tam) | 4,0 | $0,48 | $4.800 |
| Tam + duran siparişler (23 SUN) | 143.000 (tam) | 3,3 | $0,40 | $3.960 |

Optimizasyon olmayan ile tam MERX entegrasyonu arasındaki fark 10.000 parçalık bir koleksiyonda $33.000'dır. Sabit tahminlerle tek bir sağlayıcı kullanmak ile karşılaştırırken bile, MERX tam simülasyon ve fiyat toplaması yoluyla yaklaşık $2.000 tasarruf sağlar.

## Devam Eden Basım için Otomatik Energy

Platformunuz sürekli basımı destekliyorsa (sabit bir koleksiyon değil, kullanıcı tarafından yönlendirilen basımlar), basım cüzdanında otomatik energy'yi yapılandırın:

```typescript
await merx.enableAutoEnergy({
  address: MINTER_WALLET,
  min_energy: 300000,    // ~2 basım arabellegi
  target_energy: 1000000, // ~6-7 basım arabellegi
  max_price_sun: 30,
  duration: '1h'
});
```

Bu, basım cüzdanının her zaman en az iki basım için yeterli energy'ye sahip olmasını sağlar, arabellek düştüğünde otomatik olarak yenilenir. Kullanıcıya açık basım deneyimleri hızlı kalır çünkü energy talep üzerine satın alınmak yerine önceden satın alınır.

## Webhook Tarafından Yönlendirilen Basım

Basımın kullanıcı eylemleri tarafından tetiklendiği platformlar için (satın almalar, talepler), webhook tabanlı bir mimari kullanın:

```typescript
// Bir kullanıcı basım istediğinde
app.post('/api/mint', async (req, res) => {
  const { recipient, tokenId } = req.body;

  // Basım isteğini kuyruğa alın
  const mintJob = await queueMint(recipient, tokenId);

  // Energy iste
  const order = await merx.createOrder({
    energy_amount: 150000,
    duration: '5m',
    target_address: MINTER_WALLET
  });

  // Energy siparişini basım işine ilişkilendir
  await linkOrderToMint(order.id, mintJob.id);

  res.json({ status: 'processing', mintId: mintJob.id });
});

// MERX webhook: energy hazır
app.post('/webhooks/merx', async (req, res) => {
  const event = req.body;

  if (event.type === 'order.filled') {
    const mintJob = await getMintByOrderId(event.data.order_id);
    if (mintJob) {
      await executeMint(mintJob);
      await notifyUser(mintJob.userId, 'mint_complete');
    }
  }

  res.status(200).json({ received: true });
});
```

## Sonuç

TRON üzerinde NFT basımı pahalı olmak zorunda değildir. Tam energy simülasyonu ve MERX aracılığıyla çoklu sağlayıcı toplaması kombinasyonu, basımı yüksek maliyetli bir işlemden yönetilebilir bir masrafa dönüştürür.

Koleksiyon başlatmaları için tasarruflar on binlerce dolar cinsinden ölçülür. Devam eden basım platformları için, otomatik energy ve webhook entegrasyonu birim başına maliyetleri minimum'da tutarken yanıtlayıcı bir kullanıcı deneyimi korur.

Kilit anlayış kesinliktir: tam olarak ihtiyacınız olan energy'yi, mevcut en iyi fiyattan, tam olarak ihtiyacınız olduğu zaman satın alın. Tam simülasyon fazla satın almaktan gelen israfı ortadan kaldırır. Toplama aşırı ödemeyi ortadan kaldırır. Birlikte, NFT basımı maliyetlerini optimize edilmemiş yaklaşımlara kıyasla %85-90 oranında azaltırlar.

[https://merx.exchange/docs](https://merx.exchange/docs) adresinden derlemeyeye başlayın veya [https://merx.exchange](https://merx.exchange) adresindeki platformu keşfedin.


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a veya herhangi bir MCP uyumlu istemciye ekleyin -- kurulum yok, salt okuma araçları için API anahtarı gerekmez:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza sorun: "TRON energy'sinin en ucuzu şu an ne?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP belgeleri: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)