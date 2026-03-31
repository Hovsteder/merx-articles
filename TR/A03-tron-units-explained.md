# SUN, TRX, Energy, Bandwidth: Tum TRON Birimleri Aciklandi

TRON uzerinde gelistirme yapmaya veya basitce USDT gondermeye calismissaniz, muhtemelen kafa karistirici bir birim dizisiyle karsilasmissinizdir: TRX, SUN, energy, bandwidth, dondurulmus bakiye, stake edilmis kaynaklar. Dokumantasyon bu kavramlari onlarca sayfaya dagitmaktadir ve cogu egitim, uretim kodu yazarken veya maliyetleri hesaplarken gercekten onemli olan farklari gozden gecirmektedir.

Bu makale, TRON ile calismaya basladigimda var olmasini istedigim referans rehberidir. TRON ekosistemindeki her birimi, birbirleriyle nasil iliskili olduklarini ele alacak ve hemen kullanabileceginiz somut donusum ornekleri sunacagiz.

---

## TRX: The Base Currency

TRX (Tronix) is the native cryptocurrency of the TRON blockchain. It serves the same role that ETH serves on Ethereum - it is the unit of account, the medium for paying transaction fees, and the asset you stake to acquire network resources.

As of early 2026, TRX trades in the range of $0.20 to $0.30 USD, though this fluctuates. What does not fluctuate is its role in the resource model: every operation on TRON ultimately traces back to TRX, whether you spend it directly (burning) or lock it up (staking).

Key facts about TRX:

- Total supply is capped and deflationary due to burning mechanisms.
- TRX can be sent, staked, or burned.
- Minimum divisible unit is 1 SUN (more on this below).
- All on-chain fees are denominated in TRX or its sub-units.

---

## SUN: The Smallest Unit

SUN is to TRX what wei is to ETH or what satoshi is to BTC. It is the smallest indivisible unit of the TRON currency.

**1 TRX = 1,000,000 SUN**

This is not negotiable, not approximate, and not subject to market conditions. It is a fixed conversion defined at the protocol level.

### Why SUN Matters for Developers

Every serious TRON application works in SUN internally. The reason is simple: floating-point arithmetic is the enemy of financial software. When you represent 1.5 TRX as `1500000` SUN, you eliminate rounding errors entirely. Integer math is exact.

Consider this example:

```typescript
// Wrong - floating point
const amount = 1.1 + 0.2; // 1.3000000000000003

// Correct - SUN integers
const amountSun = 1_100_000 + 200_000; // 1_300_000 (exactly 1.3 TRX)
```

At MERX, every balance, every ledger entry, every price quote is stored and transmitted in SUN. The conversion to TRX only happens at the display layer, never in business logic.

### Common Conversions

| TRX | SUN |
|-----|-----|
| 0.000001 | 1 |
| 0.01 | 10,000 |
| 1 | 1,000,000 |
| 100 | 100,000,000 |
| 1,000 | 1,000,000,000 |
| 36,000 | 36,000,000,000 |

### Conversion in Code

```typescript
// TRX to SUN
function trxToSun(trx: number): bigint {
  return BigInt(Math.round(trx * 1_000_000));
}

// SUN to TRX (display only)
function sunToTrx(sun: bigint): string {
  const trx = Number(sun) / 1_000_000;
  return trx.toFixed(6);
}
```

Note the use of `BigInt` for SUN values. Standard JavaScript `number` can safely represent integers up to 2^53, which is about 9 quadrillion SUN or 9 billion TRX. For most applications this is sufficient, but `BigInt` eliminates any risk.

---

## Energy: Smart Contract Fuel

Energy is the resource consumed when executing smart contracts on TRON. Every instruction in the TRON Virtual Machine (TVM) costs a specific amount of energy, similar to how gas works on Ethereum.

### What Consumes Energy

- Calling any smart contract function (including TRC-20 token transfers like USDT)
- Deploying new smart contracts
- Any operation that involves computation on-chain

### What Does NOT Consume Energy

- Simple TRX transfers (these use bandwidth only)
- Account creation
- Voting for super representatives
- Staking and unstaking

### How Much Energy Does a USDT Transfer Cost

A standard USDT (TRC-20) transfer costs approximately **65,000 energy**. This is the single most common operation on TRON, accounting for the majority of all smart contract calls on the network.

The exact cost can vary slightly depending on the contract state - for instance, the first transfer to a new address may cost more because it creates a new storage slot. But 65,000 is the number you should use for planning.

### How to Get Energy

There are exactly two ways to obtain energy:

**1. Staking TRX (Stake 2.0)**

You lock TRX in a staking contract and receive energy in proportion to your stake relative to the total network stake. The formula is:

```
your_energy = (your_staked_trx / total_network_staked_trx) * total_energy_pool
```

As of early 2026, you need approximately **36,000 TRX** staked to receive around **65,000 energy per day** - enough for one USDT transfer daily. The exact ratio fluctuates as total network stake changes.

Important: staked energy regenerates over 24 hours. If you use all your energy at once, it takes a full day to recover. Energy from staking cannot be stockpiled.

**2. Renting from a provider (or aggregator like MERX)**

A provider stakes their own TRX and delegates the resulting energy to your address for a fee. This is the pay-per-use model. You pay a fraction of the TRX cost without locking capital.

Typical rental prices in early 2026 range from 80 to 120 SUN per energy unit, depending on the provider, duration, and volume.

### Energy Units

Energy is dimensionless - it is simply a number. There is no sub-unit. You either have 65,000 energy or you do not. It is not denominated in TRX or SUN; it is its own resource.

---

## Bandwidth: Transaction Data Allowance

Bandwidth is the resource consumed for the raw data of any transaction on TRON. Every transaction, whether it is a simple TRX transfer or a complex smart contract call, uses bandwidth proportional to its byte size.

### How Bandwidth Is Measured

Bandwidth is measured in bytes. A typical TRX transfer transaction is roughly 250-300 bytes. A USDT transfer (smart contract call) is roughly 350-400 bytes.

### Free Bandwidth

Every TRON account receives **600 free bandwidth points per day**. This regenerates over 24 hours, similar to energy. For simple TRX transfers (roughly 270 bytes each), this means you get about two free transfers per day.

### What Happens When You Run Out

When your bandwidth is depleted, the network burns TRX from your account to cover the bandwidth cost. The burn rate fluctuates but is generally modest - a few TRX at most for a standard transaction. The formula is:

```
bandwidth_cost_trx = transaction_bytes * bandwidth_price_per_byte
```

For most users doing occasional transfers, the free 600 bandwidth is sufficient. For high-volume applications, bandwidth costs become a line item worth tracking.

### Bandwidth vs Energy: The Key Difference

Every transaction uses bandwidth. Only smart contract transactions use energy. A simple TRX transfer uses bandwidth but zero energy. A USDT transfer uses both bandwidth and energy.

| Operation | Bandwidth | Energy |
|-----------|-----------|--------|
| TRX transfer | ~270 bytes | 0 |
| USDT transfer | ~350 bytes | ~65,000 |
| Contract deploy | varies | varies |
| Vote | ~270 bytes | 0 |

---

## How All Units Relate

Here is the complete picture of how TRON's units connect:

```
TRX (currency)
  |
  |-- 1 TRX = 1,000,000 SUN (sub-unit)
  |
  |-- Stake TRX --> Energy (smart contract fuel)
  |                  |
  |                  |-- Measured in units (dimensionless)
  |                  |-- ~65,000 needed per USDT transfer
  |                  |-- Regenerates over 24 hours
  |
  |-- Stake TRX --> Bandwidth (data allowance)
  |                  |
  |                  |-- Measured in bytes
  |                  |-- 600 free daily per account
  |                  |-- Used by ALL transactions
  |
  |-- Burn TRX --> Pay for bandwidth/energy directly
```

The critical insight: staking TRX gives you renewable resources (they regenerate daily), while burning TRX is a one-time payment. For high-volume users, staking or renting is dramatically cheaper than burning.

---

## Practical Cost Example

Let us walk through a concrete scenario: you need to send 100 USDT transfers per day.

### Option 1: Burn Everything

Each USDT transfer without energy costs approximately 27-30 TRX in burned fees (covering both the energy and bandwidth costs). For 100 transfers:

```
100 transfers x 28 TRX = 2,800 TRX/day
At $0.25/TRX = $700/day = $21,000/month
```

### Option 2: Stake TRX for Energy

You need approximately 65,000 energy per transfer, so 6,500,000 energy per day. At current staking ratios, this requires roughly 3,600,000 TRX staked (~$900,000 locked capital).

Obviously impractical for most users.

### Option 3: Rent Energy via MERX

At a rental rate of approximately 90 SUN per energy unit:

```
65,000 energy x 90 SUN = 5,850,000 SUN = 5.85 TRX per transfer
100 transfers x 5.85 TRX = 585 TRX/day
At $0.25/TRX = $146.25/day = $4,388/month
```

The savings versus burning: roughly 80%.

This is why the energy rental market exists. The gap between burning costs and rental costs is enormous, and it scales linearly with volume.

---

## Units in the MERX API

When working with the MERX API, all values follow consistent conventions:

```json
{
  "price_per_unit": 88000000,
  "energy_amount": 65000,
  "total_cost_sun": 5720000000,
  "total_cost_trx": 5720.0
}
```

- `price_per_unit`: Price in SUN per energy unit
- `energy_amount`: Energy units (dimensionless integer)
- `total_cost_sun`: Total cost in SUN (integer)
- `total_cost_trx`: Total cost in TRX (display convenience)

The SDK handles conversions:

```typescript
import { MerxClient } from '@merx/sdk';

const client = new MerxClient({ apiKey: 'your-key' });
const prices = await client.getPrices({ energy: 65000 });

console.log(prices.bestPrice.perUnit);    // SUN per energy unit
console.log(prices.bestPrice.totalSun);   // Total in SUN
console.log(prices.bestPrice.totalTrx);   // Total in TRX
```

Eksiksiz SDK dokumantasyonu: [https://merx.exchange/docs](https://merx.exchange/docs)

---

## Yaygin Hatalar

**Mistake 1: Mixing TRX and SUN in calculations.** Always pick one and stick with it. Internally, use SUN. Convert to TRX only for display.

**Mistake 2: Assuming energy is priced in TRX.** Energy rental prices are typically quoted in SUN per energy unit. A price of "88" usually means 88 SUN, not 88 TRX. Misreading this by a factor of 1,000,000 will ruin your day.

**Mistake 3: Forgetting bandwidth exists.** Even with full energy coverage, every transaction still consumes bandwidth. For high-volume applications, bandwidth costs add up.

**Mistake 4: Treating energy as stockpilable.** Staked energy regenerates over 24 hours. You cannot save up energy from quiet days to use on busy days. Rented energy, however, is available immediately for the rental duration.

**Mistake 5: Using floating-point for SUN values.** Use integers or BigInt. Always. No exceptions.

---

## Ozet

TRON's resource model is more nuanced than most chains, but it is not complicated once you understand the four core concepts:

- **TRX** is money.
- **SUN** is the smallest unit of money (1 TRX = 1,000,000 SUN).
- **Energy** is smart contract fuel, obtained by staking TRX or renting.
- **Bandwidth** is transaction data allowance, partially free, obtained by staking or burning.

For developers building on TRON, the most important takeaway is to always work in SUN internally and treat energy as a cost to be optimized. The difference between burning and renting energy can be 5x to 10x in cost - which is exactly why platforms like MERX exist.

Explore current energy prices and start saving on your TRON operations at [https://merx.exchange](https://merx.exchange).

---

*Bu makale, TRON altyapisi uzerine MERX bilgi serisinin bir parcasidir. MERX, tum buyuk energy saglayicilarini tek bir API'de toplayan ilk blokzincir kaynak borsasidir. Daha fazla bilgi icin [https://merx.exchange](https://merx.exchange).*
