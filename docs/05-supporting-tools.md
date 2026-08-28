# Supporting and Emerging Tools

_Part of the [NEXT System Architecture](../README.md). See also the [Architecture Overview](00-architecture-overview.md)._

Beyond the core listing pages and lead pipeline, NEXT's system includes a set of
supporting tools. Each is described below with what it does, how it connects to
the rest of the system, its technology stack, and an honest status of whether it
is **live** and in use, or still **in development**. These are deliberately
built as small, single-purpose services rather than one monolith, so each can
ship, be tested, and evolve on its own.

---

## Operations and approval board (Mission Control)

**What it does.** A visual status-and-approval surface for the automated parts
of the system. It has two views: a plain-English board that shows whether the
automations are running and healthy ("is it working"), and an approval view
where an outward-facing action (something that sends, spends, or publishes) is
staged and held until a person approves it, rather than firing on its own. The
design principle is that the automation can prepare and stage work, but a human
confirms anything that leaves the system.

**How it connects.** It reads the health and activity of the always-on
automation (the MLS listing updater, the Discord bots, the open-house worker)
and reports their state. A monitoring component watches those services and posts
a daily "systems live" green-check summary plus real-time up/down and problem
alerts into a Discord operations channel, so a non-technical owner can see at a
glance that the machinery is working.

**Stack.** Next.js and React with TypeScript for the two views, a Convex-shaped
real-time data model (missions, approvals, an append-only activity ledger,
scopes), packaged as a Docker container. The health-monitoring and status-board
component runs as a scheduled service on the Linux VPS and posts to Discord.

**Status: mixed, described honestly.** The **health monitoring and daily
status board are live internally** and running today: they watch the production
automation and post to the operations channel on a schedule. The **broader
multi-tenant approval board is in development.** The engine, data model, and
safety rules are built and tested, but the full web application (role-based
sign-in for a broker versus an operator, and the live cloud data connection)
is not yet wired to real credentials. It currently runs as a local, no-secrets
build. The approval concept is proven in tests; the production web surface is
the next step.

---

## PRIME: automated pre-listing report microsite

**What it does.** PRIME auto-generates a per-address, agent-branded pre-listing
microsite before a listing appointment. For a given property address it produces
a comparative market analysis using the **Sales Comparison Approach** (the
standard appraisal method: recent comparable sold homes, bracketed by size and
recency, adjusted on a market-derived grid, then reconciled into a value
**range**, deliberately a range and never a single recommended price). The page
presents the subject home's verified facts, the comparable sales, a value range,
a map, and an agent's branding and contact details.

**How it connects.** It merges three data sources into one report: the listing's
own facts (the property's public listing page), county public records (Maricopa
County's parcel and deed system, which supplies verified size, year built, lot,
assessed value, and recorded sale prices), and live comparable sold listings
pulled from public real-estate portals. County records act as a fact-check
against the portal data. The output is a self-contained, shareable web page an
agent can send or present.

**Stack.** Node.js (version 20) for the data and valuation layer; the microsite
is a static, self-contained branded HTML page (light and print-friendly theme,
web fonts, embedded data), deployed to Vercel. Data comes from the Maricopa
County public parcel/deed API and public portal scrapes; no paid MLS feed is
required.

**Status: live as a demo.** The valuation engine is built, tested, and
produces real reports, and a **public demo page is live**: a roughly $900K
example property is deployed and shareable at
https://next-prelisting-sierra.vercel.app. Generating a report today is a
manual, per-address run. Automatic triggering (for example, generating the
microsite the moment a listing appointment is booked) is a planned later phase,
not yet wired.

---

## Seller value and engagement connector (Seller Metrics)

**What it does.** Two related seller-facing capabilities. First, **engagement**:
for an active listing it reads how many people have viewed and saved the home on
public portals, so a seller can be shown real interest in their property.
Second, **value**: for a homeowner's own address (typically a home not yet
listed) it resolves that address to a public value estimate.

**How it connects.** Given an address, it finds the correct portal listing pages
(with a strict address-matching safeguard so it never returns the wrong home),
reads the published view/save counts and value estimates, and aggregates them
into one seller-facing result. The intended destination for these figures is the
agent's CRM (GoHighLevel), so a seller inquiry becomes a tracked contact and the
engagement numbers can feed follow-up.

**Stack.** TypeScript and Node.js, reading public portal data. Designed to hand
results into GoHighLevel with anti-abuse safeguards on any public-facing form
(required human-check, honeypot, rate limiting, and a rule never to overwrite an
existing contact).

**Status: partially built and held.** The connector engine is built and has run
live end-to-end. The **engagement half is the more ready piece:** it works live
on active listings today. The **value half is on hold** pending NEXT's MLS/Spark
data access, because MLS comparable data is likely to replace the public-estimate
source; the free public path reliably returns only a single-portal estimate with
real coverage gaps, so the data-source choice is deferred until MLS access lands.
The seller-facing forms and the live CRM wiring are **not yet built**. This tool
is in development.

---

## Competitor production X-Ray (Agent X-Ray)

**What it does.** Reconstructs a real-estate agent's production history (units
sold, sales volume, average days on market, list-to-sold ratio, price-band and
geographic distribution, and time-series trends) from closed listing records.
It is a deterministic aggregation engine: the same input always yields the same
report, with nothing estimated or fabricated.

**How it connects.** It consumes closed transaction data and rolls it up per
agent and per office, including office comparison and agent ranking. A public-web
connector reconstructs that transaction data from public agent-profile and
listing-detail pages when a direct MLS feed is not available.

**Stack.** TypeScript and Node.js. The aggregation core is standalone and
fixture-tested; the public-web connector reads public real-estate portal pages.

**Status: early, in development.** The aggregation engine and the public-web
connector are built and tested against real captured pages, but the tool is at
an early stage. Two things are still open: full production fidelity is best with
a direct MLS feed (a Spark/MLS access token), and public agent-activity feeds do
not distinguish buyer-side from seller-side transactions, which a second data
source is needed to resolve. It is not yet a live, in-use tool.

---

## New-listing onboarding tool (Listing Onboard)

**What it does.** A repeatable path for getting a brand-new listing onto the live
site. The always-on automation updates listings that already exist in the
system; a genuinely new address has no automated way in. This tool closes the
first half of that gap. Given a new address and a public listing scrape, it
produces a complete, schema-matched record for the site's database, validates
that every required field is actually present and correctly typed, and outputs
the exact write payload a person would then run.

**How it connects.** It mirrors the live listing database's schema (the same
table and field definitions the public listing pages use) and constructs the
record using the live system's own conventions (URL slug, address formatting),
so its output is shaped identically to what the automated updater would produce.
It deliberately **does not write anything.** It never calls the database and
never triggers a page rebuild. The actual write is a separate, human-run
supervised step, using the validated payload this tool hands over.

**Stack.** TypeScript and Node.js; reads a public listing scrape and produces a
validated record payload matching the listing pages' Convex schema.

**Status: dry-run, in development.** The tool builds and validates the payload
today and is fully tested, but by design it stops short of writing. It is a
dry-run producer. Connecting the validated payload to an automated (still
human-approved) live write is the remaining half, not yet built.

---

## Hosting and infrastructure

At a high level, the system runs across four hosting layers:

- **Web hosting (Vercel).** The public listing pages and the seller-handoff
  surfaces are served from Vercel. Static report pages such as the PRIME demo are
  also deployed here.
- **Real-time data (Convex).** The live data layer (listing records and related
  application state) runs on Convex, which provides the real-time database the
  listing pages and tools read from and write to.
- **Always-on automation (Node/Docker on a Linux VPS).** The continuously
  running services (the MLS listing updater, the Discord bots, the open-house
  worker, and the operations status board) run as Node.js services in Docker
  containers, with scheduled jobs, on a Linux VPS.
- **CRM (GoHighLevel).** Customer and lead records live in GoHighLevel, the CRM
  that receives leads coming back from the listing pages and ad campaigns.

Together: web pages are served from Vercel, backed by the Convex real-time data
layer; the ongoing automation runs on the VPS; and leads land in GoHighLevel.
