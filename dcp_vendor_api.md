# 1. dcp-api

<!-- TOC -->

- [1. dcp-api](#1-dcp-api)
  - [1.1. Encoding](#11-encoding)
  - [1.2. Common parameters in HTTP headers](#12-common-parameters-in-http-headers)
  - [1.3. Authentication](#13-authentication)
  - [1.4. Private API mandatory fields](#14-private-api-mandatory-fields)
  - [1.5. Signature algorithm](#15-signature-algorithm)
  - [1.6. DCP Vendor Rest API](#16-dcp-vendor-rest-api)
    - [1.6.1. Get Products](#161-get-products)
    - [1.6.2. Get Quote](#162-get-quote)
    - [1.6.3. Place Order](#163-place-order)
    - [1.6.4. Redeem Order](#164-redeem-order)
    - [1.6.5. Query Order by Order ID or Client Order ID](#165-query-order-by-order-id-or-client-order-id)
    - [1.6.6. Query Orders](#166-query-orders)
    - [1.6.7. Query Redeem Order by Redeem ID or Client Order ID](#167-query-redeem-order-by-redeem-id-or-client-order-id)
    - [1.6.8. settle price check](#168-settle-price-check)
    - [1.6.9. Check Settlement summary](#169-check-settlement-summary)
    - [1.6.10. Quarterly profit check](#1610-quarterly-profit-check)

<!-- /TOC -->

Dual-Coin Vendor API Doc
Vendor should offer the following APIs

## 1.1. Encoding

The request and response data for all interfaces is formatted in UTF-8. Content for responses in all interfaces is formatted in JSON.

## 1.2. Common parameters in HTTP headers

Below common parameters in HTTP headers are recommended to added in request, facilitating debugging and tracing each request.

| Key          | Type   | Position | Required | Description                                                     |
|--------------|--------|----------|----------|-----------------------------------------------------------------|
| x-request-id | string | header   | false    | uuid for trace the request, not included in signing the request |

## 1.3. Authentication

## 1.4. Private API mandatory fields
- Client put Access Key in http request header X-Access-Key
- Client and Server use same private secret key to sign the request.
- Client must add `timestamp` (epoch in millisecond) field in request parameter (query string for GET, json body for POST), API server will check this timestamp, if `abs(server_timestamp - request_timestamp) > 5000`, the request will be rejected.
- `timestamp` must be integer, not quoted string.
- Client must add `signature` field in request parameter (query string for GET, json body for POST).
- For POST request, Header `Content-Type` should be set as `application/json`.

| Key       | Type   | Position      | Required | Description              |
|-----------|--------|---------------|----------|--------------------------|
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

## 1.6. DCP Vendor Rest API

### 1.6.1. Get Products

- Returns a list of products

- The timeout for this interface call is 1000ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/products

- method: Get

- Parameters: in query

| Key             | Type   | Required | Description                                                                                   | Example            |
|-----------------|--------|----------|-----------------------------------------------------------------------------------------------|--------------------|
| underlying_pair | string | no       | Option's underlying currency pair, separate by "-"; Empty means all underlying currency pairs | BTC-USDT, BTC-USDC |
| type            | string | no       | option's type , in upper case; Empty means all types                                          | CALL, PUT          |

- Response: application/json

Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "items": [ //array, products
        {
            "underlying_pair": "BTC-USDC", //string, underlying pair
            "tracking_source": "deribit",  //trade source deribit or binance 
            "type": "CALL", // optionstring, type
            "settle_time_mill": 1692950400000, //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
            "strike_price": "30000", //string, strike price in string format
            "deposit_currency": "BTC", //string, deposit currency
            "min_buy": "0.1", //string, minimal buy amount
            "max_buy": "100", //string, maximal buy amount
            "mini_buy_step": "0.1", // mini buy amount step, 0.1 means deposit amount can add or reducs 0.1 pre 
            "yield_rate": "0.01", //string, yield rate; not converted:  payback_amount = deposit_amount * (1 + yield_rate) ; converted:  payback_amount = deposit_amount *(/) target_price * (1 + yield_rate)
            "redeemable":true //bool, whether this product can be early redeemed.
        },
    ]
  }
}
```

| Parameter Name         | Type   | Description                                                                                                                                                       |
|:-----------------------|:-------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| items                  | array  | products                                                                                                                                                          |
| items.underlying_pair  | string | underlying pair                                                                                                                                                   |
| items.tracking_source  | string | tracking source                                                                                                                                                   |
| items.type             | string | type                                                                                                                                                              |
| items.settle_time_mill | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.                                                                               |
| items.strike_price     | string | strike price in string format                                                                                                                                     |
| items.deposit_currency | string | deposit currency                                                                                                                                                  |
| items.min_buy          | string | minimal buy amount                                                                                                                                                |
| items.max_buy          | string | maximal buy amount, max_buy >= min_buy                                                                                                                            |
| items.mini_buy_step    | string | buy amount mini step (if set 0.1 and mini_buy set 1 then available deposit amount is 1.1,1.2,1.3,1.4....)                                                         |
| items.yield_rate       | string | yield rate; not converted: payback*amount = deposit_amount * (1 + yield_rate) ; converted: payback_amount = deposit_amount \*(/) target_price \* (1 + yield_rate) |
| items.redeemable       | bool   | whether this product can be early redeemed.                                                                                                                       |

---

### 1.6.2. Get Quote

- Get quote for placing or redeeming an order by instrument info of option.

- The quote provided by the vendor should remain valid within 1 minute; otherwise, it may result in a higher user purchase failure rate.

- The timeout for this interface call is 1000ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/quote

- method: Post

- Parameters: json in body

| Key              | Type   | Required | Description                                                                         |
|------------------|--------|----------|-------------------------------------------------------------------------------------|
| deposit_currency | string | yes      | deposit currency                                                                    |
| deposit_amount   | string | yes      | deposit_amount in string format                                                     |
| side             | string | yes      | BUY or SELL; SELL: place new order, BUY: redeem exist order                         |
| underlying_pair  | string | yes      | underlying pair                                                                     |
| tracking_source  | string | yes      | tracking source (eg. deribit, binance)                                              |
| type             | string | yes      | option type                                                                         |
| settle_time_mill | int    | yes      | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |
| strike_price     | string | yes      | strike price in string format                                                       |

- Response: application/json

Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "quote_id": "6633327782363410432", //string, quote id  (if response emtpy,place order will post an empty quote id)
    "underlying_pair": "BTC-USDC", //string, underlying pair
    "tracking_source": "deribit",  //trade source  deribit,binance etc...
    "type": "CALL", //string, option type
    "settle_time_mill": 1692950400000, //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
    "strike_price": "30000", //string, strike price in string format
    "premium_amount": "0.1", //string, premium_amount in string format
    "price_expire_time_mill": 1691727892000, //int, price expire time in millisecond,
    "side": "BUY", //string, side, BUY or SELL
    "deposit_currency": "BTC", //string, currency
    "deposit_amount": "10" //string, amount in string format
  }
}
```

| Parameter Name         | Type   | Description                                                                         |
|:-----------------------|:-------|:------------------------------------------------------------------------------------|
| quote_id               | string | quote id  (if response emtpy,place order will post an empty quote id)               |
| underlying_pair        | string | underlying pair                                                                     |
| tracking_source        | string | tracking source (eg. deribit, binance)                                              |
| type                   | string | option type   (CALL or PUT)                                                         |
| settle_time_mill       | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |
| strike_price           | string | strike price in string format                                                       |
| premium_amount         | string | premium_amount in string format                                                     |
| price_expire_time_mill | int    | price expire time in millisecond                                                    |
| side                   | string | side, BUY or SELL                                                                   |
| deposit_currency       | string | currency                                                                            |
| deposit_amount         | string | amount in string format                                                             |

Error Code:

| Code | Message                                                         |
|------|-----------------------------------------------------------------|
| 0    | Success                                                         |
| 1001 | some error (will retry several time according to product logic) |
| 1002 | other error1  eg. No price (will **NOT** retry)                 |
| 1003 | other error2  eg. price expire (will **NOT** retry)             |

### 1.6.3. Place Order

- Place Order by quote id,before place order will call quote api to get a quote.

- The timeout for this interface call is 2000ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/order

- method: Post

- Parameters: json in body

| Key              | Type   | Required | Description                                                                                               |
|------------------|--------|----------|-----------------------------------------------------------------------------------------------------------|
| quote_id         | string | no       | quote id, unique (if get quote api don't response quote id,place order will don't filled this item empty) |
| client_order_id  | string | yes      | client order id, for idempotent                                                                           |
| underlying_pair  | string | yes      | underlying pair, BTC-USDT,ETH-USDT                                                                        |
| tracking_source  | string | yes      | tracking source (eg. deribit, binance)                                                                    |
| type             | string | yes      | option type  (CALL or PUT)                                                                                |
| settle_time_mill | int    | yes      | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.                       |
| strike_price     | string | yes      | strike price in string format                                                                             |
| premium_amount   | string | yes      | premium_amount want get,in string format                                                                  |
| side             | string | yes      | side, BUY or SELL                                                                                         |
| deposit_currency | string | yes      | currency                                                                                                  |
| deposit_amount   | string | yes      | amount in string format                                                                                   |

- Response: application/json

Example:

```js
{
  "code" : 0,
  "message" : "",
  "data" : {
    "order_id" : "7080019906774802432", //string,vendor order id
    "client_order_id": "client_order_id_7080019906774802432" //string, client_order_id
  }
}
```

| Parameter Name  | Type   | Description     |
|:----------------|:-------|:----------------|
| order_id        | string | vendor order id |
| client_order_id | string | dcp order id    |

Error Code:

| Code | Message                                                         |
|------|-----------------------------------------------------------------|
| 0    | Success                                                         |
| 1001 | some error (will retry several time according to product logic) |
| 1002 | other error1  eg. No price (will **NOT** retry)                 |
| 1003 | other error2  eg. price expire (will **NOT** retry)             |

### 1.6.4. Redeem Order

- Redeem order,before call redeem order api,caller will call quote api get a redeem quote.

- The timeout for this interface call is 2000ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/order/redeem

- method: Post

- Parameters: json in body

| Key              | Type   | Required | Description                                                                                       |
|------------------|--------|----------|---------------------------------------------------------------------------------------------------|
| order_id         | string | yes      | vendor order id                                                                                   |
| client_redeem_id | string | yes      | client redeem id, for idempotent                                                                  |
| quote_id         | string | no       | quote id (if get quote api don't response quote id,place order will don't filled this item empty) |
| underlying_pair  | string | yes      | underlying pair, BTC-USDT,ETH-USDT                                                                |
| tracking_source  | string | yes      | tracking source (eg. deribit, binance)                                                            |
| type             | string | yes      | option type                                                                                       |
| settle_time_mill | int    | yes      | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.               |
| strike_price     | string | yes      | strike price in string format                                                                     |
| premium_amount   | string | yes      | premium_amount want get,in string format                                                          |
| side             | string | yes      | side, BUY or SELL                                                                                 |
| deposit_currency | string | yes      | currency  (order currency)                                                                        |
| deposit_amount   | string | yes      | amount in string format (order deposit amount)                                                    |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "order_id": "7080019906774802432", //string,dcp order id
    "redeem_id": "redeem_7080019906774802432", //string,vendor redeem id
    "client_redeem_id": "client_redeem_7080019906774802432" //string,dcp client_redeem_id
  }
}
```

| Parameter Name                 | Type   | Description                          |
|:-------------------------------|:-------|:-------------------------------------|
| order_id                       | string | order id                             |
| redeem_id                      | string | vendor redeem id                     |
| client_redeem_id               | string | dcp client redeem id, for idempotent |

Error Code:

| Code | Message                                                         |
|------|-----------------------------------------------------------------|
| 0    | Success                                                         |
| 1001 | some error (will retry several time according to product logic) |
| 1002 | other error1  eg. No price (will **NOT** retry)                 |
| 1003 | other error2  eg. price expire (will **NOT** retry)             |

### 1.6.5. Query Order by Order ID or Client Order ID

Get single order info by order_id or client_order_id

- URL: /mp/api/v1/dcp/order

- method: Get

- Parameters: in query

| Key             | Type   | Required | Description         |
|-----------------|--------|----------|---------------------|
| order_id        | string | optional | vendor order id     |
| client_order_id | string | yes      | dcp client order id |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": { //object,
    "order_id": "7080019906774802432", //string, order id
    "client_order_id": "client_order_id_7080019906774802432", //string, client_order_id
    "order_status": 0, //int, 0 : Processing, 100 : success, 110 : failed 
    "underlying_pair": "BTC-USDC" //string, underlying pair
    "tracking_source": "deribit",  //trade source
    "type": "CALL", //string, option type
    "settle_time_mill": 1692950400000, //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
    "strike_price": "30000" //string, strike price in string format
    "deposit_currency": "BTC", //string, deposit currency
    "deposit_amount": "1", //string, deposit amount in string format
    "actual_settled_time_mill": 1692950400000 //int, vendor settled time
    "actual_settled_price": "32000" //int, option settled index price in string format
    "actual_settled_currency": "BTC" //int, settled currency
    "actual_settled_amount": "1" //int, settled amount
    "premium_amount": "1" //int, premium amount
    "active_time_mill": 1692926956000, //int,active time
    "redeemable":true //bool, is order redeemable
  }
}
```

| Parameter Name                                | Type   | Description                                                                         |
|:----------------------------------------------|:-------|:------------------------------------------------------------------------------------|
| order_id                                      | string | vendor order id                                                                     |
| client_order_id                               | string | dcp client_order_id                                                                 |
| order_status                                  | int    | 0 : Processing, 100 : success, 110 : failed                                         |
| underlying_pair                               | string | underlying pair                                                                     |
| tracking_source                               | string | tracking source,deribit binance                                                     |
| type                                          | string | option type                                                                         |
| settle_time_mill                              | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |
| strike_price                                  | string | strike price in string format                                                       |
| deposit_currency                              | string | deposit currency                                                                    |
| deposit_amount                                | string | deposit amount in string format                                                     |
| active_time_mill                              | int    | when order success,filled this item with order success time                         |
| redeemable                                    | bool   | is order redeemable                                                                 |
| actual_settled_time_mill                      | int    | vendor settled time                                                                 |
| actual_settled_price                          | int    | actual option settled price in string format                                        |
| actual_settled_currency                       | int    | settled currency                                                                    |
| actual_settled_amount                         | int    | settled amount                                                                      |
| premium_amount                                | int    | premium amount                                                                      |

### 1.6.6. Query Orders

Get order list by various conditions.

- URL: /mp/api/v1/dcp/orders

- method: Get

| Parameter Name         | Type   | Required | Description                                         | Example            |
|:-----------------------|:-------|:---------|:----------------------------------------------------|:-------------------|
| underlying_pair        | string | no       | Option's underlying currency pair, separate by "-"; | BTC-USDT, BTC-USDC |
| type                   | string | no       | option's type , in upper case;                      | CALL, PUT          |
| strike_price           | string | no       | strike price in string format                       |                    |
| settle_time_mill_start | int    | no       | start option settle time in millisecond             |                    |
| settle_time_mill_end   | int    | no       | end option settle time in millisecond               |                    |
| deposit_currency       | string | no       | deposit currency                                    |                    |
| limit                  | int    | no       | limit for pagination , default is 50                |                    |
| offset                 | int    | no       | offset for pagination                               |                    |

> Above request parameters fields take effects when not empty or 0

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": { //array, order
    "count": 100, //int, total count
    "items": [ //objects, order list, order info is same as in get order api, except for redeem record list.
      { //object, order
        "order_id": "7080019906774802432", //string, order id
        "client_order_id": "client_order_id_7080019906774802432", //string, client_order_id
        "order_status": 0, //int, 0 : Processing, 100 : success, 110 : failed 
        "underlying_pair": "BTC-USDC" //string, underlying pair
        "tracking_source": "deribit",  //trade source
        "type": "CALL", //string, option type
        "settle_time_mill": 1692950400000, //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
        "strike_price": "30000" //string, strike price in string format
        "deposit_currency": "BTC", //string, deposit currency
        "deposit_amount": "1", //string, deposit amount in string format
        "actual_settled_time_mill": 1692950400000 //int, vendor settled time
        "actual_settled_price": "32000" //int, actual option settled index price in string format
        "actual_settled_currency": "BTC" //int, settled currency
        "actual_settled_amount": "1" //int, settled amount
        "premium_amount": "1" //int, premium amount
        "active_time_mill": 1692926956000, //int,active time
        "redeemable":true //bool, is order redeemable
      }
    ]
  }
}
```

| Parameter Name | Type    | Description                                                                        |
|:---------------|:--------|:-----------------------------------------------------------------------------------|
| count          | int     | total count                                                                        |
| items          | objects | order list, order info is same as in get order api, except for redeem record list. |


### 1.6.7. Query Redeem Order by Redeem ID or Client Order ID

Get single order info by redeem_id or client_redeem_id

- URL: /mp/api/v1/dcp/redeem_order

- method: Get

- Parameters: in query

| Key              | Type   | Required | Description                |
|------------------|--------|----------|----------------------------|
| redeem_id        | string | optional | vendor redeem id           |
| client_redeem_id | string | yes      | dcp client redeem order id |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": { //object, redeem order
    "order_id": "7080019906774802432", //string,vendor order id
    "client_order_id": "client_order_id_7080019906774802432", //string,dcp client_order_id
    "redeem_id": "redeem_7080019906774802432", //string,vendor redeem id
    "client_redeem_id": "redeem_7080019906774802432", //string,dcp redeem id
    "redeem_currency": "BTC", //string, redeemable currency
    "redeem_amount": "1", //string, redeem principal amount,equal order deposit amount, in string format
    "redeem_settle_amount": "1", //string, redeem settle amount in string format
    "redeem_status": 0, //int, 0 : Processing, 100 : success, 110 : failed 
    "redeem_active_time_mill": 1692926956000, //int,redeem active time
    "underlying_pair": "BTC-USDC", //string, underlying pair
    "tracking_source": "deribit",  //trade source
    "type": "CALL", //string, option type
    "settle_time_mill": 1692950400000, //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
    "strike_price": "30000", //string, strike price in string format
    "premium_amount": "-1" //int, premium amount, <= 0 
  }
}
```

| Parameter Name          | Type   | Description                                                                         |
|:------------------------|:-------|:------------------------------------------------------------------------------------|
| order_id                | string | order id                                                                            |
| client_order_id         | string | client_order_id                                                                     |
| redeem_id               | string | order id                                                                            |
| client_redeem_id        | string | redeem id                                                                           |
| redeem_currency         | string | redeemable currency                                                                 |
| redeem_amount           | string | redeem amount in string format                                                      |
| redeem_status           | int    | 0 : Processing, 100 : success, 110 : failed                                         |
| redeem_active_time_mill | int    | when order redeem success,filled this item with order redeem success time           |
| redeem_settle_amount    | string | redeem settle amount in string format                                               |
| underlying_pair         | string | underlying pair                                                                     |
| tracking_source         | string | tracking source,deribit binance                                                     |
| type                    | string | option type                                                                         |
| settle_time_mill        | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |
| strike_price            | string | strike price in string format                                                       |
| premium_amount          | string | premium amount                                                                      |


### 1.6.8. settle price check
vendor check settle price
- post matrixport settle price,vendor check and retrun check result,if check failed response an error code.

- URL: /mp/api/v1/dcp/settlement/fixing_list

- method: Post

- Parameters: json in body

| Key                    | Type   | Required | Description                                                                         |
|------------------------|--------|----------|-------------------------------------------------------------------------------------|
| settle_time_mill       | int    | yes      | option settle time in millisecond                                                   |
| infos                  | array  | yes      | infos                                                                               |
| infos.underlying_pair  | string | yes      | underlying pair                                                                     |
| infos.tracking_source  | string | yes      | tracking source eg. deribit binance...                                              |
| infos.settlement_index | string | yes      | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |

Post Data Example:

```js
{
  "settle_time_mill": 1692950400000,
  "infos": [
    {
      "underlying_pair": "BTC-USDT", // underlying pair
      "tracking_source": "deribit", // tracking source eg. deribit binance
      "settlement_index": "26000" // dcp settlement index price
    }
  ]
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
    "infos":[ 
      {
        "underlying_pair":"BTC-USDT", // underlying pair
        "tracking_source":"deribit",  // tracking source eg. deribit binance
        "settlement_index":"26000" // vendor settlement index price
      }
    ]
  }
}
```

| Parameter Name         | Type   | Description                            | Example       |
|:-----------------------|:-------|:---------------------------------------|:--------------|
| settle_time_mill       | int    | option settle time in millisecond      | 1692950400000 |
| infos                  | list   | settlement index info                  |               |
| infos.underlying_pair  | string | underlying pair                        | BTC-USDT      |
| infos.tracking_source  | string | tracking source eg. deribit binance... |               |
| infos.settlement_index | string | vendor settlement index.               | 26000         |


### 1.6.9. Check Settlement summary
vendor check settlement summary

- URL: /mp/api/v1/dcp/settlement/summary

- method: POST

| Parameter Name   | Type   | Required | Description                       | Example       |
|:-----------------|:-------|:---------|:----------------------------------|:--------------|
| settle_time_mill | int    | yes      | option settle time in millisecond | 1692950400000 |
| infos            | list   | yes      | settlement info                   |               |
| infos.currency   | string | yes      | settle currency                   |               |
| infos.amount     | string | yes      | settle amount                     |               |

Post Data Example:

```js
{
  "settle_time_mill": 1692950400000,
  "infos":[ 
    {
        "currency":"BTC", //string, settle currency
        "amount":"100", //string,dcp calc need settle amount currnecy need settle amount,if amount > 0 means vendor need to transfer to matrixport, if less than zero means matripxort need to transfer to vendor.
      }
  ]
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
    "infos":[ 
      {
        "currency":"BTC", //string, settle currency
        "amount":"100", //string, vendor calc need settle amount   
      }
    ]
  }
}
```

| Parameter Name   | Type   | Description                       | Example       |
|:-----------------|:-------|:----------------------------------|:--------------|
| settle_time_mill | int    | option settle time in millisecond | 1692950400000 |
| infos            | list   | settle info                       |               |
| infos.currency   | string | settle currency                   | BTC           |
| infos.amount     | string | currency need settle amount       | 100           |


### 1.6.10. Quarterly profit check
- post matrixport calc quarterly profit,if check failed response an error code.
- quarterly profit = (end_time_aum-start_time_aum) + last_quarter_profit - replenish_skitg
- URL: /mp/api/v1/dcp/profit/check

- method: Post

- Parameters: json in body

| Key                 | Type   | Required | Description         |
|---------------------|--------|----------|---------------------|
| start_time_mill     | int    | yes      | period start time   |
| end_time_mill       | int    | yes      | period end time     |
| start_time_aum      | string | yes      | start time aum      |
| end_time_aum        | string | yes      | end time aum        |
| last_quarter_profit | string | yes      | last quarter profit |
| replenish_skitg     | string | yes      | replenish skitg     |


- Response: application/json

Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
  }
}
```
