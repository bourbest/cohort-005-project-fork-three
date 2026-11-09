# Notification vendor landscape

Research for [#3](https://github.com/bourbest/cohort-005-project-fork-three/issues/3) (map #1, Notifications for Cadence). Sources were read on **2026-11-09**. Pricing pages carry no effective date, except Courier's ("published Nov 7, 2026"). Re-check them before committing.

Cadence constraints: React Router app, Drizzle, a single SQLite file, local-only with a DevUI user switcher, no background scheduler. Channels are an in-app feed and email. Audiences are Students and Instructors.

## Comparison

| | Knock | Novu (Cloud) | Courier | MagicBell | SuprSend |
|---|---|---|---|---|---|
| Free tier | 10k messages/mo, unlimited workflows and channels [1] | 10k workflow runs/mo, 20 workflows, 2 envs, 24h activity retention [3] | 10k sends/mo ("Developer") [6] | 1k deliveries/mo, 1 project [9] | 10k notifications/mo; in-app feed SuprSend-branded [12] |
| First paid tier | $250/mo (50k) [1] | Pro $30/mo (30k runs) [3] | $0.005/send, pay as you go [6] | $249/mo (50k) [9] | $100/mo Essentials; digest and preferences need Business at $250/mo [12] |
| React feed | `@knocklabs/react`: `KnockProvider`, `NotificationFeedPopover`, `NotificationIconButton` [2] | `@novu/react` `<Inbox/>` with Bell, Notifications, Preferences; React 18+ [4] | `@trycourier/courier-react` v9, `CourierInbox` [7] | `@magicbell/react`: `FloatingInbox`, `Inbox`, `UserPreferences`; server-signed user JWT [10] | `@suprsend/react` (the old `react-inbox` is end of support); signed JWT [13] |
| Email | BYO provider via channels | BYO provider | BYO provider | SES, Mailgun, SendGrid, SMTP [11] | BYO provider, billed separately [12] |
| Delay + cancel (inactivity nudge) | Fixed, dynamic, or relative delay up to 365d. A cancellation API with a key cancels a run [5] | Delay capped at **1 day on Free** (7d Pro, 30d Team) [4b]. **Free cannot do a 7-day nudge.** | Journeys delay by duration or until a timestamp. Cancel by token or run ID [8] | `wait` (seconds), `pause`, `abort` [11]. GA status not stated | Fixed, dynamic, or relative delay [14]. Cancellation not documented on that page |
| Digest/batch | Included on Free [1] | Digest window 1 day on Free [4c] | "Digest & batching" exists; plan unclear [6] | Not documented [11] | **Business plan only** [12] |
| Per-user preferences | Included on Free [1] | Built into the Inbox component [4] | Preference center exists; Free plan inclusion not stated [6][7] | `UserPreferences` component [10] | **Business plan only** [12] |
| Local dev/test | CLI keeps workflows as code (`pull`/`push`/`validate`), environments and branches, `workflow run --sandbox-mode --skip-delay`. **No local emulator** [15] | Self-hosted community edition needs MongoDB, Redis, and object storage, so it is heavy [16]. Framework syncs TS workflows to Cloud [17] | CLI. A test key "works with no provider to connect" [18] | CLI, test toolkit (thin docs) | Not evaluated in depth |

Every vendor is cloud-only for Cadence's purposes. None runs against the local SQLite file, and every DevUI user would have to be identified to the vendor (by user ID and a signed token).

## The inactivity nudge does not need a vendor

A rule like "no Watched time in 7 days" is a *negative* condition. A vendor only handles it if Cadence re-triggers a delayed workflow on every watch event and cancels the previous run (Knock or Courier, using `userId` as the cancellation key). The alternative is that Cadence detects inactivity itself, which needs a periodic job. Either way, Cadence owns part of the logic. A daily job that queries `max(watched_at) < now - 7d` against SQLite is simpler and testable locally. With that in place, a vendor's delay feature isn't needed.

## Minimal in-house build to match

1. `notifications` table: id, recipientUserId, type, payload JSON, createdAt, readAt, seenAt. Add `emailSentAt` if email goes through it.
2. `notification_preferences` table: userId, type, channel (in_app/email), enabled. Defaults are defined in code.
3. A `notify(userId, type, payload)` function that checks preferences, inserts the in-app row, and sends email through one provider (Resend, Postmark, or SMTP). In dev it uses a console/file transport or a local mail catcher such as Mailpit.
4. React Router loader/action for the feed: list, unread count, mark read. A bell popover component. Polling or revalidation on navigation is enough, with no websockets.
5. A scheduler for the nudge and for digests: a `npm run jobs:daily` script, or a `node-cron`/`setInterval` in the server process. Digests group unread rows per user per window. Store `lastNudgedAt` so nudges are idempotent.
6. Tests: all of it runs in Vitest against SQLite, and DevUI user switching works without changes.

Rough size: 2 tables, 1 service module, 1 route, 1 component, 1 job script. A few days of work. What's lost compared with vendors: a delivery log UI, non-dev template editing, multi-channel fan-out, and real-time push.

## Shortlist

1. **In-house (recommended for Cadence now).** It fits a local single-SQLite app, has no external accounts or keys, and works with the DevUI switcher and Vitest. Cadence needs a scheduler either way. Tradeoff: we build and own the feed UI, preferences, and digest logic.
2. **Knock.** It has the strongest free tier: digests, preferences, and scheduled workflows with long, cancellable delays, plus workflows-as-code via CLI. Tradeoff: cloud-only and no local emulator. The jump from free to paid is $250/mo.
3. **Courier.** It has a free tier with journeys, cancellation tokens, and test keys that need no provider. Tradeoff: Free inclusion of Inbox, preferences, and digest isn't confirmed on the pricing page. Docs are split between legacy Automations and Journeys.

Dropped: **Novu**, because the Free tier caps delay and digest at 1 day and self-hosting needs Mongo and Redis. **SuprSend**, because digests and preferences are paywalled at $250/mo. **MagicBell**, because of its 1k/mo free tier and thin docs on digests.

## Sources

1. Knock pricing: https://knock.app/pricing
2. Knock React feed: https://docs.knock.app/in-app-ui/react/feed
3. Novu pricing: https://novu.co/pricing
4. Novu Inbox: https://docs.novu.co/platform/inbox/overview. 4b. Delay step: https://docs.novu.co/platform/workflow/add-and-configure-steps/action-steps/delay. 4c. Limits: https://docs.novu.co/platform/developer/limits
5. Knock delay function: https://docs.knock.app/designing-workflows/delay-function
6. Courier pricing: https://www.courier.com/pricing
7. Courier Inbox: https://www.courier.com/docs/platform/inbox/inbox-overview
8. Courier cancel: https://www.courier.com/docs/platform/automations/cancel and https://courier.com/docs/automations/steps
9. MagicBell pricing: https://www.magicbell.com/pricing
10. MagicBell React: https://www.magicbell.com/docs/libraries/magicbell-react
11. MagicBell workflows and channels: https://www.magicbell.com/workflows and https://www.magicbell.com/docs
12. SuprSend pricing: https://www.suprsend.com/pricing
13. SuprSend React inbox: https://docs.suprsend.com/docs/inbox-react
14. SuprSend delay: https://docs.suprsend.com/docs/delay
15. Knock CLI: https://docs.knock.app/cli
16. Novu self-hosting: https://docs.novu.co/community/self-hosting-novu/overview
17. Novu Framework: https://docs.novu.co/framework/overview
18. Courier docs index (test key note): https://www.courier.com/docs/llms.txt
