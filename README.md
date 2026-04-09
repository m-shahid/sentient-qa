# Sentient QA — AI-Driven No-Code Test Automation Framework

An AI-powered, no-code test automation framework where testers write plain-English instructions and the framework automatically locates UI elements, executes actions, and saves tests as reusable artifacts — without any selectors, page objects, or code.

---

## What It Does

Traditional test automation breaks when UIs change because it relies on brittle XPaths and CSS selectors. This framework eliminates that problem:

- **Write tests in plain English** — no locators, no code, no page objects
- **AI resolves elements automatically** — finds buttons, fields, and labels from screenshots and DOM structure
- **Smart caching** — after the first run, cached locators are reused instantly at zero cost
- **Runs at scale** — tiered resolution keeps costs low even at thousands of tests per day

```yaml
test: Login with valid credentials
platform: android
steps:
  - launch app: com.example.app
  - tap: "Allow" button on permission dialog
  - enter: "user@test.com" in "Email" field
  - enter: "Pass1234" in "Password" field
  - tap: "Sign In" button
  - verify: "Welcome" text is displayed
```

---

## How It Works — Tiered Resolution Pipeline

Instead of calling an expensive vision model for every element, the framework cascades through increasingly powerful resolution methods:

```
Step instruction
      │
      ▼
Tier 0: Cache Lookup         ~0ms  | $0.000  | ~75% of steps
      │ miss
      ▼
Tier 1: Heuristic Fuzzy Match ~50ms | $0.000  | ~10% of steps  (no AI)
      │ low confidence
      ▼
Tier 2: Page Source + Text LLM ~300ms | ~$0.001 | ~12% of steps
      │ fail
      ▼
Tier 3: Screenshot + Vision LLM ~4s  | ~$0.020  | ~2.5% of steps
      │ fail
      ▼
Tier 4: Flag for Human Review
```

**Result:** At ~3K UI tests/day (~24K steps), the framework costs roughly **~$15/day** — compared to **~$480/day** for a pure vision approach.

---

## Repository Structure

This project follows a **hybrid multi-repo architecture**. Each repo has a single responsibility:

| Repository | Language | Role |
|---|---|---|
| [`sentient-qa`](https://github.com/m-shahid/sentient-qa) | Python / FastAPI | Tiered resolution pipeline, caching, heuristics, LLM providers, test case persistence |
| `sentient-qa-jvm` *(planned)* | Java | Appium & Selenium driver adapters, TestNG & JUnit runners |
| `sentient-qa-js` *(planned)* | TypeScript | Playwright & WebdriverIO driver adapters, Jest & Playwright Test runners |
| `sentient-qa-python` *(planned)* | Python | Selenium & Appium-Python adapters, pytest runner |

> Stack repos are thin clients. All AI logic, caching, and heuristics live exclusively in `sentient-qa`.

---

## Architecture

Full architecture documentation is at [`docs/architecture.md`](docs/architecture.md), covering:

- System component diagram
- Tiered resolution pipeline details
- Two-phase resolution protocol (page source first, screenshot only if needed)
- Cache strategy (key design, invalidation policies, cache promotion)
- Plugin interface contracts
- Deployment topology (local Docker sidecar + CI/CD)
- API contract summary (OpenAPI)
- Cost & performance projections

---

## Quick Start (Local Dev)

> Prerequisites: Docker, Java 17+ or Node 18+ depending on which stack you're running.

**1. Start the Core Service**

```bash
git clone https://github.com/m-shahid/sentient-qa
cd sentient-qa
cp .env.example .env          # add your LLM API key
docker-compose up -d
# Core Service running at http://localhost:8000
```

**2. Run a test (JVM stack)**

```bash
git clone https://github.com/m-shahid/sentient-qa-jvm
cd sentient-qa-jvm
mvn test -Dtest=LoginTest
```

**3. Run a test (JS stack)**

```bash
git clone https://github.com/m-shahid/sentient-qa-js
cd sentient-qa-js
npm install
npx playwright test login.spec.ts
```

---

## Platform Support

| Platform | Driver | Runner | Status |
|---|---|---|---|
| Android (mobile) | Appium | TestNG / JUnit5 | Planned |
| iOS (mobile) | Appium | TestNG / JUnit5 | Planned |
| Web | Playwright | Playwright Test / Jest | Planned |
| Web | Selenium | TestNG / JUnit5 | Planned |
| Web | WebdriverIO | Jest | Future |
| Desktop | WinAppDriver / Pywinauto | — | Future |

---

## Key Design Decisions

**Why Python for the Core Service?**
Fast iteration, rich AI/ML ecosystem, and direct Anthropic/OpenAI SDK support. FastAPI provides automatic OpenAPI spec generation used by all stack clients.

**Why multiple repos instead of a monorepo?**
Stack repos (JVM, JS) need to version and release independently. Teams using only Appium shouldn't pull Playwright dependencies. The OpenAPI contract is the only coupling point.

**Why two-phase resolution (page source before screenshot)?**
Screenshots are large and expensive to transmit. ~97.5% of steps resolve using only page source (Tiers 0-2). Screenshots are requested only when the cheaper tiers fail, saving bandwidth and reducing Tier 3 latency.

**Why not just self-healing locators?**
Self-healing still requires initial locator setup and only repairs existing locators. This framework never requires locators to be written — the AI resolves elements from natural-language descriptions from the start.

---

## Plugin Interfaces

Every layer is swappable via configuration:

| Interface | What it swaps |
|---|---|
| `IElementResolver` | LLM provider (Claude → GPT-4o → Gemini → local model) |
| `ILocatorCache` | Cache backend (SQLite → Redis → custom) |
| `IHeuristicMatcher` | Fuzzy matching strategy |
| `IDriverAdapter` | Test driver (Appium → Playwright → Selenium → WebdriverIO) |
| `IActionHandler` | Action implementation (tap, enter, swipe, verify, long press, …) |
| `ITestRunner` | Test runner (TestNG → JUnit → Jest → pytest) |
| `INotifier` | Tier 4 alert channel (Slack → email → webhook) |

---

## Deliverables Roadmap

| # | Deliverable | Status |
|---|---|---|
| 1 | Architecture Document & Diagrams | ✅ Done |
| 2 | Tiered Resolution Flow Diagram | Pending |
| 3 | Cache Strategy Detail | Pending |
| 4 | Heuristic Matcher Design | Pending |
| 5 | AI Prompt Templates | Pending |
| 6 | Plugin Interface Contracts | Pending |
| 7 | Data Models (OpenAPI + language classes) | Pending |
| 8 | Inter-Repo Dependency Management | Pending |
| 9 | Sample E2E Walkthrough (Login test) | Pending |
| 10 | Extension Guide | Pending |
| 11 | CI/CD & Local Dev Setup | Pending |
| 12 | Cost & Performance Analysis | Pending |
| 13 | Trade-offs & Limitations | Pending |

---

## Tech Stack

**Core Service (sentient-qa)**
- Python 3.11+, FastAPI, Pydantic v2
- SQLite (default) / Redis (team/CI)
- Jinja2 prompt templates
- Anthropic Claude API (Haiku for text, Sonnet for vision)
- OpenAI GPT-4o-mini / GPT-4o (alternative providers)

**JVM Stack (sentient-qa-jvm)**
- Java 17+, Maven
- Appium Java Client, Selenium WebDriver
- TestNG, JUnit 5
- openapi-generator-maven-plugin (client generation)

**JS Stack (sentient-qa-js)**
- TypeScript, Node 18+, npm / pnpm
- Playwright
- Jest, Playwright Test
- openapi-typescript (client generation)

---