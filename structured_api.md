# 1. API for Structured Product

<!-- TOC -->

- [1. API for Structured Product](#1-api-for-structured-product)
  - [1.1. Encoding](#11-encoding)
  - [1.2. Common parameters in HTTP headers](#12-common-parameters-in-http-headers)
  - [1.3. Authentication](#13-authentication)
  - [1.4. Private API mandatory fields](#14-private-api-mandatory-fields)
  - [1.5. Signature algorithm](#15-signature-algorithm)
  - [1.6. Structured Product Vendor RESTful API](#16-structured-product-vendor-restful-api)
    - [1.6.1. "Meta" Explained](#161-meta-explained)
    - [1.6.2. Get Products](#162-get-products)
    - [1.6.3. Get Quote](#163-get-quote)
    - [1.6.4. Place Order](#164-place-order)
    - [1.6.5. Get Redeem Quote](#165-get-redeem-quote)
    - [1.6.6. Redeem Order](#166-redeem-order)
    - [1.6.7. Query Order by Order ID or Client Order ID](#167-query-order-by-order-id-or-client-order-id)
    - [1.6.8. Query Orders](#168-query-orders)
    - [1.6.9. Query Redeem Order by Redeem ID or Client Redeem ID](#169-query-redeem-order-by-redeem-id-or-client-redeem-id)
    - [1.6.10. Check Settlement per order](#1610-check-settlement-per-order)
    - [1.6.11. Monthly profit check](#1611-monthly-profit-check)
    - [1.6.12. Renew diff check](#1612-renew-diff-check)
    - [1.6.13 Audit orders](#1613-audit-orders)

<!-- /TOC -->

Structured Product's Vendor API Doc

Vendors should provide the following APIs

## 1.1. Encoding

- The request and response data for all interfaces are formatted in UTF-8.
- Content for responses in all interfaces is formatted in JSON.

## 1.2. Common parameters in HTTP headers

Below, common parameters in HTTP headers are recommended to be added in request, facilitating debugging and tracing.

| Key          | Type   | Position | Required | Description                                                    |
| ------------ | ------ | -------- | -------- | -------------------------------------------------------------- |
| x-request-id | string | header   | false    | UUID to trace the request, not included in signing the request |

## 1.3. Authentication

## 1.4. Private API mandatory fields

- Client put Access Key in http request header X-Access-Key
- Client and Server use same private secret key to sign the request.
- Client must add `timestamp` (epoch in millisecond) field in request parameter (query string for GET, JSON body for POST), API server will check this timestamp, if `abs(server_timestamp - request_timestamp) > 5000`, the request will be rejected.
- `timestamp` must be integer, not quoted string.
- Client must add `signature` field in request parameter (query string for GET, JSON body for POST).
- For POST request, Header `Content-Type` should be set as `application/json`.

| Key       | Type   | Position      | Required | Description              |
| --------- | ------ | ------------- | -------- | ------------------------ |
| timestamp | int    | query or body | true     | epoch in millisecond     |
| signature | string | query or body | true     | signature of the request |

## 1.5. Signature algorithm

```python
import hashlib
import hmac

class Auth:
    def __init__(self, secret_key):
        self.secret_key = secret_key

    def encode_list(self, item_list):
        list_val = []
        for item in item_list:
            obj_val = self.encode_object(item)
            list_val.append(obj_val)
        output = "&".join(list_val)
        output = "[" + output + "]"
        return output

    def encode_object(self, param_map):
        sorted_keys = sorted(param_map.keys())
        # print("sorted_keys", sorted_keys)
        ret_list = []
        for key in sorted_keys:
            val = param_map[key]
            if isinstance(val, list):
                list_val = self.encode_list(val)
                ret_list.append(f"{key}={list_val}")
            elif isinstance(val, dict):
                # call encode_object recursively
                dict_val = self.encode_object(val)
                ret_list.append(f"{key}={dict_val}")
            elif isinstance(val, bool):
                bool_val = str(val).lower()
                ret_list.append(f"{key}={bool_val}")
            else:
                general_val = str(val)
                ret_list.append(f"{key}={general_val}")

        sorted_list = sorted(ret_list)
        output = "&".join(sorted_list)
        return output

    def get_signature(self, api_path, param_map):
        str_to_sign = api_path + "&" + self.encode_object(param_map)
        # print("str_to_sign = " + str_to_sign)
        sig = hmac.new(
            self.secret_key.encode("utf-8"),
            str_to_sign.encode("utf-8"),
            digestmod=hashlib.sha256,
        ).hexdigest()
        return sig
```

Explanation:

1. Request parameters: JSON Body for POST, query string for GET
1. Encode string to sign, for simple json object, sort your parameter keys alphabetically, and join them with '&' like ```'param1=value1&param2=value2'```, then get ```str_to_sign = api_path + '&' + 'param1=value1&param2=value2'```

- For nested array objects, encode each object and sort them alphabetically, join them with ```'&'``` and embraced with ```'[', ']'```
- e.g.```str_to_sign = api_path + '&' + 'param1=value1&array_key1=[array_item1&array_item2]'```.

1. ```Signature = hex(hmac_sha256(str_to_sign, secret_key))```
1. Add `signature` field to request parameter:

- for query string, add ```'&signature=YOUR_SIGNATURE'```
- for JSON body, add ```{"signature":YOUR_SIGNATURE}```

## 1.6. Structured Product Vendor RESTful API

### 1.6.1. "Meta" Explained

In our system, we are integrating multiple financial products,
including Dual-Coin, Snowball, Sharkfin and Trend, into a unified platform. It is
feasible because these products share a significant amount of common logic.
However, certain dimensions (such as settlement procedures) are product-specific,
hard to be unified completely.

To address the variations, we use "meta" to encapsulate each product's divergent
aspects (mostly algorithms). Technically, Dual-Coin is a meta-product, Sharkfin
is another meta-product. They differ in their sets of **settlement rules**,
**interest accrual rules**, **redemption policies** and **auto-renewal policies**.

For example, when we initiate the "settle" [action](#1610-check-settlement-per-order),
input data are basically the same - `settlement price` and `settlement amount`.
For Dual-Coin, the price is compared with a strike price to decide whether
the underlying option is exercised; for Sharkfin, the price is evaluated in the
yield curve function to produce a yield rate. These two different settlement algorithms
have to be linked to the "meta" layer, and retrieved by specifying the `mata_name`.

It is recommended to have a clear understanding of both the common logic and
the individual requirements of the above financial products in terms of their financial properties,
for effectively utilizing the "meta" conception.

### 1.6.2. Get Products

- Returns a list of products

- The timeout for this interface call is 1000ms. Peak QPS: 50.

- URL: /mp/api/v1/structured/products

- method: Get

- Parameters: in query

| Key             | Type   | Required | Description                                                                             | Example         |
| --------------- | ------ | -------- | --------------------------------------------------------------------------------------- | --------------- |
| meta_name       | string | yes      | name of the meta-product                                                                | dcp, snowball   |
| invest_currency | string | no       | product's investment currency. Empty means all investment currencies.                   | USDT, BTC       |
| underlying      | string | no       | product's underlying asset. Empty means all underlying assets.                          | BTC-USDT        |
| tracking_source | string | no       | the fixing convention used to settle product payment. Empty means all tracking sources. | DERIBIT,BINANCE |
| type            | string | no       | type, in uppercase. CALL=bullish products, PUT=bearish products. Empty means all types  | CALL, PUT       |

- Response: application/json

Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "snowball",
    "items": [ //array, products
        {
            "invest_currency": "USDC", //string, investment currency
            "underlying": "BTC-USDC", //string, underlying asset
            "tracking_source": "DERIBIT",  //string, trade source
            "type": "CALL", // string, type
            "term_mill": 604800000, // int, term length in millisecond. eg, 604800000 is 7 days
            "take_profit_price": "38000", //string, take-profit price in string format
            "protection_price": "30000", //string, protection price in string format
            "take_profit_apy": "0.1",
            "protection_apy": "-0.05",
            "zero_price_apy": "-0.1",
            "low_price_apy": "-0.05",
            "high_price_apy": "0.1",
            "min_buy_per_order": "0.1", //string, minimal buy amount per order
            "max_buy_per_order": "100", //string, maximal buy amount per order
            "buy_step": "0.1", // buy amount step, 0.1 means investment amount can change by 0.1
            "max_order_number_per_user": 1000000, //optional int, a user can place AT MOST how many orders
            "max_buy_per_user": "100000000", // optional string, max total buy amount per user
            "max_buy_product": "1000000000" // optional string, max total buy amount of this product
        }
    ]
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "sharkfin",
    "items": [ //array, products
        {
            "invest_currency": "USDT", //string, investment currency
            "underlying": "BTC-USDT", //string, underlying asset
            "tracking_source": "DERIBIT",  //string, tracking source
            "type": "CALL", // string, type
            "term_mill": 604800000, // int, term length in millisecond. eg, 604800000 is 7 days
            "take_profit_price": "38000", //string, take-profit price in string format
            "protection_price": "30000", //string, protection price in string format
            "take_profit_apy": "0.2",
            "protection_apy": "0.1",
            "zero_price_apy": "0.05",
            "low_price_apy": "0.05",
            "high_price_apy": "0.05",
            "min_buy_per_order": "1", //string, minimal buy amount per order
            "max_buy_per_order": "100000", //string, maximal buy amount per order
            "buy_step": "0.1", // buy amount step, 0.1 means investment amount can change by 0.1
            "max_order_number_per_user": 1000000, //optional int, a user can place AT MOST how many orders
            "max_buy_per_user": "100000000", // optional string, max total buy amount per user
            "max_buy_product": "1000000000" // optional string, max total buy amount of this product
        }
    ]
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "seagull",
    "items": [ //array, products
        {
            "invest_currency": "USDT", //string, investment currency
            "underlying": "BTC-USDT", //string, underlying asset
            "tracking_source": "BINANCE",  //string, tracking source
            "type": "CALL", // string, type
            "term_mill": 1721289600000, // int, maturity time in millisecond UNIX epoch. eg, 1721289600000 is 2024-07-18 16:00:00(UTC+8)
            "strike_convert_price": "64000", //string, the price which triggers currency conversion
            "strike_start_price": "68000", //string, the price which starts to generate additional profit
            "strike_end_price": "69000", //string, the price which yields the max profit
            "invest_amount": "1000", // string, reference investment amount from the user
            "low_price_quantity": "0.7725", // string, profit if price lower than price interval, based on invest_amount
            "high_price_quantity": "5.4161", // string, profit if price higher than price interval, based on invest_amount
            "min_buy_per_order": "1", //string, minimal buy amount per order
            "max_buy_per_order": "100000", //string, maximal buy amount per order
            "buy_step": "0.1", // buy amount step, 0.1 means investment amount can change by 0.1
            "max_order_number_per_user": 1000000, //optional int, a user can place at most how many orders
            "max_buy_per_user": "100000000", // optional string, max total buy amount per user
            "max_buy_product": "1000000000" // optional string, max total buy amount of this product
        },
        {
            "invest_currency": "BTC", //string, investment currency
            "underlying": "BTC-USDT", //string, underlying asset
            "tracking_source": "BINANCE",  //string, tracking source
            "type": "PUT", // string, type
            "term_mill": 1721289600000, // int, maturity time in millisecond UNIX epoch. eg, 1721289600000 is 2024-07-18 16:00:00(UTC+8)
            "strike_convert_price": "67000", //string, the price which triggers currency conversion
            "strike_start_price": "64000", //string, the price which starts to generate additional profit
            "strike_end_price": "63000", //string, the price which yields the max profit
            "invest_amount": "10", // string, reference investment amount from the user
            "low_price_quantity": "0.054831", // string, profit if price lower than price interval, based on invest_amount
            "high_price_quantity": "0.012106", // string, profit if price higher than price interval, based on invest_amount
            "min_buy_per_order": "0.01", //string, minimal buy amount per order
            "max_buy_per_order": "100", //string, maximal buy amount per order
            "buy_step": "0.1", // buy amount step, 0.1 means investment amount can change by 0.1
            "max_order_number_per_user": 100, //optional int, a user can place at most how many orders
            "max_buy_per_user": "10000", // optional string, max total buy amount per user
            "max_buy_product": "100000" // optional string, max total buy amount of this product
        }
    ]
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "dnt",
    "items": [ //array, products
        {
            "invest_currency": "USDT", //string, investment currency
            "underlying": "BTC-USDT", //string, underlying asset
            "tracking_source": "BINANCE",  //string, tracking source
            "term_mill": 1692950400000, // int, settle time in millisecond UNIX epoch. eg, 1692950400000 is 2023-08-25T16:00:00 +08:00
            "take_profit_price": "38000", //string, upper bound price to trigger knock event
            "protection_price": "30000", //string, lower bound price to trigger knock event
            "min_buy_per_order": "1", //string, minimal buy amount per order
            "max_buy_per_order": "100000", //string, maximal buy amount per order
            "buy_step": "0.1", // buy amount step, 0.1 means investment amount can change by 0.1
            "max_order_number_per_user": 1000000, //optional int, a user can place AT MOST how many orders
            "max_buy_per_user": "100000000", // optional string, max total buy amount per user
            "max_buy_product": "1000000000", // optional string, max total buy amount of this product
            "invest_amount": "100", // for reference, user will invest 100
            "booking_premium":"1", // the portion of invest_amount will be utilized to subscribe DNT
            "booking_quantity":"1.6" // quantity/share of the subscription
        }
    ]
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "dnt",
    "items": [ //array, products
        {
            "invest_currency": "USDT", //string, investment currency
            "underlying": "BTC-USDT", //string, underlying asset
            "tracking_source": "BINANCE",  //string, tracking source
            "term_mill": 1692950400000, // int, settle time in millisecond UNIX epoch. eg, 1692950400000 is 2023-08-25T16:00:00 +08:00
            "take_profit_price": "38000", //string, upper bound price to trigger knock event
            "protection_price": "30000", //string, lower bound price to trigger knock event
            "min_buy_per_order": "1", //string, minimal buy amount per order
            "max_buy_per_order": "100000", //string, maximal buy amount per order
            "buy_step": "0.1", // buy amount step, 0.1 means investment amount can change by 0.1
            "max_order_number_per_user": 1000000, //optional int, a user can place AT MOST how many orders
            "max_buy_per_user": "100000000", // optional string, max total buy amount per user
            "max_buy_product": "1000000000", // optional string, max total buy amount of this product
            "invest_amount": "100", // for reference, user will invest 100
            "funding_amount": "1.5", // the funding amount
            "booking_premium":"1", // the portion of funding_amount will be utilized to subscribe DNT
            "booking_quantity":"1.6" // quantity/share of the subscription
        }
    ]
  }
}
```

| Parameter Name                  | Type             | Description                                                                                                                                                                               |
| :------------------------------ | :--------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| meta_name                       | string           | name of the meta-product                                                                                                                                                                  |
| items                           | array            | products                                                                                                                                                                                  |
| items.invest_currency           | string           | investment currency                                                                                                                                                                       |
| items.underlying                | string           | underlying asset                                                                                                                                                                          |
| items.tracking_source           | string           | tracking source                                                                                                                                                                           |
| items.type                      | string           | type                                                                                                                                                                                      |
| items.term_mill                 | int              | for a fixed mature time, this is millisecond UNIX epoch, eg. 1692950400000 is 2023-08-25T16:00:00 +08:00. For a fixed term length, this is the term in milliseconds, eg. 1day is 86400000 |
| items.take_profit_price         | string           | for products with a price interval: this is the price which triggers take-profit, in string format                                                                                        |
| items.protection_price          | string           | for products with a price interval, this is the price which triggers stop-loss, in string format                                                                                          |
| items.strike_convert_price      | string           | the price which triggers currency conversion, in string format                                                                                                                            |
| items.strike_start_price        | string           | (seagull) the price which starts to generate additional profit, in string format                                                                                                          |
| items.strike_end_price          | string           | (seagull) the price which yields the max profit, in string format                                                                                                                         |
| items.apy                       | string           | for products with one annual percentage yield: in string format                                                                                                                           |
| items.take_profit_apy           | string           | for products with a price interval: the maximum annual percentage yield the product will pay within the price interval, in string format                                                  |
| items.protection_apy            | string           | for products with a price interval: the minimum annual percentage yield the product will pay within the price interval, in string format                                                  |
| items.zero_price_apy            | string           | for products with a price interval: the annual percentage yield when the price is zero (connected to low price APY to draw a line), in string format                                      |
| items.low_price_apy             | string           | for products with a price interval: the annual percentage yield when the price is smaller than the price interval, in string format                                                       |
| items.high_price_apy            | string           | for products with a price interval: the annual percentage yield when the price is larger than the price interval, in string format                                                        |
| items.min_buy_per_order         | string           | minimal buy amount per order                                                                                                                                                              |
| items.max_buy_per_order         | string           | maximal buy amount per order                                                                                                                                                              |
| items.buy_step                  | string           | buy amount step                                                                                                                                                                           |
| items.max_order_number_per_user | int, optional    | max number of orders which a user can place                                                                                                                                               |
| items.max_buy_per_user          | string, optional | max of total buy amount per user                                                                                                                                                          |
| items.max_buy_product           | string, optional | max of total buy amount of this product                                                                                                                                                   |
| items.invest_amount             | string, optional | reference amount from the user, in string format                                                                                                                                          |
| items.low_price_quantity        | string, optional | (seagull) the profit if price lower than price interval, in string format                                                                                                                 |
| items.high_price_quantity       | string, optional | (seagull) the profit if price higher than price interval, in string format                                                                                                                |
| items.funding_amount            | string, optional | the funding amount                                                                                                                                                                        |
| items.booking_premium           | string, optional | premium amount utilized to subscribe the product                                                                                                                                          |
| items.booking_quantity          | string, optional | quantity/share of the subscription                                                                                                                                                        |

---

### 1.6.3. Get Quote

- Get quote for placing an order by instrument info of a product.

- The quote provided by the vendor should remain valid within 1 minute; otherwise, it may result in a higher user purchase failure rate.

- The timeout for this interface call is 1000ms. Peak QPS: 50.

- URL: /mp/api/v1/structured/quote

- method: Get

- Parameters: json in body

| Key                  | Type   | Required | Description                                                                                        |
| -------------------- | ------ | -------- | -------------------------------------------------------------------------------------------------- |
| meta_name            | string | yes      | name of the meta-product. eg, "snowball", "dcp"                                                    |
| invest_currency      | string | yes      | investment currency                                                                                |
| underlying           | string | yes      | underlying asset                                                                                   |
| tracking_source      | string | yes      | the fixing convention used to settle product payment                                               |
| type                 | string | no       | product type, "CALL" or "PUT" (required for products with this attribute)                          |
| term_mill            | int    | yes      | product's term.                                                                                    |
| take_profit_price    | string | yes      | for products with a price interval: this is the price which triggers take-profit, in string format |
| protection_price     | string | yes      | for products with a price interval, this is the price which triggers stop-loss, in string format   |
| strike_convert_price | string | seagull  | the price which triggers currency conversion, in string format                                     |
| strike_start_price   | string | seagull  | (seagull) the price which starts to generate additional profit, in string format                   |
| strike_end_price     | string | seagull  | (seagull) the price which yields the max profit, in string format                                  |
| invest_amount        | string | yes      | amount from the user to invest in the product, in string format                                    |
| booking_premium      | string | no       | premium amount utilized for the subscription of the product                                        |
| funding_amount       | string | no       | the funding amount                                                                                 |

Request Data Example:

```js
{
  "meta_name": "dnt",
  "invest_currency": "USDT",
  "underlying": "BTC-USDT",
  "tracking_source": "BINANCE",
  "term_mill": 1692950400000,
  "take_profit_price": "38000",
  "protection_price": "30000",
  "invest_amount": "300",
  "funding_amount": "4.5",
  "booking_premium": "3"
}
```

- Response: application/json

Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "quote_id": "6634567890123456789", //string, quote id
    "meta_name": "sharkfin", // string, name of the meta-product
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source": "DERIBIT",  //trade source  DERIBIT,BINANCE etc...
    "type": "CALL", //string, product type
    "term_mill": 1209600000, //int, term. e.g. 1209600000 is 14 days
    "take_profit_price": "39000", //string, take-profit price in string format
    "protection_price": "33000", // string, protection price in string format
    "zero_price_apy": "0.03", //string, zero-price APY in string format
    "low_price_apy": "0.03", //string, low-price APY in string format
    "high_price_apy": "0.04", //string, high-price APY in string format
    "apy_points": [
        {"price": "33000", "apy": "0.1"},
        {"price": "39000", "apy": "0.2"}
    ],
    "price_expire_time_mill": 1691727892000, //int, price expire time in millisecond,
    "invest_amount": "10" //string, amount in string format
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "quote_id": "9634567890123456789", //string, quote id
    "meta_name": "seagull", // string, name of the meta-product
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source": "DERIBIT",  //trade source  DERIBIT,BINANCE etc...
    "type": "CALL", //string, product type
    "term_mill": 1721289600000, // int, maturity time in millisecond UNIX epoch. eg, 1721289600000 is 2024-07-18 16:00:00(UTC+8)
    "strike_convert_price": "64000", //string, the price which triggers currency conversion
    "strike_start_price": "68000", //string, the price which starts to generate additional profit
    "strike_end_price": "69000", //string, the price which yields the max profit
    "low_price_quantity": "0.07", //string, the profit if price lower than price interval
    "high_price_quantity": "0.51", //string, the profit if price higher than price interval
    "price_expire_time_mill": 1721289650000, //int, price expire time in millisecond UNIX epoch
    "invest_amount": "100" //string, amount in string format
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "dnt", // string, name of the meta-product
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source": "BINANCE",  //trade source  DERIBIT,BINANCE etc...
    "term_mill": 1692950400000, // int, settle time in millisecond UNIX epoch. eg, 1692950400000 is 2023-08-25T16:00:00 +08:00
    "take_profit_price": "39000", //string, upper bound price to trigger knock event
    "protection_price": "33000", // string, lower bound price to trigger knock event
    "booking_premium": "0.1", // e.g. invest_ratio=1%, 0.1 utilized to subscribe
    "booking_quantity": "0.15", // quantity/share of the subscription
    "price_expire_time_mill": 1691727892000, //int, price expire time in millisecond UNIX epoch
    "invest_amount": "10" //invest amount from user, in string format
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "dnt", // string, name of the meta-product
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source": "BINANCE",  //trade source  DERIBIT,BINANCE etc...
    "term_mill": 1692950400000, // int, settle time in millisecond UNIX epoch. eg, 1692950400000 is 2023-08-25T16:00:00 +08:00
    "take_profit_price": "39000", //string, upper bound price to trigger knock event
    "protection_price": "33000", // string, lower bound price to trigger knock event
    "booking_premium": "0.2", // e.g. 0.2 (part of the funding amonut) utilized to subscribe
    "booking_quantity": "0.3", // quantity/share of the subscription
    "price_expire_time_mill": 1691727892000, //int, price expire time in millisecond,
    "invest_amount": "10", //invest amount from user, in string format
    "funding_amount": "0.5" // the funding amount (this product has funding)
  }
}
```


| Parameter Name         | Type   | Description                                                                                                                                       |
| :--------------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| quote_id               | string | quote id                                                                                                                                          |
| meta_name              | string | name of the meta-product                                                                                                                          |
| invest_currency        | string | investment currency                                                                                                                               |
| underlying             | string | underlying asset                                                                                                                                  |
| tracking_source        | string | the fixing convention used to settle product payment                                                                                              |
| type                   | string | product type (CALL or PUT)                                                                                                                        |
| term_mill              | int    | product's term                                                                                                                                    |
| take_profit_price      | string | for products with a price interval: the price which triggers take-profit, in string format                                                        |
| protection_price       | string | for products with a price interval, the price which triggers stop-loss, in string format                                                          |
| zero_price_apy         | string | for products with a price interval: the annual percentage yield when the price is zero, in string format                                          |
| low_price_apy          | string | for products with a price interval: the annual percentage yield when the price is smaller than the price interval, in string format               |
| high_price_apy         | string | for products with a price interval: the annual percentage yield when the price is larger than the price interval, in string format                |
| apy_points             | array  | for products with a price interval: within the price interval, points of {price, annual percentage yield} construct the piecewise function of APY |
| apy_points.price       | string | price, in string format                                                                                                                           |
| apy_points.apy         | string | annual percentage yield, in string format                                                                                                         |
| strike_convert_price   | string | the price which triggers currency conversion, in string format                                                                                    |
| strike_start_price     | string | (seagull) the price which starts to generate additional profit, in string format                                                                  |
| strike_end_price       | string | (seagull) the price which yields the max profit, in string format                                                                                 |
| low_price_quantity     | string | (seagull) the profit if price lower than price interval, in string format                                                                         |
| high_price_quantity    | string | (seagull) the profit if price higher than price interval, in string format                                                                        |
| price_expire_time_mill | int    | price expire time in millisecond                                                                                                                  |
| invest_amount          | string | invest amount from user, in string format                                                                                                         |
| funding_amount         | string | the funding amount (if the product supports it)                                                                                                   |
| booking_premium        | string | the premium amount utilized for the subscription                                                                                                  |
| booking_quantity       | string | the quantity/share of the subscription                                                                                                            |

Error Code:

| Code | Message                                                |
| ---- | ------------------------------------------------------ |
| 0    | Success                                                |
| 1001 | retryable error (may retry according to product logic) |
| 1002 | other error1  eg. No price (will **NOT** retry)        |
| 1003 | other error2  eg. price expire (will **NOT** retry)    |

### 1.6.4. Place Order

- Place Order by quote id / quote params. before placing order, quote api is called to get a quote.

- The timeout for this interface call is 2000ms. Peak QPS: 50.

- URL: /mp/api/v1/structured/order

- method: Post

- Parameters: json in body

| Key               | Type   | Required           | Description                                          |
| ----------------- | ------ | ------------------ | ---------------------------------------------------- |
| meta_name         | string | yes                | name of the meta-product. eg, "snowball", "dcp"      |
| invest_amount     | string | yes                | amount in string format                              |
| quote_id          | string | follows quote      | quote id, unique                                     |
| client_order_id   | string | yes                | client order id, for idempotence                     |
| parent_order_id   | string | no                 | vendor order id of the parent order, to renew        |
| invest_currency   | string | if quote_id unused | investment currency                                  |
| underlying        | string | if quote_id unused | underlying asset                                     |
| tracking_source   | string | if quote_id unused | the fixing convention used to settle product payment |
| term_mill         | int    | if quote_id unused | product's term                                       |
| take_profit_price | string | if quote_id unused | upper bound price to trigger knock event             |
| protection_price  | string | if quote_id unused | lower bound price to trigger knock event             |
| booking_premium   | string | if quote_id unused | the premium amount utilized for the subscription     |
| booking_quantity  | string | if quote_id unused | the quantity/share of the subscription               |
| funding_amount    | string | if quote_id unused | the funding amount (if the product supports it)      |

- Response: application/json

Example:

```js
{
  "code" : 0,
  "message" : "",
  "data" : {
    "meta_name": "dcp",
    "order_id": "7080019906774802432", //string,vendor order id
    "client_order_id": "client_order_id_7080019906774802432" //string, client_order_id
  }
}
```

| Parameter Name  | Type   | Description              |
| :-------------- | :----- | :----------------------- |
| meta_name       | string | name of the meta-product |
| order_id        | string | vendor order id          |
| client_order_id | string | client order id          |

Error Code:

| Code | Message                                                |
| ---- | ------------------------------------------------------ |
| 0    | Success                                                |
| 1001 | retryable error (may retry according to product logic) |
| 1002 | other error1  eg. No price (will **NOT** retry)        |
| 1003 | other error2  eg. price expire (will **NOT** retry)    |

### 1.6.5. Get Redeem Quote

- Get quote for redeeming a previously placed order by the info of the order.

- The quote provided by the vendor should remain valid within 1 minute; otherwise, it may result in a higher failure rate.

- The timeout for this interface call is 1000ms. Peak QPS: 50.

- URL: /mp/api/v1/structured/quote/redeem

- method: Get

- Parameters: json in body

| Key           | Type   | Required | Description                                       |
| ------------- | ------ | -------- | ------------------------------------------------- |
| meta_name     | string | yes      | name of the meta-product. eg, "snowball", "dcp"   |
| order_id      | string | yes      | vendor order ID                                   |

- Response: application/json

Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "quote_id": "7633327782363410433", //string, quote id
    "meta_name": "snowball", // string, name of the meta-product
    "redeem_settle_amount": "1.2", // string, final settle amount in string format
    "price_expire_time_mill": 1691727892000, //int, price expire time in millisecond
    "order_id": "708001990677480243210", //string, vendor order ID
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "quote_id": "7633327782363410434", //string, quote id
    "meta_name": "sharkfin", // string, name of the meta-product
    "redeem_settle_amount": "1.05", // string, final settle amount in string format
    "price_expire_time_mill": 1691727892000, //int, price expire time in millisecond
    "order_id": "708001990677480243211", //string, vendor order ID
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "dnt", // string, name of the meta-product
    "redeem_settle_amount": "1.05", // string, final settle amount in string format
    "price_expire_time_mill": 1691727892000, //int, price expire time in millisecond
    "order_id": "708001990677480243212" //string, vendor order ID
  }
}
```


| Parameter Name         | Type   | Description                                                                            |
| :--------------------- | :----- | :------------------------------------------------------------------------------------- |
| quote_id               | string | quote id                                                                               |
| meta_name              | string | name of the meta-product                                                               |
| redeem_settle_amount   | string | for redemption with a final amount: amount in string format, redeem_settle_amount >= 0 |
| price_expire_time_mill | int    | price expire time in millisecond                                                       |
| order_id               | string | vendor order ID                                                                        |

Error Code:

| Code | Message                                                |
| ---- | ------------------------------------------------------ |
| 0    | Success                                                |
| 1001 | retryable error (may retry according to product logic) |
| 1002 | other error1  eg. No price (will **NOT** retry)        |
| 1003 | other error2  eg. price expire (will **NOT** retry)    |

### 1.6.6. Redeem Order

- Redeem order. before calling redeem order api, redeem quote api is called to get a redeem quote.

- The timeout for this interface call is 2000ms. Peak QPS: 50.

- URL: /mp/api/v1/structured/order/redeem

- method: Post

- Parameters: json in body

| Key                  | Type   | Required           | Description                                                                            |
| -------------------- | ------ | ------------------ | -------------------------------------------------------------------------------------- |
| meta_name            | string | yes                | name of the meta-product. eg, "snowball", "dcp"                                        |
| order_id             | string | yes                | vendor order id                                                                        |
| client_redeem_id     | string | yes                | client redeem id, for idempotence                                                      |
| quote_id             | string | follows quote      | quote id                                                                               |
| redeem_settle_amount | string | if quote_id unused | for redemption with a final amount: amount in string format, redeem_settle_amount >= 0 |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "meta_name": "dcp", // string, name of the meta-product
    "order_id": "7080019906774802432", //string, vendor order id
    "redeem_id": "redeem_7080019906774802432", //string, vendor redeem id
    "client_redeem_id": "client_redeem_7080019906774802432" //string, client_redeem_id for idempotence
  }
}
```

| Parameter Name   | Type   | Description                       |
| :--------------- | :----- | :-------------------------------- |
| meta_name        | string | name of the meta-product          |
| order_id         | string | vendor order id                   |
| redeem_id        | string | vendor redeem id                  |
| client_redeem_id | string | client redeem id, for idempotence |

Error Code:

| Code | Message                                                |
| ---- | ------------------------------------------------------ |
| 0    | Success                                                |
| 1001 | retryable error (may retry according to product logic) |
| 1002 | other error1  eg. No price (will **NOT** retry)        |
| 1003 | other error2  eg. price expire (will **NOT** retry)    |

### 1.6.7. Query Order by Order ID or Client Order ID

Get single order info by order_id or client_order_id

- URL: /mp/api/v1/structured/order

- method: Get

- Parameters: in query

| Key             | Type   | Required | Description                                     |
| --------------- | ------ | -------- | ----------------------------------------------- |
| meta_name       | string | yes      | name of the meta-product. eg, "snowball", "dcp" |
| order_id        | string | optional | vendor order id                                 |
| client_order_id | string | yes      | client order id                                 |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": { //object,
    "meta_name": "sharkfin", // string, name of the meta-product
    "order_id": "7080019906774802435", //string, order id
    "client_order_id": "client_order_id_7080019906774802435", //string, client_order_id
    "order_status": 100, //int, 0 : Processing, 100 : success, 110 : failed
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source": "DERIBIT",  //tracking source
    "type": "CALL", //string, option type
    "term_mill": 604800000, //int, term. e.g. 604800000 is 7 days
    "take_profit_price": "40000", //string, take-profit price in string format
    "protection_price": "31000", //string, protection price in string format
    "is_evaluated": true, // prices are evaluated
    "invest_amount": "10", //string, investment amount from user
    "zero_price_apy": "0.01",
    "low_price_apy": "0.01",
    "high_price_apy": "0.02",
    "apy_points": [
        {"price": "31000", "apy": "0.1"},
        {"price": "40000", "apy": "0.2"}
    ],
    "success_time_mill": 1692926956000, //int, millisecond UNIX epoch of the order being successfully placed
    "value_time_mill": 1692926956000, //int, millisecond UNIX epoch of the order starting to accure interest
    "actual_settled_time_mill": 1693555200000, //int, vendor settled time
    "actual_settled_price": "40000", //string, option settled index price in string format
    "actual_settled_currency": "USDT", //string, settled currency
    "actual_settled_amount": "10.00398429" //string, settled amount
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": { //object,
    "meta_name": "seagull", // string, name of the meta-product
    "order_id": "7080019906774802432", //string, order id
    "client_order_id": "client_order_id_7080019906774802432", //string, client_order_id
    "order_status": 0, //int, 0 : Processing, 100 : success, 110 : failed 
    "invest_currency": "USDC", //string, investment currency
    "underlying": "BTC-USDC", //string, underlying asset
    "tracking_source": "DERIBIT",  //tracking source
    "type": "CALL", //string, option type
    "term_mill": 1692950400000, //int, term
    "strike_convert_price": "34000", //string, the price which triggers currency conversion
    "strike_start_price": "38000", //string, the price which starts to generate additional profit
    "strike_end_price": "39000", //string, the price which yields the max profit
    "invest_amount": "100", //string, investment amount from user
    "low_price_quantity": "0.07", //string, the profit if price lower than price interval
    "high_price_quantity": "0.51", //string, the profit if price higher than profit interval
    "success_time_mill": 1692926956000, //int, millisecond UNIX epoch of the order being successfully placed
    "value_time_mill": 1692926956000, //int, millisecond UNIX epoch of the order starting to accure interest
    "actual_settled_time_mill": 1692950400000, //int, vendor settled time
    "actual_settled_price": "32000", //string, option settled index price in string format
    "actual_settled_currency": "BTC", //string, settled currency
    "actual_settled_amount": "0.00156359" //string, settled amount
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": { //object,
    "meta_name": "dnt", // string, name of the meta-product
    "order_id": "9080019906774802435", //string, order id
    "client_order_id": "client_order_id_9080019906774802435", //string, client_order_id
    "order_status": 100, //int, 0 : Processing, 100 : success, 110 : failed
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source": "BINANCE",  //tracking source
    "term_mill": 1692950400000, //int, term
    "take_profit_price": "40000", //string, upper bound price to trigger knock event
    "protection_price": "31000", //string, lower bound price to trigger knock event
    "invest_amount": "10", // investment amount from user, in string format
    "success_time_mill": 1692926956000, //int, millisecond UNIX epoch of the order being successfully placed
    "booking_premium": "0.2", // utilized to subscribe DNT
    "booking_quantity": "0.4", // quantity/share of the subscription of DNT
    "funding_amount": "0.5" // the funding amount (this product has funding)
    "actual_settled_time_mill": 1693555200000, //int, settled time
    "actual_settled_price": "41000", //string, option settled index price in string format
    "actual_settled_currency": "USDT", //string, settled currency
    "actual_settled_amount": "10.3" //string, settled amount
  }
}
```

| Parameter Name           | Type   | Description                                                                                                                                       |
| :----------------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| meta_name                | string | name of the meta-product                                                                                                                          |
| order_id                 | string | vendor order id                                                                                                                                   |
| client_order_id          | string | client_order_id                                                                                                                                   |
| order_status             | int    | 0 : Processing, 100 : success, 110 : failed                                                                                                       |
| invest_currency          | string | investment currency                                                                                                                               |
| underlying               | string | underlying asset                                                                                                                                  |
| tracking_source          | string | the fixing convention used to settle product payment                                                                                              |
| type                     | string | type, CALL or PUT                                                                                                                                 |
| term_mill                | int    | product's term                                                                                                                                    |
| take_profit_price        | string | for products with a price interval: the price which triggers take-profit, in string format                                                        |
| protection_price         | string | for products with a price interval, the price which triggers stop-loss, in string format                                                          |
| is_evaluated             | bool   | for products whose attributes (such as prices) are evaluated at a later time, this flag indicates whether the attributes are evaluated or not     |
| invest_amount            | string | investment amount from user, in string format                                                                                                     |
| booking_quantity         | string | for products with one booking quantity, in string format, > 0, e.g. premium amount in DCP, booking share in DNT                                   |
| booking_premium          | string | the premium amount utilized for the subscription                                                                                                  |
| funding_amount           | string | the funding amount (if the product supports it)                                                                                                   |
| zero_price_apy           | string | for products with a price interval: the annual percentage yield when the price is zero, in string format                                          |
| low_price_apy            | string | for products with a price interval: the annual percentage yield when the price is smaller than the price interval, in string format               |
| high_price_apy           | string | for products with a price interval: the annual percentage yield when the price is larger than the price interval, in string format                |
| apy_points               | array  | for products with a price interval: within the price interval, points of {price, annual percentage yield} construct the piecewise function of APY |
| apy_points.price         | string | price, in string format                                                                                                                           |
| apy_points.apy           | string | annual percentage yield, in string format                                                                                                         |
| strike_convert_price     | string | the price which triggers currency conversion, in string format                                                                                    |
| strike_start_price       | string | (seagull) the price which starts to generate additional profit, in string format                                                                  |
| strike_end_price         | string | (seagull) the price which yields the max profit, in string format                                                                                 |
| low_price_quantity       | string | (seagull) the profit if price lower than price interval, in string format                                                                         |
| high_price_quantity      | string | (seagull) the profit if price higher than price interval, in string format                                                                        |
| success_time_mill        | int    | millisecond UNIX epoch of the order being successfully placed                                                                                     |
| value_time_mill          | int    | millisecond UNIX epoch of the order starting to accure interest                                                                                   |
| actual_settled_time_mill | int    | vendor settled time                                                                                                                               |
| actual_settled_price     | string | actual settlement price in string format                                                                                                          |
| actual_settled_currency  | string | settled currency                                                                                                                                  |
| actual_settled_amount    | string | settled amount                                                                                                                                    |

### 1.6.8. Query Orders

Get order list by various conditions.

- URL: /mp/api/v1/structured/orders

- method: Get

| Parameter Name         | Type   | Required | Description                                                                                | Example            |
| :--------------------- | :----- | :------- | :----------------------------------------------------------------------------------------- | :----------------- |
| meta_name              | string | yes      | name of the meta-product.                                                                  | "snowball", "dcp"  |
| invest_currency        | string | no       | investment currency                                                                        | USDT, USDC         |
| underlying             | string | no       | underlying asset                                                                           | BTC-USDT, ETH-USDC |
| type                   | string | no       | product's type, in upper case;                                                             | CALL, PUT          |
| take_profit_price      | string | no       | for products with a price interval: the price which triggers take-profit, in string format |                    |
| protection_price       | string | no       | for products with a price interval, the price which triggers stop-loss, in string format   |                    |
| settle_time_mill_start | int    | no       | start settlement time in millisecond                                                       |                    |
| settle_time_mill_end   | int    | no       | end settlement time in millisecond                                                         |                    |
| limit                  | int    | no       | limit for pagination , default is 50                                                       |                    |
| last_order_id          | string | no       | offset for pagination                                                                      |                    |

> Above request parameters fields take effects when not empty or 0

- Response: application/json

| Parameter Name | Type    | Description                                                                        |
| :------------- | :------ | :--------------------------------------------------------------------------------- |
| count          | int     | total count                                                                        |
| items          | objects | order list, order info is same as in get order api, except for redeem record list. |

### 1.6.9. Query Redeem Order by Redeem ID or Client Redeem ID

Get single order info by redeem_id or client_redeem_id

- URL: /mp/api/v1/structured/redeem_order

- method: Get

- Parameters: in query

| Key              | Type   | Required | Description              |
| ---------------- | ------ | -------- | ------------------------ |
| meta_name        | string | yes      | name of the meta-product |
| redeem_id        | string | optional | vendor redeem id         |
| client_redeem_id | string | yes      | client redeem order id   |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": { //object, redeem order
    "meta_name": "sharkfin", // string, name of the meta-product
    "order_id": "7080019906774802433", //string,vendor order id
    "client_order_id": "client_order_id_7080019906774802433", //string, client_order_id
    "redeem_id": "redeem_7080019906774802435", //string,vendor redeem id
    "client_redeem_id": "redeem_7080019906774802435", //string, client redeem id
    "redeem_currency": "USDT", //string, redeem currency
    "redeem_settle_amount": "1.05", //string, redeem settle amount in string format
    "redeem_status": 100, //int, 0 : Processing, 100 : success, 110 : failed
    "redeem_active_time_mill": 1692926956000, //int,redeem active time
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source": "DERIBIT",  //tracking source
    "type": "CALL", //string, type
    "term_milli": 604800000, //int, term
    "take_profit_price": "40000", //string, take-profit price in string format
    "protection_price": "31000" //string, protection price in string format
  }
}
```

| Parameter Name          | Type   | Description                                                                    |
| :---------------------- | :----- | :----------------------------------------------------------------------------- |
| meta_name               | string | name of the meta-product                                                       |
| order_id                | string | order id                                                                       |
| client_order_id         | string | client_order_id                                                                |
| redeem_id               | string | redeem order id                                                                |
| client_redeem_id        | string | client redeem id                                                               |
| redeem_currency         | string | redeem currency                                                                |
| redeem_status           | int    | 0 : Processing, 100 : success, 110 : failed                                    |
| redeem_active_time_mill | int    | when order redeem success,filled this item with order redeem success time      |
| redeem_settle_amount    | string | redeem settle amount in string format                                          |
| invest_currency         | string | investment currency                                                            |
| underlying              | string | underlying asset                                                               |
| tracking_source         | string | tracking source refers to the fixing convention used to settle product payment |
| type                    | string | type                                                                           |
| term_mill               | int    | product's term                                                                 |
| take_profit_price       | string | for products with a price interval: the price which triggers take-profit       |
| protection_price        | string | for products with a price interval: the price which triggers stop-loss         |

### 1.6.10. Check Settlement per order

vendor check settlement price, settlement amount and settlement currency of an order

- post matrixport settlement price, settlement amount and settlement currency, vendor check and return check result,if check failed response is an error code.

- URL: /mp/api/v1/structured/settlement/order

- method: POST

| Parameter Name   | Type   | Required                        | Description                     | Example       |
| :--------------- | :----- | :------------------------------ | :------------------------------ | :------------ |
| meta_name        | string | yes                             | name of the meta-product        | dcp           |
| settle_time_mill | int    | yes                             | settlement time in milliseconds | 1692950400000 |
| order_id         | string | yes                             | vendor order ID                 |               |
| currency         | string | yes                             | settle currency                 | BTC           |
| vendor_net_pay   | string | yes                             | settle amount                   | 0.1           |
| settlement_index | string | yes                             | settlement price                | 38000.12      |
| initial_price    | string | only when settlment requires it | price on the first value date   | 37500.99      |

Post Data Example:

```js
{
    "meta_name": "seagull", // string, name of the meta-product
    "settle_time_mill": 1692950400000,
    "order_id": "138974182741274127",
    "currency": "BTC", //string, settle currency
    "vendor_net_pay": "100", //string, expected settlement amount. if amount > 0 vendor should transfer to matrixport, if less than zero matrixport should transfer to vendor.
    "settlement_index":"26000"// settlement index price
}
```

```js
{
    "meta_name": "snowball", // string, name of the meta-product
    "settle_time_mill": 1692950400000,
    "order_id": "138974127412741892",
    "currency": "ETH", //string, settle currency
    "vendor_net_pay": "5", //string, expected settlement amount. if amount > 0 vendor should transfer to matrixport, if less than zero matrixport should transfer to vendor.
    "settlement_index":"2100", // settlement index price
    "initial_price": "2000" // price on the first value date
}
```

```js
{
    "meta_name": "dnt", // string, name of the meta-product
    "settle_time_mill": 1692950400000,
    "order_id": "138974127412741893",
    "currency": "USDT", //string, settle currency
    "vendor_net_pay": "5", //string, expected settlement amount. if amount > 0 vendor should transfer to matrixport, if less than zero matrixport should transfer to vendor.
    "settlement_index":"2100" // settlement index price
}
```

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "settle_time_mill": 1692950400000,
    "meta_name": "seagull", // string, name of the meta-product
    "order_id": "138974182741274127",
    "valid":true,
    "settle_currency":"BTC", //string, settlement currency
    "vendor_net_pay":"100", //string, settlement amount. if amount > 0 vendor should transfer to matrixport, if less than zero matrixport should transfer to vendor. 
    "request_vendor_net_pay":"100",
    "invest_currency": "USDC", //string, investment currency
    "underlying": "BTC-USDC", //string, underlying asset
    "tracking_source":"DERIBIT",  // tracking source eg. DERIBIT BINANCE
    "settlement_index":"26000", // vendor settlement index price
    "request_settlement_index":"26000"
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "settle_time_mill": 1692950400000,
    "meta_name": "snowball", // string, name of the meta-product
    "order_id": "13892741271093",
    "valid":true,
    "settle_currency":"BTC", //string, settlement currency
    "vendor_net_pay":"100", //string, settlement amount. if amount > 0 vendor should transfer to matrixport, if less than zero matrixport should transfer to vendor. 
    "request_vendor_net_pay":"100",
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source":"DERIBIT",  // tracking source eg. DERIBIT BINANCE
    "settlement_index":"26000", // vendor settlement index price
    "request_settlement_index":"26000",
    "initial_price": "30000.5",
    "request_initial_price": "30000.5"
  }
}
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "settle_time_mill": 1692950400000,
    "meta_name": "dnt", // string, name of the meta-product
    "order_id": "138974182741278957",
    "valid":true,
    "settle_currency":"USDT", //string, settlement currency
    "vendor_net_pay":"100", //string, settlement amount. if amount > 0 vendor should transfer to matrixport, if less than zero matrixport should transfer to vendor. 
    "request_vendor_net_pay":"100",
    "invest_currency": "USDT", //string, investment currency
    "underlying": "BTC-USDT", //string, underlying asset
    "tracking_source":"BINANCE", // tracking source
    "settlement_index":"26000", // vendor settlement index price
    "request_settlement_index":"26000"
  }
}
```

| Parameter Name           | Type   | Description                     | Example       |
| :----------------------- | :----- | :------------------------------ | :------------ |
| settle_time_mill         | int    | settlement time in milliseconds | 1692950400000 |
| meta_name                | string | name of the meta-product        | dcp           |
| order_id                 | string | vendor order id                 | 1237189247194 |
| valid                    | bool   |                                 |               |
| settle_currency          | string | settlement currency             | BTC           |
| vendor_net_pay           | string | vendor's expected settle amount | 100           |
| request_vendor_net_pay   | string | client's expected settle amount |               |
| invest_currency          | string | order's investment currency     | USDT          |
| underlying               | string | order's underlying asset        | BTC-USDT      |
| tracking_source          | string | order's tracking source         | DERIBIT       |
| settlement_index         | string | vendor's expected settle index  | 26000         |
| request_settlement_index | string | client's expected settle index  | 26000         |
| initial_price            | string | vendor's initial price          | 37500.99      |
| request_initial_price    | string | client's initial price          | 37500.99      |

### 1.6.11. Monthly profit check

- post matrixport calc monthly profit,if check failed response an error code.
- matrixport monthly profit = SUM$(Premium *Platform Profit Rate* relevant settlement price of settlement currency with respect to the Structured Product on the maturity date of the Order)
- URL: /mp/api/v1/structured/monthly-profit/check

- method: Post

- Parameters: json in body
  
| Key                   | Type   | Required | Description                                               |
| --------------------- | ------ | -------- | :-------------------------------------------------------- |
| meta_name             | string | yes      | name of the meta-product. eg, "snowball", "trend"         |
| start_time_mill       | int    | yes      | start expiry time in millisecond UNIX epoch (inclusive)   |
| end_time_mill         | int    | yes      | end  expiry  time in millisecond UNIX epoch (exclusive)   |
| profit                | string | yes      | amount of usdc distributed to matrixport in string format |
| infos                 | array  | optional | price info                                                |
| infos.timestamp       | int    |          | expiry times for each day                                 |
| infos.underlying_pair | string |          | underlying pair                                           |
| infos.price           | string |          | price between underlying pair                             |

Post Data Example:

```js
{
  "meta_name": "trend",
  "start_time_mill": 1698768480000,
  "end_time_mill": 1701360480000,
  "profit": "28123.8999"
}
```

```js
Will find orders with expiration maturity_time >= 1698768480000 and maturity_time < 1701360480000
```

Response: application/json
Example:

```js
{
  "code": 0,
  "data": {
    "valid": true,
    "vendor_pay_profit": "28123.8999",
    "request_vendor_pay_profit": "28123.8999"
    "infos": [
      {
        "timestamp": 1697356800000,
        "underlying_pair": "USDCUSDT",
        "price": "1.00003033"
      }
    ]
  },
  "message": ""
}
```

| Parameter Name            | Type   | Description              |
| :------------------------ | ------ | :----------------------- |
| valid                     | bool   | true means bill check ok |
| vendor_pay_profit         | string | vendor's expect profit   |
| request_vendor_pay_profit | string | client's expect profit   |

### 1.6.12. Renew diff check

Check the total accumulated difference between mp and vendor after the final renewal order is settled.

- URL: /mp/api/v1/structured/info_renew/diff_check

- method: Post

- Parameters: json in body

| Key              | Type   | Required | Description                                       |
| ---------------- | ------ | -------- | :------------------------------------------------ |
| meta_name        | string | yes      | name of the meta-product. eg, "snowball", "trend" |
| vendor_order_id  | string | yes      | the final renewal order's vendor order id         |
| total_difference | string | yes      | accumulated difference in string format           |
| currency         | string | yes      | currency                                          |

Post Data Example:

```js
{
  "meta_name": "trend",
  "vendor_order_id": "7080019906774802432",
  "total_difference": "10.3",
  "currency": "USDT"
}
```

Response: application/json
Example:

```js
{
  "code": 0,
  "data": {
    "valid": true,
    "total_difference": "10.3",
    "request_total_difference": "10.3",
    "currency": "USDT"
  },
  "message": ""
}
```

| Parameter Name           | Type   | Description                  |
| :----------------------- | ------ | :--------------------------- |
| valid                    | bool   |                              |
| total_difference         | string | vendor's expected difference |
| request_total_difference | string | client's expected difference |
| currency                 | string | currency                     |


### 1.6.13 Audit orders

Check the total investment amount and total renewal amount between the mp and vendor for a given time range.

- URL: /mp/api/v1/structured/audit_orders

- method: Post

- Parameters: json in body

| Key                | Type   | Required | Description                                                           |
| ------------------ | ------ | -------- | :-------------------------------------------------------------------- |
| meta_name          | string | yes      | name of the meta-product. eg, "snowball", "trend"                     |
| start_time_mill    | int    | yes      | The start order successful time in millisecond UNIX epoch (inclusive) |
| end_time_mill      | int    | yes      | The end order successful time in millisecond UNIX epoch (exclusive)   |
| count              | int    | yes      | The count of successful order for a give time range                   |
| infos              | list   | optional | investment and renewal info                                           |
| infos.currency     | string | optional | investment currency                                                   |
| infos.total_amount | string | optional | total investment amount                                               |
| infos.renew_amount | string | optional | total renewal amount                                                  |

Post Data Example:

```js
{
  "meta_name": "trend",
  "start_time_mill": 1698768480000,
  "end_time_mill": 1701360480000,
  "count": 2,
  "infos": [
    {
      "currency": "USDT",
      "total_amount": "8700.5",
      "renew_amount": "1200"
    },
    {
      "currency": "ETH",
      "total_amount": "1.5",
      "renew_amount": "0.4"
    }
  ]
}
```

```js
{
  "meta_name": "dnt",
  "start_time_mill": 1719802800000,
  "end_time_mill": 1719806400000,
  "count": 5
}
```


Response: application/json
Example:

```js
{
  "code": 0,
  "data": {
    "valid": true,
    "count": 2,
    "request_count": 2,
    "infos": [
      {
        "valid": true,
        "currency": "USDT",
        "total_amount": "8700.5",
        "request_total_amount": "8700.5",
        "renew_amount": "1200",
        "request_renew_amount": "1200"
      },
      {
        "valid": true,
        "currency": "ETH",
        "total_amount": "1.5",
        "request_total_amount": "1.5",
        "renew_amount": "0.4",
        "request_renew_amount": "0.4",
      }
    ]
  },
  "message": ""
}
```

| Parameter Name             | Type   | Description                               |
| :------------------------- | ------ | :---------------------------------------- |
| valid                      | bool   |                                           |
| count                      | int    | vendor's expected order count             |
| request_count              | int    | client's expected order count             |
| infos                      | list   | investment and renewal info               |
| infos.valid                | bool   |                                           |
| infos.currency             | string | investment currency                       |
| infos.total_amount         | string | vendor's expected total investment amount |
| infos.request_total_amount | string | client's expected total investment amount |
| infos.renew_amount         | string | vendor's expected total renewal amount    |
| infos.request_renew_amount | string | client's expected total renewal amount    |
