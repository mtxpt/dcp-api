<!-- TOC -->

- [Vendor Risk](#vendor-risk)
- [SKITG](#skitg)

<!-- /TOC -->

### Vendor Risk

- Returns risk control parameters for vendor.

- URL: /dcp/api/v1/vendor_risk

- method: Get

- Parameters: in query

| Key       | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| vendor_id | int  | yes      | Vendor ID   |

- Response: application/json

Example:

```shell
curl 'https://mapi.matrixport.com/dcp/api/v1/vendor_risk?vendor_id=55'
```

```js
{
  "code": 0,
  "message": "",
  "data": {
    "margin_ratio": "0.35",
    "total_aum": "114535.7725",
    "maintenance_margin": "113635.7725",
    "mm_ratio": "0",
    "delta_cap": {
      "BTC": "0.12",
      "ETH": "0.15"
    }
  }
}
```

| Parameter Name     | Type   | Description                              |
| :----------------- | ------ | :--------------------------------------- |
| margin_ratio       | string | Margin ratio                             |
| total_aum          | string | Total assets under management (AUM)      |
| maintenance_margin | string | Maintenance margin                       |
| mm_ratio           | string | Maintenance margin ratio                 |
| delta_cap          | object | Delta for each currency (e.g., BTC, ETH) |

### SKITG

- Top up margin for vendor.

- URL: /dcp/api/v1/skitg

- method: Post

- Parameters: json in body

| Key       | Type   | Required | Description                 |
| --------- | ------ | -------- | --------------------------- |
| vendor_id | int    | yes      | Vendor ID                   |
| currency  | string | yes      | Currency symbol             |
| amount    | string | yes      | Amount in string format     |

- Response: application/json

Example:

```shell
curl --location 'https://mapi.matrixport.com/dcp/api/v1/skitg' \
--header 'USER-ID: 2917719' \
--header 'Content-Type: application/json' \
--data '{
    "vendor_id": 55,
    "currency": "ETH",
    "amount": "1.2"
}'
```

```js
{
  "code": 0,
  "data": null,
  "message": ""
}
```
