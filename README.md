# 4Partners Site API v2

Site API v2 is the API of a 4Partners partner site. Use it to build your own storefront or app on top of the site
and to sync the site with your systems: the product catalog, the cart and checkout of customers, guest carts,
orders of the site, content and settings.

- API: `https://api.4partners.io/v2`
- Interactive documentation: <https://api.4partners.io/v2>
- Documentation for language models: <https://api.4partners.io/v2/llms.txt>,
  every operation in detail: <https://api.4partners.io/v2/llms-full.txt>
- Error types: <https://api.4partners.io/api/problems>

This repository is updated automatically after each release of the API. AI agents that work with the API for you
start with [`AGENTS.md`](AGENTS.md).mPDF

## Requests and responses

Every operation is `POST https://api.4partners.io/v2/<group>/<entity>/<action>` with a JSON body
(`Content-Type: application/json`). An operation without parameters takes `{}`. Ids go in the body, keys are
`snake_case`, and an unknown key gives 422.

```sh
curl -X POST https://api.4partners.io/v2/stock/product/show \
  -H 'Content-Type: application/json' \
  -H "X-Auth-Token: $SITE_TOKEN" \
  -d '{"product_id": 123}'
```

- **Envelope.** A successful response is `{timestamp, error, data, meta}`: `error` is `false`, `data` is an object
  or an array for lists, `meta` has `request_id` and `usage`. A field without a value comes as `null`, it is never
  left out.
- **Money** is a decimal string with two digits: `"1990.00"`. Country codes are ISO 3166-1 alpha-2 in upper case
  (`RU`).
- **Pages.** Lists for display take `page` (from 1) and `per_page` and return `meta.pager` with `page`,
  `per_page`, `total` and `pages`. A full export (`stock/product/id/list`) takes `cursor` and `limit`: pass
  `next_cursor` of the answer as `cursor` until it is `null`. Page size limits are in the specification of each
  operation.
- **Errors** are `application/problem+json`: `type` is a stable identifier to branch on, `detail` says what is wrong
  and what to do, `pointer` points to the wrong value of the body (`#/items/0/quantity`). A 422 `validation-error`
  lists every wrong value in `errors` with `pointer` and `detail`. `type` is a link: open
  `https://api.4partners.io/api/problems/<slug>` for its meaning and the fix. Quote `request_id` (also in the
  `X-Request-Id` header) when you contact 4Partners support.
- **Rate limit.** 600 requests per minute, sliding window, for each operation group. With a server key the limit is
  counted per site, and all server keys of the site share one counter. With an app key it is counted per key and
  client IP address. In the customer groups it is counted per customer. The state is in
  `meta.usage` (`limit`, `used`, `remaining`, `retry_after`) and in the `X-RateLimit-*` headers. Over the limit the
  answer is 429 with `Retry-After`. Batch operations (`stock/product/batch-show`, `stock/list`,
  `order/batch-show`) save requests.
- **Language and region.** `site_variation` in the body sets the language, country and currency of data: product
  names, prices, rubrics, articles. Take the codes from `settings/site-variation/list`; without it the default
  variation is used. `Accept-Language: en` or `ru` changes only the language of error texts, English by default.

## Access

**Site key**, header `X-Auth-Token`. A site has two kinds of keys. Ask your 4Partners manager for a server key or an
app key.

- **Server key.** It opens every operation except the customer groups: orders, the deposit, products, content and
  settings.
- **App key.** It opens only the storefront and customer sign-in: reading `stock/*`, rubrics, brands, aspects,
  content blocks, articles, tags and article types, `settings/site-variation/list` and `show`, `dictionary/*`,
  `delivery/calculate`, `delivery/pick-up-point/list` and `customer-auth/*`. Other site operations give
  403 `site-token-required`; its `detail` says that the operation needs a server key. The limit of requests is counted
  per key and client IP address. An app key sends at most 10 SMS codes, sign-up codes and password reset links in total
  per hour from one IP address; over the limit the answer is 429 `sign-in-locked` with `Retry-After`.

> [!WARNING]
> The server key gives full access to the site: orders, the deposit, products, content and settings. Keep it on
> your server. Do not put it into a mobile app, browser code or any other client: your app calls your server, and
> your server calls the API. A mobile app or browser code that calls the API directly uses an app key.

**Customer token**, header `X-Customer-Auth-Token`. It opens `customer-account/*`, `customer-cart/*` and
`customer-order/*`. Your server or app gets it from `customer-auth/*` with a site key, server or app:

- `customer-auth/sign-in` by `email` and `password` returns the tokens at once. By `phone` it sends an SMS code and
  returns `confirmation_id`; `customer-auth/confirm` exchanges it and the code for the tokens.
- `customer-auth/sign-up` registers a customer by email: the code goes to the email, then `customer-auth/confirm`.
- The customer token is valid for an hour. `customer-auth/refresh` exchanges the refresh token (one-time, 30 days)
  for a new pair. A refresh token used twice ends the whole sign-in, so do not share it between parallel requests.
- `customer-auth/sign-out` with the refresh token ends the sign-in. A password change or reset and disabling the
  site key that issued the customer token also end it.

**Guest token**, the same header `X-Customer-Auth-Token`. `customer-auth/guest` issues it for a customer without
an account. It opens only `customer-cart/show` and `customer-cart/item/*`; other customer operations give 403. The
cart lives as long as the token, 90 days: call `customer-auth/guest` with `guest_token` before it expires to renew
it and keep the cart. To place an order, the guest signs in or up with a code and passes `guest_token` to
`customer-auth/confirm` (or to `customer-auth/sign-in` by email and password): the guest cart moves to the customer.

Dictionaries (`dictionary/*`) accept a site key or a customer or guest token. A token of the wrong kind gives 403
`customer-token-required` or `site-token-required`.

## Scenarios

**Catalog export.** `stock/product/id/list` page by page with `cursor` → `stock/product/batch-show` for up to 100
products at a time → `stock/list` for frequent polling of prices and stock (up to 100 products or 500 variations).

**Storefront.** `settings/site-variation/list` for the region → `content/block/list` for the banners and product
sliders of the home page → `rubric/list` for the menu → `stock/product/search` with `stock/facet/list` for the
filters of the same selection → `stock/product/suggest` for the search box → `stock/product/show` for a product card →
`stock/product/similar` for the «you may also like» block.

**Cart and checkout of a customer.**

1. `customer-auth/guest` → `customer-cart/item/create`, `customer-cart/show`.
2. `customer-auth/sign-in` by phone or `customer-auth/sign-up` by email → `customer-auth/confirm` with the code and
   `guest_token`.
3. `customer-account/address/create` → `customer-cart/checkout/update` with `address_id`.
4. `customer-cart/checkout/show` → `customer-cart/checkout/update` with the delivery method of each package, the
   pick-up point (`customer-cart/pick-up-point/list`) and `payment_method_id`.
5. `customer-cart/promo-code/apply`, if the customer has a code.
6. `customer-order/create` with `return_url` → open `payment_url` for the customer. After the bank the customer
   comes back to `return_url` with `order_id` and `payment_status`.
7. There is no payment notification: poll `is_paid` of `customer-order/show`.

**Orders of the site.** The partner orders for its own recipients and pays from the deposit:
`delivery/calculate` (and `delivery/pick-up-point/list` for pick-up methods) → `order/create` with `external_id` →
`order/pay` → `order/show`, `order/list`. `balance/show` shows the deposit. A repeated `order/create` with the `external_id` of an order that is not
canceled and the same body returns that order instead of a new one.

## This repository

- `spec/` — snapshots of the released version: `openapi.json` (the OpenAPI specification), `llms.txt` and
  `llms-full.txt`.
- `bruno/` — a [Bruno](https://www.usebruno.com/) collection with a request for every operation, grouped as in the
  documentation. The folders of the main groups describe their scenarios.

To use the collection:

1. Open `bruno/` in Bruno and select the `prod` environment.
2. Enter your server key into the secret variable `siteToken`. Bruno keeps secret values on your computer, not in
   the environment file.
3. Replace the zero ids of the environment (`productId`, `productVariationId`, `rubricId`, `deliveryMethodId`,
   `pickUpDeliveryMethodId`) with ids of your site from the list operations.
4. Do not enter `customerToken` by hand: the `Customer_Auth` sign-in, confirm and refresh requests and
   `Customer_Account` password change fill it.

Requests of the collection run against the live site: placing, paying and canceling orders, deleting objects and
sending codes really happen.
