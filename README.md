# Azan Wholesale – Reseller API Documentation

Comprehensive REST API reference and integration guide for Azan Wholesale resellers.

---

## Table of Contents

- [Overview](#overview)
- [Environments & Base URLs](#environments--base-urls)
- [Authentication & Request Headers](#authentication--request-headers)
- [Standard HTTP Response Codes](#standard-http-response-codes)
- [API Endpoints](#api-endpoints)
  - [1. Get Products API](#1-get-products-api)
  - [2. Order Product Store API](#2-order-product-store-api)
  - [3. Date-Wise New Products API](#3-date-wise-new-products-api)
- [Reseller Webhook Endpoints (Callbacks)](#reseller-webhook-endpoints-callbacks)
  - [4. Update Stock Callback](#4-update-stock-callback)
  - [5. Update Product Callback](#5-update-product-callback)
- [Integration Best Practices](#integration-best-practices)

---

## Overview

The Azan Wholesale API allows registered reseller platforms to:
- Browse and sync product catalogs with tiered wholesale pricing.
- Place dropship and bulk orders directly into the Azan Wholesale fulfillment system.
- Synchronize real-time inventory and pricing updates through automated webhooks.

---

## Environments & Base URLs

| Environment | Base URL | Usage |
| :--- | :--- | :--- |
| **Live (Production)** | `https://api.azanwholesale.com/` | Real transactions and live store fulfillment |
| **Staging (Sandbox)** | `https://betaapi.azanwholesale.com/` | Testing, staging integration, and sandbox testing |

---

## Authentication & Request Headers

All API requests must include your store credentials in custom HTTP headers. Obtain your `App-ID` and `Secret-Key` from the Azan Wholesale Reseller Admin portal.

| Header | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `App-ID` | string | **Required** | Your unique reseller application / store ID. |
| `Secret-Key` | string | **Required** | Your confidential reseller secret API key. |
| `Accept` | string | **Required** | Must be `application/json`. |
| `Content-Type` | string | **Required** (POST) | Must be `application/json` for requests with a payload body. |

> [!WARNING]
> Keep your `Secret-Key` strictly confidential. Never expose it in client-side code (browsers or mobile applications).

---

## Standard HTTP Response Codes

| Status Code | Meaning | Description |
| :--- | :--- | :--- |
| `200 OK` | Success | The request succeeded (typically for `GET` requests). |
| `201 Created` | Resource Created | The order was successfully registered in the system. |
| `400 Bad Request` | Validation Error | Request business rule failed (e.g., sale price below wholesale price). |
| `401 Unauthorized` | Auth Failed | Invalid or missing `App-ID` / `Secret-Key`. |
| `404 Not Found` | Not Found | Reseller credentials or requested resource not found. |
| `409 Conflict` | Duplicate Order | An order with the provided `platform_order_id` already exists. |
| `422 Unprocessable` | Validation Failed | Required payload parameters are missing, invalid, or products are out of stock. |
| `500 Server Error` | Server Exception | Internal server error. Please contact Azan Wholesale technical support. |

---

## API Endpoints

### 1. Get Products API

Fetch products with wholesale pricing, real-time inventory, categories, brands, and pagination.

- **HTTP Method:** `GET`
- **Route:** `/api/en/products/by-api`
- **Live URL:** `https://api.azanwholesale.com/api/en/products/by-api`
- **Staging URL:** `https://betaapi.azanwholesale.com/api/en/products/by-api`

#### Headers

| Header | Type | Requirement | Value / Example |
| :--- | :--- | :--- | :--- |
| `App-ID` | string | **Required** | `your_app_id` |
| `Secret-Key` | string | **Required** | `your_secret_key` |
| `Accept` | string | **Required** | `application/json` |

#### Query Parameters

| Parameter | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `per_page` | integer | Optional | `25` | Number of items per page (`min: 1`, `max: 500`). |
| `selected_product` | integer | Optional | `0` | `0`: Returns all wholesale products.<br>`1`: Returns only products selected in your reseller portal. |
| `category_ids` | array | Optional | `null` | Filter by category IDs (e.g., `category_ids[]=1&category_ids[]=2`). |
| `brand_ids` | array | Optional | `null` | Filter by brand IDs (e.g., `brand_ids[]=5`). |
| `product_search_name` | string | Optional | `null` | Search query matching product title/name. |
| `min_price` | numeric | Optional | `null` | Filter products with wholesale price >= `min_price`. |
| `max_price` | numeric | Optional | `null` | Filter products with wholesale price <= `max_price`. |
| `sort_by` | string | Optional | `stock_desc` | `price_high`, `price_low`, `asc`, `desc`, `stock_desc`, `stock_asc`, `name_asc`, `name_desc`. |

#### cURL Example

```bash
curl -X GET "https://api.azanwholesale.com/api/en/products/by-api?per_page=50&selected_product=0"   -H "App-ID: YOUR_APP_ID"   -H "Secret-Key: YOUR_SECRET_KEY"   -H "Accept: application/json"
```

#### Sample Response (`200 OK`)

```json
{
  "success": true,
  "data": [
    {
      "id": 3119,
      "name": "Cetaphil Gentle Skin Cleanser Dry to Normal, Sensitive Skin 125ml",
      "slug": "cetaphil-gentle-skin-cleanser-125ml",
      "sku": "8809416471112",
      "wholesale_price": 750,
      "mrp_price": 1700,
      "stock": 960,
      "manufacture_country": "Canada",
      "product_brands": "Cetaphil",
      "product_categories": "Skin Care, Cleanser",
      "short_description": "Gentle daily cleanser for sensitive skin.",
      "description": "<p>Full product description...</p>",
      "skin_types": "Dry, Normal, Sensitive",
      "pictures": [
        "https://api.azanwholesale.com/storage/products/cetaphil-1.jpg"
      ]
    }
  ],
  "meta": {
    "current_page": 1,
    "last_page": 20,
    "per_page": 50,
    "total": 1000
  }
}
```

---

### 2. Order Product Store API

Submit a customer order to Azan Wholesale for direct fulfillment.

- **HTTP Method:** `POST`
- **Route:** `/api/orders/store`
- **Live URL:** `https://api.azanwholesale.com/api/orders/store`
- **Staging URL:** `https://betaapi.azanwholesale.com/api/orders/store`

#### Headers

| Header | Type | Requirement | Value / Example |
| :--- | :--- | :--- | :--- |
| `App-ID` | string | **Required** | `your_app_id` |
| `Secret-Key` | string | **Required** | `your_secret_key` |
| `Content-Type` | string | **Required** | `application/json` |
| `Accept` | string | **Required** | `application/json` |

#### Business & Validation Rules

1. **Unique Order ID:** `platform_order_id` must be unique per reseller. Submitting an existing ID returns `409 Conflict`.
2. **Product Identification:** Each item in `order_details` must contain either `supplier_product_id` (Azan Product ID) or `sku` to identify the product.
3. **Price Validation:**
   - Reseller selling price (`unit_price`) **must not be less than** the wholesale price (`wholesale_price`).
   - Reseller selling price (`unit_price`) **must not exceed** the Maximum Retail Price (`mrp_price`).
4. **Stock Validation:** The order will be rejected with `422 Unprocessable` if requested quantity exceeds available stock.

#### Request Schema

##### Top-Level Fields

| Field | Type | Requirement | Validation Rule | Description |
| :--- | :--- | :--- | :--- | :--- |
| `platform_order_id` | integer | **Required** | `required\|integer` | Unique ID from your platform. |
| `shipping_address` | object | **Required** | `required\|array` | Customer shipping details object. |
| `order_details` | array | **Required** | `required\|array\|min:1` | List of items ordered (min 1 item). |
| `date` | string | Optional | `nullable\|date` | Order timestamp (`YYYY-MM-DD HH:mm:ss`). |
| `note` | string | Optional | `nullable\|string\|max:1000` | Order note or special delivery notes. |
| `shipping_cost` | integer | Optional | `nullable\|integer\|min:0` | Shipping charge. |
| `discount` | integer | Optional | `nullable\|integer\|min:0` | Total order discount. |
| `delivary_method` | integer | Optional | `nullable\|integer` | `0`: Picked up/received by reseller.<br>`1`: Direct delivery to customer. |
| `reseller_package_type` | string/int | Optional | `nullable` | Packaging type identifier. |
| `platform_source` | string | Optional | `nullable\|string` | Store / platform name. |
| `platform_user_id` | int/string | Optional | `nullable` | Customer ID in your system. |
| `order_source` | string | Optional | `nullable\|string` | Channel source (e.g., `web`, `app`). |
| `payment_type` | string | Optional | `nullable\|string` | Payment method (e.g., `cash_on_delivery`). |
| `payment_status` | string | Optional | `nullable\|string` | Payment status (e.g., `COD`, `PAID`). |
| `delivery_status` | string | Optional | `nullable\|string` | Delivery status (e.g., `Completed`). |
| `subtotal` | numeric | Optional | `nullable\|numeric` | Order subtotal. |
| `grand_total` | numeric | Optional | `nullable\|numeric` | Total payable amount. |
| `reward_point_used` | numeric | Optional | `nullable\|numeric` | Loyalty points used. |
| `user` | object | Optional | `nullable` | Object containing `id`, `name`, `email`. |

##### `shipping_address` Object Fields

| Field | Type | Requirement | Validation Rule | Description |
| :--- | :--- | :--- | :--- | :--- |
| `name` | string | **Required** | `required\|string\|max:255` | Recipient full name. |
| `address` | string | **Required** | `required\|string\|max:500` | Full street delivery address. |
| `phone` | string | Optional | `nullable\|string\|max:20` | Recipient phone number. |
| `email` | string | Optional | `nullable\|email\|max:255` | Recipient email address. |
| `area` | string | Optional | `nullable\|string\|max:255` | Area / Thana / Neighborhood. |
| `city` | string | Optional | `nullable\|string\|max:255` | City name. |
| `state` | string | Optional | `nullable\|string\|max:255` | State / Division. |
| `area_id` | integer | Optional | `nullable\|integer` | Courier Area ID. |
| `city_id` | integer | Optional | `nullable\|integer` | Courier City ID. |
| `zone_id` | integer | Optional | `nullable\|integer` | Courier Zone ID. |

##### `order_details.*` Item Fields

| Field | Type | Requirement | Validation Rule | Description |
| :--- | :--- | :--- | :--- | :--- |
| `quantity` | integer | **Required** | `required\|integer\|min:1` | Quantity ordered. |
| `unit_price` | numeric | **Required** | `required\|numeric\|min:0` | Reseller selling price per unit. |
| `mrp_price` | numeric | **Required** | `required\|numeric\|min:0` | Maximum Retail Price (MRP). |
| `supplier_product_id` | string/int | Optional* | `nullable\|regex:/^\d+-?\d+$/` | Azan product ID (*required if `sku` is omitted). |
| `sku` | string | Optional* | `nullable\|regex:/^\d+-?\d+$/` | Azan product SKU (*required if `supplier_product_id` is omitted). |
| `supplier` | string | Optional | `nullable\|string\|max:255` | Supplier identifier (`AzanWholeSale`). |
| `product_id` | integer | Optional | `nullable\|integer` | Reseller internal product ID. |
| `wholesale_price` | numeric | Optional | `nullable\|numeric` | Wholesale price. |
| `total_price` | numeric | Optional | `nullable\|numeric` | Item total (`unit_price * quantity`). |
| `discount` | numeric | Optional | `nullable\|numeric` | Discount applied to item. |
| `reward_point_used` | numeric | Optional | `nullable\|numeric` | Points used for this item. |

#### cURL Example

```bash
curl -X POST "https://api.azanwholesale.com/api/orders/store"   -H "App-ID: YOUR_APP_ID"   -H "Secret-Key: YOUR_SECRET_KEY"   -H "Content-Type: application/json"   -H "Accept: application/json"   -d '{
    "platform_order_id": 382969,
    "date": "2026-08-27 14:22:47",
    "note": "Fragile items. Please deliver between 2 PM - 5 PM.",
    "delivary_method": 1,
    "shipping_cost": 60,
    "discount": 0,
    "platform_source": "MyStore",
    "platform_user_id": 247094,
    "order_source": "web",
    "payment_type": "cash_on_delivery",
    "payment_status": "COD",
    "delivery_status": "Processing",
    "subtotal": 760,
    "grand_total": 820,
    "user": {
      "id": 247094,
      "name": "Niloy Roy",
      "email": null
    },
    "shipping_address": {
      "name": "Niloy Roy",
      "phone": "01700000000",
      "email": null,
      "address": "Dhaka Mohakhali Dohs, Dhaka-1206",
      "area": "Mohakhali DOHS",
      "city": "Dhaka",
      "state": "Dhaka",
      "area_id": null,
      "city_id": 3,
      "zone_id": 2642
    },
    "order_details": [
      {
        "supplier_product_id": 238,
        "sku": "8809416471112",
        "supplier": "AzanWholeSale",
        "product_id": 1097,
        "quantity": 1,
        "wholesale_price": 475,
        "unit_price": 760,
        "mrp_price": 1000,
        "total_price": 760,
        "discount": 0,
        "reward_point_used": 0
      }
    ],
    "reward_point_used": 0
  }'
```

#### Sample Responses

##### Success (`201 Created`)
```json
{
  "message": "Order created successfully",
  "order_id": 15820
}
```

##### Duplicate Order (`409 Conflict`)
```json
{
  "message": "Order already exists",
  "order_id": 382969
}
```

##### Out of Stock or Item Failure (`422 Unprocessable Entity`)
```json
{
  "message": "All products failed. ",
  "0": "Insufficient stock",
  "failed_products": [
    {
      "reason": "Insufficient stock",
      "data": {
        "supplier_product_id": 238,
        "quantity": 100
      }
    }
  ]
}
```

##### Price Constraint Violation (`400 Bad Request`)
```json
{
  "status": false,
  "message": "Your sale price can not be less than wholesale price."
}
```

---

### 3. Date-Wise New Products API

Retrieve products that have been added or updated from a specific date up to the present.

- **HTTP Method:** `GET`
- **Route:** `/api/products/date/{date}`
- **Live URL:** `https://api.azanwholesale.com/api/products/date/06-12-2025`
- **Staging URL:** `https://betaapi.azanwholesale.com/api/products/date/06-12-2025`

#### Headers

| Header | Type | Requirement | Example |
| :--- | :--- | :--- | :--- |
| `App-ID` | string | **Required** | `your_app_id` |
| `Secret-Key` | string | **Required** | `your_secret_key` |
| `Accept` | string | **Required** | `application/json` |

#### Path Parameters

| Parameter | Type | Requirement | Validation Rule | Description |
| :--- | :--- | :--- | :--- | :--- |
| `date` | string | **Required** | `date_format:d-m-Y` | Format: `DD-MM-YYYY` (e.g., `06-12-2025`). |

#### cURL Example

```bash
curl -X GET "https://api.azanwholesale.com/api/products/date/06-12-2025"   -H "App-ID: YOUR_APP_ID"   -H "Secret-Key: YOUR_SECRET_KEY"   -H "Accept: application/json"
```

#### Sample Response (`200 OK`)

```json
[
  {
    "id": 1042,
    "name": "Cosrx Low pH Good Morning Gel Cleanser 150ml",
    "slug": "cosrx-low-ph-good-morning-gel-cleanser-150ml",
    "sku": "8809416470001",
    "wholesale_price": 620,
    "mrp_price": 1250,
    "description": "<p>Gentle gel cleanser...</p>",
    "short_description": "Low pH cleanser suitable for daily use.",
    "brand": "COSRX",
    "categories": [
      "Skin Care",
      "Cleanser"
    ],
    "Stock": 450,
    "meta_title": "Cosrx Low pH Good Morning Gel Cleanser",
    "meta_description": "Gentle daily cleanser with low pH.",
    "image": "https://api.azanwholesale.com/storage/products/cosrx-cleanser.jpg"
  }
]
```

---

## Reseller Webhook Endpoints (Callbacks)

When inventory or product data changes on Azan Wholesale, our system sends automated HTTP `POST` requests to your registered webhook URLs.

> [!NOTE]
> Ensure these endpoints are accessible on your server and return a `200 OK` JSON response.

### 4. Update Stock Callback

Called whenever product inventory changes at Azan Wholesale.

- **HTTP Method:** `POST`
- **Reseller Endpoint:** `{your_base_url}/api/update-stock`

#### Incoming Headers

| Header | Value | Description |
| :--- | :--- | :--- |
| `App-ID` | `{your_app_id}` | Your assigned store API Key. |
| `Secret-Key` | `{your_secret_key}` | Your assigned store Secret Key. |
| `Content-Type` | `application/json` | Payload content format. |
| `Accept` | `application/json` | Expected response format. |

#### Webhook Payload Schema

| Field | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `sku` | string | **Required** | Product SKU identifier. |
| `supplier` | string | **Required** | Supplier identifier (`AzanWholeSale`). |
| `stock` | string/int | **Required** | Current available quantity. |
| `status` | string | **Required** | `AVAILABLE` or `OUT_OF_STOCK`. |

#### Sample Incoming Payload

```json
{
  "sku": "8809416471112",
  "supplier": "AzanWholeSale",
  "stock": "960",
  "status": "AVAILABLE"
}
```

#### Expected Reseller Response (`200 OK`)

```json
{
  "status": true,
  "message": "Stock updated successfully"
}
```

---

### 5. Update Product Callback

Called whenever product information or pricing is updated at Azan Wholesale.

- **HTTP Method:** `POST`
- **Reseller Endpoint:** `{your_base_url}/api/update-product`

#### Incoming Headers

| Header | Value | Description |
| :--- | :--- | :--- |
| `App-ID` | `{your_app_id}` | Your assigned store API Key. |
| `Secret-Key` | `{your_secret_key}` | Your assigned store Secret Key. |
| `Content-Type` | `application/json` | Payload content format. |
| `Accept` | `application/json` | Expected response format. |

#### Sample Incoming Payload

```json
{
  "sku": "8809416471112",
  "supplier": "AzanWholeSale",
  "stock": "960",
  "status": "AVAILABLE"
}
```

#### Expected Reseller Response (`200 OK`)

```json
{
  "status": true,
  "message": "Product updated successfully"
}
```

---

## Integration Best Practices

1. **Verify Webhook Headers:** Always validate incoming requests on `/api/update-stock` and `/api/update-product` against your `App-ID` and `Secret-Key` to avoid spoofed requests.
2. **Idempotent Order Submission:** Use a unique `platform_order_id` in your system before submitting to `/api/orders/store`. In case of a network timeout, re-submitting the same payload will safely return `409 Conflict` without double-ordering.
3. **Scheduled Catalog Synchronization:**
   - Fetch newly added or updated items daily using `/api/products/date/{DD-MM-YYYY}`.
   - Run a full catalog reconciliation weekly using `/api/en/products/by-api`.
4. **Price Synchronization:** Maintain your product prices such that selling price >= wholesale price and <= MRP to avoid rejected order store attempts.
