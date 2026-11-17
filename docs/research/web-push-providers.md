# Web push provider options

Research for [#10](https://github.com/bognico/software-factories-for-real-engineers/issues/10), part of map [#5](https://github.com/bognico/software-factories-for-real-engineers/issues/5). Gathered 2026-11-17 from primary sources (provider docs and pricing pages, WebKit, MDN, GitHub, the npm registry). Prices change, so check them again before deciding.

**Context.** Cadence is a web-only React Router app on a Node server with one SQLite file through Drizzle ([ADR 0006](../adr/0006-one-sqlite-file-through-drizzle.md)). v1 needs an in-app notification inbox, browser web push, per-user preferences, and maybe scheduled engagement nudges.

This note lists facts and trade-offs. It does not pick a provider.

---

## 1. The platform facts every option inherits

Every provider below ends up on the same browser primitives: a service worker, the Push API, the Notifications API, and VAPID. Whichever one we choose, these constraints apply.

- **A service worker is required to receive push.** "An app must have an active service worker to receive push messages", and it subscribes through `PushManager.subscribe()`. Messages arrive at the worker's `push` handler. ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API))
- **Subscriptions can expire or change.** `pushsubscriptionchange` fires when a subscription "has been invalidated or is about to be". The app has to resubscribe and send the new subscription to the server. ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Push_API))
- **Baseline status.** The Push API is Baseline "Widely available", working across browsers since March 2023. ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Push_API))
- **HTTPS is required.** Push needs a secure context, with an exception for `localhost` during development. OneSignal also says "Web push does not work on HTTP or in incognito/private modes." ([OneSignal web quickstart](https://documentation.onesignal.com/docs/web-push-quickstart), [FCM web client](https://firebase.google.com/docs/cloud-messaging/js/client))

### iOS / iPadOS / Safari (the big constraint)

- **iOS 16.4+ only, and only for Home Screen web apps.** "With iOS and iPadOS 16.4, we are adding support for Web Push to Home Screen web apps." A regular Safari tab on iPhone cannot subscribe. ([WebKit: Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/))
  - For Cadence this means iPhone users have to "Add to Home Screen" before they can get push, and the site needs a web app manifest ([OneSignal quickstart](https://documentation.onesignal.com/docs/web-push-quickstart)).
  - The in-app inbox is the only notification surface that reaches every iOS user.
- **A user gesture is required.** Permission has to be requested "in response to direct user interaction", such as tapping a subscribe button. A permission prompt on page load does not work. ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/), [WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/))
- **Silent push is not allowed.** Every push must show a visible notification: "The Web Push API is not an invitation for silent background runtime", and "Violations of the userVisibleOnly promise will result in a push subscription being revoked." Push cannot be used for silent background sync. ([WebKit 12945](https://webkit.org/blog/12945/meet-web-push/))
- **Apple's endpoints.** Safari uses standard VAPID web push through APNs. Servers that allowlist endpoints must allow `*.push.apple.com`. No Apple Developer Program membership is needed. ([WebKit 12945](https://webkit.org/blog/12945/meet-web-push/), [WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/))
  - OneSignal confirms that Safari on macOS 13+ and iOS 16.4+ "uses VAPID". The legacy .p12 certificate applies only to macOS 12 and earlier. ([OneSignal quickstart](https://documentation.onesignal.com/docs/web-push-quickstart))
- **Badging API.** It is supported on iOS Home Screen apps, so the app can set an unread count on the icon. That fits an inbox unread count. ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/))
- **Focus modes** can filter notifications, and users manage permission per web app ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)).
- **Declarative Web Push** (iOS/iPadOS 18.4, macOS 15.5) can display notifications "without requiring an installed service worker". The payload is a standard JSON format, and browsers without support fall back to the service worker. It is Apple-only for now, and the article does not say whether the Home Screen requirement is lifted. ([WebKit: Meet Declarative Web Push](https://webkit.org/blog/16535/meet-declarative-web-push/))

### Browser matrix for self-sent VAPID push

From the `web-push` README ([github.com/web-push-libs/web-push](https://github.com/web-push-libs/web-push)):

| Browser | Push / payload / VAPID |
| --- | --- |
| Chrome | 42+ / 50+ / 52+ |
| Edge | 17+, all features |
| Firefox | 44+ / 44+ / 46+ |
| Safari | 16+ (macOS 13+) |
| Samsung Internet | push and payload supported, no VAPID |
| Opera | Android only, no VAPID |

Firefox also limits how many push messages that don't show a notification a site can receive. Messages that do show one are exempt. ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Push_API))

---

## 2. Candidates

### A. Self-hosted Web Push: VAPID keys plus the `web-push` npm library

- **What it is.** A Node library that sends through the Web Push Protocol and encrypts payloads per the Message Encryption spec. ([README](https://github.com/web-push-libs/web-push))
- **npm metadata.**
  - Version `4.0.0`, license `MPL-2.0`, `engines.node >= 16`, last modified 2026-11-06.
  - Four dependencies: `jws`, `asn1.js`, `http_ece`, `https-proxy-agent`.
  - Source: `npm view web-push`.
- **Keys.** `webpush.generateVAPIDKeys()` or the `web-push generate-vapid-keys` CLI. Generate the keys once, store them, and reuse them. The public key becomes `applicationServerKey` in the browser. ([README](https://github.com/web-push-libs/web-push))
- **Sending.** `sendNotification(subscription, payload, options)`.
  - The subscription is `{endpoint, keys: {p256dh, auth}}`, which is exactly what `PushSubscription.toJSON()` returns.
  - Options include `TTL` (default four weeks), `urgency`, and `topic`.
  - It returns `statusCode`, `headers`, and `body`.
  - The default encoding is `aes128gcm`.
  - Source: [README](https://github.com/web-push-libs/web-push).
- **Mapping to our users.** We own it completely: a `push_subscriptions` table in SQLite with user id, endpoint, p256dh, auth, user agent, and created/last-success timestamps. One user can have many subscriptions, one per browser or device.
  - The app must prune expired subscriptions when the push service rejects them. Typically that is an HTTP 404/410 response. This behaviour comes from the Web Push protocol and is not quoted from the README.
  - The app must also handle `pushsubscriptionchange`.
- **Inbox and preferences.** None. We build both: a notifications table, read/seen state, a preferences table, and any scheduling for nudges.
- **Cost.** $0. The browser push services (FCM for Chrome, Mozilla autopush, APNs for Safari) are free to send through.
- **What we host.**
  - VAPID key pair as env secrets.
  - A service worker (`push` and `notificationclick` handlers).
  - A subscribe endpoint and a send path.
  - A job or queue for fan-out and retries, plus a scheduler for nudges.
- **Lock-in.** None. VAPID is a standard, and subscriptions are portable to any VAPID sender that holds the same key pair.

### B. Firebase Cloud Messaging (FCM)

- **Cost.** Free. Cloud Messaging is listed as "No-cost" on both the Spark and Blaze plans. ([Firebase pricing](https://firebase.google.com/pricing))
- **Client setup.**
  - It "requires a `firebase-messaging-sw.js` file" at the domain root.
  - The VAPID key pair is generated or imported in the Firebase console.
  - "The FCM SDK is supported only in pages served over HTTPS."
  - Source: [FCM web client](https://firebase.google.com/docs/cloud-messaging/js/client).
- **Identifiers.** Google now recommends Firebase Installation IDs (FIDs) through `register()` and `onRegistered()`. `getToken()` "is deprecated" and will be removed. The page warns against using both at once. ([FCM web client](https://firebase.google.com/docs/cloud-messaging/js/client))
  - This API is changing, so it adds migration work that plain VAPID does not have.
- **Browser support.** It defers to the Push API support matrix ([caniuse push-api](https://caniuse.com/#feat=push-api)). Safari is not mentioned explicitly on the client page.
- **Server SDK.** `firebase-admin` (npm `14.5.1` at time of writing) sends through the HTTP v1 API with a Google service account.
- **Mapping to our users.** We store the FID or token per user and device in our own DB. FCM has no concept of "our user".
- **Inbox and preferences.** None. Firebase's separate "In-App Messaging" product is for mobile apps and is not a web inbox.
- **Lock-in.** Moderate.
  - It adds a Google project, a service account, and the Firebase JS SDK.
  - FCM-issued tokens and FIDs only work through FCM.
  - For web-only push it mainly wraps the VAPID push we could send ourselves.
  - Its real value shows up if native mobile apps arrive later.

### C. OneSignal

- **Pricing** ([onesignal.com/pricing](https://onesignal.com/pricing)):
  - **Free: $0/mo.**
    - The page meta mentions "10,000 free web push". The plan table shows "Max 10,000 subscribers per send".
    - Email is 10k sends/mo.
    - Mobile push and in-app messaging cover up to 1,000 MAU.
    - Limits: 3 active Journeys, 6 segments, 6 data tags, 30-day reporting.
    - No frequency capping or throttling, no custom events.
    - Webhooks only on annual plans.
  - **Growth: from $19/mo** plus "$0.004 per web push subscriber". A subscriber is an opted-in device, and unsubscribed devices don't count.
  - **Professional and Enterprise:** custom pricing.
- **What we host.** `OneSignalSDKWorker.js` at the site root. It must be on our own origin, not through a CDN or redirect, and served as JavaScript. It can be a single `importScripts(...)` line. A custom path is set with `serviceWorkerPath`. We also need a `manifest.json` for iOS. ([OneSignal web quickstart](https://documentation.onesignal.com/docs/web-push-quickstart))
  - This can conflict with an app service worker we write ourselves. Only one worker controls a given scope, so this needs checking if Cadence adds its own SW.
- **Mapping to our users.** Call `OneSignal.login(<our user id>)` on the client to set the External ID. That links the user's subscriptions across devices. ([quickstart](https://documentation.onesignal.com/docs/web-push-quickstart))
  - On the server, `POST /notifications` with `include_aliases: {external_id: [...]}` and `target_channel: "push"` sends up to 20,000 users per call.
  - Auth is the header `Authorization: Key <App API key>`.
  - Source: [Create notification API](https://documentation.onesignal.com/reference/push-notification).
- **Server SDK.** `@onesignal/node-onesignal` (npm `5.20.0`), or plain REST.
- **Inbox and preferences.**
  - The pricing page does not list a notification inbox.
  - "In-App Messaging" covers modal or banner messages, not a persistent per-user inbox (inferred from the product framing; not verified in its docs).
  - Preference management beyond subscribe and unsubscribe is not covered by the sources read.
- **Scheduling and nudges.** Journeys can do this, with 3 active on the free plan.
- **Lock-in.** High for push. Subscriptions are held by OneSignal under its SDK and service worker, and segments and journeys live in its dashboard.

### D. Knock (notification orchestration)

- **Pricing** ([knock.app/pricing](https://knock.app/pricing)):
  - **Free: $0**, up to 10,000 messages/mo. There is no hard cap: Knock contacts you about billing if you go over.
  - **Starter: $250/mo**, with 50k messages and then $0.005 each.
  - **Enterprise:** custom.
  - A message is "A single message delivered to a single user on a single channel". One workflow that sends both in-app and push counts as two messages. Bounced or failed messages aren't billed.
- **Inbox.** Yes, on all plans.
  - Per-user feeds, "read, seen and archive tracking", and badge counts.
  - Real-time updates over websockets "no infrastructure setup required".
  - Drop-in React components.
  - Requests must be secured per user.
  - Sources: [Knock in-app UI](https://docs.knock.app/in-app-ui/overview), [pricing](https://knock.app/pricing).
- **Preferences.** On all plans: "Manage user preferences for your channels, notifications, and categories". ([pricing](https://knock.app/pricing))
- **Web push.** Not native. Knock's push providers are APNs, FCM, Expo, OneSignal, and Amazon SNS ([push overview](https://docs.knock.app/integrations/push/overview)).
  - Browser push goes **through FCM**, which can send "mobile or web push notifications". The FCM web token is stored on the user as channel data with `users.setChannelData`, and Knock delivers to each unique token. ([Knock FCM](https://docs.knock.app/integrations/push/firebase))
  - So Knock plus web push means Knock plus FCM (or OneSignal), and both sets of FCM setup apply.
- **Mapping to our users.** Recipients are identified by our user id. Push tokens are channel data on that user, and Knock removes a token after a bounce ([push overview](https://docs.knock.app/integrations/push/overview)).
- **Server SDK.** `@knocklabs/node` (npm `1.41.1`).
- **Lock-in.** High.
  - Workflows, templates, preference data, and the inbox message store live in Knock.
  - Cadence's SQLite would not hold notification history unless we mirror it.
- **Scheduling.** Workflow orchestration is Knock's core product. The pricing page did not detail delay, batch, or schedule steps; check the workflow docs before relying on them.

### E. Novu (orchestration, open-core)

- **Pricing** ([novu.co/pricing](https://novu.co/pricing)):
  - **Free: $0.** 10,000 workflow runs/mo, up to 5 workflows, 3 team members, 24-hour activity-feed retention. All channels are included (email, in-app, SMS, chat, push).
  - **Pro:** from $30/mo, 30k+ runs, $1.20 per extra 1k runs.
  - **Team:** from $250/mo.
  - **Enterprise:** custom.
  - Note the unit: Novu bills **workflow runs**, while Knock bills **messages per channel**.
- **Inbox and preferences.** An `<Inbox />` component and a `<Preferences />` component are listed. The flattened pricing table does not make the per-plan availability clear. ([pricing](https://novu.co/pricing))
- **Web push.** Not native per the push integrations page. Providers are FCM, Expo, APNs, OneSignal, Pusher Beams, Pushpad, and "Push Webhook" ([Novu push](https://docs.novu.co/platform/integrations/push)).
  - Browser push would go through FCM, OneSignal, or Pushpad.
  - Push Webhook could hand off to our own `web-push` sender. That is inferred from the name; the page does not describe it as web push.
- **Mapping to our users.** Subscribers are keyed by our id. Device tokens are stored "in the subscriber's profile", with a maximum of 100 per subscriber channel.
  - The credentials update **replaces** the `deviceTokens` array, so adding a token means sending the full merged list.
  - Source: [Novu push](https://docs.novu.co/platform/integrations/push).
- **Server SDK.** `@novu/api` (npm `3.20.0`).
- **License and self-hosting.**
  - The repo is MIT outside `enterprise/packages`, which is under a proprietary licence ([LICENSE-MIT on GitHub](https://github.com/novuhq/novu)).
  - The pricing page frames supported self-hosting ("On-Prem") as Enterprise.
  - Community self-hosting needs MongoDB, Redis (AOF on), S3-compatible storage, and API, Worker, WS, and dashboard services. The single-VM minimum is 4 vCPU and 8 GB RAM plus MongoDB and Redis.
  - "some features exclusive to Novu's cloud-managed solution will not be available" self-hosted.
  - Source: [Novu self-hosting](https://docs.novu.co/community/self-hosting-novu/overview).
  - That is far heavier than Cadence's one-SQLite-file stack.
- **Lock-in.** Medium on Cloud. It is lower if self-hosted, but at a big infrastructure cost.

### F. MagicBell (another inbox-plus-push candidate)

- **Pricing** ([magicbell.com/pricing](https://www.magicbell.com/pricing)):
  - **Builder: $0**, 1,000 deliveries/mo, 1 project.
  - **Startup: $249/mo**, 50k deliveries and then $0.0025 each.
  - **Enterprise:** custom.
- **Products.** It lists both "Notification Inbox" ("The Notification Inbox for web & mobile apps") and "Web Push Notifications". User preferences are implied, because "a user preference can drop a delivery".
- **Not verified.** The pricing page does not say whether web push is native VAPID or goes through FCM. Check MagicBell's docs before shortlisting it.

### Other names seen but not researched

**Pushpad** (a web push service; also a Novu provider) and **Pusher Beams** (a Novu provider) are push-only. Neither has an inbox.

---

## 3. Comparison at a glance

| | Self-hosted VAPID | FCM | OneSignal | Knock | Novu Cloud | MagicBell |
| --- | --- | --- | --- | --- | --- | --- |
| Web push route | Direct VAPID | Own VAPID wrapper | Own SDK and SW | Via FCM or OneSignal | Via FCM, OneSignal, or Pushpad | Listed; mechanism unverified |
| In-app inbox | Build it | No | No (modal IAM only) | Yes, React and real-time | Yes, `<Inbox/>` | Yes |
| Preferences | Build it | No | Subscribe only | Yes | Yes, `<Preferences/>` | Implied |
| Scheduling / nudges | Build it | No | Journeys (3 free) | Workflows | Workflows | Not checked |
| Free tier | Unlimited, $0 | Unlimited, $0 | Web push free (10k per send); IAM 1k MAU | 10k messages/mo | 10k runs/mo, 5 workflows | 1k deliveries/mo |
| Next tier | n/a | n/a | $19/mo + $0.004 per web subscriber | $250/mo | $30/mo | $249/mo |
| Node SDK | `web-push` 4.0.0 (MPL-2.0) | `firebase-admin` | `@onesignal/node-onesignal` | `@knocklabs/node` | `@novu/api` | n/a |
| Who holds subscriptions | Our SQLite | Our DB (FID/token) | OneSignal | Knock (channel data) | Novu (subscriber creds) | MagicBell |
| App must host | SW, VAPID keys, sender, queue | `firebase-messaging-sw.js`, Firebase config | `OneSignalSDKWorker.js`, manifest | SW from FCM, plus Knock keys | SW from FCM or OneSignal, plus Novu keys | Not checked |
| Lock-in | None | Moderate | High | High | Medium (MIT core) | High |

---

## 4. Trade-offs to decide on

1. **Inbox-first or push-first.** iOS users get push only after installing to the Home Screen, so the in-app inbox is the channel that reaches everyone.
   - Knock, Novu, and MagicBell bundle the inbox and preferences, but none of them delivers browser push without a second provider (FCM or OneSignal).
   - OneSignal and FCM deliver push but have no inbox.
   - No single vendor in this set covers inbox, preferences, and native VAPID web push in one product. MagicBell might, but that is unverified.
2. **Where notification data lives.** Self-hosting keeps everything in our one SQLite file, which fits ADR 0006: queryable, testable, and portable.
   - With Knock or Novu, the inbox and preference state live in the vendor, and Cadence's DB holds at most a mirror.
3. **Build cost versus product cost.**
   - Self-hosting adds:
     - A notifications table, read state, and a preferences table.
     - A subscriptions table, the service worker, and fan-out with pruning of 404/410 responses.
     - A scheduler for nudges.
   - The push transport itself is small: one dependency and one function call.
   - Orchestrators remove most of that work, but add a second vendor for push plus per-message pricing. For example, Knock counts in-app plus push as 2 messages.
4. **Service worker ownership.** OneSignal and FCM each want their own worker file at the root. If Cadence also wants its own SW (offline support, Declarative Web Push fallback), check how they coexist on one scope before choosing.
5. **Future native apps.** If iOS or Android apps are likely, FCM and the orchestrators become more attractive. For web only, plain VAPID already reaches every browser that FCM reaches.

## 5. Open questions (not answered by the sources read)

- MagicBell's web push mechanism, and its preferences and scheduling features.
- Novu's per-plan Inbox and Preferences availability, and whether Push Webhook works well as a bridge to `web-push`.
- Knock's delay, batch, and schedule steps, and their limits on the free plan.
- Apple's current per-message payload and TTL limits. Apple's developer documentation page did not render for the fetcher.
- Whether the EU's iOS Home Screen web app policy affects web push for EU users. This was not researched.
