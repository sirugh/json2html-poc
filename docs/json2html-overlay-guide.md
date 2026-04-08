# Using json2html as an Overlay for Commerce Pages on EDS

This guide walks through configuring the **json2html worker** as a BYOM overlay on an Adobe Commerce EDS storefront site. The overlay can dynamically generate Product Detail Pages (PDP) (and more!) from live Commerce data using Mustache templates.

This guide uses a mixture of placeholders and example values - make sure to double check before running the commands.

For a TLDR guide, see [the simplified version](./json2html-guide-simple.md).

Here's some example pages published through this pipeline:
 - Simple Product: https://main--json2html-poc--sirugh.aem.page/products/bezier-mega-tumbler/adb247
 - Complex Product: https://main--json2html-poc--sirugh.aem.page/products/ssg-configurable-product/ssgconfig123

---

## How It Works

The json2html worker (`json2html.adobeaem.workers.dev`) sits as a BYOM overlay. On preview or publish, EDS will check this overlay and use the HTML result instead of content from the site's content source (usually DA). If the overlay does not return a result, EDS will fall through to the site's content source. The overlay will be configured to make an API call against the configured endpoint, and render the HTML using the configured template.

<img width="2348" height="1022" alt="image" src="https://github.com/user-attachments/assets/dffcf720-39ba-4042-b2af-bdb4350f9e91" />


---

## Considerations

- The setup in this POC is not scalable because every preview and publish will hit the API separately.
- A real setup would require a system which invokes preview or publish on commerce data change.

---

## Prerequisites

- **AEM Admin API token** — needed for both Config Service and json2html configuration. Obtain from your AEM admin or the [login page](https://admin.hlx.page/login).
- **Repository access** — push rights to `<helix-org>/<helix-site>` on GitHub.
- **Commerce backend** — Adobe Commerce Catalog Service GraphQL at `https://catalog-service.adobe.io/graphql` (set store, environment, and API headers in json2html config to match your project).
- **AEM CLI** (optional) — for local development: `npm install -g @adobe/aem-cli`

---

## Step 1: Create the Mustache Templates

Templates live in your GitHub repo and are fetched by the json2html worker at render time via the site's CDN. You can validate your template with the [json2html simulator](https://tools.aem.live/tools/json2html-simulator/index.html). Pass it a GQL response from a product query, such as one from [this page](https://www.aemshop.net/products/bezier-mega-tumbler/adb247).

### 1. Product Detail Page Template

Create `templates/product-detail.html` in the repo root. Here's an example template:

```html
{{#data}}
{{#products}}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="sku" content="{{sku}}">
    <meta property="og:type" content="product">
    <meta property="og:title" content="{{#metaTitle}}{{metaTitle}}{{/metaTitle}}{{^metaTitle}}{{name}}{{/metaTitle}}">
    {{#url}}<link rel="canonical" href="{{{url}}}">{{/url}}
    {{#url}}<meta property="og:url" content="{{{url}}}">{{/url}}
    <title>{{#metaTitle}}{{metaTitle}}{{/metaTitle}}{{^metaTitle}}{{name}}{{/metaTitle}}</title>
    {{#metaDescription}}<meta name="description" content="{{metaDescription}}">{{/metaDescription}}
    {{^metaDescription}}{{#shortDescription}}<meta name="description" content="{{shortDescription}}">{{/shortDescription}}{{/metaDescription}}
    {{#metaKeyword}}<meta name="keywords" content="{{metaKeyword}}">{{/metaKeyword}}
    {{#metaDescription}}<meta property="og:description" content="{{metaDescription}}">{{/metaDescription}}
    {{^metaDescription}}{{#shortDescription}}<meta property="og:description" content="{{shortDescription}}">{{/shortDescription}}{{/metaDescription}}
    {{#images.0}}<meta property="og:image" content="{{{url}}}">{{#label}}<meta property="og:image:alt" content="{{label}}">{{/label}}{{/images.0}}
    <meta name="twitter:card" content="summary_large_image">
    <meta name="twitter:title" content="{{#metaTitle}}{{metaTitle}}{{/metaTitle}}{{^metaTitle}}{{name}}{{/metaTitle}}">
    {{#metaDescription}}<meta name="twitter:description" content="{{metaDescription}}">{{/metaDescription}}
    {{^metaDescription}}{{#shortDescription}}<meta name="twitter:description" content="{{shortDescription}}">{{/shortDescription}}{{/metaDescription}}
    {{#images.0}}<meta name="twitter:image" content="{{{url}}}">{{/images.0}}
  </head>
  <body>
    <header></header>
    <main>
      <div>
        <div class="product-details">
          <div>
            <div>
              <h1>{{name}}</h1>
            </div>
          </div>
          {{#images.0}}
          <div>
            <img src="{{{url}}}" alt="{{label}}">
          </div>
          {{/images.0}}
          {{#description}}
          <div>
            <div><h2>Description</h2></div>
            <div>{{description}}</div>
          </div>
          {{/description}}
          {{#priceRange}}
          <div>
            <div><h2>Price</h2></div>
            <div>{{#minimum}}{{#final}}{{#amount}}{{value}} {{currency}}{{/amount}}{{/final}}{{/minimum}}</div>
          </div>
          {{/priceRange}}
          {{#price}}
          <div>
            <div><h2>Price</h2></div>
            <div>{{#final}}{{#amount}}{{value}} {{currency}}{{/amount}}{{/final}}</div>
          </div>
          {{/price}}
          {{#options.length}}
          <div>
            <div><h2>Options</h2></div>
            <div>
              <ul>
                {{#options}}
                <li>
                  <h3>{{title}}</h3>
                  option id <em>{{id}}</em>
                  required <em>{{required}}</em>
                  <ul>
                    {{#values}}
                    <li>
                      {{#url}}
                      <a href="{{{url}}}">{{title}} {{#inStock}}in stock{{/inStock}}{{^inStock}}out of stock{{/inStock}}</a>
                      {{/url}}
                      {{^url}}
                      {{title}} {{#inStock}}in stock{{/inStock}}{{^inStock}}out of stock{{/inStock}}
                      {{/url}}
                    </li>
                    {{/values}}
                  </ul>
                </li>
                {{/options}}
              </ul>
            </div>
          </div>
          {{/options.length}}
          {{^options.length}}
          <div>
            <h2>Availability</h2>
            <div>{{#inStock}}in stock{{/inStock}}{{^inStock}}out of stock{{/inStock}}</div>
          </div>
          {{/options.length}}
        </div>
      </div>
    </main>
    <footer></footer>
  </body>
</html>
{{/products}}
{{/data}}
```

### Template Structure Notes

- `<header></header>` and `<footer></footer>` are empty — EDS injects the site's global header/footer automatically.
- Block markup follows EDS conventions: `<div class="block-name">` with child `<div>` rows and columns.
- `{{{description}}}` and `{{{url}}}`s uses triple-braces for unescaped HTML content.
- The PDP template wraps in `{{#data}}{{#products}}` to traverse the Catalog Service GraphQL response shape.

---

## Step 2: Commit and Push Templates

```bash
cd <your code repo>
mkdir -p templates
# (create the template files as above)
git add templates/product-detail.html
git commit -m "Add json2html Mustache template for PDP"
git push origin main
```

After pushing, the template becomes available at:
- `https://main--<helix-site>--<helix-org>.aem.live/templates/product-detail.html`

The json2html worker fetches templates from the site CDN, so they must be pushed and (optionally) previewed before the overlay will work.

---

## Step 3: Configure the Overlay via Config Service

The overlay tells EDS to fall through to the json2html worker when the primary source (DA) doesn't have a page. Overlays **must** be configured via the Config Service API. `fstab.yaml` is only used during initial onboarding; afterward it is unused, and you should delete it from the repository. Runtime routing and folder mapping come from **site configuration** in Config Service, not from `fstab.yaml`.

```bash
# Replace <YOUR_AUTH_TOKEN> with your AEM Admin API token
curl -X POST \
  "https://admin.hlx.page/config/<helix-org>/sites/<helix-site>/content.json" \
  -H "Content-Type: application/json" \
  -H "x-auth-token: <YOUR_AUTH_TOKEN>" \
  --data '{
    "overlay": {
      "url": "https://json2html.adobeaem.workers.dev/<helix-org>/<helix-site>/main",
      "type": "markup"
    }
  }'
```

### Verify the overlay configuration

```bash
curl -s "https://admin.hlx.page/config/<helix-org>/sites/<helix-site>.json" \
  -H "x-auth-token: <YOUR_AUTH_TOKEN>" | jq .
```

You should see the overlay URL in the `content` section of the response.

### Important: Folder Mapping Conflict

Your site may still have **Folder Mapping** in **site configuration** (synced via Config Service), for example:

```yaml
"folders": {
  "/products/": "/products/default"
}
```

That maps **every** `/products/*` request to the same document in DA, so the primary source always returns content for product URLs and **the overlay never runs**. To fix it, remove the `folders` property (or the `/products/` entry) from your site configuration using the same Config Service workflows you use for other site settings — that is what EDS applies at runtime.

`fstab.yaml` in the Git repository is only involved in **initial onboarding**. After onboarding, it is effectively unused for routing; live behavior comes from Config Service. You may delete `fstab.yaml` from the repo as cleanup if it is still there, but doing so does **not** by itself change folder mapping — updating site config does. Treat any remaining `fstab.yaml` as optional dead weight, not the lever for this change.

---

## Step 4: Configure the json2html Worker

This tells the json2html worker which URL patterns to handle, where to fetch data, and which template to use.

### 4a. The GraphQL Query

The PDP uses the full Catalog Service product query (the same one the storefront drop-in uses). Here it is decoded for reference:

```graphql
query GET_PRODUCT_DATA($skus: [String]) {
  products(skus: $skus) {
    ...PRODUCT_FRAGMENT
  }
}

fragment PRODUCT_FRAGMENT on ProductView {
  __typename id sku name shortDescription metaDescription metaKeyword metaTitle
  description inStock addToCartAllowed url urlKey externalId
  images(roles: []) { url label roles }
  videos { description url title preview { label roles url } }
  attributes(roles: []) { name label value roles }
  ... on SimpleProductView {
    price {
      roles
      regular { amount { value currency } }
      final { amount { value currency } }
      tiers { tier { amount { value currency } } quantity { ... on ProductViewTierRangeCondition { gte lt } } }
    }
  }
  ... on ComplexProductView {
    options { ...PRODUCT_OPTION_FRAGMENT }
    ...PRICE_RANGE_FRAGMENT
  }
}

fragment PRODUCT_OPTION_FRAGMENT on ProductViewOption {
  id title required multi
  values {
    id title inStock __typename
    ... on ProductViewOptionValueProduct {
      title quantity isDefault __typename
      product {
        sku shortDescription metaDescription metaKeyword metaTitle name
        price { final { amount { value currency } } regular { amount { value currency } } roles }
      }
    }
    ... on ProductViewOptionValueSwatch { id title type value inStock }
  }
}

fragment PRICE_RANGE_FRAGMENT on ComplexProductView {
  priceRange {
    maximum { final { amount { value currency } } regular { amount { value currency } } roles }
    minimum { final { amount { value currency } } regular { amount { value currency } } roles }
  }
}
```

The query parameter for the Catalog Service endpoint (URL-encoded, `+` for spaces):

```
query+GET_PRODUCT_DATA%28%24skus%3A+%5BString%5D%29+%7B+products%28skus%3A+%24skus%29+%7B+...PRODUCT_FRAGMENT+%7D+%7D+fragment+PRODUCT_FRAGMENT+on+ProductView+%7B+__typename+id+sku+name+shortDescription+metaDescription+metaKeyword+metaTitle+description+inStock+addToCartAllowed+url+urlKey+externalId+images%28roles%3A+%5B%5D%29+%7B+url+label+roles+%7D+videos+%7B+description+url+title+preview+%7B+label+roles+url+%7D+%7D+attributes%28roles%3A+%5B%5D%29+%7B+name+label+value+roles+%7D+...+on+SimpleProductView+%7B+price+%7B+roles+regular+%7B+amount+%7B+value+currency+%7D+%7D+final+%7B+amount+%7B+value+currency+%7D+%7D+tiers+%7B+tier+%7B+amount+%7B+value+currency+%7D+%7D+quantity+%7B+...+on+ProductViewTierRangeCondition+%7B+gte+lt+%7D+%7D+%7D+%7D+%7D+...+on+ComplexProductView+%7B+options+%7B+...PRODUCT_OPTION_FRAGMENT+%7D+...PRICE_RANGE_FRAGMENT+%7D+%7D+fragment+PRODUCT_OPTION_FRAGMENT+on+ProductViewOption+%7B+id+title+required+multi+values+%7B+id+title+inStock+__typename+...+on+ProductViewOptionValueProduct+%7B+title+quantity+isDefault+__typename+product+%7B+sku+shortDescription+metaDescription+metaKeyword+metaTitle+name+price+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+%7D+%7D+...+on+ProductViewOptionValueSwatch+%7B+id+title+type+value+inStock+%7D+%7D+%7D+fragment+PRICE_RANGE_FRAGMENT+on+ComplexProductView+%7B+priceRange+%7B+maximum+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+minimum+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+%7D+%7D
```

The variables parameter uses `"skus"` (an array). The SKU is passed via the `x-content-source-location` header (see below for why):

```
variables=%7B%22skus%22%3A%5B%22{{headers['x-content-source-location']}}%22%5D%7D
```

Which decodes to: `variables={"skus":["{{headers['x-content-source-location']}}"]}`

### 4b. Why `x-content-source-location` Instead of URL-based ID

**EDS lowercases and sanitizes all paths.** A product URL like `/products/bezier-mega-tumbler/ADB247` becomes `/products/bezier-mega-tumbler/adb247` by the time the overlay receives the request. Since the Commerce Catalog Service is **case-sensitive**, querying with `adb247` instead of `ADB247` returns no results.

The workaround: use the `x-content-source-location` header to carry the canonical (case-preserved) SKU. This header:
- Is passed when invoking preview/publish via the Admin API
- Is forwarded to the json2html worker via `forwardHeaders`
- Is injected into the endpoint URL via `{{headers['x-content-source-location']}}`

### 4c. POST the Configuration

Example configuration for POSTing to the json2html worker _(note the variable injection syntax for "x-content-source-location")_:

```bash
curl --request POST \
	--url https://json2html.adobeaem.workers.dev/config/<helix-org>/<helix-site>/main \
	--header 'Authorization: token <TOKEN_HERE>' \
	--header 'Content-Type: application/json' \
	--data '[
		{
		"path": "/products/",
		"endpoint": "https://catalog-service.adobe.io?query=query+GET_PRODUCT_DATA%28%24skus%3A+%5BString%5D%29+%7B+products%28skus%3A+%24skus%29+%7B+...PRODUCT_FRAGMENT+%7D+%7D+fragment+PRODUCT_FRAGMENT+on+ProductView+%7B+__typename+id+sku+name+shortDescription+metaDescription+metaKeyword+metaTitle+description+inStock+addToCartAllowed+url+urlKey+externalId+images%28roles%3A+%5B%5D%29+%7B+url+label+roles+%7D+videos+%7B+description+url+title+preview+%7B+label+roles+url+%7D+%7D+attributes%28roles%3A+%5B%5D%29+%7B+name+label+value+roles+%7D+...+on+SimpleProductView+%7B+price+%7B+roles+regular+%7B+amount+%7B+value+currency+%7D+%7D+final+%7B+amount+%7B+value+currency+%7D+%7D+tiers+%7B+tier+%7B+amount+%7B+value+currency+%7D+%7D+quantity+%7B+...+on+ProductViewTierRangeCondition+%7B+gte+lt+%7D+%7D+%7D+%7D+%7D+...+on+ComplexProductView+%7B+options+%7B+...PRODUCT_OPTION_FRAGMENT+%7D+...PRICE_RANGE_FRAGMENT+%7D+%7D+fragment+PRODUCT_OPTION_FRAGMENT+on+ProductViewOption+%7B+id+title+required+multi+values+%7B+id+title+inStock+__typename+...+on+ProductViewOptionValueProduct+%7B+title+quantity+isDefault+__typename+product+%7B+sku+shortDescription+metaDescription+metaKeyword+metaTitle+name+price+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+%7D+%7D+...+on+ProductViewOptionValueSwatch+%7B+id+title+type+value+inStock+%7D+%7D+%7D+fragment+PRICE_RANGE_FRAGMENT+on+ComplexProductView+%7B+priceRange+%7B+maximum+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+minimum+%7B+final+%7B+amount+%7B+value+currency+%7D+%7D+regular+%7B+amount+%7B+value+currency+%7D+%7D+roles+%7D+%7D+%7D&variables=%7B%22skus%22%3A%5B%22{{headers['"'"'x-content-source-location'"'"']}}%22%5D%7D",
		"template": "/templates/product-detail.html",
		"headers": {
		  "Magento-Store-Code": "your-store-code",
		  "Magento-Store-View-Code": "your-store-view-code",
		  "Magento-Website-Code": "your-website-code",
		  "x-api-key": "your-api-key",
		  "Magento-Environment-Id": "your-environment-id",
		  "Content-Type": "application/json"
		},
		"forwardHeaders": ["x-content-source-location"]
		}
	]'
```

> **How forwardHeaders works:** When preview is invoked for `/products/bezier-mega-tumbler/ADB247` with header `x-content-source-location: ADB247`, EDS lowercases the path but the header preserves the original SKU. The json2html worker substitutes the header value into the endpoint URL (becoming `"skus":["ADB247"]`), fetches the JSON from the Catalog Service, and renders it with the PDP template.

---

## Step 5: Test with Manual Preview/Publish

### 5a. Preview a Product Page

You need to know a valid product SKU that exists in the Commerce backend. **The `x-content-source-location` header must be included** — it carries the case-preserved SKU to the json2html worker.

```bash
# Preview a specific product page
curl -X POST \
  "https://admin.hlx.page/preview/<helix-org>/<helix-site>/main/products/bezier-mega-tumbler/adb247" \
  -H "x-auth-token: <YOUR_AUTH_TOKEN>" \
  -H "x-content-source-location: ADB247"
```

If successful, you can view the preview at:
```
https://main--<helix-site>--<helix-org>.aem.page/products/bezier-mega-tumbler/adb247
```

### 5b. Publish a Product Page

```bash
curl -X POST \
  "https://admin.hlx.page/live/<helix-org>/<helix-site>/main/products/bezier-mega-tumbler/adb247" \
  -H "x-auth-token: <YOUR_AUTH_TOKEN>" \
  -H "x-content-source-location: ADB247"
```

The published page is available at:
```
https://main--<helix-site>--<helix-org>.aem.live/products/bezier-mega-tumbler/adb247
```

### 5c. Debug Issues

If the page doesn't render correctly:

1. **Check the json2html worker directly** (pass the header manually):
   ```bash
   curl -s "https://json2html.adobeaem.workers.dev/<helix-org>/<helix-site>/main/products/bezier-mega-tumbler/adb247" \
     -H "x-content-source-location: ADB247" | head -50
   ```
   Note the lowercased path — this simulates what EDS sends. The header carries the real SKU.

2. **Check the template is accessible:**
   ```bash
   curl -s "https://main--<helix-site>--<helix-org>.aem.live/templates/product-detail.html" | head -20
   ```

3. **Check the GraphQL endpoint directly:**

For example:

   ```bash
   curl -s "https://catalog-service.adobe.io/graphql" \
     -H "Magento-Store-Code: main_website_store" \
     -H "Magento-Store-View-Code: default" \
     -H "x-api-key: 4dfa19c9fe6f4cccade55cc5b3da94f7" \
     -H "Magento-Environment-Id: f38a0de0-764b-41fa-bd2c-5bc2f3c7b39a" \
     -H "Content-Type: application/json" \
     -d '{"query":"{ products(skus: [\"ADB247\"]) { name sku } }"}' | jq .
   ```

4. **Check the overlay configuration:**
   ```bash
   curl -s "https://admin.hlx.page/config/<helix-org>/sites/<helix-site>.json" \
     -H "x-auth-token: <YOUR_AUTH_TOKEN>" | jq '.content'
   ```

---

## Step 6: Roll out to all product pages

Step 5 walks through **one** product. In production, every product URL under `/products/` must be **previewed** and, when you are ready to go live, **published** through the same Admin API pattern (`POST` preview or `POST` live, with `x-content-source-location` set to the canonical SKU for each path, matching the lowercased path segment in the URL).

- **Small catalogs:** You can repeat the preview and publish `curl` calls manually for each `urlKey` / SKU pair (or track them in a short checklist).
- **Large catalogs:** Manual calls are impractical.Build an automation script you can run against your catalog to drive bulk preview and publish.
---

## Summary Checklist

| # | Step | Command/Action |
|---|------|---------------|
| 1 | Create PDP template | `templates/product-detail.html` |
| 2 | Push template to GitHub | `git push origin main` |
| 3 | Configure overlay (Config Service) | `POST admin.hlx.page/config/<helix-org>/sites/<helix-site>/content.json` |
| 4 | Configure json2html worker | `POST json2html.adobeaem.workers.dev/config/<helix-org>/<helix-site>/main` |
| 5 | Remove folder mapping (if present) | Remove `folders` from site configuration via Config Service; optionally delete stale `fstab.yaml` from the repo |
| 6 | Preview a test product | `POST admin.hlx.page/preview/<helix-org>/<helix-site>/main/products/{urlKey}/{sku}` |
| 7 | Verify at `.aem.page` URL | Browser: `https://main--<helix-site>--<helix-org>.aem.page/products/{urlKey}/{sku}` |
| 8 | Publish when ready | `POST admin.hlx.page/live/<helix-org>/<helix-site>/main/products/{urlKey}/{sku}` |
| 9 | Roll out to all products | Preview/publish every product URL (manual for small catalogs; script or automation for large catalogs — see **Next steps: All product pages**) |

---

## Next Steps

This document details a basic implementation of the json2html overlay. There are several next steps to consider:

- Minimize the query to just the requisite data for SEO/GEO. Current query probably over-fetches.
- Fix URL generation. Currently using Commerce urls in places, ie `aemshop.net/my-product-page.html`, instead of EDS format url such as `aemshop.net/products/my-product-key/my-product-page`.
- Create an intermediate worker for the json2html worker endpoint that takes `/products?sku={{headers['x-content-source-location']}}`  and contains the minimized query.
- Add additional handling such as for more complex scenarios such as PLPs, marketing/content fragment injections, etc.

## Appendix: Key URLs

| Resource | URL |
|----------|-----|
| json2html worker (CI) | `https://json2html.adobeaem.workers.dev/<helix-org>/<helix-site>/main/PAGE_PATH` |
| json2html config endpoint | `https://json2html.adobeaem.workers.dev/config/<helix-org>/<helix-site>/main` |
| Config Service | `https://admin.hlx.page/config/<helix-org>/sites/<helix-site>` |
| Admin API (preview) | `https://admin.hlx.page/preview/<helix-org>/<helix-site>/main/PAGE_PATH` |
| Admin API (publish) | `https://admin.hlx.page/live/<helix-org>/<helix-site>/main/PAGE_PATH` |
| Preview site | `https://main--<helix-site>--<helix-org>.aem.page/` |
| Live site | `https://main--<helix-site>--<helix-org>.aem.live/` |
| Catalog Service GraphQL | `https://catalog-service.adobe.io/graphql` |
---

## Appendix: Differences from aem-commerce-prerender

| Aspect | aem-commerce-prerender | json2html overlay |
|--------|----------------------|-------------------|
| **Setup complexity** | App Builder project, Runtime actions, storage | Config API calls + templates in repo |
| **Infrastructure** | App Builder, Azure, Runtime | Cloudflare Worker (managed by Adobe) |
| **Storage** | Azure Blob Storage | None (stateless worker) |
| **Change detection** | Polling actions detect catalog changes | Requires custom change detection system |
| **Freshness** | Depends on polling interval (5-60 min) | Depends on your change detection system |
| **Rendering** | Pre-generates HTML, stores in Azure Blob | Generates HTML on-the-fly per request |
