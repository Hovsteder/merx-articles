# Uc Protokol, Tek TRON Ajan: MERX Uzerinde MCP, A2A ve ACP

Yapay zeka ajan ekosistemi protokoller arasinda parcalanmaya devam ediyor. Anthropic'in MCP'si var. Google A2A'yi baslatti. BeeAI ise ACP'yi gelistirdi. Her protokol ajan iletisimini farkli bir sekilde cozuyor ve her birinin kendine ait cerceve ve orkestrator ekosistemi bulunuyor. TRON uygulamalari gelistiren yazilimcilar icin bir protokol secmek, ayni zamanda TRON kaynaklarina hangi cercevelerin erisebilecegini secmek anlamina geliyor. MERX, her uc protokolu tek bir platformdan destekleyerek bu secimi ortadan kaldiriyor -- bunu yapan ilk ve tek TRON ajanindan bahsediyoruz.

## 2026'da Protokol Manzarasi

Yapay zeka ajan altyapisinda artik uc protokol hakim durumda:

**MCP (Model Context Protocol)** -- Anthropic tarafindan olusturuldu. Ajanlarin fonksiyonlari kesfettigi ve cagirdigi arac tabanli bir protokol. 54 arac, 30 prompt, 21 kaynak. Claude, Cursor, Windsurf ve yuzlerce MCP uyumlu istemci tarafindan kullaniliyor. Dogrudan ajan-arac etkilesimi icin en olgun protokoldur.

**A2A (Agent-to-Agent Protocol)** -- Google tarafindan olusturuldu, simdi Linux Foundation buyesinde. Orkestratoslerin gorevleri uzman ajanlara gonderip sonuclari asenkron olarak aldigi gorev tabanli bir protokol. LangChain, CrewAI, Vertex AI Agent Builder, AutoGen ve Mastra tarafindan kullaniliyor. Bir ajanin isi baska bir ajana devrettigi coklu ajan sistemleri icin tasarlandi.

**ACP (Agent Communication Protocol)** -- BeeAI (IBM) tarafindan olusturuldu. Kurumsal orkestratosler icin calistirma tabanli bir protokol. ACP artik Linux Foundation buyesinde A2A ile birlesiyor, ancak protokol ucu mevcut ACP istemcileri icin kullanilabilir olmaya devam ediyor.

Her protokolun farkli bir kesif mekanizmasi, farkli bir yurutme modeli ve farkli bir uyumlu cerceve seti var. Yalnizca MCP konusan bir TRON ajani LangChain icin gorunmezdir. Yalnizca A2A konusan bir ajan Claude icin gorunmezdir. Su ana kadar hicbir TRON projesi bu protokollerden birden fazlasini desteklemiyordu.

## MERX Simdi Neleri Destekliyor

Nisan 2026 itibariyle MERX, tek bir dagitimdan her uc protokolu destekliyor:

| Protokol | Kesif | Yurutme | Uyumlu Cerceveler |
|----------|-------|---------|-------------------|
| MCP | `merx.exchange/mcp/sse` | Arac cagrilari (istek-yanit) | Claude, Cursor, Windsurf, herhangi bir MCP istemcisi |
| A2A | `merx.exchange/.well-known/agent.json` | Gorevler (asenkron, SSE akisi) | LangChain, CrewAI, Vertex AI, AutoGen, Mastra |
| ACP | `merx.exchange/.well-known/agent-manifest.json` | Calistirmalar (asenkron, uzun yoklama) | BeeAI, IBM watsonx, ACP cerceveleri |

Her uc protokol de ayni arka ucu paylasiyor. Bir A2A gorevi `buy_energy` cagrisi yaptiginda, MCP `create_order` arac cagrisinin yurutuguyle tamamen ayni siparis yonlendirme mantigi calistirilir. Protokoller, ayni MERX toplama motoruna farkli giris noktalaridir.

## MCP: Dogrudan Entegrasyon Icin 53 Arac

MCP en derin entegrasyon noktasidir. MERX MCP sunucusu 15 kategoride organize edilmis 54 arac sunar:

- **Fiyat Istihbarati** (5 arac): tum 7 saglayicidan gercek zamanli fiyatlar, en iyi fiyat yonlendirmesi, gecmis veriler, piyasa analizi
- **Kaynak Ticareti** (4 arac): siparis olusturma, siparis listeleme, siparis detaylari, kaynak temin etme
- **Token Islemleri** (4 arac): TRX gonderme, TRC-20 token gonderme, izin verme, token bilgisi sorgulama
- **DEX Takasi** (3 arac): SunSwap teklifi alma, takas yurutme, token fiyati kontrol etme
- **Surekli Siparisler** (4 arac): fiyat tetikleyicileri, zamanlamalar, bakiye uyarilariyla sunucu tarafi otomasyon
- **Zincir Ustu Sorgular** (5 arac): hesap bilgisi, bakiyeler, islemler, blok verileri
- Artik tahmin, kolaylik, sozlesmeler, ag, baslangic, odemeler, niyet yurutme ve oturum yonetimi genelinde 28 arac daha

MCP sunucusu ayrica rehberli is akislari icin 30 prompt ve yapilandirilmis veri erisimi icin 21 kaynak sunar.

### Tek satirda baglanin

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

Kurulum yok. Salt okunur araclar icin API anahtari yok. 22 arac aninda kullanilabilir.

## A2A: Orkestrator Cerceveleri Icin 6 Beceri

A2A, MERX'i orkestratoslerin gorev devredebildigi uzman bir ajan olarak sunar. `/.well-known/agent.json` adresindeki Agent Card 7 beceri ilan eder:

| Beceri | Aciklama | Kimlik Dogrulama Gerekli |
|--------|----------|--------------------------|
| `buy_energy` | Toplanmis piyasadan devredilmis enerji satin alma | Evet |
| `get_prices` | Tum 7 saglayicidan guncel enerji fiyatlari | Hayir |
| `analyze_prices` | Trendler ve onerilerle piyasa analizi | Hayir |
| `check_balance` | Hesap bakiyesi ve zincir ustu kaynak tahsisi | Istege bagli |
| `ensure_resources` | Deklaratif kaynak provizyon (yalnizca eksigi satin al) | Evet |
| `create_standing_order` | Sunucu tarafi otomasyon kurallari | Evet |

### A2A nasil calisir

A2A protokolu gorev tabanli bir model kullanir. Bir orkestrator bir gorev gonderir, MERX bunu asenkron olarak isler ve orkestrator sonucu alir.

**Adim 1: Ajani kesfet**

```bash
curl https://merx.exchange/.well-known/agent.json
```

Bu, tum 7 beceriyi, giris semalarini, desteklenen modlari ve kimlik dogrulama gereksinimlerini iceren Agent Card'i dondurur.

**Adim 2: Gorev gonder**

```bash
curl -X POST https://merx.exchange/a2a/tasks/send \
  -H "Content-Type: application/json" \
  -d '{
    "id": "task-001",
    "message": {
      "role": "user",
      "parts": [{
        "type": "data",
        "data": { "action": "get_prices" }
      }]
    }
  }'
```

Yanit hemen `submitted` durumuyla doner. Gorev arka planda islenir.

**Adim 3: Sonucu al**

```bash
curl https://merx.exchange/a2a/tasks/task-001
```

Yanit, gorev durumunu (`completed`, `failed` vb.) ve tum 7 saglayicidan fiyat verileriyle sonuc artifaktlarini icerir.

**Adim 4: Olaylari akis olarak al (istege bagli)**

```bash
curl -N https://merx.exchange/a2a/tasks/task-001/events
```

SSE akisi gercek zamanli durum gecislerini iletir: `submitted`'dan `working`'e, oradan `completed`'a.

### Beceri yonlendirme

A2A gorevleri yapilandirilmis veri veya dogal dil kullanabilir. Gorev islemcisi otomatik olarak yonlendirir:

- **Yapilandirilmis**: `{ "action": "buy_energy", "energy_amount": 65000, "target_address": "T..." }` dogrudan buy_energy becerisine yonlendirilir
- **Dogal dil**: "What is the current energy price?" anahtar kelime desenine uyar ve get_prices'a yonlendirilir

## ACP: Kurumsal Kullanim Icin Calistirma Tabanli Yurutme

ACP, A2A'ya benzer ancak farkli bir API yuzeyine sahip calistirma tabanli bir model kullanir. `/.well-known/agent-manifest.json` adresindeki manifest ayni 7 yetenegi bildirir.

```bash
# Bir calistirma olustur
curl -X POST https://merx.exchange/acp/v1/agents/merx-tron-agent/runs \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "merx-tron-agent",
    "input": [{
      "role": "user",
      "parts": [{
        "contentType": "application/json",
        "content": "{\"action\":\"get_prices\"}"
      }]
    }]
  }'

# Sonucu yokla (uzun yoklamayla)
curl "https://merx.exchange/acp/v1/runs/{runId}?wait=true"
```

`?wait=true` parametresi uzun yoklamayi etkinlestirir: istek, calistirmanin tamamlanmasini bekleyerek 30 saniyeye kadar bloke olur ve tekrarlanan yoklama ihtiyacini azaltir.

Not: ACP, Linux Foundation buyesinde A2A ile birlesiyor. Ucu mevcut istemciler icin calismaya devam edecek, ancak yeni entegrasyonlar A2A kullanmalidir.

## Mimari: Tek Arka Uc, Uc Giris Noktasi

Her uc protokol de ayni yurutme yolunu paylasiyor:

```
MCP Tool Call ─┐
               ├──► MERX API ──► Provider Router ──► 7 Energy Providers
A2A Task ──────┤                                     (Netts, CatFee, TEM,
               ├──► MERX API                          ITRX, TronSave, Feee,
ACP Run ───────┘                                      PowerSun)
```

A2A ve ACP isleyicileri mevcut API servisi icinde calisir (`services/api/src/agent-protocols/`). MCP sunucusu ve web panosunun kullandigi ayni REST uclarina dahili HTTP cagrilari yaparlar. Bu su anlama gelir:

- **Ayni fiyatlar**: tum protokoller ayni gercek zamanli saglayici verilerini gorur
- **Ayni yonlendirme**: siparisler ayni en ucuz saglayici mantigi uzerinden gecer
- **Ayni kimlik dogrulama**: X-API-Key tum protokollerde calisir
- **Ayni guvenilirlik**: yuk devretme ve yeniden deneme mantigi esit sekilde uygulanir

Gorev ve calistirma durumu Redis'te 24 saatlik TTL ile saklanir. Protokol islemleri icin veritabani yazimi gerekmez.

## Coklu Protokol Neden Onemlidir

### Yazilimcilar icin

Bir TRON entegrasyonu gelistiriyorsunuz. Orkestrasyon cerceveniz LangChain (A2A) kullaniyor. Takim arkadasinizin botu Claude (MCP) kullaniyor. Kurumsal musteriniz BeeAI (ACP) gerektiriyor. Tek protokolle bir ajanda, ayni temel hizmete uc farkli entegrasyon yapmaniz gerekir.

MERX ile her ucu de ayni platforma baglanir. Tek API anahtari. Tek dokumantasyon seti. Tek destek kanali.

### Ajan gelistiriciler icin

Coklu ajan sistemleri standart hale geliyor. Bir planlama ajani; bir ticaret ajani, bir izleme ajani ve bir raporlama ajaniyla koordineli calisir. Bu ajanlar farkli cercevelerde calisabilir. Bir CrewAI ekibi enerji satin alimlarini A2A uzerinden MERX'e devredebilirken, bir Claude ajani fiyatlari MCP uzerinden izleyebilir.

MERX, ajanlarin birbirinin protokol tercihinden haberdar olmasi gerekmeden her ikisini de yonetir.

### TRON ekosistemi icin

Daha fazla protokol kapsami, daha fazla potansiyel entegrasyon demektir. A2A'yi destekleyen her yapay zeka cercevesi artik TRON enerji piyasalarina erisebilir. Her MCP istemcisi TRON islem maliyetlerini optimize edebilir. TRON enerji hizmetleri icin toplam hedeflenebilir pazar, desteklenen her protokolle genisler.

## MERX Nerelerde Listeleniyor

MERX, hem MCP hem de A2A dizinlerinde var olan tek TRON projesidir:

**MCP kayitlari:**
- [Glama](https://glama.ai/mcp/servers/Hovsteder/merx-mcp)
- [Smithery](https://smithery.ai/servers/powersun/merx)
- [Official MCP Registry](https://registry.modelcontextprotocol.io/servers/exchange.merx/mcp)
- mcp.so
- PulseMCP

**A2A dizinleri:**
- [awesome-a2a](https://github.com/pab1it0/awesome-a2a) (Financial Services bolumu)
- [a2aregistry.in](https://a2aregistry.in)

## Baslangic

### MCP (Claude, Cursor, Windsurf)

```json
{
  "mcpServers": {
    "merx": {
      "url": "https://merx.exchange/mcp/sse"
    }
  }
}
```

### A2A (LangChain, CrewAI, Vertex AI, AutoGen)

Discovery URL: `https://merx.exchange/.well-known/agent.json`

### ACP (BeeAI)

Discovery URL: `https://merx.exchange/.well-known/agent-manifest.json`

### Dokumantasyon

- [Agent Protocols genel bakis](https://merx.exchange/agents)
- [MCP Server (54 arac)](https://merx.exchange/mcp)
- [A2A Protocol dokumantasyonu](https://merx.exchange/docs/tools/a2a)
- [ACP Protocol dokumantasyonu](https://merx.exchange/docs/tools/acp)
- [GitHub](https://github.com/Hovsteder/merx-mcp)

---

*Tags: tron mcp server, a2a protocol, acp protocol, tron ai agent, ai agent tron energy, langchain tron, crewai tron, multi-protocol agent, merx exchange*

---

**Simdi yapay zeka ile deneyin**

MCP istemcinize ekleyin:

```json
{ "merx": { "url": "https://merx.exchange/mcp/sse" } }
```

Veya A2A ile kesfet:

```bash
curl https://merx.exchange/.well-known/agent.json
```

Sonra sorun: "Tum saglayicilardaki guncel TRON enerji fiyatlari nedir?"
