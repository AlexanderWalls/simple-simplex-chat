# Simplifying the Android UI for WhatsApp switchers — audit

**Date:** 2026-08-26
**Scope:** Android only (`apps/multiplatform/`), desktop behaviour unchanged
**Status:** Audit. No code changed.

All paths below are relative to
`apps/multiplatform/common/src/commonMain/kotlin/chat/simplex/common/` unless stated
otherwise. Line numbers were verified against the working tree at the date above.

---

## 1. Executive summary

SimpleX's Android UI is not badly designed — it is designed for a different user. It exposes
the protocol's machinery (server operators, relays, per-contact feature negotiation, incognito
profiles, multiple identities) as first-class UI because, for its current audience, that
machinery *is* the product. For a WhatsApp switcher, each of those is a decision they have no
basis to make and no motivation to learn.

Six conclusions, in order of impact:

1. **Onboarding is already fine.** Cold start to chat list is 4 screens, one text field, one
   consent gate. This is the part people assume is the problem, and it isn't. Do not spend
   effort here. (§3.1 — note this contradicts the repo's own `product/` docs, which are stale.)
2. **The friction is the 30 seconds *after* the chat list appears.** Four interruptions land
   back to back — a bare system permission dialog, a battery-optimisation alert, a layout
   preference card, and a two-page tutorial deck — before the user has done anything. This is
   the single highest-value fix and it is about *sequencing*, not removal. (§3.2)
3. **The genuine blocker is that there is no way to find anyone.** Connecting requires
   exchanging a link or QR out of band. No phone numbers, no directory. This is a deliberate
   architectural property, not a UI wart — but it means a new user's first session usually ends
   with an empty chat list. No settings change fixes this. (§3.3)
4. **Chat preference negotiation is the most confusing feature in the app** and the one with
   the least payoff for a mainstream user. Five features, each a tri-state (Always/Yes/No)
   negotiated with the other party, presented across three separate screens, with an explicit
   "Save and notify contacts" step. WhatsApp has no analogue at any level. (§4, item 1)
5. **Incognito and multiple profiles cost more than they give.** Between them they add
   surface to roughly a dozen screens — pickers, toggles, mask icons, alert variants, menu
   items. A user who wants one identity pays that tax on every connection flow. (§4, items 2–3)
6. **One "hide" is actually a default bug for this audience.** `privacyAskToApproveRelays`
   defaults to `true`, which prompts before downloading files sent via a contact's relay.
   Hiding the toggle locks that prompt on forever. It needs a default decision, not a hide.
   (§5)

**Recommended shape:** a single `simpleMode` preference (default on for new installs) that
hides advanced surface, following the existing `developerTools` pattern exactly. Estimated
~15 files touched, almost all of them one-line conditionals. Spec in Appendix A.

---

## 2. Method and what was checked

Three parallel passes over the Kotlin/Compose sources: the settings tree
(`views/usersettings/`, 9,715 lines across 15 files plus `networkAndServers/`); the
first-run and connection flows (`views/onboarding/`, `views/newchat/`, `views/contacts/`); and
the daily messaging surface (`views/chatlist/`, `views/chat/` — 25,565 lines).

Every recommendation was checked against `product/rules.md` (18 business invariants) and
against the setting's default in `AppPreferences` (`model/SimpleXAPI.kt:101-390`).

**Two caveats found during research, both worth recording:**

- **`product/views/onboarding.md` and `product/flows/onboarding.md` are stale.** They describe
  pre-7.0 mandatory screens for address creation and notification mode. In the code those
  stages carry `// deprecated` comments (`views/onboarding/OnboardingView.kt:9-10`) and route
  to a single collapsed screen (`App.kt:194-196`). `product/views/settings.md` is stale too —
  it predates the "Advanced settings" section at `views/usersettings/SettingsView.kt:106`.
  Trust source over `product/` until these are refreshed.
- **The build cannot be verified in a cloud session.** No Android SDK, and Gradle cannot
  resolve `com.android.application:8.7.0` through the proxy. Every change from this audit needs
  a local `./gradlew :android:assembleDebug`.

---

## 3. The switcher's journey, ranked by where friction lands

### 3.1 Cold start → chat list: already short, leave it alone

| # | Screen | File | Mandatory? |
|---|---|---|---|
| 1 | "Be free in your network" → **Get started** | `views/onboarding/SimpleXInfo.kt:44` | Yes |
| 2 | "Your profile" — display name only | `views/WelcomeView.kt:192` | Yes |
| 3 | "Your network" → **Continue** | `views/onboarding/YourNetwork.kt:42` | Yes, but its two detours ("Setup routers", "Setup notifications") are optional |
| 4 | "Network commitments" → **Accept** | `views/onboarding/ChooseServerOperators.kt:42` | Yes — hard consent gate |

Four screens, one text field, one legal accept. Notably the team has already done this work:
three former stages collapsed into screen 3, and new users are quietly opted out of the
delivery-receipts prompt (`platform/Core.kt:146`), the What's New dialog
(`views/WelcomeView.kt:306`), and the file-encryption indicator (`views/WelcomeView.kt:395`).

**Recommendation: no change.** The only defensible tweak is copy on screen 4, and consent
wording is the last thing to touch casually.

### 3.2 The interruption stack — highest-value fix in this audit

Immediately after the chat list renders, in this order:

| # | Interruption | File |
|---|---|---|
| 1 | Android `POST_NOTIFICATIONS` system dialog, **with no in-app explanation first** | `androidMain/.../onboarding/SetNotificationsMode.android.kt:12-39` |
| 2 | Battery-optimisation / background-restriction alert, possibly leading to system settings | `android/src/main/java/chat/simplex/app/SimplexService.kt:430-468` |
| 3 | One-hand-UI card — asks the user to pick a toolbar layout | `views/chatlist/ChatListView.kt:94-144` |
| 4 | Two-page "Talk to someone" card deck | `views/newchat/OnboardingCards.kt:315-425` |

A WhatsApp user's model is "I installed it, it works." Four modal-ish demands before sending a
single message reads as the app being broken or needy. Note #1 in particular: Android shows a
bare system dialog, and a user who taps "Don't allow" has silently disabled the app's core
value, with `canAskToEnableNotifications` (`model/SimpleXAPI.kt:111`) meaning they may never be
asked again.

**Recommendation — sequence, don't remove:**

- Precede #1 with a one-line in-app explanation of *what breaks* without it. This is the
  standard Android pattern and the app already has the screen for it
  (`views/onboarding/SetNotificationsMode.kt:25`), just not wired into this path.
- Defer #2 until the user has at least one conversation. It is meaningless before then.
- Hide #3 in simple mode — pick a default layout and commit to it. `oneHandUI` already
  defaults to `true` (`model/SimpleXAPI.kt:269`).
- Keep #4. It is the only thing on screen telling the user what to do next, and it is already
  written in plain language.

### 3.3 Reaching a first conversation — the real gap

Contact discovery is link-based only: 1-time invitation links, contact address links, group
links. Entry points are well built (`views/newchat/NewChatSheet.kt:186-207`) and the
paste-a-link path is genuinely good — the chat-list search field doubles as a link acceptor
(`views/chatlist/ChatListView.kt:751`).

But there is no way to answer "is my sister on this?" A "SimpleX name" handle exists
(`views/usersettings/UserAddressView.kt:374-399`) and is resolvable from any search box
(`views/chatlist/ChatListView.kt:1056-1067`), but it is labelled BETA and defaults to a
`.testing` TLD (`:1043`).

**Recommendation: out of scope for UI simplification, but name it.** No amount of hiding fixes
this. The honest options are (a) accept it and make the invite flow as frictionless as
possible, or (b) lean hard on the SimpleX name feature as the primary identity story. Worth a
separate decision.

---

## 4. Catalogue: what to remove, hide, or reword

Recommendation key: **REMOVE** = delete from the fork · **HIDE** = gate behind `simpleMode`
· **REWORD** = keep, change copy · **KEEP** = leave alone.

### Tier 1 — highest confusion, lowest mainstream value

| # | Feature | Where | Rec. | Notes |
|---|---|---|---|---|
| 1 | **Chat preference negotiation.** 5 features × tri-state Always/Yes/No, in three screens, plus "Save and notify contacts" | `views/usersettings/Preferences.kt:65-176` (global), `views/chat/ContactPreferences.kt:77-158` (per-contact), `views/chat/group/GroupPreferences.kt:97-251` (per-group) | **HIDE** global + per-contact; **KEEP** group | The tri-state is genuinely hard: `ALWAYS` = allow even if they don't, `YES` = only if they also do, `NO` = prohibit. Per-contact adds a 4th value, `default (…)`. Group prefs are the one case with a real analogue (group admin settings) — keep for owners. |
| 2 | **Incognito.** Profile picker in new-chat, group-creation toggle, composer picker, contact-info section, "Accept incognito"/"Join incognito" menu items, mask icons on rows, 3-way connection alerts | `views/newchat/NewChatView.kt:375-405`, `views/chat/ComposeContextProfilePickerView.kt:165-222`, `views/chatlist/ChatPreviewView.kt:552-563`, `views/chatlist/ChatListNavLinkView.kt:821-881` | **HIDE** | Default is already off (`model/SimpleXAPI.kt:189`). `ActiveProfilePicker` already takes a `showIncognito` flag (`views/newchat/NewChatView.kt:293`) — reuse it. |
| 3 | **Multiple chat profiles.** Switcher, `UserProfilesView`, hidden profiles with passwords, per-profile mute, per-profile themes | `views/chatlist/UserPicker.kt:252-302`, `views/usersettings/UserProfilesView.kt`, `views/usersettings/HiddenProfileView.kt` | **HIDE** | Collapses the user picker to: address, settings, use-from-desktop. Careful: `UserProfilesView.kt:360` resets onboarding when the last profile is deleted — that path must stay reachable. |
| 4 | **Network & servers**, entire subtree: operators, chat relays, SMP/XFTP server lists, SOCKS proxy, onion modes, and raw `TCP_KEEPIDLE`/`TCP_KEEPINTVL`/`TCP_KEEPCNT`, per-KB timeouts, private-routing mode pickers | `views/usersettings/networkAndServers/` — worst offender `AdvancedNetworkSettings.kt:229-347` | **HIDE** the top-level row (`SettingsView.kt:107`) | One conditional hides thousands of lines of UI. Defaults are all sane. |
| 5 | **WebRTC ICE servers** — free-text server-list editor | `views/usersettings/RTCServers.kt:93-123` | **HIDE** | Reachable from two places: `CallSettings.kt:43` and `NetworkAndServers.kt:302`. |
| 6 | **Self-destruct passcode** | `views/usersettings/PrivacySettings.kt:681-723` | **HIDE** | Alarming rather than reassuring for this audience. Already double-gated (needs app lock on + passcode mode). Note RULE-04 governs behaviour, not visibility — hiding is safe. |
| 7 | **Crowdfunding / "invest" row** in Settings | `views/usersettings/SettingsView.kt:116-125` | **REMOVE** | A fork has no reason to ship an investment pitch. Deleting the block is a clean 10-line removal; `GetStakeView` then has one remaining caller in `WhatsNewView.kt`. |

### Tier 2 — meaningful clutter

| # | Feature | Where | Rec. | Notes |
|---|---|---|---|---|
| 8 | **Delivery-receipts tri-state** + "override for all contacts" alerts | `views/usersettings/PrivacySettings.kt:364-443` | **HIDE** | New users are already force-set to on (`platform/Core.kt:146`). RULE-08 governs consistency, not visibility. |
| 9 | **Chat lists / tags** — custom tag chips, emoji picker, reordering, collapsing | `views/chatlist/TagListView.kt:297`, `views/chatlist/ChatListView.kt:1185-1443` | **HIDE** user tags; **KEEP** the unread filter | Preset tags (Contacts/Groups/Favorites) are harmless; user-defined tag creation and reordering are power-user features. |
| 10 | **Custom theme editor** — 8 colour rows, wallpaper scale/tint, per-user theme destination, font scale, toolbar transparency + blur | `views/usersettings/Appearance.kt:445-1188` | **HIDE** editor; **KEEP** light/dark + wallpaper presets | The single densest settings screen in the app. |
| 11 | **Live messages** (send-button dropdown + composer bolt button) | `views/chat/SendMsgView.kt:190-200`, `:499-521` | **HIDE** | Character-by-character live typing. No WhatsApp analogue; surprising when triggered accidentally. |
| 12 | **`//` bot-commands button** in composer | `views/chat/ComposeView.kt:1268-1281` | **KEEP** | Already conditional on `cInfo.useCommands` and an empty/`/`-prefixed draft — invisible in ordinary chats. Verified, no change needed. |
| 13 | **Content filters** — 6-way media/file/link filter menu in chat toolbar | `views/chat/ChatView.kt:3955-3973`, menu at `:1458-1514` | **HIDE** or reduce to "Media" | Six options where WhatsApp offers one "Media, links and docs" screen. |
| 14 | **`MorePrivacy` second tier** — show-last-messages, message draft, verify SimpleX names, encrypt local files, protect IP, show encryption | `views/usersettings/PrivacySettings.kt:114-244` | **HIDE** most | This screen is itself evidence of the problem — someone already built a hand-rolled "advanced" tier. Simple mode should absorb it. See §5 for the `protect_ip_address` caveat. |
| 15 | **Message context menu**: Report, Moderate, Archive report | `views/chat/item/ChatItemView.kt:492-494`, `:1045-1057` | **KEEP** | Already conditional on group + role + Reports feature. Not visible in 1:1 chats. |

### Tier 3 — reword, don't hide

| # | Item | Where | Notes |
|---|---|---|---|
| 16 | **Notification service mode** (Off / Periodic / Always-on) | `views/usersettings/NotificationsSettingsView.kt:38-58` | Unavoidable without FCM — RULE-16 makes the foreground service the delivery mechanism. Default is already `SERVICE` (`model/SimpleXAPI.kt:8258`), the right choice. Reword outcome-first: "Get messages instantly" / "Check every 10 minutes" / "Only when app is open". |
| 17 | **"1-time link" vs "SimpleX address"** | `views/newchat/NewChatView.kt`, `views/usersettings/UserAddressView.kt` | The `onboarding = true` copy already solves this ("Send the link via any messenger — it's secure. Ask to paste into SimpleX."). Reuse it rather than writing new strings. |
| 18 | **"Chat data"** (database passphrase, export, import, delete) | `views/database/DatabaseView.kt:107` | **KEEP, reword.** This is the only backup mechanism. A WhatsApp user *expects* backup and will look for it. Hiding it would be actively harmful; "Chat data" just doesn't read as "backup". |

**Translation cost note:** the app ships 3,204 strings across 42 locales
(`common/src/commonMain/resources/MR/`). Every *new* string ships untranslated in 41 languages.
This is why the audit prefers hiding over rewording, and why items 16–17 should reuse existing
`onboarding_*` strings wherever possible.

---

## 5. Defaults that hiding would lock in

Hiding a toggle freezes its default permanently for simple-mode users. Checked:

| Preference | Default | Verdict when hidden |
|---|---|---|
| `incognito` (`:189`) | `false` | Safe |
| `networkUseSocksProxy` (`:153`) | `false` | Safe |
| `performLA` (`:117`) | `false` | Safe — app lock stays user-reachable anyway |
| `notificationsMode` (`:8258`) | `SERVICE` | Safe, and correct for this audience |
| `webrtcPolicyRelay` (`:115`) | `true` | Safe — relays calls, protecting IP |
| `privacyAcceptImages` (`:123`) | `true` | Safe, matches WhatsApp |
| `privacyMediaBlurRadius` (`:138`) | `0` | Safe |
| `privacyEncryptLocalFiles` (`:136`) | `true` | Safe |
| **`privacyAskToApproveRelays` (`:137`)** | **`true`** | **NOT safe — see below** |

### The one that needs a decision, not a hide

`privacyAskToApproveRelays` defaults to `true`. Per **RULE-15**
(`product/rules.md:185`), the app must then prompt before downloading a file sent via a
relay the user hasn't approved — i.e. most files from most contacts. The flag is consumed at
`model/SimpleXAPI.kt:2124`.

For a WhatsApp switcher, "tap to approve a server before you can see the photo your friend
sent" is exactly the kind of interruption this whole exercise exists to remove. But turning it
off trades their IP address to a relay chosen by the sender — a real privacy reduction, and one
the user would not know they had accepted.

**This is your call, not a mechanical one.** Three options:

1. **Leave `true`, hide the toggle** — safest privacy, keeps the prompts. Least surprising to
   SimpleX's existing values.
2. **Default to `false` in simple mode, keep the toggle visible** — smoothest experience, and
   the user can still find the control. My recommendation.
3. **Default to `false` and hide** — smoothest, but silently weakens a privacy property. I'd
   avoid this one.

Do not bundle this into a general "hide advanced settings" commit; it changes behaviour, and it
should be visible as its own decision in the history.

---

## 6. What not to touch

Worth stating explicitly, since a simplification pass tends to over-reach:

- **Onboarding (§3.1)** — already short.
- **Anything that changes what is negotiated with contacts.** Hiding a chat *preference* is
  safe; changing its *default* changes what your users' contacts experience, silently, on both
  ends.
- **Database export / passphrase** (item 18) — the only backup path.
- **The security-code verification flow** (`views/chat/ChatInfoView.kt:596`) — WhatsApp has
  this too, and users who look for it are the ones who need it.
- **Group preferences for owners** (item 1) — the one place feature control has a real
  mainstream analogue.
- **Report / moderate** (item 15) — already invisible outside groups.

---

## Appendix A — `simpleMode` implementation spec

**1. Define the preference** — `model/SimpleXAPI.kt`, alongside `developerTools` at `:148`:

```kotlin
val simpleMode = mkBoolPreference(SHARED_PREFS_SIMPLE_MODE, true)
```

with `private const val SHARED_PREFS_SIMPLE_MODE = "SimpleMode"` in the companion object near
`:433`. Default `true` — new installs get the simple UI.

**2. Mirror it into `AppSettings`** (`model/SimpleXAPI.kt:8290+`): add the field, the diff line
(~`:8327`), the apply line (~`:8375`), the default (~`:8413`), and the read (~`:8452`). Skipping
this means the setting is lost on device migration — `developerTools` does all five, so copy it.

**3. Read it at the point of use**, never through parameters:

```kotlin
if (!appPrefs.simpleMode.state.value) { /* advanced row */ }
```

`AdvancedNetworkSettings.kt` threads `developerTools` through its layout signature (`:39`,
`:166`, `:201`) and then never reads it in the body (`:229-347`) — a dead gate. Don't repeat it.

**4. Guard to Android** where the surface is shared, using the existing idiom:

```kotlin
val simple = appPlatform.isAndroid && appPrefs.simpleMode.state.value
```

**5. The escape hatch.** Put the toggle in Appearance (`views/usersettings/Appearance.kt`,
Interface section) — discoverable but not prominent. Not in Developer tools; a curious user
should be able to find it without being told it's for developers.

**6. Reuse what exists.** `NewChatView` and `UserAddressView` already take an
`onboarding: Boolean` that strips tab bars, titles, section headers and the incognito picker,
and substitutes plain-language copy (`views/newchat/NewChatView.kt:110-118`, `:468-498`;
`views/usersettings/UserAddressView.kt:325-337`). That is the desired simple-mode behaviour,
already written and already translated into 42 languages. Feed `simpleMode` into that parameter
rather than building a parallel mechanism.

**Estimated footprint:** ~15 files, most changes a single conditional. The two larger pieces are
the user picker (item 3) and the incognito surface (item 2), which touch several call sites each.

---

## Appendix B — documentation debt found

Independent of any UI change, these are wrong today and will mislead the next person (or model)
that reads them, per the workflow mandated in `apps/multiplatform/CODE.md`:

- `product/views/onboarding.md` — describes deprecated stages as mandatory screens; predates the
  collapse into `YourNetworkView`.
- `product/flows/onboarding.md` — same, including a stale `OnboardingStage` enum listing.
- `product/views/settings.md` — predates the "Advanced settings" section
  (`views/usersettings/SettingsView.kt:106`); also lists rows that now live in `UserPicker.kt`.

Also worth fixing while nearby: the dead `developerTools` gate in
`views/usersettings/networkAndServers/AdvancedNetworkSettings.kt` (`:39`, `:166`, `:201`), and
the unused `showSettingsModalWithSearch` parameter in `views/usersettings/SettingsView.kt:84`.
