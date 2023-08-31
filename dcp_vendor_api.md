# Dual-Coin Vendor Access Framework

## Vendor API

Vendor should offer the following APIs:

## Encoding

The request and response data for all interfaces is formatted in UTF-8. Content for responses in all interfaces is formatted in JSON.

## Common parameters in HTTP headers

Below common parameters in HTTP headers are recommend to added in request, facilitating debugging and tracing each request.

| Key          | Type   | Position | Required | Description                                                     |
| ------------ | ------ | -------- | -------- | --------------------------------------------------------------- |
| x-request-id | string | header   | false    | uuid for trace the request, not included in signing the request |

# Authentication

## Private API mandatory fields

- Client and Server use same private secret key to sign the request.
- Client must add `timestamp` (epoch in millisecond) field in request parameter (query string for GET, json body for POST), API server will check this timestamp, if `abs(server_timestamp - request_timestamp) > 5000`, the request will be rejected.
- `timestamp` must be integer, not quoted string.
- Client must add `signature` field in request parameter (query string for GET, json body for POST).
- For POST request, Header `Content-Type` should be set as `application/json`.

| Key       | Type   | Position      | Required | Description              |
| --------- | ------ | ------------- | -------- | ------------------------ |
| timestamp | int    | query or body | true     | epoch in millisecond     |
| signature | string | query or body | true     | signature of the request |

## Signature algorithm

```python
#########
# Python code to calc Matrixport API signature
#########
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

    #########
    # END
    #########

```

### Explanation:

1. Request parameters: JSON Body for POST, query string for GET
2. Encode string to sign, for simple json object, sort your parameter keys alphabetically, and join them with '&' like 'param1=value1&param2=value2', then get str_to_sign = api_path + '&' + 'param1=value1&param2=value2'
3. For nested array objects, encode each object and sort them alphabetically, join them with '&' and embraced with '[', ']', e.g.
   str_to_sign = api_path + '&' + 'param1=value1&array_key1=[array_item1&array_item2]', see example below.
4. Signature = hex(hmac_sha256(str_to_sign, secret_key))
5. Add `signature` field to request parameter:
   - for query string, add '&signature=YOUR_SIGNATURE'
   - for JSON body, add {"signature":YOUR_SIGNATURE}

<br>
<br>

### 1. Get Products

- Returns a list of products

- The timeout for this interface call is 1000ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/products

- method: Get

- Parameters: in query

| Key             | Type   | Required | Description                                                                                   | Example            |
| --------------- | ------ | -------- | --------------------------------------------------------------------------------------------- | ------------------ |
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
            "underlying_pair": "BTC-USDC" //string, underlying pair
            "type": "CALL" // optionstring, type
            "settle_time_mill": 1692950400000 //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
            "strike_price": "30000" //string, strike price in string format

            "deposit_currency": "BTC", //string, deposit currency
            "min_buy": "0.1" //string, minimal buy amount
            "max_buy": "100" //string, maximal buy amount
            "yield_rate": "0.01" //string, yield rate; not converted:  payback_amount = deposit_amount * (1 + yield_rate) ; converted:  payback_amount = deposit_amount *(/) target_price * (1 + yield_rate)
            "redeemable":true, //bool, whether this product can be early redeemed.
        },
    ]
  }
}
```

| Parameter Name         | Type   | Description                                                                                                                                                       |
| :--------------------- | :----- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| items                  | array  | products                                                                                                                                                          |
| items.underlying_pair  | string | underlying pair                                                                                                                                                   |
| items.type             | string | type                                                                                                                                                              |
| items.settle_time_mill | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.                                                                               |
| items.strike_price     | string | strike price in string format                                                                                                                                     |
| items.deposit_currency | string | deposit currency                                                                                                                                                  |
| items.min_buy          | string | minimal buy amount                                                                                                                                                |
| items.max_buy          | string | maximal buy amount, max_buy >= min_buy                                                                                                                            |
| items.yield_rate       | string | yield rate; not converted: payback*amount = deposit_amount * (1 + yield_rate) ; converted: payback_amount = deposit_amount \*(/) target_price \* (1 + yield_rate) |
| items.redeemable       | bool   | whether this product can be early redeemed.                                                                                                                       |

---

### 2. Get Quote

- Get quote for placing or redeeming an order by instrument info of option.

- The quote provided by the vendor should remain valid within 1 minute; otherwise, it may result in a higher user purchase failure rate.

- The timeout for this interface call is 1000ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/quote

- method: Post

- Parameters: json in body

| Key              | Type   | Required | Description                                                                         |
| ---------------- | ------ | -------- | ----------------------------------------------------------------------------------- |
| deposit_currency | string | yes      | deposit currency                                                                    |
| deposit_amount   | string | yes      | deposit_amount in string format                                                     |
| side             | string | yes      | BUY or SELL; SELL: place new order, BUY: redeem exist order                         |
| underlying_pair  | string | yes      | underlying pair                                                                     |
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
    "quote_id": "6633327782363410432", //string, quote id

    "underlying_pair": "BTC-USDC" //string, underlying pair
    "type": "CALL" //string, option type
    "settle_time_mill": 1692950400000 //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
    "strike_price": "30000" //string, strike price in string format

    "premium_amount": "0.1" //string, premium_amount in string format
    "price_expire_time_mill": 1691727892000 //int, price expire time in millisecond,
    "side": "BUY" //string, side, BUY or SELL

    "deposit_currency": "BTC", //string, currency
    "deposit_amount": "10", //string, amount in string format
  }
}
```

| Parameter Name         | Type   | Description                                                                         |
| :--------------------- | :----- | :---------------------------------------------------------------------------------- |
| quote_id               | string | quote id                                                                            |
| underlying_pair        | string | underlying pair                                                                     |
| type                   | string | option type                                                                         |
| settle_time_mill       | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |
| strike_price           | string | strike price in string format                                                       |
| premium_amount         | string | premium_amount in string format                                                     |
| price_expire_time_mill | int    | price expire time in millisecond                                                    |
| side                   | string | side, BUY or SELL                                                                   |
| deposit_currency       | string | currency                                                                            |
| deposit_amount         | string | amount in string format                                                             |

Error Code:

| Code     | Message                                                            |
| -------- | ------------------------------------------------------------------ |
| 0        | Success                                                            |
| 12000003 | Invalid parameters                                                 |
| 1001     | No price (will **NOT** retry)                                      |
| 1002     | Unknown error (will retry several time according to product logic) |

### 3. Place Order

- Place Order by quote id.

- The timeout for this interface call is 2000ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/order

- method: Post

- Parameters: json in body

| Key             | Type   | Required | Description                     |
| --------------- | ------ | -------- | ------------------------------- |
| quote_id        | string | yes      | quote id, unique                |
| client_order_id | string | yes      | client order id ,for idempotent |

- Response: application/json

Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "quote_id": "6633327782363410432", //string, quote id
    "order_id": "7080019906774802432", //string, order id
    "client_order_id": "client_order_id_7080019906774802432", //string, client_order_id

    "underlying_pair": "BTC-USDC" //string, underlying pair
    "type": "CALL" //string, option type
    "settle_time_mill": 1692950400000 //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
    "strike_price": "30000" //string, strike price in string format

    "filled_qty": "100" //string, filed qty
    "filled_avg_price": "0.01" //string, filled average price in string format
    "side": "BUY" //string, side, BUY or SELL

    "premium_amount": "0.1" //string, premium_amount in string format

    "deposit_currency": "BTC", //string, deposit currency
    "deposit_amount": "1", //string, deposit amount in string format

  }
}
```

| Parameter Name         | Type   | Description                                                                         |
| :--------------------- | :----- | :---------------------------------------------------------------------------------- |
| quote_id               | string | quote id                                                                            |
| underlying_pair        | string | underlying pair                                                                     |
| type                   | string | option type                                                                         |
| settle_time_mill       | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |
| strike_price           | string | strike price in string format                                                       |
| premium_amount         | string | premium_amount in string format                                                     |
| price_expire_time_mill | int    | price expire time in millisecond                                                    |
| side                   | string | side, BUY or SELL                                                                   |
| deposit_currency       | string | currency                                                                            |
| deposit_amount         | string | amount in string format                                                             |

Error Code:

| Code     | Message                                                       |
| -------- | ------------------------------------------------------------- |
| 0        | Success                                                       |
| 12000003 | Invalid parameters, such as quote id not valid                |
| 1101     | Quote price expired                                           |
| 1102     | Can not satisfy the requested yield rate (will **NOT** retry) |
| 1103     | Unknown error (will retry until price expire)                 |

### 4. Get Redeemable Info

- Returns the total amount that can be obtained for redeeming the specified order in advance. The currency is always the same as the investment currency, i.e., no currency exchange.

- The accepted time range for redemption is from 24 hours after the order is purchased to 24 hours before the order expires. The timeout for this interface call is 200ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/order/redeemable

- method: get

- Parameters: in query

| Key      | Type   | Required | Description |
| -------- | ------ | -------- | ----------- |
| order_id | string | yes      | order id    |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "order_id": "7080019906774802432", //string, order id
    "client_order_id": "client_order_id_7080019906774802432", //string, client_order_id

    "redeem_currency": "BTC", //string, redeemable currency
    "min_redeemable_amount": "0.1", //string, min redeemable amount in string format, eg. 0.1
    "max_redeemable_amount": "1", //string, max redeemable amount in string format
  }
}
```

| Parameter Name        | Type   | Description                            |
| :-------------------- | :----- | :------------------------------------- |
| order_id              | string | order id                               |
| client_order_id       | string | client order id                        |
| redeem_currency       | string | redeemable currency                    |
| max_redeemable_amount | string | max redeemable amount in string format |
| min_redeemable_amount | string | min redeemable amount in string format |

### 5. Redeem Order

- Redeem order by redeem quote id.

- The timeout for this interface call is 2000ms. Peak QPS: 50.

- URL: /mp/api/v1/dcp/order/redeem

- method: Post

- Parameters: json in body

| Key              | Type   | Required | Description                      |
| ---------------- | ------ | -------- | -------------------------------- |
| order_id         | string | yes      | order id                         |
| client_redeem_id | string | yes      | client redeem id, for idempotent |
| quote_id         | string | yes      | quote id                         |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": {
    "order_id": "7080019906774802432", //string, order id
    "redeem_id": "redeem_7080019906774802432", //string, redeem id
    "client_redeem_id": "client_redeem_7080019906774802432", //string, client_redeem_id

    "redeem_currency": "BTC", //string, redeemable currency
    "redeem_amount": "1", //string, redeem amount in string format

    "redeem_option":{ //object, option traded for redeem
        "underlying_pair": "BTC-USDC" //string, underlying pair
        "type": "CALL" //string, option type
        "settle_time_mill": 1692950400000 //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
        "strike_price": "30000" //string, strike price in string format

        "premium_amount": "-0.1" //string, premium_amount in string format

        "filled_qty": "100" //string, filed qty
        "filled_avg_price": "0.01" //string, filled average price in string format
        "side": "BUY" //string, side, BUY or SELL

    }

  }
}
```

| Parameter Name                 | Type   | Description                                                                         |
| :----------------------------- | :----- | :---------------------------------------------------------------------------------- |
| order_id                       | string | order id                                                                            |
| redeem_id                      | string | redeem id                                                                           |
| client_redeem_id               | string | client redeem id, for idempotent                                                    |
| redeem_currency                | string | redeemable currency                                                                 |
| redeem_amount                  | string | redeem amount in string format                                                      |
| redeem_settle_amount           | string | redeem settle amount in string format                                               |
| redeem_option                  | object | option traded for redeem                                                            |
| redeem_option.underlying_pair  | string | underlying pair                                                                     |
| redeem_option.type             | string | option type                                                                         |
| redeem_option.settle_time_mill | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |
| redeem_option.strike_price     | string | strike price in string format                                                       |
| redeem_option.filled_qty       | string | filed qty                                                                           |
| redeem_option.filled_avg_price | string | filled average price in string format                                               |
| redeem_option.side             | string | BUY or SELL                                                                         |
| redeem_option.premium_amount   | string | premium_amount in string format                                                     |

Error Code:

| Code     | Message                                                  |
| -------- | -------------------------------------------------------- |
| 0        | Success                                                  |
| 12000003 | Invalid parameters, such as order id, quote id not valid |
| 1201     | Redeem Amount error                                      |
| 1202     | Quote expired                                            |
| 1203     | Balance not enough (will retry in several hours)         |
| 1204     | Unknown error (will **NOT** retry)                       |

### 6. Query Order by Order ID or Client Order ID

Get single order info by order_id or client_order_id

- URL: /mp/api/v1/dcp/order

- method: Get

- Parameters: in query

| Key             | Type   | Required | Description     |
| --------------- | ------ | -------- | --------------- |
| order_id        | string | no       | order id        |
| client_order_id | string | no       | client order id |

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": { //object, order
    "quote_id": "6633327782363410432", //string, quote id
    "order_id": "7080019906774802432", //string, order id
    "client_order_id": "client_order_id_7080019906774802432", //string, client_order_id

    "order_status": 1, //int, 0 : Processing, 100 : Fund freezing, 110 : Vendor order processing, 120 : Vendor order failed, 130 : Canceled, 200 : Closing, 210 : Pending settlement, 220 : Settlement, 230 : settled, 240 : full redemption

    "underlying_pair": "BTC-USDC" //string, underlying pair
    "type": "CALL" //string, option type
    "settle_time_mill": 1692950400000 //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
    "strike_price": "30000" //string, strike price in string format

    "filled_qty": "100" //string, filed qty
    "filled_avg_price": "0.01" //string, filled average price in string format
    "filled_time_mill":  1692950400000//int,
    "side": "BUY" //string, side, BUY or SELL

    "deposit_currency": "BTC", //string, deposit currency
    "deposit_amount": "1", //string, deposit amount in string format

    "true_settled_time_mill": 1692950400000 //int, true option settled time
    "true_settled_price": "32000" //int, true option settled price in string format
    "true_settled_currency": "BTC" //int, settled currency
    "true_settled_amount": "1" //int, settled amount

    "premium_amount": "1" //int, premium amount

    "redeemable":true, //bool,
    "redeem_records":[
        {
        "order_id": "7080019906774802432", //string, order id
        "redeem_id": "redeem_7080019906774802432", //string, redeem id

        "redeem_currency": "BTC", //string, redeemable currency
        "redeem_amount": "1", //string, redeem amount in string format
        "redeem_settle_amount": "1", //string, redeem settle amount in string format

        "redeem_option":{ //object, option traded for redeem
            "underlying_pair": "BTC-USDC" //string, underlying pair
            "type": "CALL" //string, option type
            "settle_time_mill": 1692950400000 //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
            "strike_price": "30000" //string, strike price in string format

            "premium_amount": "-1" //int, premium amount

            "filled_qty": "100" //string, filed qty
            "filled_avg_price": "0.01" //string, filled average price in string format
            "filled_time_mill":  1692950400000//int,
            "side": "BUY" //string, side, BUY or SELL
        }
        }
    ],

  }
}
```

| Parameter Name                                | Type   | Description                                                                                                                                                                                                    |
| :-------------------------------------------- | :----- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| quote_id                                      | string | quote id                                                                                                                                                                                                       |
| order_id                                      | string | order id                                                                                                                                                                                                       |
| client_order_id                               | string | client_order_id                                                                                                                                                                                                |
| order_status                                  | int    | 0 : Processing, 100 : Fund freezing, 110 : Vendor order processing, 120 : Vendor order failed, 130 : Canceled, 200 : Closing, 210 : Pending settlement, 220 : Settlement, 230 : settled, 240 : full redemption |
| underlying_pair                               | string | underlying pair                                                                                                                                                                                                |
| type                                          | string | option type                                                                                                                                                                                                    |
| settle_time_mill                              | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.                                                                                                                            |
| strike_price                                  | string | strike price in string format                                                                                                                                                                                  |
| filled_qty                                    | string | filed qty                                                                                                                                                                                                      |
| filled_avg_price                              | string | filled average price in string format                                                                                                                                                                          |
| side                                          | string | side, BUY or SELL                                                                                                                                                                                              |
| deposit_currency                              | string | deposit currency                                                                                                                                                                                               |
| deposit_amount                                | string | deposit amount in string format                                                                                                                                                                                |
| true_settled_time_mill                        | int    | true option settled time                                                                                                                                                                                       |
| true_settled_price                            | int    | true option settled price in string format                                                                                                                                                                     |
| true_settled_currency                         | int    | settled currency                                                                                                                                                                                               |
| true_settled_amount                           | int    | settled amount                                                                                                                                                                                                 |
| premium_amount                                | int    | premium amount                                                                                                                                                                                                 |
| redeem_records.order_id                       | string | order id                                                                                                                                                                                                       |
| redeem_records.redeem_id                      | string | redeem id                                                                                                                                                                                                      |
| redeem_records.redeem_currency                | string | redeemable currency                                                                                                                                                                                            |
| redeem_records.redeem_amount                  | string | redeem amount in string format                                                                                                                                                                                 |
| redeem_records.redeem_settle_amount           | string | redeem settle amount in string format                                                                                                                                                                          |
| redeem_records.redeem_option                  | object | option traded for redeem                                                                                                                                                                                       |
| redeem_records.redeem_option.underlying_pair  | string | underlying pair                                                                                                                                                                                                |
| redeem_records.redeem_option.type             | string | option type                                                                                                                                                                                                    |
| redeem_records.redeem_option.settle_time_mill | int    | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.                                                                                                                            |
| redeem_records.redeem_option.strike_price     | string | strike price in string format                                                                                                                                                                                  |
| redeem_records.redeem_option.premium_amount   | int    | premium amount                                                                                                                                                                                                 |
| redeem_records.redeem_option.filled_qty       | string | filed qty                                                                                                                                                                                                      |
| redeem_records.redeem_option.filled_avg_price | string | filled average price in string format                                                                                                                                                                          |
| redeem_records.redeem_option.side             | string | side, BUY or SELL                                                                                                                                                                                              |

### 7. Query Orders

Get order list by various conditions.

- URL: /mp/api/v1/dcp/orders

- method: Get

| Parameter Name         | Type   | Required | Description                                         | Example            |
| :--------------------- | :----- | :------- | :-------------------------------------------------- | :----------------- |
| underlying_pair        | string | no       | Option's underlying currency pair, separate by "-"; | BTC-USDT, BTC-USDC |
| type                   | string | no       | option's type , in upper case;                      | CALL, PUT          |
| strike_price           | string | no       | strike price in string format                       |
| settle_time_mill_start | string | no       | start option settle time in millisecond             |
| settle_time_mill_end   | string | no       | end option settle time in millisecond               |
| deposit_currency       | string | no       | deposit currency                                    |
| limit                  | int    | no       | limit for pagination , default is 50                |
| offset                 | int    | no       | offset for pagination                               |

> Above request parameters fields take effects when not empty or 0

- Response: application/json
  Example:

```js
{
  "code": 0,
  "message": "",
  "data": { //array, order
    "count":100, //int, total count
    "items":[ //objects, order list, order info is same as in get order api, except for redeem record list.
      {
        "quote_id": "6633327782363410432", //string, quote id
        "order_id": "7080019906774802432", //string, order id
        "client_order_id": "client_order_id_7080019906774802432", //string, client_order_id

        "order_status": 1, //int, 0 : Processing, 100 : Fund freezing, 110 : Vendor order processing, 120 : Vendor order failed, 130 : Canceled, 200 : Closing, 210 : Pending settlement, 220 : Settlement, 230 : settled, 240 : full redemption

        "underlying_pair": "BTC-USDC" //string, underlying pair
        "type": "CALL" //string, option type
        "settle_time_mill": 1692950400000 //int, option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000.
        "strike_price": "30000" //string, strike price in string format

        "filled_qty": "100" //string, filed qty
        "filled_avg_price": "0.01" //string, filled average price in string format
        "filled_time_mill":  1692950400000//int,
        "side": "BUY" //string, side, BUY or SELL

        "deposit_currency": "BTC", //string, deposit currency
        "deposit_amount": "1", //string, deposit amount in string format

        "true_settled_time_mill": 1692950400000 //int, true option settled time
        "true_settled_price": "32000" //int, true option settled price in string format
        "true_settled_currency": "BTC" //int, settled currency
        "true_settled_amount": "1" //int, settled amount

        "premium_amount": "1" //int, premium amount

        "redeemable":true, //bool,
      }
    ],
  }
}
```

| Parameter Name | Type    | Description                                                                        |
| :------------- | :------ | :--------------------------------------------------------------------------------- |
| count          | int     | total count                                                                        |
| items          | objects | order list, order info is same as in get order api, except for redeem record list. |

### 8. settle price check 
vendor check settle price 
- post matrixport settle price,vendor check and retrun check result,if check failed response an error code.

- URL: /mp/api/v1/dcp/settlement/fixing_list

- method: Post

- Parameters: json in body

| Key                    | Type   | Required | Description                                                                         |
| ---------------------- | ------ | -------- | ----------------------------------------------------------------------------------- |
| settle_time_mill       | int    | yes      | option settle time in millisecond                                                   |
| infos                  | array  | yes      | infos
| infos.underlying_pair  | string | yes      | underlying pair                                                                     |
| infos.tracking_source  | string | yes      | traceing source eg deribit binance...                                               |
| infos.settlement_index | string | yes      | option settle time in millisecond, eg. 2023-08-25T16:00:00 +08:00 is 1692950400000. |

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
        "underlying_pair":"BTC-USDT", // nderlying pair
        "settlement_index":"26000", // vendor settlement index price
      }
    ],
  }
}
```

| Parameter Name         | Type   |  Description                                         | Example            |
| :--------------------- | :----- |  :-------------------------------------------------- | :----------------- |
| settle_time_mill       | int    |  option settle time in millisecond                   | 1692950400000      |
| infos                  | list   |  settlement index info                               |                    |
| infos.underlying_pair  | string |  underlying pair                                     | BTC-USDT           |
| infos.settlement_index | string |  vendor settlement index.                            | 26000              |


### 9. Check Settlement summary
vendor check settlement summary 

- URL: /mp/api/v1/dcp/settlement/summary

- method: POST

| Parameter Name         | Type   | Required | Description                                         | Example            |
| :--------------------- | :----- | :------- | :-------------------------------------------------- | :----------------- |
| settle_time_mill       | int    | yes      | option settle time in millisecond                   | 1692950400000      |
| infos                  | list   | yes      | settlement info                                     |                    |
| infos.currency         | string | yes      | settle currency                                     |                    |
| infos.amount           | string | yes      | settle amount                                       |                    |


```js
{
  "settle_time_mill": 1692950400000,
  "infos":[ 
    {
        "currency":"BTC", //string, settle currency
        "amount":"100", //string, currnecy need settle amount   
      }
  ],
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
    ],
  }
}
```

| Parameter Name         | Type   |  Description                                         | Example            |
| :--------------------- | :----- |  :-------------------------------------------------- | :----------------- |
| settle_time_mill       | int    |  option settle time in millisecond                   | 1692950400000      |
| infos                  | list   |  settle info                                         |                    |
| infos.currency         | string |  settle currency                                     | BTC                |
| infos.amount           | string |  currnecy need settle amount                         | 100                |


### 10. quarterly profit check 
- post matrixport calc quarterly profit,if check failed response an error code.
- quarterly profit = (end_time_aum-start_time_aum) + last_quarter_profit - replenish_skitg
- URL: /mp/api/v1/dcp/profit/check

- method: Post

- Parameters: json in body

| Key                    | Type   | Required | Description                   |
| ---------------------- | ------ | -------- | ----------------------------- |
| start_time_mill        | int    | yes      | period start time             |
| end_time_mill          | int    | yes      | period end time               |
| start_time_aum         | string | yes      | start time aum                |
| end_time_aum           | string | yes      | end time aum                  |
| last_quarter_profit    | string | yes      | last quarter profit           |
| replenish_skitg        | string | yes      | replenish skitg               |


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

