# Notification delivery: build vs. buy

Research for [#3](https://github.com/adam-mrachek/cohort-005-project/issues/3), part of the notifications map [#2](https://github.com/adam-mrachek/cohort-005-project/issues/2). This note gives a short-list and trade-offs. It does **not** pick one. The choice belongs to #6 ("Build or buy, and which vendor").

Researched 2026-11-18 from vendor pricing pages and docs, MDN, WebKit and Postmark. Prices are list prices in USD and change often. Check them again before #6 is decided.

## Starting point in this repo

- **Stack:** React Router 8 with SSR (`react-router.config.ts`), a Node server (`react-router-serve`), and one SQLite file (`data.db`) opened through `better-sqlite3` and Drizzle in `app/db/index.ts`. There is no queue, worker or cron process today.
- **Event sources that exist:** `comments` (one level of replies, soft delete through `deletedAt`), `enrollments`, `lessonProgress` and `videoWatchEvents` (each has `userId`, `lessonId` and `createdAt`). The "no watch in 7 days" signal can come from `videoWatchEvents`.
- **Email already exists:** `app/lib/mailer.server.ts` sends transactional email through Postmark (the `outbound` stream) when `POSTMARK_SERVER_TOKEN` is set, and otherwise writes to `.outbox/`. Every option below can reuse it, either directly (build) or by connecting Postmark as the vendor's email provider. One-SQLite-file is recorded in ADR 0006.
- **Web push prerequisites missing:** `public/` has no web app manifest and no service worker. Every option needs both before push can work on iOS (see below).

## Web push facts that apply to every option

- The Push API needs an active service worker. The page calls `PushManager.subscribe()`, gets back a `PushSubscription` (an endpoint plus encryption keys), and the server sends to that endpoint. The service worker's `push` event then calls `showNotification()`. MDN marks the API "Baseline: Widely available" since March 2023. [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- MDN warns that the endpoint must be kept secret and that subscription endpoints need CSRF protection. [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- **iOS/iPadOS:** web push needs version 16.4 or later, and only works for a web app that has been **added to the Home Screen**. The manifest must set `display` to `standalone` or `fullscreen`, and the permission request must come from a direct user action such as a tap. No Apple Developer Program membership is needed. If you filter outbound traffic, allow `*.push.apple.com`. The Badging API also works for Home Screen apps. [WebKit blog](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)
- **What this means:** no vendor can get around the iOS Home Screen rule. In every option, Cadence has to ship the manifest and service worker and design the "add to Home Screen" step itself. Vendors only take over the server-side send and storing subscriptions.
- **Sending it ourselves:** the `web-push` npm library generates VAPID keys (`generateVAPIDKeys()`, "create these keys once"), signs and encrypts the payload, and supports `TTL` (default four weeks), `urgency` and `topic`. Failed sends reject with a `statusCode`. Clearing out expired subscriptions after a 404 or 410 is our job. [web-push-libs/web-push](https://github.com/web-push-libs/web-push)

## Email facts that apply to every option

- **None of the vendors send email themselves.** Knock, Novu, Courier and SuprSend all route email through a provider you connect, such as Postmark. MagicBell's email docs list SES, Mailgun, SendGrid, SMTP and Ping Email, but **not** Postmark (SMTP is the fallback). So buying adds to the email provider's bill rather than replacing it. [Knock Postmark](https://docs.knock.app/integrations/email/postmark), [Novu Postmark](https://docs.novu.co/platform/integrations/email/postmark), [MagicBell docs](https://www.magicbell.com/docs)
- **Postmark streams:** Postmark keeps transactional email (the "outbound" stream, e.g. "comment notifications") apart from broadcast email (e.g. "daily digests"), including separate IP ranges. Broadcast email **requires** an opt-out link, and Postmark says you manage your lists outside Postmark. [Postmark message streams](https://postmarkapp.com/message-streams)
  - What this means for Cadence: a reply to a comment fits the transactional stream. Digests, and very likely re-engagement nudges, fit the broadcast stream. Either way, unsubscribe handling has to be built or bought.
- By default, Knock sends to Postmark without a `MessageStream`, so email goes out on "outbound". You can override it per channel or per step with `{"MessageStream": "broadcasts"}`. [Knock Postmark](https://docs.knock.app/integrations/email/postmark)
- **Postmark price:** Basic is $15/mo, Pro $16.50/mo and Platform $18/mo, each starting at 10,000 emails, with overage of $1.80, $1.30 and $1.20 per 1,000. The free developer tier covers 100 emails a month with no overage. [Postmark pricing](https://postmarkapp.com/pricing)

## The scheduled nudge applies to every option

"If no watch in 7 days, send X, and cancel if they come back" depends on `videoWatchEvents`, which only lives in our SQLite file. There are two ways to do it:

1. **Re-arm on activity.** Each watch event cancels the pending run and triggers a new delayed one, keyed per Student and course. Knock (cancellation key, delays up to 365 days), Novu (`DELETE /v1/events/trigger/{transactionId}`, with the delay capped by plan) and Courier (cancel by `cancelation_token`, delays up to 93 days) all support this. It means an API call to the vendor on every watch event, or at least the first one each day.
2. **Detect it ourselves.** A periodic job queries SQLite for inactive Students and triggers a notification. This needs a scheduler in our process (or an external cron hitting an endpoint), whether we buy or build.

Sources: [Knock delay](https://docs.knock.app/designing-workflows/delay-function), [Novu cancel](https://docs.novu.co/api-reference/events/cancel-triggered-event), [Novu limits](https://docs.novu.co/platform/developer/limits), [Courier cancel](https://www.courier.com/docs/platform/automations/cancel)

## Short-list

### A. Build on our own stack

- **In-app:** a `notifications` table in SQLite (recipient, type, payload, `readAt`, `createdAt`), a bell made with shadcn, and a loader and action for the list and mark-as-read. It is fully styleable. For "real-time", the options are refreshing on navigation, polling, or SSE from the Node server.
- **Push:** our own manifest, service worker, `push_subscriptions` table and the `web-push` library. Browser push services cost nothing.
- **Email:** the existing Postmark mailer (`app/lib/mailer.server.ts`), extended with a broadcast stream for nudges and digests.
- **Delays, nudges and digests:** we build them. We need a durable job runner. With one SQLite file and one Node process, the simplest form is a `jobs` table (`runAt`, `status`) polled by an in-process interval or a separate `tsx` worker against the same file. WAL mode is already on.
- **Preferences:** our own table keyed by notification type and channel, plus a `/settings` page.
- **Run ourselves:** the job runner or cron, retries, removing dead push subscriptions, unsubscribe links, and observability.
- **Cost:** Postmark only. At about 10 emails per MAU per month, list price is roughly $15 to $180 a month between 1k and 10k MAU. 100k MAU means about 1M emails, around $1.3k a month on Pro at list price or a custom quote.
- **Lock-in and residency:** none. All data stays in our SQLite file.
- **Main risk:** engineering time and owning reliability. Digests, preference rules and nudges are the hard parts.

### B. Knock (cloud)

- **In-app:** `@knocklabs/react` has ready-made components plus headless hooks (`useNotifications`, `useNotificationStore` with `unread_count` and `unseen_count`) backed by the Feed API and Knock's real-time service. The headless docs show `markAsSeen` and `markAllAsSeen`. Mark-as-read is in the Messages API. [Knock headless feed](https://docs.knock.app/in-app-ui/react/headless/feed)
- **Push:** the providers listed are APNs, FCM, Expo, OneSignal and Amazon SNS. **There is no direct VAPID web push**, so browser push would go through FCM (or OneSignal), which adds a Firebase dependency. [Knock push](https://docs.knock.app/integrations/push/overview)
- **Email:** has a Postmark integration and supports `MessageStream` overrides.
- **Delays and cancellation:** delays of up to 365 days, cancelled with a cancellation key. This fits the re-arm pattern well.
- **Preferences, batching:** included on every plan, including Free. [Knock pricing](https://knock.app/pricing)
- **Pricing:** billed per message **per recipient per channel**. In-app plus email counts as 2. Messages suppressed by preferences, and failed messages, are not billed. Free: 10k messages. Starter: $250/mo with 50k included, then $0.005 each. Enterprise is custom. Log retention is 30 days, or 90 on Enterprise.
- **Residency:** the pricing page lists no EU region option.

### C. Novu (cloud, or self-hosted community edition)

- **In-app:** the `@novu/react` `<Inbox />` and `<Preferences />` components, plus headless hooks (`useNotifications`, `useCounts`, `useNovu`) with read, unread, archive and live WebSocket updates. [Novu useNotifications](https://docs.novu.co/platform/sdks/react/hooks/use-notifications)
- **Push:** FCM, APNs, Expo, OneSignal, Pusher Beams, Pushpad and a push webhook. No VAPID web push is documented. Web goes through FCM or Pushpad, or through the Push Webhook to our own `web-push` sender. [Novu push](https://docs.novu.co/platform/integrations/push)
- **Email:** has a Postmark integration.
- **Delays and digests:** these are **capped by plan**: Free 1 day, Pro 7 days, Team 30 days. A 7-day nudge needs Pro at least. Runs can be cancelled by `transactionId`. Free also limits you to 5 workflows. [Novu limits](https://docs.novu.co/platform/developer/limits)
- **Pricing (cloud):** billed per **workflow run**, meaning one trigger to one subscriber, whatever the channels. [Novu billing](https://docs.novu.co/platform/account/billing) Free: 10k runs. Pro: from $30/mo with 30k+ runs. Team: from $250/mo with 250k+ runs. Extra runs on Pro and Team cost $1.20 per 1k. US and EU data residency are available even on Free. [Novu pricing](https://novu.co/pricing)
- **Self-hosted community edition:** the core is MIT, and the `enterprise/` folder is under a commercial licence. It needs API, Worker, WS and Dashboard services, **MongoDB, Redis (with AOF) and S3**. That means at minimum one 4 vCPU / 8 GB VM. You get one team member, no snooze and no branding removal. Digests and preferences are included. **This is a large ops footprint next to a single SQLite file.** [Novu self-hosting](https://docs.novu.co/community/self-hosting-novu/overview), [Novu comparison](https://docs.novu.co/community/self-hosted-and-novu-cloud), [Novu repo](https://github.com/novuhq/novu)

### D. MagicBell

- **In-app:** an inbox plus React components. The docs we read did not confirm headless hooks.
- **Push:** **native VAPID web push**, the only vendor here that has it. You store browser `PushSubscription`s through its API. iOS and service worker guidance is not covered on the channel page. [MagicBell web push](https://www.magicbell.com/docs/channels/web_push)
- **Email:** **Postmark is not listed** (only SES, Mailgun, SendGrid, SMTP and Ping Email).
- **Workflows:** `wait` (delay), `abort` and conditions. **Digests or batching were not found** in the docs. [MagicBell workflows](https://www.magicbell.com/workflows)
- **Preferences:** by category and channel, through `/channels/user_preferences`.
- **Pricing:** billed per delivery. Builder: $0 with 1k deliveries. Startup: $249/mo with 50k, then $0.0025 each. [MagicBell pricing](https://www.magicbell.com/pricing)

### E. Courier

- **In-app:** `<CourierInbox />` from `@trycourier/courier-react`, themeable or with custom views, real-time, with read and archive. Low-level APIs exist for a fully custom UI. [Courier Inbox](https://www.courier.com/docs/platform/inbox/inbox-overview)
- **Push:** browser push goes through FCM, OneSignal, Pusher Beams or NowPush. There is no direct VAPID. [Courier push](https://www.courier.com/docs/external-integrations/push/intro-to-push)
- **Delays and cancellation:** Journeys have delays of up to 93 days and can be cancelled by `cancelation_token` or `run_id`.
- **Digests and batching:** the pricing page lists them only on **Enterprise**. [Courier pricing](https://www.courier.com/pricing)
- **Pricing:** Developer: free for 10k sends. Business: $0.005 for **every** send, including the first 10k. EU data residency comes with Business.

### F. SuprSend

- Includes an embeddable inbox and a headless inbox, and web push on every plan. [SuprSend pricing](https://www.suprsend.com/pricing)
- **Preferences and digests are gated to Business**, at $250/mo with 50k notifications, then $5 per 1k. Essentials is $100/mo with 50k, then $2 per 1k, but has no preferences or digests. Free covers 10k notifications.
- Self-hosting, BYOC and custom data region are Enterprise only.
- We did not go deeper on SuprSend (push mechanism, Postmark support). Do that if it makes the final round.

## Rough monthly cost

Assumptions: about 10 notification events per MAU per month. Each one goes to in-app and email, and about half also go to push, so about 25 per-channel messages per MAU. Postmark (or another email provider) is **extra** in every vendor row.

| Option | Billing unit | 1k MAU | 10k MAU | 100k MAU |
| --- | --- | --- | --- | --- |
| A. Build | Postmark emails (~10/MAU) | ~$15 | ~$135–180 | ~$1.3k or custom quote |
| B. Knock | per channel message (25k / 250k / 2.5M) | $250 (Starter) | ~$1,250 | ~$12.5k list (Enterprise quote) |
| C. Novu Cloud | workflow run (10k / 100k / 1M) | $0 Free, but 1-day delay cap, so Pro ~$30 for nudges | ~$114 (Pro) | ~$1,150 (Team) |
| C'. Novu self-host | infrastructure | VM + MongoDB + Redis + S3 | same | larger cluster |
| D. MagicBell | delivery (assumed per channel) | $249 | ~$750 | ~$6.4k |
| E. Courier | send (assumed per channel) | ~$125 | ~$1,250 | ~$12.5k (Enterprise needed for digests) |
| F. SuprSend | notification (assumed per channel), Business for preferences/digests | $250 | ~$1,250 | ~$12.5k |

Digests lower the message count for every per-message vendor. Knock does not say whether one batched feed item counts as one message or several.

## Trade-offs in one view

| | In-app headless React | Web push (VAPID) | Postmark | 7-day nudge + cancel | Preferences | Digests | We run |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Build | ours | ours (`web-push`) | yes | ours (job table + poller) | ours | ours | everything |
| Knock | yes | via FCM/OneSignal | yes | yes (365 d) | all plans | all plans | trigger calls, SW |
| Novu Cloud | yes | via FCM/Pushpad/webhook | yes | Pro+ (7 d cap) / Team | yes | plan-capped window | trigger calls, SW |
| Novu self-host | yes | same | yes | yes | yes | yes | Mongo, Redis, S3, 4 services |
| MagicBell | components (headless unclear) | **native** | **no** | wait + abort | yes | not found | trigger calls, SW |
| Courier | yes | via FCM/OneSignal/etc. | yes (provider) | yes (93 d) | yes | Enterprise only | trigger calls, SW |
| SuprSend | yes | yes | not checked | not checked | Business+ | Business+ | trigger calls, SW |

## Gaps no option fills well

- **iOS web push is a product problem.** Every option requires Home Screen install, a manifest and a service worker that Cadence owns.
- **Nudges depend on our data.** Watch activity lives in SQLite. Buying gives us delays and cancellation, but we still trigger on watch events or run our own inactivity job.
- **There is no VAPID web push next to Postmark.** Only MagicBell has native VAPID web push, and it lacks Postmark. Every vendor with Postmark routes browser push through FCM or OneSignal, which adds a third party. Novu's Push Webhook is the exception: it can hand off to our own `web-push` sender.
- **Novu self-hosted conflicts with ADR 0006 ("one SQLite file through Drizzle")** because it needs MongoDB, Redis and S3.
