# multi-creator-chatbot

An AI-driven fan engagement and monetisation system for OnlyFans creators, built on Spring Boot. It listens to OnlyFans webhooks, holds natural conversations with fans using Claude, and escalates them through a scripted content ladder toward pay-per-view (PPV) purchases, tips, and custom orders.

The system is multi-tenant: a single deployment serves many creators, each with their own API credentials, voice, content vault, and analytics.

---

## Stack

| Component | Choice |
| --- | --- |
| Runtime | Java 17, Spring Boot 4.0.2 |
| Persistence | PostgreSQL via Spring Data JPA (`ddl-auto=update`) |
| AI | Anthropic Claude (`claude-sonnet-4-5-20250929`) |
| Integrations | OnlyFans API (onlyfansapi.com), GoHighLevel, Instagram |
| Deploy | Docker (multi-stage, Temurin Alpine), Railway |
| Port | 8081 |

---

## How a message flows

```
OnlyFans webhook
      |
      v
[ BOT_ENABLED gate ]  ---- false ----> ack only, no processing
      |
      v
resolve Creator by account_id
      |
      v
async thread ---> filter: self-sent? spam?
      |
      v
MessageBatchingService      collects rapid-fire messages into one turn
      |
      v
ScriptEngineService         analyse state, detect preferences, pick script
      |
      v
AnthropicService            generate reply in creator voice
      |
      v
MessageSanitizerService     strip OnlyFans restricted words
MessageSplittingService     break into human-sized messages
ResponseTimingService       typing indicators + natural delay
      |
      v
OnlyFansApiService          send, persist, update state
```

Fans move through eight conversation states: `CASUAL`, `PLAYFUL`, `FLIRTY`, `SUGGESTIVE`, `INTIMATE`, `EXPLICIT`, `SEXTING_SESSION`, `SELLING`.

---

## Webhook events handled

| Event | Behaviour |
| --- | --- |
| `subscriptions.new` | Create fan record, open conversation, send AI welcome |
| `messages.received` | Batch, analyse, generate and send reply |
| `transactions.new` | Record spend, update fan value tier |
| `messages.ppv.unlocked` | Mark PPV purchased, trigger post-purchase ladder |
| `tips.received` | Log tip, may advance a pending custom request |
| `subscriptions.renewed` | Refresh subscription state |

---

## REST endpoints

### Webhooks
| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/api/webhook/onlyfans` | Main OnlyFans event ingest |
| GET | `/api/webhook/onlyfans/health` | Health check |
| POST | `/webhook/instagram` | Instagram DM ingest |
| GET | `/webhook/health` | Health check |

### Creators
| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/api/creators/onboard` | Register a creator (name, OF URL, API key, account id, tone, tracking code) |
| GET | `/api/creators` | List all creators |
| GET | `/api/creators/{creatorId}` | Fetch one creator |
| PUT | `/api/creators/{creatorId}` | Update creator settings |
| DELETE | `/api/creators/{creatorId}` | Remove a creator |

A browser onboarding form is served at `/creator-onboarding.html`.

### Analytics
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/analytics/dashboard/{creatorId}` | Aggregate performance dashboard |
| GET | `/api/analytics/category/{creatorId}/{scriptCategory}` | Per-category script breakdown |
| GET | `/api/analytics/recent/{creatorId}` | Recent script performance |
| GET | `/api/analytics/recommendations/{creatorId}` | Suggested script and pricing changes |

### Content vault
| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/vault/media` | List synced vault media |
| POST | `/api/vault/download` | Pull all vault content from OnlyFans |
| GET | `/api/vault/media/{mediaId}` | Fetch one media item |
| GET | `/api/vault/debug/lists` | Inspect raw vault list sync |

---

## Services

### Conversation core
| Service | Responsibility |
| --- | --- |
| `OnlyFansChatbotService` | Main orchestrator. Handles new subscriptions and incoming messages end to end |
| `InstagramChatbotService` | Equivalent flow for Instagram DMs |
| `ScriptEngineService` | Analyses conversation state, detects fan preferences, selects script category, advances framework stages |
| `ScriptService` | Manages escalating content offers L1 to L7 with topic matching and explicitness gating |
| `ConversationService` | Creates and retrieves conversation threads |
| `MessageService` | Persists inbound and outbound messages |
| `FanService` | Fan records, state, spend totals |
| `CreatorService` | Multi-tenant creator resolution and credentials |

### AI and message shaping
| Service | Responsibility |
| --- | --- |
| `AnthropicService` | All Claude calls: replies, state analysis, intent classification, boolean gating prompts |
| `MessageSanitizerService` | Replaces OnlyFans restricted words that trigger 400 errors |
| `MessageSplittingService` | Splits one AI reply into several natural messages |
| `MessageBatchingService` | Groups rapid consecutive fan messages so the bot answers once |
| `ResponseTimingService` | Human-like delays, typing indicators, delayed PPV sends, cancellation of stale replies |

### Monetisation
| Service | Responsibility |
| --- | --- |
| `PPVService` | Builds and sends PPV offers, tracks every offer |
| `PPVLadderService` | Post-purchase upsell ladder |
| `PPVPurchaseService` | Records unlocks and purchase outcomes |
| `PPVResponseService` | Interprets fan replies to an offer (accept, decline, negotiate) |
| `PricingLadderService` | Category-aware price ladders, start-price selection, tier rounding |
| `PeakInterestDetectionService` | Detects the moment a fan is most likely to buy |
| `TipPromptService` | Decides when to ask for a tip and sends the prompt |
| `CustomRequestService` | Custom content orders: intake, duration parsing, quoting, approval, completion |
| `OrganicContentSuggestionService` | Surfaces content that fits the conversation naturally |

### Content
| Service | Responsibility |
| --- | --- |
| `ContentVaultService` | Syncs OnlyFans vault lists, selects media by price tier, level, or fan interests, excludes already-sent items |
| `ContentCategoryResolver` | Maps vault folder names to content categories |
| `VaultDownloadService` | Bulk vault media download |
| `FanInterestService` | Tracks per-fan interests to personalise bundles |

### Retention and ops
| Service | Responsibility |
| --- | --- |
| `FollowUpService` | Follow-ups on unanswered messages and re-engagement passes |
| `ReEngagementService` | Daily sweep of inactive fans |
| `ScriptAnalyticsService` | Records script usage and conversion for analytics |
| `ErrorLogService` | Structured error persistence |
| `GoHighLevelService` | GoHighLevel CRM sync |
| `OnlyFansApiService` | Typed client for the OnlyFans API |

---

## Scheduled jobs

| Job | Schedule | Purpose |
| --- | --- | --- |
| `FollowUpService.processFollowUps` | every 60s | Chase conversations the fan left hanging |
| `FollowUpService.processReengagement` | every 30 min | Nudge cooling conversations |
| `ReEngagementService.processInactiveFans` | daily 10:00 | Re-open dormant fans |

All scheduled jobs no-op while `BOT_ENABLED` is false.

---

## Pricing model

Every price ends in `.95`. Ladders are per category:

| Category | Tiers |
| --- | --- |
| Solo / Couple | 9.95, 29.95, 49.95, 99.95, 149.95, 199.95 |
| Full sextape | 99.95, 199.95 |
| Bundle | 19.95, 29.95, 49.95 |
| Custom | 49.95 minimum |
| Teaser | free or 9.95 |
| GFE / rapport | 9.95, 19.95, 29.95 |

Start price is chosen from fan spend history, reply recency, whether a free teaser was already sent, purchases in the current session, and whether the fan is being re-engaged after going cold. A decline triggers a two hour cooldown before any new offer.

---

## Script system

Loaded from `src/main/resources/scripts/`.

**Categories:** `WELCOME`, `TEASE`, `MENU_TEASE`, `PPV_OFFER`, `CUSTOM_OFFER`, `ENGAGEMENT`, `RE_ENGAGEMENT`, `OBJECTION_HANDLERS`, `SEXTING_SESSION`, `GFE_SCRIPTS`, `DOMINANT_ROLEPLAY`, `SUBMISSIVE_ROLEPLAY`, `FANTASY_FULFILLMENT`, `RELATIONSHIP_BUILDING`, `ANTICIPATION_BUILDING`

**Frameworks:** `SELL` and `PARE`, staged per conversation.

Creator voice is tuned via `src/main/resources/voice-examples.txt`.

---

## Data model

`Creator`, `Fan`, `Conversation`, `ConversationState`, `Message`, `ContentVault`, `VaultMedia`, `ContentCategory`, `PPVOffer`, `FanPurchase`, `FanInterest`, `FanScriptProgress`, `CustomRequest`, `ScriptPerformance`, `ErrorLog`.

---

## Configuration

Set as environment variables. Nothing is hardcoded.

| Variable | Default | Purpose |
| --- | --- | --- |
| `BOT_ENABLED` | `false` | Master kill switch |
| `ANTHROPIC_API_KEY` | required | Claude access |
| `ONLYFANS_API_KEY` | required | OnlyFans API access |
| `ONLYFANS_ACCOUNT_ID` | required | Creator account id |
| `ONLYFANS_API_BASE_URL` | `https://app.onlyfansapi.com/api` | API base |
| `SPRING_DATASOURCE_URL` | local postgres | Database |
| `SPRING_DATASOURCE_USERNAME` | `postgres` | Database user |
| `SPRING_DATASOURCE_PASSWORD` | `postgres` | Database password |
| `GOHIGHLEVEL_API_TOKEN` | `none` | CRM sync, optional |

Tuning knobs in `application.properties`:

| Property | Default | Meaning |
| --- | --- | --- |
| `pricing.active-reply-minutes` | 10 | Window that counts a fan as an active replier |
| `pricing.session-hours` | 2 | Session window for multi-purchase escalation |
| `pricing.cold-days` | 2 | Inactivity before a fan counts as cold |
| `purchase-intent.ppv.delay-seconds` | 90 | Delay before sending PPV on buy intent |
| `purchase-intent.typing-indicator-count` | 3 | Typing pulses before that send |
| `script.force-all-fans` | false | Testing only: force script on every fan |
| `vault.sync.on-startup` | enabled | Set false when the OnlyFans API is unreachable |

### Kill switch

`BOT_ENABLED` defaults to **false**. While false, webhooks return `{"status":"bot_disabled"}` without processing and scheduled jobs do nothing. Set `BOT_ENABLED=true` to go live.

---

## Running

Local:

```bash
export ANTHROPIC_API_KEY=...
export ONLYFANS_API_KEY=...
export ONLYFANS_ACCOUNT_ID=...
export BOT_ENABLED=true

./mvnw spring-boot:run
```

Docker:

```bash
docker build -t multi-creator-chatbot .
docker run -p 8081:8081 --env-file .env multi-creator-chatbot
```

Point the OnlyFans webhook at `https://<host>/api/webhook/onlyfans`.

---

## Repository notes

Branches: `init-branch` is the default and the working branch; `main` mirrors it.

Fan conversation exports (`clients-history`, `csv.file`, any `*.csv`) are gitignored as personally identifiable information and must never be committed.
