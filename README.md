<div align="center">

# 🛍️ Jhola

### One grocery list. Three apps. The cheapest *real* basket.

Type your list the way you'd say it: `doodh 2 packet, atta 5kg, maggi 4, dhaniya`.<br/>
Jhola checks live prices on **Blinkit**, **Zepto** and **Swiggy Instamart**, works out the cheapest way to buy everything **after delivery, handling and small-cart fees**, fills the carts on your own accounts, and **stops at the payment page**.

**You approve. You pay. Jhola never pays.**

![Node.js](https://img.shields.io/badge/Node.js-ESM-339933?logo=nodedotjs&logoColor=white) ![Express](https://img.shields.io/badge/Express-5-000000?logo=express&logoColor=white) ![Playwright](https://img.shields.io/badge/Playwright-CDP-2EAD33?logo=playwright&logoColor=white) ![TinyFish](https://img.shields.io/badge/TinyFish-Agent%20%C2%B7%20Browser%20%C2%B7%20Search%20%C2%B7%20Fetch-FF6B35) ![Tests](https://img.shields.io/badge/tests-21%20passing-brightgreen)

</div>

## 💡 Why Jhola

Quick-commerce prices aren't the same across apps, and the sticker price isn't what you pay. A ₹2 cheaper item can end up ₹30 more expensive once a delivery fee or small-cart fee kicks in. Finding the real cheapest basket means checking three apps item by item and doing fee maths in your head. Most people don't, and older parents definitely don't.

**Jhola is built for them.** A family member connects the apps once. After that, Mom or Dad just types (or pastes) the weekly list in plain Hinglish and taps **Approve**.

---

## ✨ What it does

| Tab | What you get |
|---|---|
| 🧺 **Basket** (Jhola) | One list → live prices from 3 apps → the cheapest plan after fees (one app or a split) → carts filled on your accounts → you pay |
| 🛒 **Order** | *"2 Vaseline lip balm from Blinkit"*: finds the product, asks once ("Is this it?") and stops searching the moment you say yes |
| 🔍 **Compare** | Best in-stock price for any product across online stores, with a screenshot of the winning listing as proof |
| 📝 **Forms** | Two-phase form filling. A dry run shows every value it *would* type, and it submits only after you approve |
| 🕘 **History** | Every run with its live log, results and TinyFish cost |
| ⚙️ **Setup** | Connect Blinkit / Zepto / Instamart once, with your own phone and OTP |
| 👤 **Me** | Name, phone, address and PIN code, used for forms and location |

### 👵 Caretaker mode
- **Default stores**, in order, for when no store is named
- **Simple mode**: bigger text and fewer tabs
- **Caretaker PIN**, so settings can't be changed by accident

---

## 🔄 How a basket gets planned

```
 "doodh 2 packet, atta 5kg, maggi 4, dhaniya"
                   │
                   ▼
 ┌────────────────────────────┐
 │ 1. PARSE (rules, no LLM)   │  milk 2 × 500 ml · atta 5 kg · maggi ×4 · coriander leaves
 └─────────────┬──────────────┘
               ▼
 ┌────────────────────────────┐   One Agent run per app, staggered 4 s apart,
 │ 2. COLLECT                 │   stealth browser, India-proxy profile.
 │   Blinkit │ Zepto │ Insta  │   Top 3 results per item + any fees seen.
 └─────────────┬──────────────┘   Blocked? → scripted Browser-API scan fallback
               ▼
 ┌────────────────────────────┐   Pack sizes → price per kg / L / piece,
 │ 3. MATCH                   │   packs needed, exact vs size variant vs substitute
 └─────────────┬──────────────┘
               ▼
 ┌────────────────────────────┐   All 7 app combinations + local search
 │ 4. OPTIMISE                │   to cross free-delivery / small-cart thresholds
 └─────────────┬──────────────┘
               ▼
 ┌────────────────────────────┐   Live browser panels, then the basket table.
 │ 5. PLAN  ← you review      │   Swap products, drop items, force one app, edit fees.
 └─────────────┬──────────────┘   Totals update instantly in the browser.
               ▼  Approve
 ┌────────────────────────────┐   Clears the cart, adds exact products,
 │ 6. FILL                    │   reads the real bill. Stops before payment.
 └─────────────┬──────────────┘
               ▼
 ┌────────────────────────────┐   Planned vs actual. More than ₹10 apart → asks you again.
 │ 7. CHECK + LEARN           │   Real fees are saved for the next plan.
 └─────────────┬──────────────┘
               ▼
 ┌────────────────────────────┐
 │ 8. PAY  ← you, in your app │   Your logged-in session opens in a Jhola window
 └────────────────────────────┘
```

---

## 🧮 The optimiser

Shared by the server and the browser (`public/lib/basket.js`, pure functions), so every swap or drop re-plans instantly with no round trip.

1. **Pack maths:** parses `"500 ml"`, `"4 x 70 g"`, `"Pack of 2"`, `"1 L"` and so on, then works out how many packs cover what you asked for. Each offer gets a unit price (₹/kg, ₹/L, ₹/pc).
2. **Match quality:** every result is classed as **exact**, **size variant** or **substitute**. Items marked `only` / `sirf` / `no substitute` never get substitutes.
3. **Fee model** per app: delivery fee, free-delivery threshold, handling fee, small-cart fee and threshold. Starts from estimates, then **learns the real fees** from your actual carts.
4. **Search:** tries every non-empty subset of apps (7 combinations). For each, it assigns items to their cheapest app, then runs a **local search** that moves single items between apps to cross free-delivery or small-cart thresholds, or to empty an app entirely.
5. **Ranking:** covers the most items first, then the lowest total. Plans within **₹5** of each other go to the better-rated app.
6. **Savings:** compares the best plan with the best single-app plan, so you can see what the split is worth.

---

## 🗣️ Hinglish list parser

No LLM, just fast, testable rules (`lib/basket/parse.js`):

| You type | Jhola understands |
|---|---|
| `doodh 2 packet` | milk · 2 × 500 ml = 1 L |
| `atta 5kg` | atta · 5000 g |
| `ek darjan ande` | eggs · 12 pcs |
| `amul butter sirf` | butter · brand **Amul** · no substitutes |
| `pyaz aur tamatar` | onion, tomato *(splits on `,` `;` `+` `and` `aur`)* |

- **~95 Hinglish words:** doodh, cheeni, haldi, pyaz, dhaniya and more
- **Hindi number words:** ek, do, teen, aadha, darjan
- **~90 Indian brands** recognised
- **Typical pack sizes** for "1 packet" of milk, curd, paneer and butter

---

## 🛡️ Safety by design

| Rule | How it's enforced |
|---|---|
| **Never pays** | Every agent goal forbids Pay / Place Order / Checkout. You pay in the app yourself |
| **Nothing in a cart without approval** | Carts are filled only after you tap **Approve** on the plan |
| **No surprises** | Substitutes and size changes are flagged on the plan. The approval dialog warns that existing cart items will be cleared |
| **Never handles secrets** | Agents never type passwords, OTPs, card, UPI or bank details. Forms lock out sensitive fields (password, OTP, CVV, Aadhaar, PAN and others) |
| **Your login, typed by you** | You enter your phone and OTP straight into the app's own page in a streamed window. Jhola only keeps the saved TinyFish browser profile |
| **`COMPLETED` ≠ success** | Every result is checked for login walls, CAPTCHAs, "access denied" pages and empty data |
| **Forms submit once** | Dry run first, you approve every value, and the form is submitted at most once |
| **Local data** | Profile, settings, fees and history live in `./data` as JSON on your machine |

---

## 🧯 Failure handling

| Failure | What Jhola does |
|---|---|
| `logged_out` | Marks that app's login as expired and points you to **Setup** |
| `blocked` / `SITE_BLOCKED` / CAPTCHA | Retries with a scripted **Browser-API scan**, then plans without that app |
| Item not on one app | Treated as unavailable on that app |
| Item not on any app | Listed as **Not available** on the plan |
| SSE stream drops | Polls `GET /v1/runs/{id}` until the run finishes |
| Transient API errors (429 / 5xx) | Retries with exponential backoff, respecting `Retry-After` |
| Cart total ≠ plan (more than ₹10 off) | Shows planned vs actual side by side and asks you again |
| Out of credit | Plain-language message pointing to your TinyFish wallet |

---

## 🐟 Built on TinyFish

| TinyFish API | Used for |
|---|---|
| **Agent** (`run-sse`) | Searching each app, filling carts, finding products and reading forms. Uses `output_schema` for typed results, `browser_profile: stealth`, and live `streaming_url` panels |
| **Browser Context Profiles** | One saved login per app with `proxy_country_code: "IN"` (`/v1/profiles`, `/setup-session`, `/save`), reused through `use_profile` + `profile_id` |
| **Browser** (CDP) | The streamed login and payment window, the fallback price scan through Playwright, and proof screenshots |
| **Search** | Finding store pages for **Compare** |
| **Fetch** | Cheap triage that drops blogs, reviews and listing pages before spending agent steps |

Every job tracks its own usage (agent runs, steps, browser minutes) and shows the estimated cost in **History**.

---

## 🚀 Getting started

### Prerequisites
- Node.js 18+
- A TinyFish API key from [agent.tinyfish.ai/api-keys](https://agent.tinyfish.ai/api-keys)

### Install and run
```bash
git clone https://github.com/shauryasuyal/Jhola-TinyFish.git
cd Jhola-TinyFish
npm install
cp .env.example .env       # add TINYFISH_API_KEY
npm start                  # → http://localhost:4321
```

### Test
```bash
npm test                   # 21 tests: parser, matcher, optimiser and safety (node --test)
```

### First run
1. **Me:** add your name, phone, address and PIN code.
2. **Setup:** connect Blinkit / Zepto / Instamart. A TinyFish browser (India) opens inside Jhola. Log in with your phone and OTP, check the delivery address, and press **Save & close**.
3. **Basket:** type your list, adjust items if needed (brand, size, "no subs"), and press **Find cheapest basket**.

> Without a connected app, Jhola still compares prices using your PIN code. It just can't fill that app's cart.

### Environment

| Variable | Default | Purpose |
|---|---|---|
| `TINYFISH_API_KEY` | none | **Required.** Your TinyFish key |
| `PORT` | `4321` | Server port |
| `ERRAND_DATA_DIR` | `./data` | Where profile, settings, fees, history and screenshots are stored |

---

## 🗂️ Project structure

```
Jhola-TinyFish/
├── server.js               Express server: REST API, job runner and static UI
├── lib/
│   ├── tinyfish.js         TinyFish client: Search, Fetch, Agent (SSE + polling), Browser, Profiles
│   ├── accounts.js         One-time app logins: streamed browser window, saved profiles
│   ├── jobs.js             Background jobs: live logs, approvals, SSE stream, cost tracking
│   ├── safety.js           Shared guardrails, sensitive-field and block detection
│   ├── stores.js           Store directory: aliases and on-site search URLs
│   ├── proof.js            Proof screenshots through the Browser API
│   ├── orderParse.js       "2 Amul butter from Blinkit" → items + store
│   ├── store.js            Local JSON persistence in ./data
│   ├── basket/
│   │   ├── jhola.js        Collect → plan → fill → check pipeline
│   │   ├── parse.js        Hinglish list parser
│   │   └── scan.js         Browser-API fallback scan
│   └── chores/
│       ├── order.js        Order: find, confirm once, hand off
│       ├── priceHunt.js    Compare: Search → Fetch → Agent → proof
│       └── formFill.js     Forms: dry run → approve → submit once
├── public/
│   ├── index.html · app.js · styles.css
│   └── lib/basket.js       Optimiser shared by server and browser
└── test/                   node --test suites
```

---

## 🔌 API reference

<details>
<summary><b>Show endpoints</b></summary>

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/status` | Server and TinyFish key status |
| `POST` | `/api/jhola/parse` | Parse a free-text list into items |
| `POST` | `/api/jhola` | Start a basket job |
| `GET` | `/api/jhola/fees` | Current fee model (estimates and learned) |
| `GET` | `/api/accounts` | Connected apps |
| `POST` | `/api/accounts/:app/open` | Open the streamed login window |
| `GET` | `/api/accounts/window/:sid/frame` | Latest frame of the streamed window |
| `POST` | `/api/accounts/window/:sid/input` | Forward clicks and keys to the window |
| `POST` | `/api/accounts/window/:sid/close` | Close the window (optionally save the login) |
| `DELETE` | `/api/accounts/:app` | Disconnect an app |
| `POST` | `/api/order/parse` · `/api/order` | Parse or start an Order job |
| `POST` | `/api/price-hunt` | Start a Compare job |
| `POST` | `/api/form-fill` | Start a Forms job |
| `GET` | `/api/jobs` · `/api/jobs/:id` | List jobs or get one job |
| `GET` | `/api/jobs/:id/stream` | Live job updates (SSE) |
| `POST` | `/api/jobs/:id/approve` | Approve or reject a plan, cart diff or form |
| `POST` | `/api/jobs/:id/rerun` | Re-run a job |
| `DELETE` | `/api/jobs/:id` | Delete a job |
| `GET` `PUT` | `/api/settings` | Caretaker settings (PIN-protected) |
| `POST` | `/api/settings/unlock` | Check the caretaker PIN |
| `GET` `PUT` | `/api/profile` | Your profile |

</details>

---

<div align="center">

**Jhola** (झोला): the cloth bag every Indian household takes to the sabzi mandi.

</div>
