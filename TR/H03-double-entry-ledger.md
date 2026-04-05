# Blockchain için Çift Taraflı Defter: MERX Muhasebe Mimarisi

Çift taraflı muhasebe 13. yüzyılda İtalya'da icat edildi. O zamandan beri her finansal yenilikten sağ salim çıktı -- kâğıt para, borsa, merkez bankacılığı, elektronik fon transferi ve şimdi blockchain. Dayanıklı olmasının bir sebebi var: işe yarıyor. Her işlem dengeli bir giriş çifti üretir ve herhangi bir dengesizlik hemen bir hatayı gösterir.

MERX için muhasebe sistemini inşa ettiğimizde -- TRX yatırımları, enerji satın alımları, sağlayıcı uzlaşmaları ve çekilişleri yöneten bir platform -- çift taraflı defter tasarımını tartışmadan seçtik. Alternatif, güncellenen bakiye ile tek taraflı izleme, çoğu kripto platformunun muhasebe yönetme şeklidir. Aynı zamanda çoğu kripto platformunun açıklanamayan bakiye tutarsızlıklarıyla nasıl sonuç verdiğidir.

Bu makale MERX'in ardındaki defter mimarisini açıklamaktadır: neden çift taraflı defter blockchain platformları için önemlidir, defter tablosu nasıl tasarlanmıştır, neden kayıtlar değişmezdir, borç ve alacak hesapları nasıl etkileşim kurar ve bakiyeleri SELECT FOR UPDATE kullanarak nasıl uzlaştırırız.

## Kripto için Neden Çift Taraflı Defter?

Tek taraflı bir sistem banka hesap özeti gibi çalışır: bir sütun, bir çalışan toplam. Bir kullanıcı 100 TRX yatırdığında, hesaplarına 100 eklersiniz. 5 TRX harcadıklarında 5 çıkarırsınız. Basit.

Tek taraflı sistemiyle ilgili sorunlar stres altında ortaya çıkar:

**Eşzamanlı değişiklikler.** İki emir eş zamanlı olarak yürütülür. Her ikisi de kullanıcının bakiyesini 100 TRX olarak okur. Her ikisi de 5 TRX çıkarır. Her ikisi de 95 TRX yazar. Kullanıcı bir kez yerine iki kez ücretlendirilmiştir. Daha kötüsü: bir yazma diğerini üzerine yazar ve platform bir işlemi tamamen takip edemez.

**Denetim kaydı eksik.** Bir kullanıcının bakiyesi 47,3 TRX'dir. Oraya nasıl geldi? Tek taraflı sistemde, bireysel işlem kayıtlarından bakiyeyi yeniden oluşturmalısınız -- bu kayıtlar tamamlanmamış olabilir ve yerleşik bir bütünlük kontrolü yoktur.

**Uzlaştırma başarısızlığı.** Tüm kullanıcı bakiyelerinin toplamı platformun TRX varlıklarına eşit olmalıdır. Tek taraflı sistemde, bunu doğrulamak her kullanıcının bakiyesini toplayıp hazinesiz karşılaştırmayı gerektirir. Sayılar eşleşmezse, tutarsızlığı bulmanın sistematik bir yolu yoktur.

Çift taraflı defter tüm bu sorunları yapısal olarak çözer. Her bakiye mutasyonu sıfıra toplamması gereken iki giriş oluşturur. Bütünlük kontrolü her işlemin içine yerleştirilir.

## Defter Tablosu

MERX defteri tek bir PostgreSQL tablodur:

```sql
CREATE TABLE ledger (
  id            BIGSERIAL PRIMARY KEY,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  account_id    UUID NOT NULL REFERENCES accounts(id),
  entry_type    TEXT NOT NULL,
  amount_sun    BIGINT NOT NULL,
  direction     TEXT NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
  reference_id  UUID,
  reference_type TEXT,
  description   TEXT,
  idempotency_key TEXT UNIQUE
);

CREATE INDEX idx_ledger_account ON ledger(account_id);
CREATE INDEX idx_ledger_reference ON ledger(reference_id);
CREATE INDEX idx_ledger_created ON ledger(created_at);
```

### Sütun Tasarımı

**amount_sun**: Tüm tutarlar SUN'da depolanır (1 TRX = 1.000.000 SUN). En küçük birimi kullanmak kayan nokta aritmetiğini tamamen ortadan kaldırır. Hiçbir ondalık miktar, yuvarlama hatası, hassasiyet kaybı yoktur. Her hesaplama tamsayı aritmetiğidir.

**direction**: DEBIT ya da CREDIT. Anlam hesap türüne bağlıdır:

- Kullanıcı hesapları için: CREDIT bakiyeyi arttırır, DEBIT azaltır
- Uzlaştırma hesapları için: DEBIT bakiyeyi arttırır, CREDIT azaltır
- Gelir hesapları için: CREDIT bakiyeyi arttırır

**entry_type**: Defter girişini kategorilendiriyor. Örnekler:

```
DEPOSIT              Kullanıcı TRX'i MERX hesabına yatırır
WITHDRAWAL           Kullanıcı MERX hesabından TRX çeker
ORDER_PAYMENT        Kullanıcı enerji siparişi için ödeme yapar
ORDER_REFUND         Sipariş başarısız, ödeme kullanıcıya iade edilir
PROVIDER_SETTLEMENT  Yerine getirilen sipariş için enerji sağlayıcısına ödeme
X402_PAYMENT         x402 protokolü yoluyla alınan zincir üstü ödeme
```

**reference_id** ve **reference_type**: Defter girişini onu tetikleyen iş nesnesine bağlarlar (bir sipariş, bir yatırım, bir çekilişi). Bu çift yönlü bir denetim izi oluşturur: defter girişinden siparişi bulabilirsiniz ve siparişten defter girişlerini bulabilirsiniz.

**idempotency_key**: Yinelenen girişleri önler. Aynı işlem iki kez işlenirse (yeniden denemeden dolayı, ağ zaman aşımından, yinelenen webhook), idempotency_key'deki benzersiz kısıtlama yalnızca bir girişin oluşturulmasını sağlar.

## Değişmezlik Kuralı

Defter kayıtları asla güncelleştirilmez. Asla silinmez. Bu veritabanı seviyesinde uygulanır:

```sql
-- Defter kayıtlarına yapılan güncellemeleri engelle
CREATE OR REPLACE FUNCTION prevent_ledger_update()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Ledger records cannot be updated';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_ledger_update
  BEFORE UPDATE ON ledger
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_update();

-- Defterden silmeleri engelle
CREATE OR REPLACE FUNCTION prevent_ledger_delete()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'Ledger records cannot be deleted';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_ledger_delete
  BEFORE DELETE ON ledger
  FOR EACH ROW
  EXECUTE FUNCTION prevent_ledger_delete();
```

Bu tetikleyiciler değişmezlik kuralını veritabanı seviyesinde kırılmaz hale getirir. Doğrudan SQL çalıştıran bir yönetici bile önce tetikleyiciyi devre dışı bırakmadan bir defter girişini değiştiremez ya da kaldıramaz -- bu işlem veritabanı denetim günlüklerinde görülür olabilir.

### Neden Değişmezlik Önemlidir

**Denetim bütünlüğü.** Defter kayıtları değiştirilirse, bir saldırgan (ya da bir hata) platformun finansal tarihini değiştirebilir. Değişmez kayıtlar, tarihin kalıcı ve kurcalamaya dayanıklı olduğu anlamına gelir.

**Düzenleyici uyum.** Finansal kayıt tutma düzenlemeleri evrensel olarak işlem kayıtlarının korunmasını gerektirir. Bunları silmek veya değiştirmek uyum ihlalidir.

**Hata ayıklama.** Bir şeyler ters gittiğinde -- ve gerçek para işlemi yapan bir sistemde işler ters gidecektir -- değişmez kayıtlar olayların tam, değiştirilmemiş bir zaman çizelgesini sağlar. Tarihi tamamen gerçekleştiği gibi yeniden oynatabilirsiniz.

### Düzeltmeler ve Tersine Çevirmeler

Bir defter girişinin "düzeltilmesi" gerekiyorsa (örneğin, bir geri ödeme), orijinal girişi güncellemezsiniz. Ters yönde yeni bir giriş oluşturursunuz:

```sql
-- Orijinal sipariş ödemesi
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($user_id, 'ORDER_PAYMENT', 1820000, 'DEBIT', $order_id);

INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement, 'ORDER_PAYMENT', 1820000, 'CREDIT', $order_id);

-- Sipariş başarısız, geri ödeme ver (yeni girişler, orijinallar kalıyor)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($user_id, 'ORDER_REFUND', 1820000, 'CREDIT', $order_id);

INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id)
VALUES ($provider_settlement, 'ORDER_REFUND', 1820000, 'DEBIT', $order_id);
```

Geri ödemeden sonra, orijinal ödeme girişleri hala mevcuttur. Kullanıcının hesaplanan bakiyesi hem ödeme hem de geri ödemeyi yansıtır: net sıfır. Denetim izi tam olarak ne olduğunu ve ne zaman olduğunu gösterir.

## Borç ve Alacak Hesapları

MERX çift taraflı sistemde kendi rolü olan birkaç hesap türü kullanır:

### Kullanıcı Hesapları

Her MERX kullanıcısının bir hesabı vardır. Bakiyeleri defter girişlerinden hesaplanır:

```sql
SELECT
  COALESCE(SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END), 0) -
  COALESCE(SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END), 0)
  AS balance_sun
FROM ledger
WHERE account_id = $user_id;
```

Alacaklar bakiyeyi arttırır (yatırımlar, geri ödemeler). Borçlar azaltır (sipariş ödemeleri, çekilişler).

### Sağlayıcı Uzlaştırma Hesabı

Bir kullanıcı enerji satın aldığında, ödeme sağlayıcıya ulaşması gerekir. Sağlayıcı uzlaştırma hesabı MERX'in her sağlayıcıya borçlu olduğunu takip eder:

```
Kullanıcı sipariş için ödeme yapar:
  Kullanıcı hesabı:               DEBIT  1.820.000 SUN
  Sağlayıcı uzlaştırması (Ücret): CREDIT 1.820.000 SUN

MERX sağlayıcı ile anlaşır:
  Sağlayıcı uzlaştırması (Ücret): DEBIT  1.820.000 SUN
  Hazine:                         CREDIT 1.820.000 SUN
```

Herhangi bir zamanda, sağlayıcı uzlaştırma hesabı bakiyesi MERX'in o sağlayıcıya borçlu olduğu ve henüz uzlaştırmadığı toplam tutarı gösterir.

### Hazine Hesabı

Hazine hesabı MERX'in zincir üstü TRX varlıklarını temsil eder. Yatırımlar hazineyi alacaklandırır (alınan TRX). Çekilişler ve sağlayıcı uzlaştırmaları hazineyi borçlandırır (gönderilen TRX).

### Temel Denklem

Her zaman:

```
Tüm ALINAN'ların Toplamı = Tüm BORÇ'ların Toplamı
```

Bu denklem başarısız olursa, sistemin bir hatası vardır. MERX periyodik olarak bir uzlaştırma denetimi çalıştırır:

```sql
SELECT
  SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END) AS total_credits,
  SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END) AS total_debits
FROM ledger;

-- total_credits MUST equal total_debits
-- Değilse, derhal uyarı ver
```

## Tam İşlem Akışı

İşte tipik bir enerji satın almanın defterden nasıl akış yapacağıdır:

### 1. Kullanıcı TRX Yatırır

Yatırım monitörü MERX yatırım adresine gelen TRX transferini algılar:

```sql
BEGIN;

-- Kullanıcının hesabını alacaklandır (bakiyesi artıyor)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($user_id, 'DEPOSIT', 100000000, 'CREDIT', $deposit_id, 'DEPOSIT', $tx_hash);

-- Hazineyi borçlandır (alınan TRX, hazine sorumluluğu kabul eder)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($treasury_id, 'DEPOSIT', 100000000, 'DEBIT', $deposit_id, 'DEPOSIT', $tx_hash || '_treasury');

COMMIT;
```

### 2. Kullanıcı Enerji Satın Alır

Kullanıcı 65.000 enerji için sipariş verir, 28 SUN/birim:

```sql
BEGIN;

-- Satırı kilitleyerek bakiye kontrol et
SELECT balance_sun FROM account_balances
WHERE account_id = $user_id
FOR UPDATE;

-- Yeterli bakiye doğrula
-- 65.000 * 28 = 1.820.000 SUN = 1,82 TRX

-- Kullanıcı hesabını borçlandır (bakiye azalır)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($user_id, 'ORDER_PAYMENT', 1820000, 'DEBIT', $order_id, 'ORDER', $idempotency_key);

-- Sağlayıcı uzlaştırmasını alacaklandır (MERX şimdi sağlayıcıya borçlu)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type, idempotency_key)
VALUES ($provider_settlement_id, 'ORDER_PAYMENT', 1820000, 'CREDIT', $order_id, 'ORDER', $idempotency_key || '_settlement');

COMMIT;
```

### 3. Sipariş Başarısız Olur (Geri Ödeme)

Sağlayıcı enerjiyi devretmede başarısız olursa, sipariş iade edilir:

```sql
BEGIN;

-- Kullanıcı hesabını alacaklandır (bakiye geri yüklenir)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type)
VALUES ($user_id, 'ORDER_REFUND', 1820000, 'CREDIT', $order_id, 'ORDER');

-- Sağlayıcı uzlaştırmasını borçlandır (MERX artık sağlayıcıya borçlu değil)
INSERT INTO ledger (account_id, entry_type, amount_sun, direction, reference_id, reference_type)
VALUES ($provider_settlement_id, 'ORDER_REFUND', 1820000, 'DEBIT', $order_id, 'ORDER');

COMMIT;
```

Orijinal ödeme girişleri kalıyor. Geri ödeme girişleri yeni kayıtlardır. Bu sipariş için kullanıcının net bakiye değişikliği sıfırdır: 1.820.000 SUN borçlandırıldı, sonra 1.820.000 SUN alacaklandırıldı.

## SELECT FOR UPDATE ile Bakiye Uzlaştırması

Herhangi bir finansal sistemdeki en tehlikeli an, kesintiden önceki bakiye kontrolüdür. Uygun kilitleme olmaksızın, eşzamanlı isteklerin her ikisi de bakiye kontrolünü geçebilir ve her ikisi de kesintiye uğrayabilir, negatif bakiye ile sonuçlanabilir.

### Yarış Koşulu

```
Thread A: SELECT balance WHERE user_id = 1  -> 10 TRX
Thread B: SELECT balance WHERE user_id = 1  -> 10 TRX
Thread A: 8 TRX çıkar, yeni bakiye = 2 TRX
Thread B: 8 TRX çıkar, yeni bakiye = 2 TRX
Sonuç: Kullanıcı 10 TRX'i vardı, 16 TRX harcadı, bakiye 2 TRX'i gösteriyor
```

Platform 6 TRX kaybetti.

### Çözüm: SELECT FOR UPDATE

```sql
BEGIN;

-- Satırı kilitle. Bu satırı FOR UPDATE ile okumaya çalışan
-- başka herhangi bir işlem bu işlem bitene kadar engellenir.
SELECT balance_sun FROM account_balances
WHERE account_id = $user_id
FOR UPDATE;

-- Artık özel bir kilide sahibiz. Bakiyeyi güvenle kontrol et.
-- Yetersiz ise, ROLLBACK et.
-- Yeterli ise, defter girişlerine devam et.

INSERT INTO ledger ...;

COMMIT;
-- Kilit serbest bırakıldı. Sonraki bekleyen işlem devam edebilir.
```

`FOR UPDATE` ile senaryo şu hale gelir:

```
Thread A: SELECT ... FOR UPDATE  -> 10 TRX (satır kilitlendi)
Thread B: SELECT ... FOR UPDATE  -> ENGELLEDI (A'nın kilidini bekliyor)
Thread A: 8 TRX çıkar, COMMIT  -> bakiye = 2 TRX (kilit serbest)
Thread B: SELECT ... FOR UPDATE  -> 2 TRX (kilit kazanıldı)
Thread B: 8 TRX çıkar?         -> YETERSİZ BAKIYE, ROLLBACK
```

Harcama yok. Kayıp fon yok. `SELECT FOR UPDATE`'in serileştirme garantisi bakiye kontrolleri ve kesintilerin atomik olmasını sağlar.

### Performans Etkileri

`SELECT FOR UPDATE` hesap başına işlemleri serileştirir. İki kullanıcı eşzamanlı işlem yapabilir birbirini engelle (farklı satırları kilitlerler). Ama aynı kullanıcı için iki eşzamanlı sipariş sırada beklemelidir.

Pratikte bu bir darboğaz değildir. Bireysel kullanıcılar nadiren gerçekten eşzamanlı istekler gönderir. Göndererlerse (örneğin, yanlış yapılandırılmış bir bot), serileştirme doğru davranıştır -- istediğiniz bu isteklerin paralel değil sırayla işlenmesidir.

## Periyodik Uzlaştırma

İşlem başına bütünlüğün ötesinde, MERX tüm defteri doğrulayan periyodik bir uzlaştırma çalıştırır:

```sql
-- 1. Küresel bakiye denklemini doğrula
SELECT
  SUM(CASE WHEN direction = 'CREDIT' THEN amount_sun ELSE 0 END) AS credits,
  SUM(CASE WHEN direction = 'DEBIT' THEN amount_sun ELSE 0 END) AS debits
FROM ledger;
-- credits MUST equal debits

-- 2. Hesap bakiyelerini zincir üstü duruma karşı doğrula
-- Tüm kullanıcı bakiyelerinin toplamı hazine varlıklarına eşit olmalı
-- eksi bekleyen sağlayıcı uzlaştırmaları

-- 3. Yetim başvuruları kontrol et
-- Her reference_id geçerli bir siparişi, yatırımı veya çekilişi işaret etmelidir
SELECT l.reference_id, l.reference_type
FROM ledger l
LEFT JOIN orders o ON l.reference_id = o.id AND l.reference_type = 'ORDER'
LEFT JOIN deposits d ON l.reference_id = d.id AND l.reference_type = 'DEPOSIT'
WHERE o.id IS NULL AND d.id IS NULL AND l.reference_type IS NOT NULL;
```

Herhangi bir kontrol başarısız olursa, sistem derhal uyarır. Yanıt inceleme ve düzeltmedir (yeni defter girişleri yoluyla), asla mevcut kayıtların değiştirilmesi değil.

## Özet

MERX çift taraflı defteri sağlar:

1. **Bütünlük**: Her işlem dengeli bir çifttir. Dengesizlikler derhal algılanır.
2. **Değişmezlik**: Kayıtlar değiştirilemez ya da silinmez. Tarih kalıcıdır.
3. **Eşzamanlılık güvenliği**: SELECT FOR UPDATE bakiye kontrolleri hakkında yarış koşullarını önler.
4. **Denetlenebilirlik**: Çift yönlü referanslarla tam finansal tarih.
5. **Uzlaştırma**: Periyodik kontroller tüm sistem durumunu doğrular.

Bu yeni değildir. Blockchain platformuna uygulanan 700 yıllık muhasebe tekniğidir. Yenilik, çoğu kripto platformunun bunu atlaması -- ve kaybolan fonlar, açıklanamayan tutarsızlıklar ve denetçi kâbusleriyle fiyatı ödemesidir.

Platform: [https://merx.exchange](https://merx.exchange)
Dokümantasyon: [https://merx.exchange/docs](https://merx.exchange/docs)


## Şimdi AI ile Deneyin

MERX'i Claude Desktop'a ya da herhangi bir MCP uyumlu istemciye ekleyin -- sıfır kurulum, salt oku araçlar için API anahtarı gerekli değildir:

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

AI ajanınıza sorun: "Şu anda en ucuz TRON enerjisi nedir?" ve tüm bağlı sağlayıcılardan canlı fiyatlar alın.

Tam MCP dokümantasyonu: [merx.exchange/docs/tools/mcp-server](https://merx.exchange/docs/tools/mcp-server)