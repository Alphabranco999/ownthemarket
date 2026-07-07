# Connecting Your Website to PesaPal (MTN & Airtel Money)

Your website (`index.html`) currently **simulates** the payment so you can see and demo the full flow. To take real UGX 20,000 payments from MTN and Airtel, you need to connect it to PesaPal, which is the payment gateway that actually talks to MTN Mobile Money and Airtel Money in Uganda. Here is the full path, in order.

## Why you need a backend, not just a website

A plain HTML/CSS/JS site (like `index.html`) can only be hosted as static files — it cannot safely hold your PesaPal secret key. If you put your secret key in JavaScript, anyone who views your page's source code can steal it and take payments in your name. That's why `server-example.js` exists: it's a small server that sits between your website and PesaPal, and it's the only place your secret key ever lives.

You don't need to be a programmer to get this running — you (or anyone you hire on Upwork/locally) just needs to follow the steps below in order.

## Step 1: Create your PesaPal account
1. Go to **pesapal.com** and sign up as a merchant in Uganda.
2. Provide your business details (or personal ID/NIN if you're selling as an individual) and bank details — this is where your money will settle.
3. Wait for PesaPal to verify your account. This can take a day or two.

## Step 2: Get your API credentials (start in Sandbox)
1. Go to **developer.pesapal.com** and log in with your merchant account.
2. Create an app — this gives you a **Consumer Key** and **Consumer Secret**.
3. Start in **Sandbox mode** first (test money, no real charges) before going live. Sandbox base URL: `https://cybqa.pesapal.com/pesapalv3`.

## Step 3: Put your site on a server that can run code
Static hosting (like GitHub Pages) won't work for the payment part, because it can't run `server-example.js`. Cheap, simple options:
- **Render.com** or **Railway.app** — free/cheap tiers, deploy `server-example.js` directly.
- A basic **VPS** (e.g. DigitalOcean) if you want full control.

Once deployed, you'll have a real URL, e.g. `https://ownthebuyer.com`.

## Step 4: Register your IPN URL (one-time step)
PesaPal needs a URL on your server to notify when a customer finishes paying.
1. Set `IPN_URL` in your `.env` file to `https://yourdomain.com/api/pesapal/ipn`.
2. Visit `https://yourdomain.com/api/pesapal/register-ipn` once in your browser.
3. It returns an `ipn_id` — copy that into your `.env` as `PESAPAL_IPN_ID`. You only do this once.

## Step 5: Fill in your `.env` file
Create a file called `.env` next to `server-example.js`:
```
PESAPAL_ENV=sandbox
PESAPAL_CONSUMER_KEY=your_key_here
PESAPAL_CONSUMER_SECRET=your_secret_here
CALLBACK_URL=https://yourdomain.com/payment-complete
IPN_URL=https://yourdomain.com/api/pesapal/ipn
PESAPAL_IPN_ID=the_id_from_step_4
```

## Step 6: Connect the "Pay Now" button
In `index.html`, the `startPayment()` function currently fakes the confirmation with a timer. Replace that fake part with a real call to your server:
```js
const res = await fetch('/api/pesapal/initiate', {
  method: 'POST',
  headers: {'Content-Type':'application/json'},
  body: JSON.stringify({ phone, network })
});
const { redirect_url } = await res.json();
window.location.href = redirect_url;
```
This sends the buyer to a PesaPal-hosted page, which is what actually pushes the MTN/Airtel USSD PIN prompt to their phone. You don't have to build that prompt screen yourself — PesaPal handles it.

## Step 7: Only unlock the download after real payment
When PesaPal confirms payment through your `/api/pesapal/ipn` endpoint, mark that order as paid in your own records (even a simple spreadsheet-style database works to start). Then either:
- Redirect the buyer to `/download/:orderId`, which checks they actually paid before sending the PDF (already sketched out in `server-example.js`), or
- Email them the PDF automatically once payment is confirmed.

Never make the download link guessable or public before payment is confirmed — that's the part that actually makes this a paywall.

## Step 8: Test with real phones in Sandbox
PesaPal's sandbox lets you simulate MTN/Airtel payments with test numbers before risking real money. Confirm the whole flow works end-to-end: pay → IPN received → download unlocked.

## Step 9: Go live
1. Ask PesaPal to move your account to **Live** status (they may ask a few more verification questions).
2. Get your **live** Consumer Key/Secret from developer.pesapal.com.
3. Update `.env`: set `PESAPAL_ENV=live`, swap in the live key/secret, and re-register your IPN against the live base URL (`https://pay.pesapal.com/v3`).
4. Do one small real test payment yourself before announcing the site publicly.

## Quick summary of the money flow
```
Buyer clicks "Pay" on your site
        ↓
Your server asks PesaPal for a payment link (SubmitOrderRequest)
        ↓
Buyer is sent to PesaPal's page → USSD prompt hits their phone
        ↓
Buyer enters their MTN/Airtel PIN
        ↓
PesaPal notifies your server (IPN) that payment is COMPLETED
        ↓
Your server marks the order paid and unlocks the real PDF download
```

If you get stuck on any single step, PesaPal's own support team (via the merchant dashboard) is quite responsive — the account verification step (Step 1) is usually the slowest part, so start that first even before touching any code.
