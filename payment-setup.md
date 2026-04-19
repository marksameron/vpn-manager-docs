# Payment Setup

VPN Management Studio supports 3 payment providers for your client portal. Your end-users can pay for VPN subscriptions using cryptocurrency or bank cards.

---

## Supported Providers

| Provider | Payment Types | Currencies | Setup Time |
|----------|--------------|------------|------------|
| **CryptoPay** | Crypto via Telegram @CryptoBot | USDT, BTC, TON, ETH | 5 min |
| **PayPal** | Bank cards (Visa, Mastercard), PayPal balance | USD, EUR, GBP | 10 min |
| **NOWPayments** | 100+ cryptocurrencies | BTC, ETH, USDT, USDC, LTC, XMR, TON, SOL... | 10 min |

You can enable one, two, or all three. If multiple providers are configured, your users will see a provider selection screen when paying.

---

## CryptoPay (Telegram)

Payments via Telegram's built-in @CryptoBot. Best for Telegram-centric VPN services.

### Setup

1. Open [@CryptoBot](https://t.me/CryptoBot) in Telegram
2. Go to **Crypto Pay** → **My Apps** → **Create App**
3. Copy the **API Token**
4. Add to your `.env` file:

```env
CRYPTOPAY_API_TOKEN=your_token_here
CRYPTOPAY_TESTNET=false
```

5. Restart services: `vpnmanager services restart`

### How it works

- User selects a plan in the client portal or Telegram client bot
- CryptoBot generates a payment link
- User pays in USDT, BTC, TON, or ETH
- Payment is confirmed automatically
- Subscription activates instantly

---

## PayPal (Bank Cards)

Accept Visa, Mastercard, and PayPal payments. Best for international customers who prefer traditional payments.

### Setup

1. Go to [developer.paypal.com](https://developer.paypal.com/dashboard/applications)
2. Log in with your PayPal Business account (or create one)
3. Click **Apps & Credentials** → **Create App**
4. Name: `VPN Management Studio` (or your brand name)
5. Copy **Client ID** and **Client Secret**
6. Add to your `.env` file:

```env
PAYPAL_CLIENT_ID=your_client_id
PAYPAL_CLIENT_SECRET=your_client_secret
PAYPAL_SANDBOX=false
```

7. Restart services: `vpnmanager services restart`

### Webhook (optional but recommended)

For automatic payment confirmation:

1. In PayPal Developer Dashboard → **Webhooks** → **Add Webhook**
2. URL: `https://YOUR_DOMAIN:10443/client-portal/webhook/paypal`
3. Events: select **CHECKOUT.ORDER.APPROVED** and **PAYMENT.CAPTURE.COMPLETED**
4. Copy the **Webhook ID** and add to `.env`:

```env
PAYPAL_WEBHOOK_ID=your_webhook_id
```

### Alternative: configure via Admin Panel

Instead of editing `.env`, go to **Admin Panel → Settings → Payment** and enter your PayPal credentials there. Changes apply immediately without restart.

### Sandbox testing

Set `PAYPAL_SANDBOX=true` to test with PayPal sandbox accounts before going live. Create test accounts at [sandbox.paypal.com](https://www.sandbox.paypal.com).

---

## NOWPayments (Cryptocurrency)

Accept 100+ cryptocurrencies with automatic conversion. Best for privacy-focused VPN services.

### Setup

1. Register at [nowpayments.io](https://nowpayments.io)
2. Go to **Settings → API Keys** → generate a new key
3. Go to **Settings → IPN** → copy the **IPN Secret Key**
4. Set the **IPN Callback URL**: `https://YOUR_DOMAIN:10443/client-portal/webhook/nowpayments`
5. Add to your `.env` file:

```env
NOWPAYMENTS_API_KEY=your_api_key
NOWPAYMENTS_IPN_SECRET=your_ipn_secret
NOWPAYMENTS_SANDBOX=false
```

6. Restart services: `vpnmanager services restart`

### Currency settings

In your NOWPayments dashboard:

- **Coin Settings** → enable the cryptocurrencies you want to accept
- **Auto Conversion** (optional) → convert all incoming payments to a single currency (e.g., USDT)
- **Payout wallet** → set your wallet address for receiving funds

### Sandbox testing

Set `NOWPAYMENTS_SANDBOX=true` to use the sandbox environment at [sandbox.nowpayments.io](https://sandbox.nowpayments.io) (separate account required).

---

## Verifying Configuration

After setup, verify in the Admin Panel:

1. Go to **Settings → Payment**
2. Each configured provider shows a green **Connected** badge
3. If a provider shows **Not configured**, check your `.env` values and restart

You can also check via CLI:

```bash
vpnmanager status
```

The output includes payment provider status.

---

## Pricing & Plans

Payment providers work with subscription plans configured in the Admin Panel:

1. Go to **Admin Panel → Subscriptions**
2. Create plans with monthly/quarterly/yearly pricing
3. Set traffic limits, bandwidth, device count per plan
4. Users see these plans in the client portal payment modal

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Provider shows "Not configured" | Check `.env` for correct key names and values, restart services |
| PayPal "401 Unauthorized" | Verify Client ID and Secret match your app in PayPal dashboard |
| NOWPayments webhook not received | Verify IPN Callback URL is accessible from the internet (port 10443 open) |
| CryptoPay "Invalid token" | Regenerate token in @CryptoBot → Crypto Pay → My Apps |
| Payments not auto-confirming | Ensure webhook URLs are correctly configured in provider dashboards |

---

## Support

Need help? Contact [support@flirexa.biz](mailto:support@flirexa.biz) or visit [flirexa.biz](https://flirexa.biz).
