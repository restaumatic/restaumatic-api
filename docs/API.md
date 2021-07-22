# 🛰️ Restaumatic Webhook API
This API is provided to enable other services to receive real-time information about new orders placed through Restaumatic system. The intended use cases include, but are not limited to integration with point of sale systems and delivery tracking.

# Development infrastructure

Please contact us to get access to development infrastracture (test restaurant with your development endpoint).

# Enabling integration

Integration has to be configured and enabled for given restaurant by Restaumatic support. In the future, we may add this option to restaurant’s panel to allow self-service.

API may be used in two modes: push, where configured webhook is called on order status change, or poll, when client periodically checks for incoming order. For push mode we need webhook URL in order to enable integration. Upon enabling integration one should receive an API Key that is required to perform API calls as well as authenticate webhook calls. This should be provided as a query parameter `apiKey` when calling Restaumatic endpoints as indicated in the documentation.

# Receiving orders

In order for Restaumatic to send webhook payloads, your server needs to be accessible from the Internet. We also highly suggest using SSL so that we can send encrypted payloads over HTTPS. The order data will be sent to configured URLs as a POST request with `application/json` payload. API key will be provided as a `apiKey` query parameter. Restaumatic expects `200 OK` response. In case of failure, Restaumatic will retry calling a webhook.

The webhook will be triggered when:

- new order is placed and it needs to be confirmed,
- an order was accepted,
- an order was cancelled.

Order may be accepted or cancelled with other channel and the webhook client should take this into account. Also, some orders (e.g. entered manually by restaurant staff) trigger only accepted event (they are already in `Completed` state), so integration should not rely on receiving orders in `WaitingForConfirmation` state in order to get all orders.

**Note:** Restaumatic automatically cancels orders that are not confirmed within a specified period (approximately 15 minutes, is subject to change). In such case, the customer is notified about this and is refunded (if paid online). Likely, an external system should reflect this state change.

## Data format

**General notes**

- All timestamps in UTC (ISO8601 format).
- Fields marked as nullable may be not present in JSON objects.
- Not nullable fields are in **bold**.
- We may add more fields in the future. We will do our best to maintain backward compatibility.
- `productId` is unique for each specific variant (size) of a menu item.
- In case of online payment we call webhooks after payment was successfully processed.

**Half and half pizzas**

Half and half pizzas are encoded as ordinary items. However, they use fixed UUID for `productId` and `variant.id`. The latter may be used to obtain information on pizza size. The information about parts, i.e. base pizza name, product and variant ids and topping adjustments are encoded in `extra.parts` attribute of an item in similar fashion to regular pizzas. Half and half pizzas may be also disabled by the restaurant in the admin panel.

Split pizza product UUID: `bb696623-ac72-5850-81bf-759b54e23b27`

Variant id by Restaumatic size in Restaumatic system:
1. `3909adee-2fd3-5af8-bd5f-03e33ed48b7d`
1. `7bbf5ddc-a0b4-5d8e-9733-83640c3a394f`
1. `e9d0b342-c380-52dc-ba53-8ada2c20f1fb`
1. `517ccb0b-c4a9-5bcc-8e06-89861bb54ff7`
1. `84383e8f-d602-55a6-842a-6feecd5b287d`

**Split items for generic products**

Split items are encoded as ordinary items. However, they use fixed UUID for `productId` and have no `variant.id`. The information about parts (variant, addons) are encoded in `extra.parts` attribute of an item in similar fashion to regular products. Ability to split selected may be enabled by the restaurant in the admin panel.

Split product (new menu) UUID: `beb19f5e-6c14-4a75-b1fd-356ebf0a5bec`

**Combos**

Combos are represented as separate products.

**Order (webhook payload)**

| **Field**                | **Type**            |                                                                                      |
| ------------------------ | ------------------- | ------------------------------------------------------------------------------------ |
| **id**                   | UUID                | UUID of an order                                                                     |
| **orderedAt**            | DateTime            | When the order was successfully placed & payment was complete                        |
| **timezone**             | TZLabel             | Time zone of an restaurant, e.g. `Europe/Warsaw`                                     |
| **state**                | OrderState          | Current state of an order, enum: `WaitingForConfirmation, Completed, Cancelled`    |
| **origin**               | OrderOrigin         | Origin of an order, `Online, Phone, Bar`, etc; new origins will be added without notice    |
| **paymentMethod**        | PaymentMethod       | Enum: `Cash, Online, Card`                                                         |
| confirmation             | Confirmation or Null | Null when order state is `WaitingForConfirmation`                                    |
| **customer**             | Customer            | Information about customer                                                           |
| **fulfillmentMethod**    | FulfillmentMethod   |                                                                                      |
| requestedFulfillmentTime | DateTime or Null     | Null for deliver ASAP orders, otherwise requested time for "deliver at XX:YY" orders |
| **total**                | Number              | Order total incl. discounts and delivery                                             |
| **currency**             | String              | ISO 4217, e.g. `PLN`, `EUR`, etc.                                                    |
| **items**                | Array of Item       |                                                                                      |
| **discounts**            | Array of Discount   | Discounts that were applied to whole order (not to specific items)                   |
| userNote                 | String or Null       | Note that user provided during checkout                                              |
| **callback**             | String              | Callback URL to order confirmation endpoint.                                         |
| vatId                    | String or Null      | Vat ID if user has requested invoice                                                 |

**Confirmation**

| **Field**       | **Type**        |                                          |
| --------------- | --------------- | ---------------------------------------- |
| **confirmedAt** | DateTime        | When the order was confirmed             |
| **status**      | Status          | Enum: `Accepted`, `Rejected`             |
| deliveryTime    | DateTime or Null | Expected delivery time (when applicable) |
| message         | String or Null   | Optional message for the customer.       |


**Customer**

There may be more variants in the future, API client should check `tag` attribute value.

| **Field** | **Type**     |                           |
| --------- | ------------ | ------------------------- |
| **tag**   | CustomerType | Constant `OnlineCustomer` |
| **name**  | String       |                           |
| **email** | String       |                           |
| **phone** | String       | E164 formatted            |


**Fulfillment method**

There are two variants that should be distinguished by `tag` attribute (possible values: `Takeaway`, `Delivery`).

| **Field** | **Type**              |                     |
| --------- | --------------------- | ------------------- |
| **tag**   | FulfillmentMethodType | Constant `Takeaway` |

| **Field**           | **Type**              |                     |
| ------------------- | --------------------- | ------------------- |
| **tag**             | FulfillmentMethodType | Constant `Delivery` |
| **deliveryCost**    | Number                |                     |
| **deliveryAddress** | DeliveryAddress       |                     |


**Delivery address**

| **Field**        | **Type**      |                   |
| ---------------- | ------------- | ----------------- |
| **street**       | String        |                   |
| **streetNumber** | String        |                   |
| apartmentNumber  | String or Null |                   |
| floor            | String or Null |                   |
| postCode         | String or Null |                   |
| **city**         | String        |                   |
| **country**      | String        | ISO3166-1 Alpha 2 |
| coordinates      | Coordinates or Null |                   |

**Coordinates**

| **Field** | **Type** |           |
| --------- | -------- | --------- |
| **lat**   | Number   | Latitude  |
| **lon**   | Number   | Longitude |


**Item**

| **Field**     | **Type**          |                                                                                                                                        |
| ------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **productId** | UUID              | UUID of a menu item                                                                         |
| **name**      | String            | Primary product name                                                                                                                   |
| variant       | Variant or Null    | Variant (e.g. size) of an item                                                                                                         |
| description   | String or Null     | Textual product description (size, sides, modification of ingredients), formatting is subject to change without notice                                                                 |
| extra         | ItemExtra or Null  | More fine-grained and structured info                                                                                                  |
| **quantity**  | Int               |                                                                                                                                        |
| **subtotal**  | Number            | Unit price times quantity. No discounts applied. This price includes all extra customizations (e.g. added toppings, side dishes, etc.) |
| **total**     | Number            | Subtotal minus discounts to this item.                                                                                                 |
| **discounts** | Array of Discount | Discounts that were applied to this item                                                                                               |


**Variant**

| **Field** | **Type** |                           |
| --------- | -------- | ------------------------- |
| **id**    | UUID     | UUID of a variant         |
| **name**  | String   | Display name of a variant |



**ItemExtra**
All fields are optional

| **Field**      | **Type**                  |                                                                                      |
| -------------- | ------------------------- | ------------------------------------------------------------------------------------ |
| size           | String or Null             | Product size name (when applicable)                                                  |
| includedAddons | Array of AddonInfo or Null | Addons that are included by default (e.g. toppings for non-custom pizza)             |
| addedAddons    | Array of AddonInfo or Null | Addons that were added by the user (e.g. sides or extra pizza toppings)              |
| removedAddons  | Array of AddonInfo or Null | Addons that were removed by the user (e.g. pizza toppings that customer didn't like) |
| parts          | Array of PartExtra or Null | Parts of an item, applicable to half-and-half pizzas |

**PartExtra (only half and half pizzas)**

| **Field**      | **Type**                  |                                                                                      |
| -------------- | ------------------------- | ------------------------------------------------------------------------------------ |
| **productId** | UUID              | UUID of a menu item                                                                         |
| **name**      | String            | Primary product name                                                                         |
| variant       | Variant or Null    | Variant (e.g. size) of an item                                                                  |
| includedAddons | Array of AddonInfo or Null | Addons that are included by default (e.g. toppings for non-custom pizza)             |
| addedAddons    | Array of AddonInfo or Null | Addons that were added by the user (e.g. sides or extra pizza toppings)              |
| removedAddons  | Array of AddonInfo or Null | Addons that were removed by the user (e.g. pizza toppings that customer didn't like) |


**AddonInfo**

| **Field**     | **Type**  |                                                                                                                                        |
| ------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **productId** | UUID      |                                                                                                                                        |
| **name**      | String    | Addon name                                                                                                                             |
| **addonType** | AddonType | Addon type. Addons may have different semantics:

- `Side` – side dish (deprecated)
- `Topping` – pizza ingredient (deprecated)
- `PizzaPan` – type of pizza pan (deprecated)
- `Modifier` – modifier item


**Discount**

| **Field**   | **Type**      |                                 |
| ----------- | ------------- | ------------------------------- |
| **name**    | String        | Primary discount/promotion name |
| description | String or Null | Additional information          |
| **value**   | Number        | Discount value                  |



# Accepting & rejecting orders

The confirmation data shall be sent to callback URL provided in webhook data as a POST request with `application/json` payload. Currently we use the following endpoints:

Testing: `https://www.manca.ro/api/v1/integrations/orders/ORDER_ID?apiKey=API_KEY`

Production: `https://www.skubacz.pl/api/v1/integrations/orders/ORDER_ID?apiKey=API_KEY`

Order id has to be provided both in URL as well as in request body.

**Accept the order**

    {
      "orderId": "0241c2a3-ee54-4c43-bad0-89a2e446e2b4",
      "status": "Accepted",
      "deliveryTime": "2018-10-03T23:16:04.022526Z"
    }

**Reject the order**

    {
      "orderId": "0241c2a3-ee54-4c43-bad0-89a2e446e2b4",
      "status": "Rejected",
      "comment": "We've run out of pizza dough."
    }


# Debugging webhooks

We provide special endpoint for webhook debugging. It will present some diagnostic information about recent webhook calls. Note, as this is debugging API we may change and improve this from time to time without notice.

Testing: `https://www.manca.ro/api/v1/integrations/log?apiKey=API_KEY`

Production: `https://www.skubacz.pl/api/v1/integrations/log?apiKey=API_KEY`


# Retrieving recent orders (polling)

If using webhooks is not suitable for your use case, we make polling endpoint available. It will return recent orders, including confirmed or rejected ones. Remember, orders may be cancelled either automatically by Restaumatic or by the user using other methods.

Testing: `https://www.manca.ro/api/v1/integrations/orders?apiKey=API_KEY`

Production: `https://www.skubacz.pl/api/v1/integrations/orders?apiKey=API_KEY`

# Getting product list

In some cases one may require to map products from Restaumatic to products in the external system. To make this possible we provide an extra endpoint to retrieve full list of products including available variants. Then the mapping procedure has to be implemented in the external system.

| **Field**     | **Type**                |                                                                                            |
| ------------- | ----------------------- | ------------------------------------------------------------------------------------------ |
| **productId** | UUID                    | Product id in Restaumatic                                                                  |
| **kind**      | ProductKind             | Indicates type of product, Enum: Product, Modifier, Dish, Drink, Pizza, Freebie, Side, Topping, PizzaPan |
| group         | String or Null           | Optional group of products (e.g. dish group, product category, modifier name), for informational purpose |
| **name**      | String                   | Product name                                                                               |
| variants      | Array of Variant or Null | List of possible variants (see Variant type from webhook). Applicable to some products.   |

Testing: `https://www.manca.ro/api/v1/integrations/menu?apiKey=API_KEY`

Production: `https://www.skubacz.pl/api/v1/integrations/menu?apiKey=API_KEY`


# Example webhook payload
See [example.json](example.json).

# Changelog

## July 2021

* Support for generic product representation.

## January 2021

* Added `vatId` field to the order.

## September 2019

#### Support for half-and-half pizzas

Information about pizza parts is now encoded in new `parts` attribute of ItemExtra.
