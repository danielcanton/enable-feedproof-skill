---
name: enable-feedproof
description: Connect a Vercel Shop storefront to FeedProof. Reports AI-agent visits (GPTBot, ClaudeBot, PerplexityBot and others) from the proxy, and adds FeedProof's product details (material, care, origin, dimensions, warranty, condition, GTIN) to the product Markdown and Product JSON-LD.
---

# Enable FeedProof

FeedProof (getfeedproof.com) is a Shopify app that fixes product data for AI shopping agents. This skill connects a Vercel Shop storefront to it in two parts:

- **Agent visits.** `proxy.ts` tells FeedProof when an AI agent fetches a page, which page it was, and whether it read HTML, Markdown or a text file. FeedProof shows the merchant which agents read which products. The report runs in `event.waitUntil` and never delays or fails the response.
- **Product details.** The product query reads the `feedproof.*` metafields that FeedProof writes, plus the default variant's barcode. The product Markdown gets a `## Details` section. Product JSON-LD gets `material`, `countryOfOrigin`, `gtin`, `offers.itemCondition` and `additionalProperty`.

The agent-visit code belongs to Next.js (the request boundary). The metafield read belongs to Shopify (fragment, transform, domain type). Markdown and JSON-LD belong to Next.js. No Eve changes.

## Before you start

Ask the user one question, then carry on whatever the answer. The code builds and runs without a key:

1. **Has the merchant connected the storefront in FeedProof?** In the Shopify admin: FeedProof → Settings → Headless storefront → Connect storefront. Connecting does two things. It shows a `FEEDPROOF_INGEST_KEY`, once, and it makes the `feedproof.*` metafields readable by the Storefront API. Until it runs, the metafields come back `null` and the Details section stays empty. Everything still builds.

The key is a server-only secret. Never write it into source, `.env.example` or the output. The user sets it in Vercel → Settings → Environment Variables, or in `.env.local` for local work.

---

## Part A: Report AI-agent visits

### A1. Create `lib/agent-visits/index.ts`

`lib/agent-visits/` belongs to Next.js (the request boundary), not Eve, despite sitting beside `lib/agent/`.

```ts
export type AgentVisitRepresentation = "html" | "markdown" | "text";

// Deliberately broad: FeedProof classifies the exact agent and drops the rest.
const AUTOMATED_USER_AGENT =
  /bot|crawl|spider|agent|-user|gpt|claude|anthropic|openai|perplexity|cohere|mistral|meta-external|bytespider|ccbot|duckassist/i;

const TEXT_PATHS = new Set(["/llms.txt", "/robots.txt", "/sitemap.md", "/sitemap.xml"]);

export function agentVisitRepresentation(
  pathname: string,
  markdownRewritePath: string | null,
): AgentVisitRepresentation {
  if (markdownRewritePath) return "markdown";
  return TEXT_PATHS.has(pathname) ? "text" : "html";
}

export function isAgentVisitCandidate(
  userAgent: string,
  representation: AgentVisitRepresentation,
): boolean {
  return representation === "markdown" || AUTOMATED_USER_AGENT.test(userAgent);
}
```

### A2. Create `lib/agent-visits/server.ts`

```ts
import type { NextFetchEvent, NextRequest } from "next/server";

import { agentVisitRepresentation, isAgentVisitCandidate } from "@/lib/agent-visits";

const DEFAULT_INGEST_URL = "https://feedproof.fly.dev/api/agent-visits";
const REPORT_TIMEOUT_MS = 2000;

// Never blocks or fails the response, and drops the query string, which can carry search terms.
export function reportAgentVisit(
  request: NextRequest,
  event: NextFetchEvent,
  markdownRewritePath: string | null,
): void {
  const key = process.env.FEEDPROOF_INGEST_KEY;
  if (!key || (request.method !== "GET" && request.method !== "HEAD")) return;

  const userAgent = request.headers.get("user-agent") ?? "";
  const { pathname } = request.nextUrl;
  const representation = agentVisitRepresentation(pathname, markdownRewritePath);
  if (!userAgent || !isAgentVisitCandidate(userAgent, representation)) return;

  event.waitUntil(
    fetch(process.env.FEEDPROOF_INGEST_URL || DEFAULT_INGEST_URL, {
      body: JSON.stringify({
        visits: [{ at: new Date().toISOString(), path: pathname, representation, userAgent }],
      }),
      headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
      method: "POST",
      signal: AbortSignal.timeout(REPORT_TIMEOUT_MS),
    }).then(
      () => undefined,
      () => undefined,
    ),
  );
}
```

### A3. Call it from `proxy.ts`

Take the `NextFetchEvent` as the proxy's second argument. Compute the Markdown rewrite once, report the visit with it, then reuse it for the rewrite. The report goes at the top so it also covers requests that Hydrogen handlers answer later on.

```diff
--- a/proxy.ts
+++ b/proxy.ts
@@ -6,8 +6,9 @@ import {
   CUSTOMER_ACCOUNT_LOGOUT_PATH,
   CUSTOMER_ACCOUNT_REFRESH_PATH,
 } from "@shopify/hydrogen/customer-account";
-import { NextResponse, type NextRequest } from "next/server";
+import { type NextFetchEvent, NextResponse, type NextRequest } from "next/server";
 
+import { reportAgentVisit } from "@/lib/agent-visits/server";
 import {
   createCustomerRequestContext,
   createCustomerSessionManager,
@@ -37,8 +38,10 @@ const NOOP_SESSION_MANAGER = {
   setSessionItem: () => {},
 };
 
-export async function proxy(request: NextRequest): Promise<Response> {
+export async function proxy(request: NextRequest, event: NextFetchEvent): Promise<Response> {
   const pathname = request.nextUrl.pathname;
+  const markdownRewritePath = getMarkdownRewritePath(request);
+  reportAgentVisit(request, event, markdownRewritePath);
   const requestContext = createCustomerRequestContext(request);
   const isAuthPath = shopConfig.auth.isEnabled && AUTH_PATHS.has(pathname);
   const customerSession =
@@ -61,7 +64,7 @@ export async function proxy(request: NextRequest): Promise<Response> {
 
   const headers = requestContext.getForwardedRequestHeaders();
   const rewritePath =
-    getMarkdownRewritePath(request) ?? getRefinedCollectionPath(pathname, request.nextUrl.search);
+    markdownRewritePath ?? getRefinedCollectionPath(pathname, request.nextUrl.search);
   const response = rewritePath
     ? NextResponse.rewrite(new URL(`${rewritePath}${request.nextUrl.search}`, request.url), {
         request: { headers },
```

If the storefront's `proxy.ts` has drifted from the template, keep the rules: report before any early return, call `getMarkdownRewritePath(request)` only once, and never `await` the report.

### A4. Environment

Add the rows to the `# Optional` block of `.env.example`, keeping it alphabetical:

```bash
# FEEDPROOF_INGEST_KEY="your-feedproof-ingest-key-here" # Report AI-agent visits to FeedProof; leave unset to skip reporting.
# FEEDPROOF_INGEST_URL="https://feedproof.fly.dev/api/agent-visits" # Override the FeedProof ingest endpoint.
```

Add `FEEDPROOF_INGEST_KEY` and `FEEDPROOF_INGEST_URL` to `turbo.json` `globalEnv`, keeping it alphabetical.

---

## Part B: FeedProof details in product Markdown and JSON-LD

Use `shopify-graphql-reference` for placement. The selection below is valid against Storefront API 2026-10. To check it with the Shopify AI Toolkit, validate this standalone query:

```graphql
query FeedProofCheck($handle: String!) {
  product(handle: $handle) {
    feedproofAttributes: metafields(
      identifiers: [
        { namespace: "feedproof", key: "material" }
        { namespace: "feedproof", key: "dimensions" }
        { namespace: "feedproof", key: "care_instructions" }
        { namespace: "feedproof", key: "country_of_origin" }
        { namespace: "feedproof", key: "warranty" }
        { namespace: "feedproof", key: "condition" }
      ]
    ) {
      key
      value
    }
    selectedOrFirstAvailableVariant {
      barcode
      sku
    }
  }
}
```

### B1. Product fragment (`lib/shopify/fragments/product/index.ts`)

Add `barcode` to the `selectedOrFirstAvailableVariant` selection in `ProductFields`, and add the aliased metafields after `category`:

```diff
--- a/lib/shopify/fragments/product/index.ts
+++ b/lib/shopify/fragments/product/index.ts
@@ -80,6 +80,7 @@ export const PRODUCT_FRAGMENT = gql(
     }
     selectedOrFirstAvailableVariant {
       ...ProductVariantFields
+      barcode
       sku
     }
     options {
@@ -110,6 +111,19 @@ export const PRODUCT_FRAGMENT = gql(
     category {
       ...TaxonomyCategoryFields
     }
+    feedproofAttributes: metafields(
+      identifiers: [
+        { namespace: "feedproof", key: "material" }
+        { namespace: "feedproof", key: "dimensions" }
+        { namespace: "feedproof", key: "care_instructions" }
+        { namespace: "feedproof", key: "country_of_origin" }
+        { namespace: "feedproof", key: "warranty" }
+        { namespace: "feedproof", key: "condition" }
+      ]
+    ) {
+      key
+      value
+    }
   }
 `,
   [PRODUCT_VARIANT_FRAGMENT, TAXONOMY_CATEGORY_FRAGMENT],
```

The metafields return `null` for any key without storefront access, so a store FeedProof hasn't connected is unaffected.

### B2. Domain type (`lib/product/types.ts`)

```diff
--- a/lib/product/types.ts
+++ b/lib/product/types.ts
@@ -90,10 +90,21 @@ export interface ProductPage {
   products: ProductCard[];
 }
 
+export interface ProductAttributes {
+  careInstructions?: string;
+  condition?: string;
+  countryOfOrigin?: string;
+  dimensions?: string;
+  material?: string;
+  warranty?: string;
+}
+
 export interface ProductDetails extends ProductCard {
   /** Sparse variant cache around the default variant; feeds Hydrogen's product form store. */
   adjacentVariants: ProductVariant[];
   allVariantsInStock: boolean;
+  /** FeedProof `feedproof.*` metafields; empty until FeedProof exposes them to the Storefront API. */
+  attributes: ProductAttributes;
   category?: Category | null;
   compareAtPriceRange?: {
     maxVariantPrice: Money;
@@ -101,6 +112,7 @@ export interface ProductDetails extends ProductCard {
   };
   currencyCode: string;
   defaultVariant?: ProductVariant;
+  defaultVariantBarcode?: string;
   defaultVariantSku?: string;
   description: string;
   descriptionHtml: string;
```

### B3. Transform (`lib/shopify/transforms/product/index.ts`)

```diff
--- a/lib/shopify/transforms/product/index.ts
+++ b/lib/shopify/transforms/product/index.ts
@@ -5,6 +5,7 @@ import type {
   Category,
   OptionValue,
   OptionValueSwatch,
+  ProductAttributes,
   ProductCard,
   ProductDetails,
   ProductOption,
@@ -206,6 +207,26 @@ function hasUniformPriceRange(product: ShopifyProduct): boolean {
   return compareAtPriceRange.minVariantPrice.amount === compareAtPriceRange.maxVariantPrice.amount;
 }
 
+const FEEDPROOF_ATTRIBUTE_KEYS = {
+  care_instructions: "careInstructions",
+  condition: "condition",
+  country_of_origin: "countryOfOrigin",
+  dimensions: "dimensions",
+  material: "material",
+  warranty: "warranty",
+} as const satisfies Record<string, keyof ProductAttributes>;
+
+function transformAttributes(metafields: ShopifyProduct["feedproofAttributes"]): ProductAttributes {
+  const attributes: ProductAttributes = {};
+  for (const metafield of metafields) {
+    const name =
+      metafield && FEEDPROOF_ATTRIBUTE_KEYS[metafield.key as keyof typeof FEEDPROOF_ATTRIBUTE_KEYS];
+    const value = metafield?.value.trim();
+    if (name && value) attributes[name] = value;
+  }
+  return attributes;
+}
+
 export function transformShopifyProductDetails(product: ShopifyProduct): ProductDetails {
   const variants = product.variants
     ? flattenConnection(product.variants).map(transformVariant)
@@ -225,6 +246,7 @@ export function transformShopifyProductDetails(product: ShopifyProduct): Product
     availableForSale: product.availableForSale,
     isGiftCard: product.isGiftCard,
     adjacentVariants: product.adjacentVariants.map(transformVariant),
+    attributes: transformAttributes(product.feedproofAttributes),
     allVariantsInStock:
       !product.encodedVariantExistence ||
       product.encodedVariantExistence === product.encodedVariantAvailability,
@@ -232,6 +254,7 @@ export function transformShopifyProductDetails(product: ShopifyProduct): Product
     variantsCount: product.variantsCount?.count ?? variants?.length ?? 0,
     defaultVariant,
     defaultVariantSelectedOptions: defaultVariant?.selectedOptions ?? [],
+    defaultVariantBarcode: product.selectedOrFirstAvailableVariant?.barcode || undefined,
     defaultVariantSku: product.selectedOrFirstAvailableVariant?.sku || undefined,
     description: product.description,
     descriptionHtml: product.descriptionHtml,
```

### B4. Markdown (`lib/markdown/product/index.ts`)

The Details section goes after About and before Options. Only filled attributes are listed. GTIN is listed only for a single-variant product, because the barcode belongs to the default variant.

```diff
--- a/lib/markdown/product/index.ts
+++ b/lib/markdown/product/index.ts
@@ -3,10 +3,19 @@ import { escapeMarkdown, markdownFrontmatter } from "@/lib/markdown";
 import { formatPrice } from "@/lib/money";
 import type { Money } from "@/lib/money/types";
 import { isMarkedDown } from "@/lib/product";
-import type { ProductDetails } from "@/lib/product/types";
+import type { ProductAttributes, ProductDetails } from "@/lib/product/types";
 
 const SUMMARY_MAX_LENGTH = 200;
 
+const ATTRIBUTE_LABELS = {
+  careInstructions: "Care",
+  condition: "Condition",
+  countryOfOrigin: "Country of origin",
+  dimensions: "Dimensions",
+  material: "Material",
+  warranty: "Warranty",
+} as const satisfies Record<keyof ProductAttributes, string>;
+
 function summarize(product: ProductDetails): string {
   const description = product.description.replace(/\s+/g, " ").trim();
   const seoDescription = product.seo.description.replace(/\s+/g, " ").trim();
@@ -83,6 +92,17 @@ export function productToMarkdown(product: ProductDetails, locale: string): stri
     sections.push("## About", "", escapeMarkdown(product.description), "");
   }
 
+  const details = (Object.keys(ATTRIBUTE_LABELS) as Array<keyof ProductAttributes>)
+    .filter((key) => product.attributes[key])
+    .map(
+      (key) =>
+        `- **${ATTRIBUTE_LABELS[key]}**: ${escapeMarkdown((product.attributes[key] ?? "").replace(/\s+/g, " "))}`,
+    );
+  if (product.defaultVariantBarcode && product.variantsCount === 1) {
+    details.push(`- **GTIN**: ${escapeMarkdown(product.defaultVariantBarcode)}`);
+  }
+  if (details.length > 0) sections.push("## Details", "", ...details, "");
+
   if (product.options.length > 0) {
     sections.push("## Options", "");
     for (const option of product.options) {
```

### B5. Product JSON-LD (`components/product-detail/schema.tsx`, `components/product-detail/product-detail-section.tsx`)

`gtin` follows the same single-offer rule as `sku`. `itemCondition` is set only when the condition is one of Schema.org's four values.

```diff
--- a/components/product-detail/schema.tsx
+++ b/components/product-detail/schema.tsx
@@ -1,9 +1,12 @@
 import { shopConfig } from "@/lib/config";
 import type { Image } from "@/lib/media/types";
 import type { Money } from "@/lib/money/types";
+import type { ProductAttributes } from "@/lib/product/types";
 
 interface ProductSchemaData {
+  attributes: ProductAttributes;
   availableForSale: boolean;
+  barcode?: string;
   currencyCode: string;
   description: string;
   handle: string;
@@ -18,6 +21,19 @@ interface ProductSchemaData {
   vendor?: string;
 }
 
+const ITEM_CONDITIONS: Record<string, string> = {
+  damaged: "https://schema.org/DamagedCondition",
+  new: "https://schema.org/NewCondition",
+  refurbished: "https://schema.org/RefurbishedCondition",
+  used: "https://schema.org/UsedCondition",
+};
+
+const PROPERTY_LABELS = {
+  careInstructions: "Care instructions",
+  dimensions: "Dimensions",
+  warranty: "Warranty",
+} as const satisfies Partial<Record<keyof ProductAttributes, string>>;
+
 interface ProductSchemaProps {
   product: ProductSchemaData;
 }
@@ -28,6 +44,17 @@ export function ProductSchema({ product }: ProductSchemaProps) {
     ? "https://schema.org/InStock"
     : "https://schema.org/OutOfStock";
   const singleOffer = product.offerCount === 1;
+  const { attributes } = product;
+  const itemCondition = attributes.condition
+    ? ITEM_CONDITIONS[attributes.condition.trim().toLowerCase()]
+    : undefined;
+  const additionalProperty = (Object.keys(PROPERTY_LABELS) as Array<keyof typeof PROPERTY_LABELS>)
+    .filter((key) => attributes[key])
+    .map((key) => ({
+      "@type": "PropertyValue",
+      name: PROPERTY_LABELS[key],
+      value: attributes[key],
+    }));
   const schema = {
     "@context": "https://schema.org",
     "@type": "Product",
@@ -40,10 +67,15 @@ export function ProductSchema({ product }: ProductSchemaProps) {
           name: product.vendor,
         }
       : undefined,
+    additionalProperty: additionalProperty.length > 0 ? additionalProperty : undefined,
+    countryOfOrigin: attributes.countryOfOrigin,
+    gtin: singleOffer ? product.barcode : undefined,
+    material: attributes.material,
     offers: singleOffer
       ? {
           "@type": "Offer",
           availability,
+          itemCondition,
           price: product.priceRange.minVariantPrice.amount,
           priceCurrency: product.currencyCode,
           url,
@@ -51,6 +83,7 @@ export function ProductSchema({ product }: ProductSchemaProps) {
       : {
           "@type": "AggregateOffer",
           availability,
+          itemCondition,
           highPrice: product.priceRange.maxVariantPrice.amount,
           lowPrice: product.priceRange.minVariantPrice.amount,
           offerCount: product.offerCount,
```

```diff
--- a/components/product-detail/product-detail-section.tsx
+++ b/components/product-detail/product-detail-section.tsx
@@ -64,6 +64,8 @@ export function ProductDetailSection({
           offerCount: product.variantsCount,
           availableForSale: product.availableForSale,
           sku: product.defaultVariantSku,
+          attributes: product.attributes,
+          barcode: product.defaultVariantBarcode,
         }}
       />
       <ProductOpenGraph
```

---

## Verify

Run the existing checks. Don't add tests (see AGENTS.md). The template pins pnpm 12; with an older global pnpm, use `npx -y pnpm@12.8.2` in place of `pnpm`:

```bash
pnpm typecheck && pnpm lint && pnpm build
```

Then start the build and check the behaviour, not just the status codes. To watch the reports locally, point `FEEDPROOF_INGEST_URL` at any request logger:

```bash
pnpm start &
curl -s -o /dev/null -A "Mozilla/5.0; compatible; GPTBot/1.2; +https://openai.com/gptbot" localhost:3000/products/<handle>   # one report, representation "html"
curl -s -o /dev/null -A "ClaudeBot/1.0" -H "Accept: text/markdown" localhost:3000/products/<handle>                        # one report, "markdown"
curl -s -o /dev/null -A "Mozilla/5.0 (Macintosh) Chrome/141.0 Safari/537.36" localhost:3000/products/<handle>                # no report
curl -s localhost:3000/products/<handle>.md | sed -n '/## Details/,/## Options/p'                                            # FeedProof details; also one report, "markdown"
```

Against the real FeedProof endpoint, a report answers `{"ok":true,"accepted":1,"dropped":0}`. A browser that slips through the broad filter shows as `dropped`. The visit appears in FeedProof → Settings → Headless storefront within seconds.

Say which checks ran against a real store and which against mock data. The Details section needs a store that FeedProof has connected and enriched. On mock.shop it stays empty.

## Guardrails

- The ingest key is server-only. Never put it in a `NEXT_PUBLIC_*` variable, source, a log line or the PR body.
- The report never blocks, retries or throws. Keep it in `event.waitUntil`, with the 2-second timeout and the swallowed rejection.
- Send the pathname only. Query strings can carry search terms.
- Keep the user-agent filter broad and let FeedProof classify. Don't hard-code an allowlist of agents in the storefront: FeedProof updates its list without a storefront redeploy.
- Don't route Eve or Shopify's agent endpoints (`/api/mcp`, `/api/ucp/mcp`) through this code. Those are POSTs and are skipped by design.
- Commerce facts still come from Shopify: the attributes are Shopify metafields read through the Storefront API, never hand-entered values.
