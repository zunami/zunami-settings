# Grocy Optimal Setup Guide
### Real-world experience with Grocy 4.6.0 — Price tracking, quantity units, barcodes, parent products
https://grocy.info/de 
https://github.com/grocy/grocy

This guide is based on extensive hands-on testing. It covers the most common setup mistakes and how to avoid them, with correct Grocy UI terminology throughout.

---

## 1. First Steps — Correct Setup Order

Always set up master data in this order. Products cannot be created without locations and quantity units.

1. **Locations** (required field on every product)
2. **Quantity units** + global conversions
3. **Product groups**
4. **Shopping locations** (stores)
5. **Products**

---

## 2. Quantity Units — The Anchor Principle

### The golden rule

Always stock in the **smallest unit you need for recipes**. Grocy calculates reliably from small to large (e.g. g → kg), but struggles with large to small (rounding errors).

```
✅ Stock unit: g → recipe uses 250 g → clean whole number
❌ Stock unit: kg → recipe uses 0.250 kg → ugly decimal, error-prone
```

### Recommended quantity units

**Weight (anchor: g)**
| Name | Abbreviation |
|---|---|
| Gram | g |
| Kilogram | kg |
| Dekagram | dag *(important in Austria/Germany)* |

**Volume (anchor: ml)**
| Name | Abbreviation |
|---|---|
| Milliliter | ml |
| Liter | l |
| Deciliter | dl |

**Countable**
| Name |
|---|
| Piece |

**No global conversion — product-specific only**
| Name | Reason |
|---|---|
| Tablespoon | 1 tbsp flour ≠ 1 tbsp water — density varies |
| Teaspoon | Same reason |
| Pinch | Not measurable |
| Bottle | Contents vary per product |
| Can | Contents vary per product |
| Package | Contents vary per product |

### Global QU conversions — only these four

Navigate to: **Master Data → Quantity Units → Edit unit → Save & continue → Add QU conversion**

> ⚠️ **Critical:** Always enter conversions **from the smaller unit to the larger unit**.
> Never the other way around — entering kg → g with factor 1000 causes decimal rounding errors in Grocy.

| From (smaller) | To (larger) | Factor | Math check |
|---|---|---|---|
| Gram | Kilogram | `0.001` | 1000 g × 0.001 = 1 kg ✅ |
| Gram | Dekagram | `0.1` | 10 g × 0.1 = 1 dag ✅ |
| Milliliter | Liter | `0.001` | 1000 ml × 0.001 = 1 l ✅ |
| Milliliter | Deciliter | `0.01` | 100 ml × 0.01 = 1 dl ✅ |

> **Performance warning:** Too many global conversions cause serious performance problems. Grocy resolves all indirect chains — at 95 units × 350 conversions this creates millions of resolved rows and crashes the recipes page. Keep global conversions minimal.

---

## 3. The Decimal Places Problem — Critical Fix

### What happens with default settings

Grocy stores all prices internally per **stock quantity unit** (e.g. per gram).
With the default 2 decimal places display setting:

```
€1.15 for 1 bottle (690g)
→ Internally: 1.15 ÷ 690 = 0.001666... €/g
→ Stored/displayed as: 0.00 €/g  ← TRUNCATED — price is lost!
→ Shown as: 0.00 €/kg            ← Completely useless
```

### The fix

For Docker installations (linuxserver/grocy), set via environment variables in your stack:

```yaml
environment:
  - GROCY_STOCK_DECIMAL_PLACES_AMOUNTS=4
  - GROCY_STOCK_DECIMAL_PLACES_PRICES_INPUT=6
  - GROCY_STOCK_DECIMAL_PLACES_PRICES_DISPLAY=4
```

**Why 6 for input?**
```
1.15 ÷ 690 = 0.001667 €/g (needs 6 places to preserve precision)
× 1000 = 1.667 €/kg ✅ (correctly rounded for display)
```

**Why not 8?** For €/kg display, 4 decimal places is already more than sufficient.

> **Note:** These are `DefaultUserSetting` values in config.php — they can also be overridden via environment variables with the `GROCY_` prefix.

---

## 4. The Unit Price vs. Total Price Bug

### Confirmed behavior (Grocy 4.6.0)

When using a non-SI unit (e.g. Bottle, Can, Package) as the purchase unit, entering the **Total price** gives wrong results:

| Price entry method | Input | Displayed price | Correct? |
|---|---|---|---|
| **Unit price** (per bottle) | €1.15 | 0.001667 €/g = **1.67 €/kg** | ✅ |
| **Total price** (21 bottles) | €24.15 | 0.0015 €/g = **1.50 €/kg** | ❌ Wrong |

The total price option appears to apply the QU conversion factor twice when the purchase unit differs from the stock unit. This is a known issue — see GitHub #722, #1460, #1715, #1706.

### Workaround

Always use **Unit price** when purchasing with non-SI units (Bottle, Can, Package).

**Set this per product so it pre-selects automatically:**
Navigate to: Product edit → **Default purchase price type** → select **Unit price**

This way the correct option is always pre-selected when purchasing that product.

---

## 5. Barcode Strategy for Variable Package Sizes

### The problem

One product (e.g. tomato sauce) from multiple suppliers with different sizes:
- Supplier A: 690g bottle (EAN: 4067796208788)
- Supplier B: 500g bottle (EAN: 1234567890123)
- Supplier C: 250g bottle (EAN: 9876543210987)

### Solution: One barcode entry per package size

On the product edit page, add a barcode for each package size:

| Barcode | Amount | Quantity unit | Store |
|---|---|---|---|
| 4067796208788 | 690 | Gram | DM |
| 1234567890123 | 500 | Gram | Rewe |
| 9876543210987 | 250 | Gram | Spar |

When scanning, Grocy pre-fills the correct gram amount automatically. You only confirm or adjust the quantity.

### Advanced: product-specific QU conversion for "Bottle"

If you want to enter quantities as "bottles" rather than grams:

1. Create quantity unit "Bottle" (no global conversion!)
2. On the product: **QU Conversions** → Add product-specific conversion:
   - From: Gram → To: Bottle → Factor: `0.001449` (= 1/690)
   - From: Bottle → To: Gram → Factor: `690`
3. Set barcode: Amount = 1, Quantity unit = Bottle

Scanning then shows "1 Bottle" and Grocy knows this = 690g.

> ⚠️ With this setup, always use **Unit price** (not Total price) when entering prices — see Section 4.

---

## 6. Drained Weight (Abtropfgewicht) Rule

For products with liquid (canned tomatoes, preserved vegetables, etc.):

Austrian/German law (Preisauszeichnungsgesetz): When a product shows a **drained weight**, the unit price (Grundpreis) **must be based on the drained weight**, not the total fill weight. This matches what you see on supermarket shelf labels.

**Example: Canned tomatoes**
- Total fill weight: 400g (tomatoes + liquid)
- Drained weight: 290g (tomatoes only)
- **Set barcode amount to: 290g** (not 400g)
- Price per kg is then correctly calculated on the drained weight

Add a barcode note: "Can 400g / Drained weight 290g" for clarity.

---

## 7. Parent Products and Sub Products

### When to use this feature

Use parent/sub products when:
- The same base product comes from multiple suppliers
- Different package sizes exist
- You want one combined stock view and one recipe ingredient
- You want separate price history per supplier

### Correct structure

```
Parent product: "Passata" (stock: Gram)
  ↑ Used in recipes
  ↑ Shows combined stock from all sub products
  ↑ Has product-specific ml/g QU conversion for recipes

  Sub product: "Passata DM Bio 690g"
    → Own barcode (690g)
    → Own price history per store
    → Product-specific: Bottle → 690g
    → "Hide on stock overview" = checked

  Sub product: "Passata Rewe 500g"
    → Own barcode (500g)
    → Own price history per store
    → Product-specific: Bottle → 500g
    → "Hide on stock overview" = checked
```

### Required settings on sub products

| Field | Value | Why |
|---|---|---|
| Parent product | Name of parent | Creates the link |
| Quantity unit stock | Same as parent (e.g. Gram) | Required for stock aggregation to work |
| Hide on stock overview | ✅ checked | Only parent shows in main overview |
| Default purchase price type | Unit price | Prevents the total price bug (Section 4) |

### Required settings on parent product

| Field | Value |
|---|---|
| Accumulate min. stock amount of sub products | ✅ checked |
| QU Conversion (if needed for recipes) | e.g. Gram → Milliliter: 0.952 |

### Price behavior with parent/sub products

Prices are tracked **per sub product**, not rolled up to the parent automatically.
The parent shows the last price from any sub product transaction.
For price comparison between suppliers: check individual sub product entries.

> This is a known limitation — GitHub Issue #1797 requests cost aggregation from sub products to parent.

### Important limitation: consuming from parent

You cannot directly consume stock from the parent product — stock lives in the sub products.
Consuming must be done on the sub product level, or via a recipe that uses the parent product.
See GitHub Issue #899.

---

## 8. Optimal Docker Environment Variables

Complete recommended configuration for linuxserver/grocy:

```yaml
services:
  grocy:
    image: ghcr.io/linuxserver/grocy:latest
    container_name: grocy
    environment:
      - PUID=1000          # Your system user ID
      - PGID=1000          # Your system group ID
      - TZ=Europe/Vienna   # Your timezone

      # Currency and locale
      - GROCY_CURRENCY=EUR
      - GROCY_CULTURE=de   # de, en, fr, etc.

      # Calendar
      - GROCY_CALENDAR_FIRST_DAY_OF_WEEK=1  # 1 = Monday
      - GROCY_CALENDAR_SHOW_WEEK_OF_YEAR=true

      # Barcode lookup (Open Food Facts)
      - GROCY_STOCK_BARCODE_LOOKUP_PLUGIN=OpenFoodFactsBarcodeLookupPlugin

      # Decimal places — CRITICAL for correct price display
      - GROCY_STOCK_DECIMAL_PLACES_AMOUNTS=4
      - GROCY_STOCK_DECIMAL_PLACES_PRICES_INPUT=6
      - GROCY_STOCK_DECIMAL_PLACES_PRICES_DISPLAY=4

      # Purchase/consume defaults
      - GROCY_STOCK_DUE_SOON_DAYS=7
      - GROCY_STOCK_DEFAULT_PURCHASE_AMOUNT=1
      - GROCY_STOCK_DEFAULT_CONSUME_AMOUNT=1

    volumes:
      - /path/to/grocy/data:/config
    ports:
      - 9111:80
    restart: unless-stopped
```

> **Important:** Always configure via environment variables, not by editing config.php inside the container. Changes to config.php are overwritten on every container update.

---

## 9. Purchase Workflow — Best Practices

### Fastest correct workflow

1. Open **Purchase** page
2. Scan barcode → Grocy pre-fills amount and quantity unit
3. Enter price → select **Unit price**
4. Enter best before date
5. OK

### What to avoid

- ❌ **Never use Total price** when purchase unit is non-SI (Bottle, Can, Package) — see Section 4
- ❌ **Never type the product name manually** if you have a barcode — always scan
- ❌ **Never change the stock quantity unit** after a product has been added to stock — this cannot be undone

### Mobile app timeout

The Grocy Android app (by Patrick Zedler) may time out on slower NAS hardware.
If you get network errors: App Settings → Network → increase timeout to 60 seconds.
Running Grocy on dedicated hardware (separate from NAS) significantly improves performance.

---

## 10. Known Limitations (Grocy 4.6.0)

| Limitation | Workaround |
|---|---|
| Total price entry gives wrong €/kg with non-SI units | Always use Unit price (Section 4) |
| Prices not aggregated from sub products to parent | Check individual sub product price history |
| No per-barcode QU conversion factor | Use product-specific QU conversions + separate barcodes per size |
| Stock quantity unit cannot be changed after first stock entry | Plan carefully — set correctly before first purchase |
| Decimal truncation with small per-unit prices (e.g. €/g) | Set `GROCY_STOCK_DECIMAL_PLACES_PRICES_INPUT=6` (Section 3) |
| Cannot consume directly from parent product stock | Consume from sub products, or use recipes |
| Performance problems with too many global QU conversions | Keep global conversions to minimum (Section 2) |

---

*Tested on Grocy 4.6.0 (linuxserver Docker image).
Related GitHub issues: #722, #878, #899, #998, #1460, #1715, #1743, #1797, PR #801*
