# DNS & Domains

Forte does two related things with domains, both **account-level** (peers to projects, not nested in
one) and both requiring a **verified billing method**:

1. **DNS hosting (beta)** — make Forte authoritative for a domain you own.
2. **Domain registration** — buy a new domain through Forte.

Both are managed under **`/console/domains`**. DNS hosting also has a **`forte dns` CLI**; domain
registration is **console-only**. Neither has an SDK — do not invent `forte.projects.dns.*` or similar.

## Three ways to point a domain at Forte

Don't conflate these — pick the one that matches what the customer wants:

- **Custom domain (CNAME).** Keep DNS wherever it is; add a custom domain to a **service** or **website**
  and point a `CNAME` at the `domains.tryforte.dev` endpoint shown. Forte provisions/renews TLS. This is
  the lightest option and needs no DNS hosting. (`forte websites domains ...`, or the service/website
  page in the console.)
- **DNS hosting.** Delegate the domain's nameservers to Forte so Forte hosts the whole zone. Connecting a
  website/service on that domain then wires the records automatically — nothing to copy.
- **Buy the domain from Forte.** Registration sets Forte DNS hosting up automatically.

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

Customers can **buy a domain directly in Forte**: open `/console/domains` and choose **Register a
domain**. This is **console-only** — there is **no** `forte domains` CLI command and no SDK. The only
requirement is a **verified billing method** (registration spends real money).

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
4. **Done in about a minute.** Forte registers the name, creates its DNS zone, and points the domain's
   nameservers at Forte. Connect a website or service on it and the records wire themselves — there are
   no nameservers to copy anywhere.

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

## What Forte is *not*

- For a domain you already own **elsewhere**, Forte is **not your registrar** — you can't transfer the
  registration in to pay Forte for it. Bring the domain and either add it as a **custom domain (CNAME)**
  or move its **DNS hosting** to Forte.
- DNS hosting does not give a website Forte end-user auth, request logs, or metrics — those are
  **service** features. Website traffic still bypasses the gateway (see `references/setup-walkthrough.md`).

Docs: [Custom domains — services](https://forteplatforms.com/docs/core-concepts/services) ·
[Custom domains — websites](https://forteplatforms.com/docs/core-concepts/websites) ·
Help: [DNS hosting](https://forteplatforms.com/help/deployments/dns-hosting) ·
[Custom domain](https://forteplatforms.com/help/deployments/custom-domain)
