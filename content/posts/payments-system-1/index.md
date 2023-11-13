---
author: "Abhay"
title: Building a Modular and Extensible Payments System [1]
description: "In this part we'll focus on the logic and code of handling our payments"
date: "2023-11-11T00:00:00+00:00"
template: "post"
draft: true
slug: "payments-system-1"
series:
  - "payments-system"
tags:
  - "backend"
  - "nodejs"
  - "typescript"
  - "systems-design"
  - "low-level-design"
hidemeta: false
comments: false
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "posts/payments-system-0/thumbnail.png" # image path/url
    alt: "Thumbnail: Server with various payment gateways" # alt text
    caption: "A Server with various payment gateways" # display caption under cover
    relative: true # when using page bundles set this to true
    hidden: false # only hide on current single page
---

## Components
{{<img "images/payments-system-components-overview.png" "Components of payments system" >}}

The flow of payment goes like this.
1. Create payment url
    1. User selects the product they want and initiate payment.
    2. Concerned application code creates and order and requests a payment Url from PaymentGatewaysAbstractionLayer for given order.
    3. PGAL create a transaction to mark a payment attempt.
    4. PGAL passes this transaction to PaymentGateways wrapper to get a URL 
    5. URL is returned all the way back to user(front-end)
2. User visit the page and pays and payment Gateway lets backend know about the success/failure through Webhooks.
3. Payment processor marks the transaction marks the transaction as failed/completed.
4. Payment processor runs the application code to allot the bought product to the user.

### A. Application Code
To state the obvious this contains the business logic of how features work, what products do we sell, what all is given to a buyer etc. etc.

**Step 1:**
User select a product to buy and the application invokes payment collection logic.

### B. Payment Gateways Abstraction Layer
{{<img "images/b-payment-gateway-abstraction-layer.png" "Payment Gateways Abstraction Layer" >}}
*Pa.G.A.L.* :P, we don't call it that, but it's funny xD.

What this layer does is, it takes details about the payment request and gives us a url to payment page provided by the selected Gateway. Our team chose this approach because of significantly lower front-end effort.

We have multiple features that require payments system and adding PG selection logic to all of them is violation of DRY.
So this layer is abstracts away this responsibility. This decides which payment gateway to use depending on:
1. Location of customer.
2. Amount & Currency of transaction.
3. Health and availability of payment gateways.

Other rules can be added to optimise the fee we pay to PGs, let say by using rarely available but cheaper PG.

```typescript
export class PaymentGatewaysAbstractionLayer {
  constructor(
    stripe: Stripe, // stripe wrapper
    phonepe: Phonepe,
    juspay: Juspay,
    conversionRateSevice: ConversionRateSevice
  ) {
    ...
  }

  async createOrder(payload: {
    buyer: MongooseTypes.ObjectId;
    seller: MongooseTypes.ObjectId;
    productType: IProductType;
    productId: MongooseTypes.ObjectId;
    amount: number;
    currency: ICurrencyCode;
    data: any;
  }) {
    return this.orderModel.create(payload);
  }

  async createPaymentUrl(...) {
    ...
  }
}
```

`createOrder` is just a wrapper over order model, we needed to dissociate this from `createPaymentUrl` in order to provide feature to retry payment for already created order.

Let's take a deeper look into how we `createPaymentUrl`

First we gather required data
```typescript
async createPaymentUrl({
  orderId,
  ip,
  redirectUrl,
}: {
  orderId: Id,
  ip: string | null,
  redirectUrl: string,
}) {
  const country = getCountry(ip); // string | null
  const order = getOrder({id: orderId});
```

Select appropriate payment gateway
```typescript
  if(country === 'IN') {
    // use phonepe/juspay for UPI
    if (product.amount > 2500) {
      // use juspay
    } else {
      // use phonepe
    }
  }
  // use stripe for foreign customers
}
```

Now we'll take a slight detour into interface land.
#### Payment Gateway Interface
Payment gateways have their own ~~hoops and hurdles~~ way of doing things, this would cause a lot of headache for the API/SDK users if every payment gateway provide their own set of API. Sadly they do just that. So now our payment gateway abstraction layer has two do two things 
1. Gateway selection logic
2. Parameter customisation logic for using different gateways
This voids SRP of SOLID, i.e. single responsibility principle, the code complexity shoots through roof.

So we wrap the gateway specific logic in wrappers written by us that follow a contract, in typescript land these are called `Interfaces`. This ensures that all wrappers take same parameters and return same output. Making PaymentGatewaysAbstractionLayer's developers job less painful in this case it was me, so I made my life less painful :D.

**The Contract:**
```typescript
export interface IPaymentGatewayInterface {
  async createPaymentUrl({
    product: ProductDetails, // To be shown on payment screen
    transaction: Transaction,
    redirectUrl: string,
  }): Promise<{url: string}>;

  async refund({transaction: Transaction});
}
```

When a wrapper implements the contract the typescript compiler ensures these methods are implemented.

When the PaymentGatewaysAbstractionLayer developers use any wrapper they just call `.createPaymentUrl(...)` and don't have to care about the internals of PaymentGateways.

### Payment Completion Webhooks Listener
{{<img "images/c-webhooks-listener.png" "Payment Completion Webhooks Illustration" >}}
