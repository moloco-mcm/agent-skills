---
name: catalog-feed
description: Convert a product catalog into Moloco MCM's catalog feed format (SSPI / MSPI / Delivery) and validate it before submission. Use when building or debugging catalog feed CSVs, mapping a third-party or in-house catalog to Moloco's fields, or fixing catalog validation errors.
---

# MCM Catalog Feed

You are helping a partner developer produce a valid Moloco MCM **catalog feed**. The feed is a CSV (or gzip-compressed `.csv.gz`) of item and seller metadata that Moloco ingests to train ranking models and to sync inventory for ad campaigns. A correctly formatted feed is a prerequisite for running ads.

Treat the public documentation as authoritative. Where this skill summarizes rules, links into `mcm-docs.moloco.com` are provided in [Sources](#sources). Always validate a generated feed with the Catalog Feed Validator before submission.

## Pick the feed type first

| Feed type | When to use | Feeds produced |
|---|---|---|
| **SSPI Common** (Single Seller Per Item) | Each item has exactly one seller | One feed (item + seller + price in the same row) |
| **SSPI Delivery** | Food delivery / restaurants — an SSPI variant | One feed; uses `average_order_value` instead of `price` / `sale_price` |
| **MSPI** (Multiple Sellers Per Item) | An item can be sold by many sellers | **Two feeds**: an Item feed + a Seller feed, joined on item id |

Delivery is documented as one of two SSPI variants ("Common" and "Delivery") on the SSPI specification page; it is not an independent feed type alongside SSPI/MSPI.

Moloco recommends **MSPI even for current single-seller platforms** as the industry-standard, future-proof choice. Migrating SSPI → MSPI later involves significant coordination and potential downtime.

For each feed type, validate with the matching tab on the Catalog Feed Validator (SSPI, MSPI Item, MSPI Seller, Delivery, Google, Naver — see [Validation](#validation)).

## Field reference

Notation: **Required** fields fail validation if missing or empty. **Recommended** fields improve ad performance and should be included when available. **Optional** fields are validated only when present. Header names are matched case-insensitively; use the lowercase names below. To represent a null value, omit the field entirely — an empty string in a required field fails validation (except for `category`, per the construction guidelines).

### SSPI Item (Common)

**Required:** `id` (≤50), `title` (≤200), `image_link` (https URL, ≤2000), `seller_id` (≤50, ID format), `seller_title` (≤200), `price` (numeric), `sale_price` (numeric), `category` (≤750, category format)

**Recommended:** `availability` (`in_stock`|`out_of_stock`|`preorder`|`backorder`), `rating` (one decimal, scale of 5 or 10), `review_count` (integer ≥0), `brand` (≤70), `brand_id` (≤50), `location` (≤750, location format), `adult` (`Y`|`N`), `created_time` (ISO 8601), `updated_time` (ISO 8601)

**Optional:** `business_type`, `description` (≤5000), `blocked` (≤50), `link` (https, ≤1024), `mobile_link` (https, ≤1024), `additional_image_link` (https, ≤1024), `shipping_charge` (numeric), `reward_point` (≤100), `google_product_category` (≤750), `item_group_id` (≤50), `color` (≤100), `gender` (`unisex`|`male`|`female`), `size` (≤100), `material` (≤200), `pattern` (≤100), `condition` (`new`|`used`|`refurbished`), `age_group` (`all`|`adult`|`teen`|`kids`|`toddler`|`infant`|`newborn`), `class` (`N`|`U`|`D`), `delivery_option` (`one_day`|`same_day`|`regular`|`dawn`), `is_bundle` (`Y`|`N`), `gtin` (≤200), `custom_label_0`…`custom_label_4` (each ≤200)

### SSPI Delivery

A variant of SSPI for food-delivery use cases. Same shape as SSPI Common except:

**Required:** `id`, `title`, `image_link` (https), `seller_id` (ID format), `seller_title`, `average_order_value` (numeric — replaces `price`/`sale_price`), `category`

**Recommended:** `delivery_fee_avg` (numeric)

**Delivery-only optional:** `delivery_charge`, `menus` (≤20000)

### MSPI Item

The Item feed holds **product metadata only — no seller or price fields**. Those live in the Seller feed.

**Required:** `id` (≤50), `title` (≤200), `image_link` (https, ≤2000), `category` (≤750, category format)

**Recommended:** `rating`, `review_count`, `brand` (≤70), `brand_id` (≤70), `adult` (`Y`|`N`), `created_time`, `updated_time`

**Optional:** the SSPI Item optional set, with the seller- and price-related fields removed. The MSPI Item feed must **not** contain `seller_id`, `seller_title`, `price`, or `sale_price`.

### MSPI Seller

**Required:** `item_id` (≤50, joins to Item feed `id`), `seller_id` (≤50, ID format), `seller_title` (≤200), `price` (numeric), `sale_price` (numeric)

**Recommended:** `updated_time`

**Optional:** `business_type`, `location`, `availability`, `blocked`, `shipping_charge`, `condition`, `delivery_option`, `class` (`N`|`U`|`D`). If absent in the Seller feed, the value from the Item catalog is used.

## Format rules

These rules are documented on the SSPI / MSPI specification pages and the construction guidelines.

- **URLs (`image_link`, `link`, `mobile_link`, `additional_image_link`)** — must begin with `https://` and follow RFC 2396 / RFC 1738.
- **`seller_id`** — only `a-z A-Z 0-9 - _ .` (alphanumerics, hyphen, underscore, dot).
- **`category` / `location`** — `>` separates hierarchy depth, `;` separates multiple values. No segment may be empty: `A>>B`, `A;`, and leading/trailing delimiters fail. Example: `Electronics>Accessories>Cables;Computers>Components`.
- **`price` / `sale_price` / `shipping_charge` / `average_order_value`** — numeric ≥ 0. **For KRW and JPY: whole numbers only (no decimals).** Other currencies allow up to 2 decimal places.
- **`rating`** — round to one decimal place; use a scale of 5 or 10.
- **Datetimes (`created_time`, `updated_time`)** — ISO 8601, max 25 characters. Documented patterns include `YYYY-MM-DDThh:mmZ` and `YYYY-MM-DDThh:mm+hhmm` (example: `2023-02-24T11:07+0100`).
- **Multi-value fields** — entries are separated by `;`. The construction guidelines cap multi-value array size at 50 entries (except `menus`).

**Currency** — the currency used in the catalog feed must match the currency set on the platform with Moloco. Do not mix currencies within a feed. The full list of currencies Moloco supports is documented on the Multi-Currency Support page (see [Sources](#sources)).

### Construction guidelines

- Encode the file as **UTF-8**.
- Use `;` to separate list entries within a field.
- Quote any field that contains a comma (`"a, b"`).
- Trim leading/trailing whitespace; keep single spaces between words.
- Minimum update interval: once per hour.

## Update and delete rows

Include a `class` column to mark each row: `N` = new, `U` = update, `D` = delete. Required columns for non-`N` rows depend on the feed:

- **SSPI Item / MSPI Item — update (`U`) or delete (`D`)**: only `id` (plus `class`) is required.
- **MSPI Seller — delete (`D`)**: `item_id`, `seller_id`, and `class` are required (the join key cannot be inferred from `id` alone).

## Validation

Validate a feed with Moloco's public **Catalog Feed Validator** before submission. It exposes one tab per feed type: SSPI, MSPI Item, MSPI Seller, Delivery, Google, Naver. The tool handles up to **100 lines or 10 MB** of data per check.

When generating a feed, self-check every row against the [Format rules](#format-rules) before pointing the partner at the validator.

## Converting a third-party feed

The validator accepts two common third-party formats directly, so a partner can validate their native feed without rewriting it.

**Google Merchant** — Recommended path: pair the Google feed with MSPI and add a separate Seller feed (no edits to the Google file). For SSPI you must append `seller_id` + `seller_title`. Enum differences vs Moloco-native: Google uses `yes`/`no` for `adult` and `is_bundle` (mapped to `Y`/`N`), and `condition` order is `new`|`refurbished`|`used`. Category comes from `product_type` (primary) or `google_product_category`.

**Naver EP** — SSPI only (MSPI not supported). Field mapping: `normal_price` → `price`, `price_pc` → `sale_price`; `category_name1..4` merge hierarchically with `>`; use `update_time` (which maps to canonical `updated_time`). Enums are Korean: `gender` ∈ 남자|여자|남녀공용, `condition` ∈ 신상품|중고, `age_group` ∈ 유아|아동|청소년|성인. Use `;` for multi-value (Naver's native `|` delimiter is not supported).

For any other in-house catalog, map your fields onto the SSPI or MSPI schema above, then validate with the matching tab.

## Delivering the feed to Moloco

Production ingestion is **file-based**, not a row-by-row API:

- **Transport:** Amazon S3, Google Cloud Storage, or an HTTPS URL.
- **Auth:** IP Allow List, Access Credentials, GCS Federated Identity, HTTP Authentication, or Public Access. (HTTP Authentication and IP Allow List cannot be combined.)
- **Format:** CSV or `.csv.gz`, UTF-8. Example path: `s3://<bucket>/catalog-item/full-update/latest.csv.gz`.
- **Setup:** create the bucket, grant Moloco read access, upload, then share the path and export schedule with your Moloco account manager.

## Common pitfalls

1. **`http://` image URLs** — must be `https://`.
2. **KRW/JPY prices with decimals** — `19000.00` fails; use `19000`. Other currencies allow up to 2 decimals.
3. **Empty category/location segments** — `A>>B`, `A;`, or a trailing `>` fail; every segment must be non-empty.
4. **Rating with two decimals** — round to one decimal (`4.5`), not `4.55`.
5. **Seller/price fields in the MSPI Item feed** — those belong only in the Seller feed. The MSPI Item feed has no price columns.
6. **Empty string vs null** — omit optional fields entirely to mean null; a blank value in a required field is an error (except `category`, per construction guidelines).
7. **Mixing currencies in one feed** — the feed currency must match the ad account currency; don't mix.
8. **MSPI Seller delete row with only `id`** — the Seller feed's delete row needs `item_id` + `seller_id` + `class`.

## Worked examples

Header + first row for each feed type. The values demonstrate format rules; field names and order can be adjusted to fit your data.

### SSPI Item (Common)

```csv
id,title,image_link,seller_id,seller_title,price,sale_price,category,availability,rating,review_count,brand,brand_id,location,adult,created_time,updated_time
SKU123,"Wireless Mouse",https://example.com/img/sku123.png,seller_42,Example Seller,19,15,Electronics>Accessories>Mice,in_stock,4.5,120,ExampleBrand,EXB123,US>CA>LosAngeles,N,2023-01-01T10:00+0000,2023-01-15T12:30+0000
```

### MSPI Item

```csv
id,title,image_link,category,rating,review_count,brand,adult,updated_time
ITEM_XYZ,"Cotton T-shirt",https://example.com/img/xyz.png,Apparel>Clothing>T-shirts,4.7,250,ExampleBrand,N,2025-04-29T14:55+0900
```

### MSPI Seller

```csv
item_id,seller_id,seller_title,price,sale_price,updated_time,availability
ITEM_XYZ,seller_abc,"Example Seller",35,30,2025-04-29T14:50+0900,in_stock
```

### SSPI Delivery

```csv
id,title,image_link,seller_id,seller_title,average_order_value,category,delivery_fee_avg,availability
STORE_007,"Example Restaurant",https://example.com/img/store007.jpg,seller_d7,Example Restaurant,28000,Korean>Stew;Intl>Thai,3500,in_stock
```

## Sources

This skill is derived from Moloco's public catalog documentation. The validator tool is the authoritative check — when in doubt, validate against it. If any of these pages change, re-check the affected section here.

- Overview & feed types: <https://mcm-docs.moloco.com/docs/catalog-feed>, <https://mcm-docs.moloco.com/docs/sspi-vs-mspi>
- Field specifications: <https://mcm-docs.moloco.com/docs/sspi-catalog-feed-specification>, <https://mcm-docs.moloco.com/docs/mspi-catalog-feed-specification>
- Construction & validation: <https://mcm-docs.moloco.com/docs/catalog-construction-guidelines>, <https://mcm-docs.moloco.com/docs/catalog-feed-validator>
- Delivery to Moloco: <https://mcm-docs.moloco.com/docs/catalog-feed-integration>
- Third-party feeds: <https://mcm-docs.moloco.com/docs/third-party-catalog-feeds>
- Field relevance & filtering: <https://mcm-docs.moloco.com/docs/how-mcm-uses-catalog-fields-for-relevance-and-filtering>
- Multi-currency support: <https://mcm-docs.moloco.com/docs/multi-currency-support>
