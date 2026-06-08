---
name: catalog-feed
description: Convert a product catalog into Moloco MCM's catalog feed format (SSPI / MSPI / Delivery) and validate it before submission. Use when building or debugging catalog feed CSVs, mapping a third-party or in-house catalog to Moloco's fields, or fixing catalog validation errors.
---

# MCM Catalog Feed

You are helping a partner developer produce a valid Moloco MCM **catalog feed**. The feed is a CSV (or gzip-compressed `.csv.gz`) of item and seller metadata that Moloco ingests to train ranking models and to sync inventory for ad campaigns. A correctly formatted feed is a prerequisite for running ads, and small format mistakes (wrong URL scheme, wrong price format, empty category segments) are the most common cause of rejected feeds.

The field schemas and validation rules below are the exact rules enforced by Moloco's Catalog Feed Validator. Treat them as authoritative.

## Pick the feed type first

| Feed type | When to use | Feeds produced |
|---|---|---|
| **SSPI** (Single Seller Per Item) | Each item has exactly one seller | One feed (item + seller + price in the same row) |
| **MSPI** (Multiple Sellers Per Item) | An item can be sold by many sellers | **Two feeds**: an Item feed + a Seller feed, joined on item id |
| **Delivery** | Food delivery / restaurants | One feed (uses `average_order_value` instead of price) |

Moloco recommends **MSPI even for current single-seller platforms** — it is the industry-standard, future-proof choice. Migrating SSPI → MSPI later requires significant coordination and potential downtime.

For each feed type there is a matching validator route (see [Validation](#validation)): `sspi-item`, `mspi-item`, `mspi-seller`, `delivery-item`.

## Field reference

Notation: **Required** fields fail validation if missing or empty. **Recommended** fields improve ad performance and should be included when available. **Optional** fields are validated only when present. Header names are matched case-insensitively; use the lowercase names below. To represent a null value, omit the field entirely — an empty string in a required field fails validation.

### SSPI Item (`sspi-item`)

**Required:** `id` (≤50), `title` (≤200), `image_link` (https URL, ≤2000), `seller_id` (≤50, ID format), `seller_title` (≤200), `price` (numeric), `sale_price` (numeric), `category` (≤750, category format)

**Recommended:** `availability` (`in_stock`|`out_of_stock`|`preorder`|`backorder`), `rating` (0–10, ≤1 decimal), `review_count` (integer ≥0), `brand` (≤70), `brand_id` (≤50), `location` (≤750, location format), `adult` (`Y`|`N`), `created_time` (ISO 8601), `updated_time` (ISO 8601)

**Optional:** `business_type` (≤750), `description` (≤5000), `blocked` (≤50), `link` (https, ≤1024), `mobile_link` (https, ≤1024), `additional_image_link` (https, ≤1024), `shipping_charge` (numeric), `reward_point` (≤100), `google_product_category` (≤750), `item_group_id` (≤50), `color` (≤100), `gender` (`unisex`|`male`|`female`), `size` (≤100), `material` (≤200), `pattern` (≤100), `condition` (`new`|`used`|`refurbished`), `age_group` (`all`|`adult`|`teen`|`kids`|`toddler`|`infant`|`newborn`), `class` (`N`|`U`|`D`), `delivery_option` (`one_day`|`same_day`|`regular`|`dawn`), `is_bundle` (`Y`|`N`), `gtin` (≤200), `custom_label_0`…`custom_label_4` (each ≤200)

### MSPI Item (`mspi-item`)

The Item feed holds **product metadata only — no seller or price fields**. Those live in the Seller feed.

**Required:** `id` (≤50), `title` (≤200), `image_link` (https, ≤2000), `category` (≤750, category format)

**Recommended:** `rating`, `review_count`, `brand` (≤70), `brand_id` (≤70), `adult` (`Y`|`N`), `created_time`, `updated_time`

**Optional:** same set as SSPI Item optional, **plus** `video_availability` (`Y`|`N`) and `video_id` (id-list format). Cross-field rule: `video_availability` must be `Y` if and only if `video_id` is populated. The MSPI Item feed must **not** contain `seller_id`, `seller_title`, `price`, or `sale_price`.

### MSPI Seller (`mspi-seller`)

**Required:** `item_id` (≤50, joins to Item feed `id`), `seller_id` (≤50, ID format), `seller_title` (≤200), `price` (numeric), `sale_price` (numeric)

**Recommended:** `updated_time`

**Optional:** `business_type`, `location`, `availability`, `blocked`, `shipping_charge`, `condition`, `delivery_option`, `class` (≤32, `N`|`U`|`D`). If absent, these inherit from the Item feed.

### Delivery Item (`delivery-item`)

**Required:** `id`, `title`, `image_link` (https), `seller_id` (ID format), `seller_title`, `average_order_value` (numeric — replaces `price`/`sale_price`), `category`

**Recommended:** `delivery_fee_avg` (numeric)

**Delivery-only optional:** `delivery_charge`, `menus` (≤20000)

## Format rules

These are the exact value-format rules the validator enforces. Most rejected feeds fail one of these.

- **URLs (`image_link`, `link`, `mobile_link`, `additional_image_link`)** — must start with `https://` and have a valid host. `http://` and protocol-relative `//` URLs are rejected.
- **`seller_id`** — only `a-z A-Z 0-9 - _ .` (alphanumerics, hyphen, underscore, dot).
- **`category` / `location`** — `>` separates hierarchy depth, `;` separates multiple values. No segment may be empty: `A>>B`, `A;`, and leading/trailing delimiters all fail. Example: `Electronics>Accessories>Cables;Computers>Components`.
- **`price` / `sale_price` / `shipping_charge` / `average_order_value`** — numeric ≥ 0. **For KRW and JPY: integers only (no decimal point).** All other currencies: at most 2 decimal places. The currency is set on the Moloco side per ad account.
- **`rating`** — number 0–10; if a decimal point is present, exactly one decimal digit (`4.5` ok, `4.55` fails).
- **Datetimes (`created_time`, `updated_time`)** — ISO 8601 / RFC 3339, max 25 characters. Accepted forms include `2023-01-15T12:30:00Z`, `2023-01-15T12:30:00+09:00`, `2023-01-15T12:30+0000`.
- **`business_type`** — `;`-delimited, each segment ≤100 chars.
- **`video_id`** — `;`-delimited, up to 50 IDs, each ≤128 chars.

Supported currencies: USD, KRW, JPY, EUR, GBP, SEK, INR, THB, IDR, CNY, CAD, RUB, BRL, SGD, HKD, AUD, PLN, DKK, VND, MYR, PHP, TRY, VES, AED.

### Construction guidelines

- Encode the file as **UTF-8**.
- Use `;` to separate list entries within a field.
- Quote any field that contains a comma (`"a, b"`).
- Trim leading/trailing whitespace; keep single spaces between words.
- Minimum update interval: once per hour.

## Update and delete rows

Include a `class` column to mark each row: `N` = new, `U` = update, `D` = delete.

- For an **update** (`U`) or **delete** (`D`) row, only `id` (plus `class`) is required — other fields can be omitted.
- A delete row validates `id` only.

## Validation

Before submitting a feed, validate it with Moloco's **Catalog Feed Validator** (see `mcm-docs.moloco.com/docs/catalog-feed-validator`). The validator enforces exactly the rules above.

- **Endpoint:** `POST /validate/{feedType}` where `feedType` ∈ `sspi-item`, `mspi-item`, `mspi-seller`, `delivery-item`, `google-item`, `naver-item`. The request body is the raw CSV content. Max body size **10 MB** (the online tool caps the number of rows it previews; the full file goes through production ingestion).
- **Response:** a JSON `summary` (`validation_result`: `ok` | `warning` | `error`, plus column count, processed record count, error/warning counts), `header_validations`, and `record_validations`.
- **Error message shape:** `Row 'N': Field 'X' is violating rule 'R' with value 'V'`.
- **Severity:** any error → `error`; otherwise any warning → `warning`; otherwise `ok`. Unknown columns and missing required columns are reported as header validations.

When generating a feed, self-check every row against the [Format rules](#format-rules) before pointing the partner at the validator.

## Converting a third-party feed

The validator accepts two common third-party formats directly, so a partner can validate their native feed without rewriting it.

**Google Merchant (`google-item`)** — Recommended path: pair the Google feed with MSPI and add a separate Seller feed (no edits to the Google file). For SSPI you must append `seller_id` + `seller_title`. Note the enum differences: Google uses `yes`/`no` for `adult` and `is_bundle` (not `Y`/`N`), and `condition` order is `new`|`refurbished`|`used`. Category comes from `product_type` or `google_product_category`.

**Naver EP (`naver-item`)** — SSPI only (MSPI not supported). Field mapping: `normal_price` → `price`, `price_pc` → `sale_price`; `category_name1..4` merge hierarchically with `>`; use `update_time` (not `updated_time`). Enums are Korean: `gender` ∈ 남자|여자|남녀공용, `condition` ∈ 신상품|중고, `age_group` ∈ 유아|아동|청소년|성인. Use `;` for multi-value (not `|`).

For any other in-house catalog, map your fields onto the SSPI or MSPI schema above, then validate with the matching route.

## Delivering the feed to Moloco

Production ingestion is **file-based**, not a row-by-row API:

- **Transport:** Amazon S3, Google Cloud Storage, or an HTTPS URL.
- **Auth:** IP allow list, access credentials, GCS federated identity, or HTTP authentication. (HTTP Authentication and IP Allow List cannot be combined.)
- **Format:** CSV or `.csv.gz`, UTF-8. Example path: `s3://<bucket>/catalog-item/full-update/latest.csv.gz`.
- **Setup:** create the bucket, grant Moloco read access, upload, then share the path and export schedule with your Moloco account manager.

## Common pitfalls

1. **`http://` image URLs** — must be `https://`. Plain `http` or `//` fails.
2. **KRW/JPY prices with decimals** — `19000.00` fails; use `19000`. Other currencies allow up to 2 decimals.
3. **Empty category/location segments** — `A>>B`, `A;`, or a trailing `>` fail; every segment must be non-empty.
4. **Rating with two decimals** — `4.55` fails; use a single decimal (`4.5`) or integer.
5. **Seller/price fields in the MSPI Item feed** — those belong only in the Seller feed. The MSPI Item feed has no price columns.
6. **`video_availability` / `video_id` mismatch** — set `video_availability=Y` exactly when `video_id` is populated; one without the other fails.
7. **Empty string vs null** — omit optional fields entirely to mean null; a blank value in a required field is an error.
8. **Mixing currencies in one feed** — the feed currency must match the ad account currency; don't mix.

## Worked examples

Real validator-passing header + first row for each native feed type.

### SSPI Item

```csv
id,title,image_link,seller_id,seller_title,price,sale_price,category,availability,rating,review_count,brand,brand_id,location,adult,created_time,updated_time
SKU123,"Wireless Mouse",https://example.com/img/sku123.png,seller_42,Example Seller,19,15,Electronics>Accessories>Mice,in_stock,4.5,120,ExampleBrand,EXB123,US>CA>LosAngeles,N,2023-01-01T10:00:00Z,2023-01-15T12:30:00Z
```

### MSPI Item

```csv
id,title,image_link,category,rating,review_count,brand,adult,updated_time,video_availability,video_id
ITEM_XYZ,"Cotton T-shirt",https://example.com/img/xyz.png,Apparel>Clothing>T-shirts,4.7,250,ExampleBrand,N,2025-04-29T14:55:00+09:00,Y,vid1;vid2
```

### MSPI Seller

```csv
item_id,seller_id,seller_title,price,sale_price,updated_time,availability
ITEM_XYZ,seller_abc,"Example Seller",35,30,2025-04-29T14:50:00+09:00,in_stock
```

### Delivery Item

```csv
id,title,image_link,seller_id,seller_title,average_order_value,category,delivery_fee_avg,availability
STORE_007,"Example Restaurant",https://example.com/img/store007.jpg,seller_d7,Example Restaurant,28000,Korean>Stew;Intl>Thai,3500,in_stock
```
