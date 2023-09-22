---
author: "Abhay"
title: Building a Modular and Extensible Payments Service [0]
description: "Learn how to structure your payments service with modularity in mind, allowing for easy integration and scalability as your application grows. Learn the art of building an extensible foundation that can adapt to changing business needs, ensuring robustness and agility, to, at the end of the day, maximize profit."
date: "2023-09-17T00:00:00+00:00"
template: "post"
draft: false
slug: "payments-service-0"
series:
  - "payments-service"
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
    image: "images/tumbnail.webp" # image path/url
    alt: "Thumbnail: Server with various payment gateways" # alt text
    caption: "Server with various payment gateways" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/rathod-sahaab/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

## Foreword
[Skip to requirements](#what-do-we-want)

This is a story of technical debt and its inevitable payoff. It's not a cautionary tale and let's not take away that technical debt is bad, technical debt is much like financial debt and most of our modern economy is built on debt. Debt is not bad as long as you can pay it off, tell that to the governments around the world.

## Let's start from the middle
In my day job at [Tealfeed](https://tealfeed.com), we added a revenue-generating feature, with **stripe** as the payment gateway. Everything worked well, and we lived happily ever after. Right? Wrong! First, we added a few more features and then a few more payment gateways all in good faith and name of business needs, and then, the code was, to put it lightly, BLOATED. Mathematically speaking the code scaled with the order of `A × B` *(cross product)* `A` being the number of features and `B` being the number of payment gateways.

A few hair-pulling problems arose due to this, concoction of features and payment gateways.
1. Bugs fixed once in payment logic still occurred somewhere overlooked.
2. Payment logic polluted the business logic allowing for blunder-full programming experience.
3. Code for resolution on which payment to use had to be done in multiple places. So business agility went out the window.
4. Each of our feature's database models contained information regarding payment voiding the much beloved SRP(Single Responsibility Principle) from SOLID.
    1. Not all features had all the payment info as adding the payment info code was a manual task prone to human error.
    2. Made the db models bloated, and fetch code where payment status had to be checked either in query or in code.
    3. Our analytics capabilities were limited due to payment related data being distributed.
5. Ghost purchases: as the payment info was stored in feature models we had to create a ghost purchase just to store payment data which were marked successful in some way if the payment was successful. If payment was not successful the purchases still lingered. All our queries had to account for that.

```typescript
class Call {
  ...callData,
  status: 'pending' | 'success' | 'failure';
  amount: number, // TODO: string 
  currency: ICurrencyCode,
  clientIp: string,
  country: string | null,
  paymentGateway: 'stripe' | 'phonepe';
  paymentGatewayTransactionId: string;
}

class Package {
  ...packageData,
  status: 'pending' | 'success' | 'failure';
  amount: number, // TODO: string 
  currency: ICurrencyCode,
  paymentGateway: 'stripe' | 'phonepe';
  paymentGatewayTransactionId: string;
}

class Webinar {
  ...webinarData,
  status: 'pending' | 'success' | 'failure';
  amount: number, // TODO: string 
  currency: ICurrencyCode,
  paymentGateway: 'stripe' | 'phonepe';
  paymentGatewayTransactionId: string;
}
```

And we don't want that.

## What do we want?
Our payments service/module
1. Should centralize all of the payment gateway selection logic.
2. Be open to the addition of new payment gateways with minimal effort.
3. Should not depend on the features and be agnostic of them in a functional sense.
4. Should not depend on payment gateway specific features *(vendor workaround code is not fun)*.
5. Should centralize all the payments-related data
6. No ghost purchases.

## Data
Any good back-end feature design starts with the data, so let's build up our data model. There are two major parts to our payments data, Orders and Transactions.
1. Order: is like a shopping cart, it remembers what you are trying to buy.
2. Transactions: these store the data about attempts of payments for a particular order.

An Order can have multiple Transactions, in case the customer retries failed payments, but, in order(pun intended) to state the obvious only one successful Transaction.

### Order
First things first, we need to know 
1. who is buying.
2. what is being bought?
2. how much is to be paid?
3. and from whom.
'Whom' can be technically optional as the product contains data about the seller but it makes analytics a lot easier.
```typescript
class Order {
  // who
  buyer: User;

  // what
  productType: ProductTableName; // 'call_type' | 'package' | 'webinar'
  productId: Id;

  // how much
  amount: number; // TODO: string
  currency: ICurrencyCode; // 'INR' | 'USD',

  // whom
  seller: User;
}
```

Then we need to know whether this purchase was successful, failed, other little goody statuses, and which was the successful transaction if any.
```typescript
class Order {
  ...prev,
  status: 'created' | 'paid' | 'canceled' | 'refunded' | 'failed';
  successfulTransaction?: TransactionId;
}
```

<!-- whether this order was paid out to the creator. -->
<!-- ```typescript -->
<!-- class Order { -->
<!--   ...prev, -->
<!--   paidOut: boolean; -->
<!-- } -->
<!-- ``` -->
<!---->
Extra data: some of the products like calls also require specific data like start & end times. Which is not shared in other products. This was earlier stored in either ghost purchases or stripe meta-data. But as the requirements were 'no ghost purchases' and no payment gateway-specific features we need to store that data too. Hence order has a meta-data field to store.
```typescript
class Order {
  ...prev,
  data: any;
}
```

This data is generally called upon when the payment gateway indicates a successful payment.

### Transaction
These store the data about attempts of payments for a particular order.

#### Parent Order and Status
We need to know which order this transaction belongs to, and what's the status of the transaction.

```typescript
class Transaction {
  order: Order;
  status: 'created' | 'paid' | 'failed';
}
```

#### Amount
We also need to store the actual amount paid for the transaction by the user. This is more for the case of international transactions, where the amount of the order would be in dollars but the actual amount received by us will be INR(Rupee) as paid by the customer.

This might also depend on the payment gateway, as Stripe will charge users in dollars and Phonepe might charge users in rupees.

```typescript
class Transaction {
  ...prev,
  amount: number; // TODO: string
  currency: ICurrencyCode; // 'INR' | 'USD',

  conversionRate: // just for analytics
}
```
**This will be crucial for refunds** as conversion rates may change and the actual amount refunded might be less or greater than the amount spent by the user if we go by the amount stored in order only, which maps 1-to-1 with the cost of services.

#### Payment Gateways
As we have already made quite a big deal about payment gateways it's obvious we have to store those, to track where the payment came from. This makes sense at the transaction level because the same order can be tried to be paid by different PGs depending on payment method (cards/UPI etc.), and failures.

To reiterate each payment attempt will have a separate transaction.

We store the PG through which the payment came and also the Transaction ID so we can look it up on PG's console.

```typescript
class Transaction {
  ...prev,
  paymentGateway: 'stripe' | 'phonepe' | 'razorpay';
  paymentGatewayTransactionId: string;
}
```

#### Refunds
We also need to track refunds somehow, and due to my bias towards not mutating the data I can #functional-programming. Each refund also creates a new transaction because as you can imagine refunds can fail too (due variety of factors). Also, there might be future requests for a partial refund feature.
All in all, there was a significant overlap between normal transactions and refunds. So we just add another field to determine whether it's a `payment` transaction or a `refund` transaction.


```typescript
class Transaction {
  ...prev,
  type: 'payment' | 'refund';
}
```

So that was all the data regarding receiving payments. To 

### Compiled

Mostly order is used for our application flows, and transactions are just to store data for invoicing, analytics and debugging purposes.

```typescript
class Order {
  // who
  buyer: User;

  // what
  productType: ProductTableName; // 'call_type' | 'package' | 'webinar'
  productId: Id;

  // how much
  amount: number; // TODO: string
  currency: ICurrencyCode; // 'INR' | 'USD',

  // whom
  seller: User;

  // status
  status: 'created' | 'paid' | 'canceled' | 'refunded' | 'failed';
  successfulTransaction?: TransactionId;

  // meta-data
  data: any;
}

class Transaction {
  order: Order;
  status: 'created' | 'paid' | 'failed';

  amount: number; // TODO: string
  currency: ICurrencyCode; // 'INR' | 'USD',

  conversionRate: // just for analytics

  // payment gateway related information
  paymentGateway: 'stripe' | 'phonepe' | 'razorpay';
  paymentGatewayTransactionId: string;

  // refund or payment
  type: 'payment' | 'refund';
}
```

