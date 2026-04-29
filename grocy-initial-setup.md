# Grocy Initial Setup Guide

A community-tested setup guide for Grocy. Covers quantity units, unit conversions, storage locations, and product groups.

> **Key principle:** Always enter unit conversions from the **smaller unit → larger unit** (e.g. Gram → Kilogram).
> Never the other way around — converting downward (e.g. Kilogram → Gram with factor 1000) causes decimal rounding errors in Grocy's internal calculations.

---

## 1. Quantity Units

Go to: **Master Data → Quantity Units**

Create the following units. Name in singular and plural are usually identical for metric units.

### Weight

| Name (Singular) | Name (Plural) | Note |
|---|---|---|
| Gram | Gram | ⚓ Stock anchor for all weight-based products |
| Kilogram | Kilogram | |
| Dekagram | Dekagram | Common in Austria/Germany (1 dag = 10 g) |

### Volume

| Name (Singular) | Name (Plural) | Note |
|---|---|---|
| Milliliter | Milliliter | ⚓ Stock anchor for all liquid products |
| Liter | Liter | |
| Deciliter | Deciliter | |

### Countable

| Name (Singular) | Name (Plural) | Note |
|---|---|---|
| Piece | Pieces | ⚓ Anchor for countable items |

### Cooking measures (no global conversion — product-specific only)

| Name (Singular) | Name (Plural) | Why no global conversion? |
|---|---|---|
| Tablespoon | Tablespoons | 1 tbsp flour ≠ 1 tbsp water — set per product |
| Teaspoon | Teaspoons | Same reason |
| Pinch | Pinches | Not measurable |

### Packaging (no global conversion — product-specific only)

| Name (Singular) | Name (Plural) | Why no global conversion? |
|---|---|---|
| Package | Packages | Contents vary per product |
| Bottle | Bottles | Contents vary per product |
| Can | Cans | Contents vary per product |
| Cup | Cups | Contents vary per product |

---

## 2. Quantity Unit Conversions (Global)

Go to: **Master Data → Quantity Units → Edit unit → Save & continue → Add conversion**

> ⚠️ **Important:** Always enter conversions **from smaller → larger unit**.
> This avoids decimal rounding issues in Grocy's stock calculations.
>
> Example: Enter `Gram → Kilogram (factor 0.001)` — NOT `Kilogram → Gram (factor 1000)`.

### Weight conversions

| From (smaller) | To (larger) | Factor | Calculation check |
|---|---|---|---|
| Gram | Kilogram | `0.001` | 1000 g × 0.001 = 1 kg ✅ |
| Gram | Dekagram | `0.1` | 10 g × 0.1 = 1 dag ✅ |

### Volume conversions

| From (smaller) | To (larger) | Factor | Calculation check |
|---|---|---|---|
| Milliliter | Liter | `0.001` | 1000 ml × 0.001 = 1 l ✅ |
| Milliliter | Deciliter | `0.01` | 100 ml × 0.01 = 1 dl ✅ |

### No global conversion for:
- Tablespoon, Teaspoon, Pinch → add per product if needed
- Package, Bottle, Can, Cup → add per product (contents vary)

---

## 3. Storage Locations

Go to: **Master Data → Locations**

| Name | Description |
|---|---|
| Fridge | Main refrigerator |
| Freezer | Deep freezer / freezer compartment |
| Pantry | Dry goods, canned goods, non-perishables |
| Cellar | Long-term storage, wine, preserves |
| Bathroom | Personal care, medicine |
| Cleaning supplies | Detergents, cleaning agents |

> **Note:** "Location" is a required field when creating a product in Grocy. Set up locations before adding any products.

---

## 4. Product Groups

Go to: **Master Data → Product Groups**

| Name | Description |
|---|---|
| Baking ingredients | Flour, sugar, yeast, baking powder, starch, chocolate |
| Dairy products | Milk, butter, cheese, yogurt, cream, eggs |
| Meat & cold cuts | Fresh meat, cold cuts, sausage, poultry |
| Vegetables & fruit | Fresh and stored vegetables and fruit |
| Beverages | Water, juices, soft drinks, beer, wine, coffee, tea |
| Frozen food | Frozen vegetables, meat, ready meals, ice cream |
| Canned & jarred goods | Canned goods, pickled vegetables, jams, sauces |
| Spices & oils | Salt, pepper, spices, cooking oil, vinegar |
| Bread & bakery | Bread, pastries, crispbread, rusks |
| Cleaning | Detergent, dish soap, cleaning agents, sponges |
| Hygiene | Soap, shampoo, dental care, toilet paper |
| Household & misc | Batteries, candles, aluminum foil, freezer bags, miscellaneous |

---

## 5. Recommended Product Setup

When creating a product, use these settings for weight-based items (e.g. sugar, flour):

| Field | Recommended value | Reason |
|---|---|---|
| **QU Stock** | Gram | Enables recipe use in grams |
| **QU Purchase** | Gram | Consistent with stock unit |
| **Factor** | 1 | No conversion needed |
| **Default location** | Pantry (or appropriate) | Required field |
| **Product group** | Baking ingredients (or appropriate) | For filtering |

For different package sizes of the same product, add **barcodes with individual amounts**:

| Barcode | Amount | QU |
|---|---|---|
| EAN of 150g package | 150 | Gram |
| EAN of 250g package | 250 | Gram |
| EAN of 500g package | 500 | Gram |

Grocy will automatically pre-fill the correct amount when scanning — no manual entry needed.

---

## 6. App Settings

Go to: **Settings (top right gear icon)**

| Setting | Recommended value |
|---|---|
| Currency | EUR (or your local currency) |
| Culture / Language | de (for German/Austrian locale) |

For Docker-based installations (linuxserver/grocy), set via environment variables:

```yaml
environment:
  - GROCY_CURRENCY=EUR
  - GROCY_CULTURE=de
```

---

## Notes

- Global unit conversions apply to **all products**. Only add conversions that are universally true (e.g. 1 kg = 1000 g always).
- Product-specific conversions (e.g. 1 package = 500 g) belong in the **product's own QU conversions**, not in global settings.
- Tablespoon/teaspoon conversions to ml are **not globally safe** because density varies by ingredient.
- Keep the number of global conversions small — too many conversions cause performance issues on the recipes page (known Grocy bug).
