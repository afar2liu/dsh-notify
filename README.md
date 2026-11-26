# Notify for Chrome

<!-- skeleton:
version: 2.0
kind: readme
status: active
updated: 2026-11-26
summary: Operator manual and behavior contract for Notify for Chrome (Chinese: Chrome 通知) — Chrome/Web Notification with synthesized chimes, silent in-app toast fallback, native DSH settings rows and typography, Windows/install-mode notification branches.
-->

> Status: active · Updated: 2026-11-26 · Package: `@nolit/dsh-notify` v`0.0.1-rc.6` · Chinese display name: **Chrome 通知** · Parent: `README.md` in the `dsh-plugins` workspace

**Notify for Chrome** turns DeepSeek Harness events into **Chrome notifications** (the Web
Notification API, `new Notification(...)`). It has no native app, no ports, no IPC, no file queue
and no system notifier. When Chrome cannot show a notification, the plugin falls back to a
**silent in-app toast**.

## 0. Version numbering — why `0.0.1-rc.6` is not lower than `0.2.1`

The version number went **down** against this project's own history, on purpose. Two separate
facts explain it, and until 2026-11-06 neither was written down anywhere.

**`0.2.x` was never this plugin's version.** The `v0.2.0` and `v0.2.1` releases belonged to the
**Windows PowerShell/WPF prototype** (a tray widget polling the DeepSeek billing window). The
browser plugin lived inside that repository under `plugin/` and happened to carry the same
number — by accident, not by design.

**The `@nolit` ladder was reset by owner decision on 2026-11-05.** Every `@nolit` package moved
to one scheme, `0.0.1-rc.N`, in which only the `0.0.1-rc.6` moves. `0.1.8` and `0.2.1` are not
milestones under that scheme; the scheme replaced them.

| Surface | Before | After |
| :--- | :--- | :--- |
| Version scheme | per-track, drifting (`0.1.6 → 0.1.7 → 0.1.8 → 0.2.1`) | **`0.0.1-rc.N`** |
| Package | `@nolit/dsh-notify-web` @ `0.2.1`, nested in `plugin/` | **`@nolit/dsh-notify`** @ `0.0.1-rc.6`, repository root |
| Repository | Windows prototype at root | plugin at root; the Windows prototype was removed |

The `-web` suffix came from the 2026-10-24 recovery and was never the owner's choice;
`PROVENANCE.md` documented the package as `@nolit/dsh-notify` the entire time while the files
said `-web`. The rename landed in `0.0.1-rc.6`; `0.0.1-rc.6` is the corrected build.

Version numbers are the owner's decision. An agent must not choose, bump or "stabilise" one.

## 1. Behavior

| DSH event | Channel |
|---|---|
| `api-session/status` (`running=false`) | Chrome notification "Turn complete", root sessions only (unless `Include subagents` is on), 5 s cooldown (`running=true` clears that session's completion cooldown and error suppression) |
| `api-session/error` | Chrome notification "Turn failed", root sessions only (unless `Include subagents` is on), message truncated to 160 chars; suppresses the completion its own idle transition would raise |
| `approval/request` | Chrome notification "Approval needed", deduplicated per request ID for 30 s; `next()` passes through immediately |
| `user-questions/request` | Chrome notification "Question waiting" (or "Plan review needed" / "Approval needed" per intent), deduplicated per request ID for 30 s; `next()` passes through immediately |

- **Chrome first.** The plugin calls the Notification API. A granted permission and a secure
  context are the only prerequisites.
- **Silent toast fallback.** If the permission is denied/pending, the origin is not a secure
  context, the Notification API is unavailable, or the constructor/`onerror` fails, the event is
  shown as the **native DSH toast** and nothing else. The toast itself is silent; there is no second
  notification channel and no title-bar marker.
- **One tab delivers.** Multiple tabs elect a leader (Web Locks, localStorage lease fallback) so an
  event is not announced once per tab. A focused tab always delivers.
- **One alert per turn.** The host reports a failure and the idle transition it causes as two
  separate events (`agent/error`, then `agent/status` → `api-session/status(false)`). A delivered
  "Turn failed" alert therefore suppresses the "Turn complete" that would echo the same failed turn
  for the cooldown window, per session. The suppression is consumed by that one transition — the
  next turn that actually runs and finishes still notifies — and setting the cooldown to `0`
  disables it.
- **Named input requests.** A plan review raised through user questions is announced as "Plan review needed", and an approval raised the same way as "Approval needed"; other questions keep "Question waiting".
- **Synthesized chimes, no audio files.** Each completion, approval and question plays a short chime generated with Web Audio; the plugin ships and stores no audio file. When the chime plays, its notification is `silent` so you hear one sound. When the browser has locked audio or sound is off, the operating system's own notification sound plays instead.

## 2. Windows and install-environment behavior

**Which OS does the environment card report?** The **browser's** OS — the machine running the tab
that shows the notification — not the OS running the `dsh` host process. The two are independent:
a Windows Chrome can be pointed at a DSH server on Linux (the default remote setup), and the
notification rules that matter (Action Center persistence, `requireInteraction` support, the
installed-app branch) are properties of the browser's OS.

The sensitivity is per-platform, measured:

| Browser OS | Installed app | Behaviour |
|---|---|---|
| Windows | yes | `requireInteraction: true` for every kind; the app window and Action Center keep them |
| Windows | no | `requireInteraction` follows the setting, approval/question only |
| Linux | either | same as the Windows browser tab |
| macOS | either | `requireInteraction` is not honoured by Chrome, so it stays `false` |
| Android / other | either | `requireInteraction: false`; `silent` follows chime; unique `tag`; no persistence |

Because a value like "Windows" next to a Linux host looks wrong, the row is labelled
**Browser OS** (浏览器 OS) and carries that explanation inline.

The Chrome notification parameters branch on the **runtime platform** and the **install mode**.
Platform detection order: DSH's `document.documentElement.dataset.platform` (the native shell sets
it; native code also tests it for `darwin`), then `navigator.userAgentData.platform`, then
`navigator.platform`, then `navigator.userAgent`. Install mode is `installed` when the page runs in
a standalone/minimal-ui/fullscreen display mode (or `navigator.standalone`), otherwise `browser`.

| Environment | `requireInteraction` | `silent` | `tag` |
|---|---|---|---|
| **Windows, installed app** | `true` for every kind — the Action Center keeps the alert while the app window is minimized, so a completion cannot be auto-dismissed | `true` when the chime played, else `false` | unique per event |
| **Windows, browser tab** | follows the "Keep approval alerts open" setting; approval/question only | `true` when the chime played, else `false` | unique per event |
| **macOS** | `false` — Chrome does not honour `requireInteraction`, so claiming it would misrepresent the behavior | `true` when the chime played, else `false` | unique per event |
| **Linux / other** | follows the setting; approval/question only | `true` when the chime played, else `false` | unique per event |

- **Why a unique tag on Windows:** Windows replaces an existing notification that reuses a `tag`,
  which silently drops repeated events. Every event gets a fresh tag so completions accumulate in
  the Action Center instead of being replaced.
- **Permission guidance is platform-specific:** a Windows profile points at Chrome's site settings
  and **Windows Settings → System → Notifications**; other platforms point at the browser's site
  settings. The guidance is shown next to the status row and is the only place a blocked
  permission is explained.
- Nothing else is Windows-specific: event filtering, cooldowns, tab leadership, locale and the
  toast fallback are generic.

## 3. Install

> [!NOTE]
> **Test project / local install only:** This package has **not** been published to the npm
> registry and does **not** support online installation (`dsh plugin --profile web add @nolit/dsh-notify`
> will not work). To use it, clone the repository locally and install it with `link:`:

```bash
git clone https://github.com/afar2liu/dsh-notify.git /path/dsh-notify
dsh plugin --profile web add "link:/path/dsh-notify"
```

The package name stays `@nolit/dsh-notify`; only the **display identity** changed to
"Notify for Chrome" / "Chrome 通知". Renaming the npm package would break the live profile's
`link:` dependency.

The package must be listed in the profile's `dsh.profile.bundles`. Reload the DSH page after
installing (and run `npm run build` first if you edit `lib/client.src.js` or `locale/*.json`). If
`lib/client.js` has a syntax error only this plugin is disabled; the DSH core keeps working.

## 4. Settings

**Settings → Notify for Chrome** (English; the label is **Chrome 通知** when the host locale is
Chinese) renders native DSH settings surfaces only:

- a native `Tag` for the Chrome-notification status, native `Button` actions, native `Switch`
  toggles and a native `Input` for the cooldown;
- the section navigation icon is a **DSH icon** (`IconAlarmClockOutlineRegular`), registered through
  `settings.section.icon`; the plugin ships no icon set of its own;
- typography comes from the live DSH tokens `--dsh-content-font-size` /
  `--dsh-content-font-size-secondary` and their `--dsh-content-font-delta*` line-height deltas;
  colours and surfaces come from `--dsw-*` tokens. The plugin **injects no stylesheet**, sets **no
  font-family** and hardcodes **no pixel font size**.

The page is grouped into five cards: status, when to notify, sound, browser support and environment.

**When to notify**

| Row | Default | Meaning |
|---|---|---|
| Turn complete / Approval needed / Question waiting / Turn error | on | which events notify |
| Include subagents | off | also notify for subagent complete/error |
| Hide completion alerts while focused | off | suppress completion while the DSH tab is focused |
| Keep approval alerts open | on | `requireInteraction` for approval/question where supported (Windows/Linux) |
| Only alert while this tab is in the background | off | suppress every alert while the DSH tab is focused |
| Completion cooldown | 5 s | per-session grouping window |

**Sound rows**

| Row | Default | Meaning |
|---|---|---|
| Play a chime | on | a synthesized chime with each completion, approval and question |
| Chime volume | 60 | 0–100 |
| Turn complete chime | chime | `chime` (ascending) or `cascade` (descending) |
| Input needed chime | triple | `triple` (urgent beeps) or `pulse` (double pulse) |

The Sound card also provides **Play complete sample** and **Play input sample** buttons, and the Environment card includes **Reset settings** to restore defaults.

Browsers unlock audio only after a click or key press on the page. Until then the chime stays silent and the OS sound plays instead.

Preferences persist in the browser's `localStorage` under `dsh-notify-web:settings` (version 2).
They are browser preferences, not Host configuration: the plugin's Host half is intentionally a
no-op and exposes no settings namespace. No audio file is read, stored or referenced; chimes are generated in the browser.

Guidance rules:

- **Permission pending** → the Enable button is shown.
- **Denied / insecure / unsupported** → the Enable button is **hidden** (a capability that is gone
  is not shown as a broken control); the row explains what to change, and the test action still
  exercises the toast fallback.
## 5. Localization

English is the default. Chinese is used **only** while the host locale (`ctx.locale.active`) is
Chinese (`zh` / `zh-CN` etc.); a locale change switches the next notification and the settings
strings live.

- Source of truth: [`locale/en.json`](locale/en.json) and [`locale/zh.json`](locale/zh.json).
- `scripts/gen-locale.mjs` bakes them into `lib/locale-table.js`; `scripts/build-client.mjs` inlines
  that table into `lib/client.js` (the browser loader cannot import JSON).
- No hardcoded Chinese exists outside the baked dictionary (a test enforces this).
- Display identity: English **Notify for Chrome**, Chinese **Chrome 通知**.

## 6. Limitations

### Browser support — Chrome / Chromium required

The plugin is built for **Chrome and Chromium-based browsers** (Edge, Brave, Opera, Vivaldi, Arc).
Those are the engines where the whole surface below exists at once. **On any other browser, correct
operation is not guaranteed and is not tested.**

| Capability | Chrome / Chromium | Firefox | Safari |
|---|---|---|---|
| `new Notification()` | yes | yes | 16+ only |
| `AudioContext` (synthesized chimes) | yes | yes | yes (with `webkitAudioContext` fallback) |
| `navigator.locks` (tab leader) | 69+ | yes | no (`localStorage` lease fallback) |
| `requireInteraction` | Windows/Linux | **ignored** | **ignored** |
| Installed-app notification branch | yes | n/a | n/a |

**Engine detection vs. feature detection.** The plugin detects the browser engine family
(`chromium`, `firefox`, `safari`, `other` from `userAgent`, `vendor`, and `userAgentData.brands`)
to display the Browser support card, and never claims a non-Chromium browser is supported.
Runtime capabilities (`Notification`, `isSecureContext`, `navigator.locks`, `AudioContext`) are
feature-detected and degrade gracefully. A non-Chromium browser may look functional while behaving
differently — for example Firefox accepts `requireInteraction` and ignores it, so an approval alert
auto-dismisses even though the setting is on.

### Security constraints that can stop a notification appearing

These are browser/OS security rules, not plugin bugs. Each one degrades to the **silent in-app
toast** rather than failing loudly, which is why the toast fallback exists.

| Constraint | What the user sees | Measured behaviour |
|---|---|---|
| **Permission not granted** (`default`) | nothing from the OS | `new Notification(...)` **does not throw** — it looks successful and shows nothing. The plugin checks `Notification.permission === 'granted'` *before* constructing, so it routes to the toast instead. |
| **Permission blocked** (`denied`) | nothing, and no prompt returns | `requestPermission()` resolves `'denied'` immediately; Chrome will not prompt again for the origin. Only the user can undo it in site settings. The plugin hides the Enable button, because a control that cannot work is not shown as a broken one. |
| **Insecure origin** | nothing | `http://<LAN-IP>:3080` is not a secure context; Chrome refuses OS notifications. Use `https://` or `localhost`. Reported as `insecure`. A self-signed certificate needs to be trusted once in the browser; do **not** browse with certificate errors bypassed — notifications need a genuinely secure context. |
| **No user gesture** | prompt never appears | `requestPermission()` needs a click. On the first real event with `permission === 'default'`, the plugin attempts `requestPermission()` once per page; if the browser blocks it without a gesture, the toast fallback covers it until the user presses the Enable button in Settings. |
| **OS Do Not Disturb** / Focus Assist | nothing, or a silent banner | The OS owns this; the page cannot override it and cannot detect it. |
| **Browser-level site mute** | nothing | Silent skip at the Chrome layer; indistinguishable from a delivered notification. |
| **Tab closed** | nothing | No service-worker backend, so no delivery once the DSH tab is gone. |

- Notifications work **only while the DSH tab is open**. There is no service-worker push backend.
- Background-tab throttling, browser notification policy and OS Focus Assist / Do Not Disturb can
  change what is shown.
- Approval/question notifications do not replace the DSH approval UI; the event handler passes
  `next()` through immediately.

## 7. Privacy

- No `XMLHttpRequest`, `WebSocket`, `EventSource` or local port; no filesystem access.
- **No network requests.** The browser half makes no outbound request of any kind.
- The only storage is the settings object and the tab-leader lease key, both in `localStorage`.
- Notification bodies are fixed copy plus the session title (80 chars max); prompts, code and tool
  arguments are never included.
- Notification clicks only focus the DSH tab; no external link is opened.

## 8. Development

```bash
cd dsh-notify
npm test          # pretest runs gen-locale + build-client, then node --test
npm run build     # regenerate lib/locale-table.js and lib/client.js
node --check lib/client.js
node --check lib/index.js
```

Tests use mocked browser APIs (`Notification`, `AudioContext`, `localStorage`, Web Locks) and assert
event filtering, cooldowns, error-to-completion suppression, approval/question pass-through,
user-questions intent naming, synthesized chimes, `onlyWhenHidden`, the Chrome-vs-toast channel
decision, Windows/macOS/Linux parameter branches, locale switching, settings persistence,
native-primitive rendering, the typography token contract, and the absence of every removed legacy
path.

## 9. Structure

- `lib/client.src.js` — editable browser-half source (baked by `scripts/build-client.mjs`).
- `lib/client.js` — shipped browser entry (generated; do not edit by hand).
- `lib/locale-table.js` — generated locale dictionary (from `locale/*.json`).
- `lib/index.js` — no-op Host half; registers the package only. No services, no I/O.
- `cordis.patch.yml` — bundle row and default settings.
- `market/index.json` — market/index entry (`@nolit/dsh-notify`, format 1).

## 10. License and assets

- Code: MIT. See [LICENSE](LICENSE).
- The package bundles **no fonts, logos or audio**. The manifest icon is the neutral clock SVG at
  [`icon.svg`](icon.svg)/[`market/icons/dsh-notify.svg`](market/icons/dsh-notify.svg).
- The historical `docs/` tree is kept as history (included in `package.json` `files`), and is not
  referenced by the notification code path.
- Upstream / reference repositories:
  - <https://github.com/Nolitor/dsh-notify/>
  - <https://github.com/dshiq04/dsh-notify>

---


## 中文摘要

**Notify for Chrome（Chrome 通知）** 通过浏览器的 **Chrome 通知（Web Notification API）** 提醒 DeepSeek Harness 的回合完成、工具审批、提问与错误事件；当 Chrome 拦截或无法显示通知时，会自动回退为**静默的 DSH 应用内 Toast**。提示音通过浏览器的 Web Audio API 实时合成，不包含任何音频文件、音频存储、自定义 CSS、本地应用、端口或文件队列。

- 当前为测试项目，尚未发布到 npm 仓库，不支持在线安装。使用时需先将仓库 clone 到本地，再通过 `dsh plugin --profile web add "link:/path/dsh-notify"` 安装。
- 设置入口位于 **设置 → Chrome 通知**，仅使用 DSH 原生组件（Switch / Button / Tag / Input / Toast）和 DSH 设计与排版 Token。
- 默认语言为英文，仅当宿主语言（locale）为中文时切换为中文显示。
- Windows 下的通知参数会根据安装环境（已安装应用 / 浏览器标签页）有所区别：已安装应用下所有通知均保持常驻直到关闭，浏览器标签页下仅审批和提问通知保持常驻；macOS 下不使用 `requireInteraction`。
- 项目参考源：
  - <https://github.com/Nolitor/dsh-notify/>
  - <https://github.com/dshiq04/dsh-notify>
