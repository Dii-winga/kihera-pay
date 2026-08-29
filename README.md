# Invoice payment public page

Static, framework-free page behind the **Pay this invoice** link and QR on an
invoice (`https://pay.kihera-farms.com/i/{token}`). See
`docs/domain/PAYMENTS.md`.

## Source of truth & the public mirror

**This directory is the source of truth**, exactly as `web/trace/` is. The page
is *deployed* from a separate public repo, because Netlify's free tier cannot
build the private `kihera-app`. That repo is a **generated mirror** — its
`index.html` and `_redirects` are copies of the two deployable files here.
**Never edit the mirror by hand; edit these files.**

Keeping the two in step follows the trace page's discipline: prefer a workflow
that pushes the deployable files on every change to `web/pay/**` on `main`;
failing that, copy `index.html` and `_redirects` into the mirror and push, or
the live site keeps serving the old version. `README.md` is not mirrored.

**Neither the mirror nor the DNS record has been created.** That is a
deployment step (PAYMENTS.md §9), and this repository deliberately does not
create or modify a remote repository.

## What the page does

Reads the token from the path (`/i/{token}`, or `?i=` dev fallback) and calls
the Supabase RPC `public.payment_link_summary(token)` with the
**publishable/anon key** — public by design, the same key the trace page and the
mobile app ship with. Everything the page may know is decided server-side by
that function: the issuing farm's name, the invoice number, the currency, the
amount **currently** due, the due date, the state, and whether the payer must
supply an email. Nothing identifies the customer.

**Pay** posts the token (and, if needed, an email) to the `pay-init` Edge
Function, which derives the amount, initializes Paystack and returns a hosted
checkout URL. The page validates that the URL is HTTPS on a Paystack host
before following it.

**The browser decides nothing about money.** No amount is sent, and none is
accepted from the query string. The return from Paystack is informational: the
page re-reads state from the server and never marks itself paid because a query
parameter said so.

## Deploy layout

The mirror's root **is** the publish root, so `/index.html` is served at the
site root and the redirect in `_redirects` (`/i/* → /index.html`, 200) makes
every token path serve the page.

**No logo, and no farm branding beyond the name in the payload.** Same decision
as 0163 for the trace page: this page serves the platform, and it names the
seller in text rather than printing one tenant's mark on another tenant's
invoice.

## Secrets

There are none here, and there must never be any. The only key in this
directory is the public anon key. The Paystack secret lives in the Edge
Function environment; a page that could reach it would be a page an attacker
could read it from.
