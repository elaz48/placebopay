# PlaceboPay

**The payment gateway that doesn't pay.** A fake, hosted-checkout placeholder for demos, prototypes and client previews. One URL, three outcomes, zero signup.

![PlaceboPay: a fake hosted checkout with approve, decline and cancel buttons](docs/og.png)

Real payment providers require an account, business details and sandbox credentials before you can test a simple redirect flow. PlaceboPay requires a URL.

- **Hosted version:** [placebopay.dev](https://placebopay.dev) (free, no signup, nothing stored)
- **Self-hosted version:** this repository. A single `index.php`, no dependencies, no database.

> Also searchable as: fake payment gateway, mock payment gateway, dummy payment gateway, payment gateway placeholder, sandbox checkout page.

## Why

You are building a demo, a prototype, or a client preview. The checkout flow needs to *exist* and *look real*, but choosing and onboarding a payment provider is premature. Wiring up a real sandbox means accounts, API keys, SDKs and webhooks, all for a button that needs to say "payment successful" in tomorrow's demo.

PlaceboPay is the placeholder: a realistic hosted checkout page where the visitor explicitly picks the outcome (approve, decline, cancel) and gets sent back to your app with the result on the query string. Exactly like a hosted Stripe or PayPal checkout, minus the paperwork and the money.

## Quick start (hosted)

Point your "Pay now" button at:

```
https://placebopay.dev/pay/
  ?name=Demo+Shop
  &amount=49.90
  &currency=EUR
  &ref=ORDER-1042
  &delay=2
  &success_url=https://your-app.test/checkout/return
```

The visitor sees a checkout page with the merchant name, the amount and three buttons. Whichever they click, they return to your URL with:

```
https://your-app.test/checkout/return
  ?placebopay=1
  &status=success        (or: failed | cancelled)
  &txn_id=PLB_9f2c41d8a03b57e6
  &ref=ORDER-1042
```

Existing query strings and URL fragments on your `success_url` are preserved.

## Quick start (self-hosted)

```bash
mkdir -p pay
curl -o pay/index.php https://raw.githubusercontent.com/elaz48/placebopay/main/index.php
```

Upload the `pay/` folder to any PHP host: shared hosting, DirectAdmin, cPanel, `php -S localhost:8000`, anything running PHP 7.4+ (8.x recommended). There is nothing to install and nothing to configure.

Optionally, open `index.php` and edit the `CONFIG` block at the top:

| Constant | Default | Purpose |
|---|---|---|
| `ALLOWED_RETURN_HOSTS` | `[]` (any) | Lock redirects to your own domains, e.g. `['myapp.test', 'staging.example.com']`. Recommended for private instances. |
| `MAX_DELAY_SECONDS` | `10` | Upper bound for the `?delay=` parameter. |
| `REPORT_URL` | GitHub issues | "Report misuse" link shown in the footer. Point it at your own repo or contact page. |
| `DOCS_URL` | `https://placebopay.dev` | Docs link shown when the page is opened without parameters. |

## Parameters

| Parameter | Required | Description |
|---|---|---|
| `success_url` | **yes** | Return URL. Must be `http` or `https`. `localhost` and private hosts are fine (the server never fetches this URL, it is only a client-side link). |
| `fail_url` | no | Return URL for declined payments. Defaults to `success_url`. |
| `cancel_url` | no | Return URL when the visitor cancels. Defaults to `success_url`. |
| `name` | no | Merchant name displayed on the page. Max 60 characters. |
| `amount` | no | Displayed amount, e.g. `49.90`. Digits with an optional 2-decimal part. |
| `currency` | no | 3-letter code, e.g. `EUR`, `HUF`, `USD`. Display only. |
| `ref` | no | Your order reference. Max 40 chars, `A-Z a-z 0-9 . _ -`. Echoed back on return. |
| `delay` | no | Simulated processing time in seconds after the click, clamped to 0-10. Cosmetic, JS-only. |

### Return parameters

| Parameter | Description |
|---|---|
| `placebopay` | Always `1`, so you can detect the return unambiguously. |
| `status` | `success`, `failed` or `cancelled`. |
| `txn_id` | Random per-page-load ID in the form `PLB_` + 16 hex chars. Not stored anywhere. |
| `ref` | Your `ref`, echoed back if you sent one. |

### Example

A plain HTML button is a complete integration:

```html
<a href="https://placebopay.dev/pay/?name=My+Shop&amount=12000&currency=HUF&ref=ORDER-7&success_url=https://myshop.test/thanks">
  Pay now
</a>
```

If the visitor clicks **Approve payment**, they land on:

```
https://myshop.test/thanks?placebopay=1&txn_id=PLB_9f2c41d8a03b57e6&ref=ORDER-7&status=success
```

A minimal return handler in PHP (works on 7.4+):

```php
if (($_GET['placebopay'] ?? '') === '1') {
    switch ($_GET['status'] ?? '') {
        case 'success':   show_thank_you_page($_GET['ref'] ?? ''); break;
        case 'failed':    show_payment_failed_page(); break;
        case 'cancelled': redirect_back_to_cart(); break;
        default:          show_error_page();
    }
}
```

## Abuse resistance

A page that imitates payments will attract the wrong crowd, so PlaceboPay is engineered to be worthless to scammers. If you fork this project, **please keep these properties intact**:

1. **Zero input fields.** There is nothing to type anywhere on the page, so there is nothing to phish. The card shown is a static, well-known test number (`4242 4242 4242 4242`).
2. **No automatic redirects.** Every redirect is an explicit, clearly labelled click, and the destination domain is printed under the buttons. The page cannot be used as a silent open-redirect hop.
3. **Permanent test banner.** Every render states in plain words that the payment is simulated and no card data is collected. No parameter can hide or alter it.
4. **Strict URL validation.** Only `http`/`https` schemes are accepted; URLs with embedded credentials (`https://user:pass@host`) are rejected; all output is HTML-escaped.
5. **No iframes, no indexing.** `X-Frame-Options: DENY` blocks embedding (clickjacking), `noindex` keeps checkout pages out of search engines, `Referrer-Policy: no-referrer` avoids leaking your URLs.
6. **Nothing is stored.** No accounts, no database, no logs of parameters, no cookies. All state lives in the URL of a single request.
7. **Host allowlist for private instances.** Set `ALLOWED_RETURN_HOSTS` and your instance will only ever redirect to domains you chose.

Seen the hosted instance being misused anyway? [Open an issue](https://github.com/elaz48/placebopay/issues) and it will be acted on.

## What PlaceboPay is not

- It is **not a sandbox**. There are no API keys, no webhooks, no server-side verification. If you need to test webhook handling or a specific provider's API shapes, use that provider's real sandbox.
- It is **not for staging payment logic you plan to ship**. It is a placeholder for the phase *before* you pick a provider.
- It **never touches money**, card data, or personal data. By design it cannot.

## FAQ

**Does it work with localhost?**
Yes. `success_url=http://localhost:3000/return` is the primary use case. The redirect happens in the visitor's browser, so the server never needs to reach your machine.

**Can I make a specific outcome automatic, e.g. always fail?**
No, and that is intentional: automatic redirects are the number one abuse vector for a page like this. The outcome is always a human click. In a demo that is a feature, since you choose the story live.

**Why PHP?**
Because the target deployment is "any shared hosting on the planet, in one minute, with zero build steps". A single PHP file is still unbeatable at that.

**Is the txn_id stable or verifiable?**
No. It is random per page load and stored nowhere. Treat it as demo garnish, not as proof of anything.

## License

MIT. Do what you want, keep the abuse-resistance intact, and never present PlaceboPay to anyone as a real payment page.
