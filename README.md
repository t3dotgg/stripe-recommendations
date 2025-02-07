# How I Stay Sane Implementing Stripe

> [!NOTE]  
> **Update (2025-02-07)**  
> Stripe invited me to speak with the CEO at their company-wide all hands meeting. They were super receptive to my feedback, and I see a bright future where none of this is necessary. Until then, I still think this is the best way to set up payments in your SaaS apps.

I have set up Stripe far too many times. I've never enjoyed it. I've talked to the Stripe team about the shortcomings and they say they'll fix them...eventually.

Until then, this is how I recommend setting up Stripe. I don't cover everything - check out [things that are still your problem](#things-that-are-still-your-problem) for clarity on what I'm NOT helping with.

> If you want to stay sane implementing file uploads, check out my product [UploadThing](https://uploadthing.com/).

### Pre-requirements

- TypeScript
- Some type of JS backend
- Working auth (that is verified on your JS backend)
- A KV store (I use Redis, usually [Upstash](https://upstash.com/?utm_source=theo), but any KV will work)

### General philosophy

IMO, the biggest issue with Stripe is the "split brain" it inherently introduces to your code base. When a customer checks out, the "state of the purchase" is in Stripe. You're then expected to track the purchase in your own database via webhooks.

There are [over 258 event types](https://docs.stripe.com/api/events/types). They all have different amounts of data. The order you get them is not guaranteed. None of them should be trusted. It's far too easy to have a payment be failed in stripe and "subscribed" in your app.

These partial updates and race conditions are obnoxious. I recommend avoiding them entirely. My solution is simple: _a single `syncStripeDataToKV(customerId: string)` function that syncs all of the data for a given Stripe customer to your KV_.

The following is how I (mostly) avoid getting Stripe into these awful split states.

## The Flow

This is a quick overview of the "flow" I recommend. More detail below. Even if you don't copy my specific implementation, you should read this. _I promise all of these steps are necessary. Skipping any of them will make life unnecessarily hard_

1. **FRONTEND:** "Subscribe" button should call a `"generate-stripe-checkout"` endpoint onClick
1. **USER:** Clicks "subscribe" button on your app
1. **BACKEND:** Create a Stripe customer
1. **BACKEND:** Store binding between Stripe's `customerId` and your app's `userId`
1. **BACKEND:** Create a "checkout session" for the user
   - With the return URL set to a dedicated `/success` route in your app
1. **USER:** Makes payment, subscribes, redirects back to `/success`
1. **FRONTEND:** On load, triggers a `syncAfterSuccess` function on backend (hit an API, server action, rsc on load, whatever)
1. **BACKEND:** Uses `userId` to get Stripe `customerId` from KV
1. **BACKEND:** Calls `syncStripeDataToKV` with `customerId`
1. **FRONTEND:** After sync succeeds, redirects user to wherever you want them to be :)
1. **BACKEND:** On [_all relevant events_](#events-i-track), calls `syncStripeDataToKV` with `customerId`

This might seem like a lot. That's because it is. But it's also the simplest Stripe setup I've ever seen work.

Let's go into the details on the important parts here.

### Checkout flow

The key is to make sure **you always have the customer defined BEFORE YOU START CHECKOUT**. The ephemerality of "customer" is a straight up design flaw and I have no idea why they built Stripe like this.

Here's an adapted example from how we're doing it in [T3 Chat](https://t3.chat).

```ts
export async function GET(req: Request) {
  const user = auth(req);

  // Get the stripeCustomerId from your KV store
  let stripeCustomerId = await kv.get(`stripe:user:${user.id}`);

  // Create a new Stripe customer if this user doesn't have one
  if (!stripeCustomerId) {
    const newCustomer = await stripe.customers.create({
      email: user.email,
      metadata: {
        userId: user.id, // DO NOT FORGET THIS
      },
    });

    // Store the relation between userId and stripeCustomerId in your KV
    await kv.set(`stripe:user:${user.id}`, newCustomer.id);
    stripeCustomerId = newCustomer.id;
  }

  // ALWAYS create a checkout with a stripeCustomerId. They should enforce this.
  const checkout = await stripe.checkout.sessions.create({
    customer: stripeCustomerId,
    success_url: "https://t3.chat/success",
    ...
  });
```

### syncStripeDataToKV

This is the function that syncs all of the data for a given Stripe customer to your KV. It will be used in both your `/success` endpoint and in your `/api/stripe` webhook handler.

The Stripe api returns a ton of data, much of which can not be serialized to JSON. I've selected the "most likely to be needed" chunk here for you to use, and there's a [type definition later in the file](#custom-stripe-subscription-type).

Your implementation will vary based on if you're doing subscriptions or one-time purchases. The example below is with subcriptions (again from [T3 Chat](https://t3.chat)).

```ts
// The contents of this function should probably be wrapped in a try/catch
export async function syncStripeDataToKV(customerId: string) {
  // Fetch latest subscription data from Stripe
  const subscriptions = await stripe.subscriptions.list({
    customer: customerId,
    limit: 1,
    status: "all",
    expand: ["data.default_payment_method"],
  });

  if (subscriptions.data.length === 0) {
    const subData = { status: "none" };
    await kv.set(`stripe:customer:${customerId}`, subData);
    return subData;
  }

  // If a user can have multiple subscriptions, that's your problem
  const subscription = subscriptions.data[0];

  // Store complete subscription state
  const subData = {
    subscriptionId: subscription.id,
    status: subscription.status,
    priceId: subscription.items.data[0].price.id,
    currentPeriodEnd: subscription.current_period_end,
    currentPeriodStart: subscription.current_period_start,
    cancelAtPeriodEnd: subscription.cancel_at_period_end,
    paymentMethod:
      subscription.default_payment_method &&
      typeof subscription.default_payment_method !== "string"
        ? {
            brand: subscription.default_payment_method.card?.brand ?? null,
            last4: subscription.default_payment_method.card?.last4 ?? null,
          }
        : null,
  };

  // Store the data in your KV
  await kv.set(`stripe:customer:${customerId}`, subData);
  return subData;
}
```

### `/success` endpoint

> [!NOTE]
> While this isn't 'necessary', there's a good chance your user will make it back to your site before the webhooks do. It's a nasty race condition to handle. Eagerly calling syncStripeDataToKV will prevent any weird states you might otherwise end up in

This is the page that the user is redirected to after they complete their checkout. For the sake of simplicity, I'm going to implement it as a `get` route that redirects them. In my apps, I do this with a server component and Suspense, but I'm not going to spend the time explaining all that here.

```ts
export async function GET(req: Request) {
  const user = auth(req);
  const stripeCustomerId = await kv.get(`stripe:user:${user.id}`);
  if (!stripeCustomerId) {
    return redirect("/");
  }

  await syncStripeDataToKV(stripeCustomerId);
  return redirect("/");
}
```

Notice how I'm not using any of the `CHECKOUT_SESSION_ID` stuff? That's because it sucks and it encourages you to implement 12 different ways to get the Stripe state. Ignore the siren calls. Have a SINGLE `syncStripeDataToKV` function. It will make your life easier.

### `/api/stripe` (The Webhook)

This is the part everyone hates the most. I'm just gonna dump the code and justify myself later.

```ts
export async function POST(req: Request) {
  const body = await req.text();
  const signature = (await headers()).get("Stripe-Signature");

  if (!signature) return NextResponse.json({}, { status: 400 });

  async function doEventProcessing() {
    if (typeof signature !== "string") {
      throw new Error("[STRIPE HOOK] Header isn't a string???");
    }

    const event = stripe.webhooks.constructEvent(
      body,
      signature,
      process.env.STRIPE_WEBHOOK_SECRET!
    );

    waitUntil(processEvent(event));
  }

  const { error } = await tryCatch(doEventProcessing());

  if (error) {
    console.error("[STRIPE HOOK] Error processing event", error);
  }

  return NextResponse.json({ received: true });
}
```

> [!NOTE]
> If you are using Next.js Pages Router, make sure you turn this on. Stripe expects the body to be "untouched" so it can verify the signature.
>
> ```ts
> export const config = {
>   api: {
>     bodyParser: false,
>   },
> };
> ```

### `processEvent`

This is the function called in the endpoint that actually takes the Stripe event and updates the KV.

```ts
async function processEvent(event: Stripe.Event) {
  // Skip processing if the event isn't one I'm tracking (list of all events below)
  if (!allowedEvents.includes(event.type)) return;

  // All the events I track have a customerId
  const { customer: customerId } = event?.data?.object as {
    customer: string; // Sadly TypeScript does not know this
  };

  // This helps make it typesafe and also lets me know if my assumption is wrong
  if (typeof customerId !== "string") {
    throw new Error(
      `[STRIPE HOOK][CANCER] ID isn't string.\nEvent type: ${event.type}`
    );
  }

  return await syncStripeDataToKV(customerId);
}
```

### Events I Track

If there are more I should be tracking for updates, please file a PR. If they don't affect subscription state, I do not care.

```ts
const allowedEvents: Stripe.Event.Type[] = [
  "checkout.session.completed",
  "customer.subscription.created",
  "customer.subscription.updated",
  "customer.subscription.deleted",
  "customer.subscription.paused",
  "customer.subscription.resumed",
  "customer.subscription.pending_update_applied",
  "customer.subscription.pending_update_expired",
  "customer.subscription.trial_will_end",
  "invoice.paid",
  "invoice.payment_failed",
  "invoice.payment_action_required",
  "invoice.upcoming",
  "invoice.marked_uncollectible",
  "invoice.payment_succeeded",
  "payment_intent.succeeded",
  "payment_intent.payment_failed",
  "payment_intent.canceled",
];
```

### Custom Stripe subscription type

```ts
export type STRIPE_SUB_CACHE =
  | {
      subscriptionId: string | null;
      status: Stripe.Subscription.Status;
      priceId: string | null;
      currentPeriodStart: number | null;
      currentPeriodEnd: number | null;
      cancelAtPeriodEnd: boolean;
      paymentMethod: {
        brand: string | null; // e.g., "visa", "mastercard"
        last4: string | null; // e.g., "4242"
      } | null;
    }
  | {
      status: "none";
    };
```

## More Pro Tips

Gonna slowly drop more things here as I remember them.

### DISABLE "CASH APP PAY".

I'm convinced this is literally just used by scammers. over 90% of my cancelled transactions are Cash App Pay.
![image](https://github.com/user-attachments/assets/c7271fa6-493c-4b1c-96cd-18904c2376ee)

### ENABLE "Limit customers to one subscription"

This is a really useful hidden setting that has saved me a lot of headaches and race conditions. Fun fact: this is the ONLY way to prevent someone from being able to check out twice if they open up two checkout sessions 🙃 More info [in Stripe's docs here](https://docs.stripe.com/payments/checkout/limit-subscriptions)

## Things that are still your problem

While I have solved a lot of stuff here, in particular the "subscription" flows, there are a few things that are still your problem. Those include...

- Managing `STRIPE_SECRET_KEY` and `STRIPE_PUBLISHABLE_KEY` env vars for both testing and production
- Managing `STRIPE_PRICE_ID`s for all subscription tiers for dev and prod (I can't believe this is still a thing)
- Exposing sub data from your KV to your user (a dumb endpoint is probably fine)
- Tracking "usage" (i.e. a user gets 100 messages per month)
- Managing "free trials"
  ...the list goes on

Regardless, I hope you found some value in this doc.
