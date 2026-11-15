# DNS & Domains

Forte does two related things with domains, both **account-level** (peers to projects, not nested in
one) and both requiring a **verified billing method**:

1. **DNS hosting (beta)** — make Forte authoritative for a domain you own.
2. **Domain registration** — buy a new domain through Forte, or transfer one you own to Forte.

Both are managed under **`/console/domains`**. DNS hosting also has a **`forte dns` CLI**; domain
registration is **paid for in the console only**, though the Forte MCP tools can search names and
prices and link to the checkout (see "Buying a domain for a user" below). Neither has an SDK — do not invent `forte.projects.dns.*` or similar.

## Three ways to point a domain at Forte

Don't conflate these — pick the one that matches what the customer wants:

- **Custom domain (CNAME).** Keep DNS wherever it is; add a custom domain to a **service** or **website**
  and point a `CNAME` at the `domains.tryforte.dev` endpoint shown. Forte provisions/renews TLS. This is
  the lightest option and needs no DNS hosting. (`forte websites domains ...`, or the service/website
  page in the console.)
  **Root domain and `www`:** `example.com` and `www.example.com` are each their own custom domain
  **connection** on the website or service. `www` is part of the domain the customer already has, so
  there is **nothing more to buy or register**. When you connect a root domain, **connect `www` too**
  (the MCP tools `create_web_app_custom_domain` / `create_service_custom_domain` and the CLI do this
  by default; in the console use **Connect www**) unless the customer says not to.
  Describe it as a second connection, never as "a separate domain".
- **DNS hosting.** Delegate the domain's nameservers to Forte so Forte hosts the whole zone. Connecting a
  website/service on that domain then wires the records automatically — nothing to copy.
- **Buy the domain from Forte.** Registration sets Forte DNS hosting up automatically.
- **Transfer the domain to Forte.** Forte becomes its registrar and hosts its DNS (see "Transferring
  a domain to Forte" below).

## DNS hosting (beta)

Point a domain's nameservers at Forte and Forte becomes authoritative for it.

- **Delegation.** Forte creates a hosted zone and assigns **four vanity nameservers**
  (`ns1`–`ns4.tryforte.dev`). The customer replaces their domain's nameservers with these **at their
  registrar**. The zone lifecycle is `CREATING → PENDING_DELEGATION → ACTIVE` (or `FAILED`); it goes
  `ACTIVE` once Forte observes the delegation live. `forte dns sync <zoneId>` re-checks and activates.
- **Whole domain or subdomain.** Host `example.com` (the whole tree) or `app.example.com` (a subdomain,
  leaving the rest on the current provider). Forte hosts **one zone per domain tree**: adding
  `example.com` when you already host `app.example.com` **merges** them into one zone (records carry
  over; names under the absorbed subdomain can be briefly unreachable, ~1 minute, while the zone
  rebuilds). The `--merge` flag / merge-on-create handles this; absorbing a zone requires delete rights
  on it. To keep only a subdomain on Forte, host the parent elsewhere first.
- **Scan & import.** On create, Forte scans the domain's current DNS as a **starting point — not a
  complete copy**. Compare against the current provider and **import any missing records before
  switching nameservers**. A pasted **BIND zone file** can be previewed and imported in bulk.
- **DNSSEC.** If the domain has **DNSSEC enabled at the registrar, turn it off before changing
  nameservers** or the domain stops resolving. Forte warns when it detects DNSSEC on a domain being
  moved.
- **Managed records.** Connecting a website or service on a hosted domain **auto-writes the traffic and
  TLS-validation records**. Those are marked **managed** and **locked** — shown in the console/CLI but
  not editable by hand.
- **Record types** you can edit: `A`, `AAAA`, `CNAME`, `MX`, `TXT`, `NS`, `CAA`, `SRV`.
- **History.** Each zone keeps a change history (lifecycle + record edits, including Forte's managed
  edits), newest first.

CLI: `forte dns list | create <domain> [--merge] | get <zoneId> | sync <zoneId> | delete <zoneId> [--yes]`
and `forte dns records list|add|remove <zoneId> ...` (see `references/cli.md`). Console:
`/console/domains`.

## Domain registration

Customers can **buy a domain directly in Forte**: open `/console/domains`, choose **Add a domain**,
then **Buy a new domain**. This is **console-only** — there is **no** `forte domains` CLI command and no SDK. The only
requirement is a **verified billing method** (registration spends real money).

### Buying a domain for a user

When a user asks you to buy, register or find a domain, you search and they pay:

1. If the Forte MCP tools are connected, call **`list_domain_availability`** with up to 20 full names
   (`example.com`, `example.dev`); a bare word with no ending comes back as invalid, so propose the endings
   yourself. It needs a verified billing method and a connection that covers all projects.
2. Show the available names with their registration and renewal prices (returned in USD cents per
   term) and let the user pick.
3. Give the user that name's **`checkoutUrl`**. It opens the console with the name already selected;
   the user confirms the owner contact, accepts the registration terms and pays there.
4. Once they've paid, connect the domain to their website or service with
   **`create_web_app_custom_domain`** / **`create_service_custom_domain`** (this also connects `www`).
   No need to wait for DNS first; see "How long a new domain takes to start working" below.

The tools **can't buy, reserve, renew or transfer** a domain. Never say a domain is bought or held
until the user has finished the console checkout: price and availability are confirmed again there.
Without the MCP tools, send the user to `/console/domains` → **Add a domain**.

How it works:

1. **Search.** Type a name; Forte shows it across common endings with the price for each. The price
   shown is the whole cost of the term, and the renewal price is shown beside it. Most endings are
   1-year terms; **`.ai` is sold in 2-year terms**, charged upfront.
2. **Owner details.** Registries require real contact details for a domain's owner. The form is
   pre-filled from the account and can be edited; saved contacts can be reused. The customer is the
   legal owner (registrant) of the domain, not Forte.
3. **Pay.** The saved card is charged immediately, with the bank's 3-D Secure check. It appears in
   billing history as its own invoice and in the cost explorer under **Domains**. Domain purchases
   **can't be refunded** once the domain is registered.
4. **Registered in about a minute.** Forte registers the name, creates its DNS zone, and points the
   domain's nameservers at Forte. Connect a website or service on it and the records wire themselves —
   there are no nameservers to copy anywhere.

### How long a new domain takes to start working

Registered is not the same as resolving. After the purchase completes:

- **The registry publishes the domain** a few minutes later (about 10 minutes is normal). Until then
  the name does not exist in DNS at all, and a lookup anywhere returns "no such domain".
- **Forte's zone becomes active** once Forte sees that delegation. It checks every 15 minutes, so allow
  up to about 30 minutes after purchase.
- **A custom domain connected on it** stays in "pending DNS verification" until the zone is active.
  Forte then writes the traffic and verification records itself and rechecks every 10 minutes, then
  issues the certificate. Connecting the website right after buying is fine; nothing has to be redone.
- **A resolver that asked too early** can keep answering "no such domain" for up to an hour, so the
  customer's own machine may lag behind the rest of the internet.

So a domain bought and connected in one sitting **normally starts serving within an hour**. Inside that
window, missing records, a pending custom domain, or "no such domain" are **expected**. Don't call it a
bug, and don't remove and re-add the custom domain: that restarts verification and gains nothing. Check
status again later (`list_web_app_custom_domains` / `list_service_custom_domains`, or `forte websites
domains list`). If the zone is still not Active after about an hour, or the custom domain is still
pending an hour after the zone went Active, that is worth a support case.

What's included and what isn't:

- **WHOIS privacy is always on** and free. Contact details are never shown in public WHOIS, and this
  can't be turned off.
- **Auto-renewal** is on by default and can be switched off. Forte emails about 30 days and 7 days
  before expiry, charges the saved card about two weeks before expiry, and retries if the card fails. A
  failed renewal **never suspends the account**; the domain simply expires if it isn't renewed.
- **"Let it expire"** stops renewal. The domain keeps working until its expiry date and can be switched
  back on any time before then. There is no way to delete a domain early, and nothing is refunded.
- **Transfer out** is always available from the domain's page (it unlocks the domain and shows a
  transfer code), except in the **first 60 days** after registration, which ICANN rules forbid.
- **DNSSEC is not available** for domains registered through Forte.
- **Which endings.** About 80 common ones (`.com`, `.net`, `.org`, `.dev`, `.app`, `.io`, `.co`,
  `.ai`, `.cloud`, `.tech` and similar). Country endings with residency rules (`.us`, `.ca`,
  `.uk`, `.de`, `.eu`) aren't offered. Names priced above **$200** per term, including premium
  names, aren't sold. An account can hold **3 registered domains** by default; support can raise that.

**The registrar is Spaceship.** Forte partners with Spaceship, an ICANN-accredited registrar, for the
registration itself. Customers **will get emails from Spaceship** about their domain, most importantly
one asking them to **verify their email address**. They must do that within **15 days** or the registry
suspends the domain until they do. If a customer asks why "Spaceship" is emailing them about a domain
they bought from Forte, this is why — it's expected and legitimate. Everything else (renewals, DNS,
billing, support) happens in Forte.

If a purchase is charged but the registration can't be completed, Forte opens a **support case
automatically** and the console takes the customer straight to it; the team either finishes the
registration or refunds in full.

## Transferring a domain to Forte

A customer can move a domain they already own to Forte, so Forte is its registrar and renews it:
`/console/domains` → **Add a domain** → **Transfer a domain you own**. Console-only (no CLI, SDK or
MCP tool), and it needs a verified billing method.

1. **Check the domain.** Forte confirms it can take the domain and shows the transfer price, which
   also extends the expiry by one term, and the renewal price.
2. **Prepare at the current registrar.** Unlock the domain, request the **transfer code** (EPP or
   authorization code), and make sure email to the registrant address is readable. **Turn off DNSSEC
   first**, or the domain stops resolving when its nameservers switch.
3. **Bring DNS records over.** Forte copies the domain's current records into a Forte zone before the
   transfer, so nothing breaks when it completes. If Forte already hosts the domain's DNS, that zone is
   kept as is.
4. **Owner details, terms, pay.** Same as buying: the saved card is charged, with the bank's 3-D
   Secure check.

A transfer **can take up to 7 days**, and the current registrar may email the customer to approve it.
Until it completes, the existing DNS keeps serving the domain. It **can't be cancelled from Forte**
once submitted (decline it at the current registrar instead), and a transfer the registrar or
registry rejects is **refunded in full**. After it completes, the domain can't be transferred again
for **60 days**.

Forte won't take a domain when: its ending isn't one Forte offers; nobody has registered it (buy it
instead); it is already with Forte or another transfer for it is in progress; the account is at its
domain limit; or another account hosts DNS for it on Forte.

## What Forte is *not*

- Transferring is optional. To keep a domain at its current registrar, add it as a **custom domain
  (CNAME)** or move only its **DNS hosting** to Forte.
- DNS hosting does not give a website Forte end-user auth, request logs, or metrics — those are
  **service** features. Website traffic still bypasses the gateway (see `references/setup-walkthrough.md`).

Docs: [Custom domains — services](https://forteplatforms.com/docs/core-concepts/services) ·
[Custom domains — websites](https://forteplatforms.com/docs/core-concepts/websites) ·
Help: [DNS hosting](https://forteplatforms.com/help/deployments/dns-hosting) ·
[Custom domain](https://forteplatforms.com/help/deployments/custom-domain)
