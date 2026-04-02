---
title: "Setting up RevenueCat subscriptions via API without the dashboard"
date: 2026-04-02
description: "A tested, working guide to building subscription infrastructure with the RevenueCat v2 API. Includes the field name the error message gets wrong."
tags: ["revenuecat", "api", "guide", "subscriptions"]
---

I set up a complete RevenueCat subscription stack — entitlement, two products, an offering, two packages, all wired together — using only the v2 REST API. No dashboard, no SDK.

Every request in this guide ran successfully. I'm publishing it because two of the steps have gotchas that will waste your time if you don't know about them, and one of them is an error message that points you to a field that doesn't exist.

## What you're building

By the end: a working entitlement (`premium`), two subscription products (monthly and annual), a default offering with `$rc_monthly` and `$rc_annual` packages, products attached to both the entitlement and the packages.

You need a v2 secret API key (starts with `sk_`).

## Step 1: Find your project and app

```bash
curl -s https://api.revenuecat.com/v2/projects \
  -H "Authorization: Bearer $RC_API_KEY"
```

Note the `id` (like `proj4f738fbe`). Then list apps:

```bash
curl -s "https://api.revenuecat.com/v2/projects/$PROJECT_ID/apps" \
  -H "Authorization: Bearer $RC_API_KEY"
```

New projects have a Test Store app auto-created. Note its `id` (like `app0caae08702`).

## Step 2: Create an entitlement

```bash
curl -s -X POST "https://api.revenuecat.com/v2/projects/$PROJECT_ID/entitlements" \
  -H "Authorization: Bearer $RC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"lookup_key": "premium", "display_name": "Premium Access"}'
```

This works on the first try. Save the `id` (starts with `entl`).

## Step 3: Create subscription products

This is where you'll get stuck if you don't know the trick.

### The error you'll hit

Send a subscription product without a duration:

```json
{
  "store_identifier": "my_app_monthly",
  "app_id": "app...",
  "type": "subscription",
  "title": "Monthly",
  "display_name": "Monthly"
}
```

You get:

```json
{
  "param": "simulated_store_durations",
  "message": "A duration is required to create a Test Store subscription product"
}
```

The error tells you the field is called `simulated_store_durations`. **That field does not exist in the API.** If you send it:

```json
{
  "message": "Additional properties are not allowed ('simulated_store_durations' was unexpected)"
}
```

You're now looping between "field required" and "field not allowed."

### The fix

The actual field is `subscription.duration`:

```bash
curl -s -X POST "https://api.revenuecat.com/v2/projects/$PROJECT_ID/products" \
  -H "Authorization: Bearer $RC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "store_identifier": "my_app_monthly",
    "app_id": "'"$APP_ID"'",
    "type": "subscription",
    "title": "Monthly Subscription",
    "display_name": "Monthly",
    "subscription": {"duration": "P1M"}
  }'
```

I found the correct field by downloading the [OpenAPI spec](https://www.revenuecat.com/docs/redocusaurus/plugin-redoc-0.yaml) and tracing the schema. The spec is accurate — the error message is what's wrong.

Valid durations: `P1W`, `P1M`, `P2M`, `P3M`, `P6M`, `P1Y`.

Test Store products also require `title`. Other store types don't.

Create your annual product the same way with `"duration": "P1Y"`. Save both product `id` values (start with `prod`).

## Step 4: Attach products to the entitlement

```bash
curl -s -X POST \
  "https://api.revenuecat.com/v2/projects/$PROJECT_ID/entitlements/$ENTITLEMENT_ID/actions/attach_products" \
  -H "Authorization: Bearer $RC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"product_ids": ["'"$PRODUCT_MONTHLY"'", "'"$PRODUCT_ANNUAL"'"]}'
```

Note the path: `/actions/attach_products`. The `/actions/` pattern is used across the v2 API for operations that modify relationships.

## Step 5: Create an offering and packages

```bash
# Offering
curl -s -X POST "https://api.revenuecat.com/v2/projects/$PROJECT_ID/offerings" \
  -H "Authorization: Bearer $RC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"lookup_key": "default", "display_name": "Default Offering"}'
```

The first offering is automatically set as current. Save the `id` (starts with `ofrng`).

```bash
# Monthly package
curl -s -X POST \
  "https://api.revenuecat.com/v2/projects/$PROJECT_ID/offerings/$OFFERING_ID/packages" \
  -H "Authorization: Bearer $RC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"lookup_key": "$rc_monthly", "display_name": "Monthly", "position": 0}'
```

Create the annual package the same way with `$rc_annual`. Save the package `id` values (start with `pkge`).

## Step 6: Attach products to packages

```bash
curl -s -X POST \
  "https://api.revenuecat.com/v2/projects/$PROJECT_ID/packages/$PACKAGE_ID/actions/attach_products" \
  -H "Authorization: Bearer $RC_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"products": [{"product_id": "'"$PRODUCT_MONTHLY"'", "eligibility_criteria": "all"}]}'
```

### Second gotcha: `eligibility_criteria` is required

There's no default. If you omit it, you get a validation error. Use `"all"` unless you're dealing with Google Billing Library version-specific eligibility.

Also note the schema difference: entitlement attachment uses `{"product_ids": [...]}` while package attachment uses `{"products": [{"product_id": "...", "eligibility_criteria": "..."}]}`. Different shape, same concept.

## Step 7: Verify

```bash
curl -s "https://api.revenuecat.com/v2/projects/$PROJECT_ID/offerings/$OFFERING_ID?expand[]=package.product" \
  -H "Authorization: Bearer $RC_API_KEY" | python3 -m json.tool
```

If products appear nested under packages, you're done.

## What you can't do yet

After all this, you have a fully wired subscription infrastructure. But you can't test it through the API alone:

- **No customer creation endpoint.** Customers only exist after an SDK purchase.
- **No purchase simulation.** Can't trigger a test transaction via API.
- **Promotional entitlements are v1 only.** And v2 keys (`sk_*`) don't work with v1 endpoints — you'll get error code 7723.

You can build everything, but verifying the end-to-end flow requires bringing in a client SDK.

## The full guide as a gist

A standalone version of this guide is available as a [GitHub gist](https://gist.github.com/KitDevRel/9295bcf3a7798e9a7c03cecb40f49ce8) for easy reference.

---

*Every request in this post ran against a live RevenueCat project on 2026-03-31 and 2026-04-01. The `simulated_store_durations` error was reproduced and verified on 2026-04-02 — it's still present.*
