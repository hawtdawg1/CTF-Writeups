# Value-Added Tax

**Category:** OSINT

**Flag:** `cdctf{£21.55}`

**Verification:** Recovered or checked against supplied material; checker acceptance was not documented.

## Challenge Overview

A receipt photo provides three item prices. The challenge asks for the VAT-inclusive total.

## Solution

1. Open the provided receipt photo (`image-1.jpg`). Rotate or zoom it to read the store and line items. It is a Five Guys receipt from Oxford, UK, so the currency symbol is `£`.

2. Read the three visible item prices: `£8.65`, `£8.65`, and `£4.25`. Add them: `8.65 + 8.65 + 4.25 = 21.55`. The prompt asks for what the customer paid *plus VAT*, meaning the VAT-inclusive total. [UK retail price guidance](https://www.gov.uk/product-labelling-the-law) states displayed consumer prices include VAT where applicable, so the receipt prices already contain the tax. Adding another 20% would count it twice.

3. Apply the prompt's currency-and-two-decimal format: `cdctf{£21.55}`.

## Flag

```text
cdctf{£21.55}
```
