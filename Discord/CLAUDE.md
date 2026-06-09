# Discord Utils — CLAUDE.md

Single-file web app (`index.html`) with Discord utilities. No frameworks, no build tools, no dependencies besides CDN-loaded fonts and Bootstrap Icons.

---

## Stack & Files

```
C:\Users\Administrator\Desktop\DiscordUtils\
└── index.html   ← entire app (HTML + CSS + JS)
```

**CDN dependencies (head)**
- Bootstrap Icons: `https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css`
- Fira Code font: `https://fonts.googleapis.com/css2?family=Fira+Code:wght@400;500;600;700&display=swap`

---

## App Architecture

### Two-screen model

Both screens are `position: fixed; inset: 0` and overlay the full viewport.

| Element | State before login | State after login |
|---------|-------------------|-------------------|
| `#pre`  | visible (no class) | `#pre.gone` → fade out |
| `#app`  | `opacity:0; pointer-events:none` | `#app.here` → fade in |

**Transition sequence (login):**
```javascript
g('pre').classList.add('gone');          // fade #pre out
setTimeout(() => g('app').classList.add('here'), 80);  // fade #app in
```

**Transition sequence (logout):**
```javascript
g('app').classList.remove('here');
setTimeout(() => { g('pre').classList.remove('gone'); /* reset state */ }, 290);
```

### App layout

```
#app (position:fixed, display:flex, height:100dvh)
├── #sb (sidebar, 288px wide, flex column)
│   ├── .sb-scroll (flex:1, overflow-y:auto)
│   │   ├── .sb-prof (avatar, name, handle, bio, meta grid)
│   │   └── nav.sb-nav (tab buttons with data-tab)
│   └── .sb-foot (logout button)
└── #main (flex:1, display:flex, flex-direction:column)
    ├── #tp-friends  (.tp.on by default)
    ├── #tp-servers  (.tp)
    ├── #tp-dms      (.tp)
    └── #tp-delete   (.tp)
```

Tab switching: clicking `.sb-item[data-tab]` toggles `.on` on the matching `.tp#tp-{tab}`.

---

## Discord API

**Base URL:** `https://discord.com/api/v10`

**Auth:** Raw token in `Authorization` header — no `Bot` or `Bearer` prefix. User tokens only.

```javascript
async function api(path, opts = {}) {
    const h = { Authorization: S.token };
    if (opts.json !== undefined) {
        h['Content-Type'] = 'application/json';
        opts.body = JSON.stringify(opts.json);
        delete opts.json;
    }
    return fetch(API + path, { ...opts, headers: h });
}
```

**Session object:** `S = { token, user }` — set after successful login, `null` when logged out.

### Endpoints used

| Feature | Method | Path |
|---------|--------|------|
| Login/auth | GET | `/users/@me` |
| Friends list | GET | `/users/@me/relationships` → filter `type === 1` |
| Remove friend | DELETE | `/users/@me/relationships/{uid}` |
| Open DM channel | POST | `/users/@me/channels` body `{recipients:[uid]}` |
| Servers list | GET | `/users/@me/guilds?limit=200` |
| Leave server | DELETE | `/users/@me/guilds/{gid}` |
| Mute server | PATCH | `/users/@me/guilds/{gid}/settings` |
| DM list | GET | `/users/@me/channels` → filter `type 1 or 3` |
| Close DM | DELETE | `/channels/{cid}` |
| Mute DM | PATCH | `/users/@me/guilds/@me/settings` |
| Fetch messages | GET | `/channels/{cid}/messages?limit=100&before={id}` |
| Delete message | DELETE | `/channels/{cid}/messages/{msgId}` |
| Search guild msgs | GET | `/guilds/{gid}/messages/search?author_id={uid}&limit=25` |

**Mute server body:**
```json
{ "muted": true, "mute_config": { "selected_time_window": -1, "end_time": null } }
```
Unmute: `{ "muted": false, "mute_config": null }`

**Mute DM body:**
```json
{ "channel_overrides": [{ "channel_id": "{cid}", "muted": true, "mute_config": {...} }] }
```

### Rate limit handling

```javascript
if (r.status === 429) {
    const d = await r.json();
    await sl((d.retry_after || 1) * 1000 + 300);
    continue; // retry
}
```

### Snowflake → creation date

```javascript
function snowDate(id) {
    return new Date(Number(BigInt(id) >> 22n) + 1420070400000)
        .toLocaleDateString('pt-BR', { day: '2-digit', month: 'short', year: 'numeric' });
}
```

### Avatar URL

```javascript
function avUrl(uid, hash, sz = 80) {
    if (!hash) return `https://cdn.discordapp.com/embed/avatars/${Number(BigInt(uid) >> 22n) % 6}.png`;
    return `https://cdn.discordapp.com/avatars/${uid}/${hash}.${hash.startsWith('a_') ? 'gif' : 'png'}?size=${sz}`;
}
```

### Guild icon URL

```javascript
function icUrl(gid, hash, sz = 64) {
    if (!hash) return '';
    return `https://cdn.discordapp.com/icons/${gid}/${hash}.${hash.startsWith('a_') ? 'gif' : 'png'}?size=${sz}`;
}
```

### Guild message search response format

```javascript
data.messages.flat().filter(m => m.author?.id === S.user.id)
// each hit also has m.channel_id for cross-channel deletion
```

---

## CSS Architecture

### Color variables (`:root`)

```css
--bg0: #050507  /* deepest background */
--bg1: #0b0b12  /* sidebar background */
--bg2: #101018  /* card/surface */
--bg3: #161622  /* input background, nested surface */
--bg4: #1c1c2c  /* hover states, skeleton base */
--bg5: #232335  /* borders active, badge bg */
--bd:  #262638  /* default border */
--bd2: #30304a  /* elevated border */

--tx:  #eaeaf4  /* primary text */
--tx2: #8888a8  /* secondary text */
--tx3: #484868  /* muted/placeholder text */

--ac:    #5865f2  /* Discord blue — primary accent */
--ac-dk: #4752c4  /* accent hover */
--ac-lt: #7983f5  /* accent light */
--ac-bg: #0c0e24  /* accent tinted background */
--ac-bd: #1e2260  /* accent tinted border */
--ac-tx: #a0aaff  /* accent text on dark bg */

--rd: #e84244 / --rd-dk / --rd-bg / --rd-bd / --rd-tx  /* red/danger */
--am: #f5a623 / --am-dk / --am-bg / --am-bd / --am-tx  /* amber/warning/mute */
--gn: #3ba55c / --gn-dk / --gn-bg / --gn-bd / --gn-tx  /* green/success */
```

### Button classes

| Class | Use |
|-------|-----|
| `.btn.btn-primary` | Main CTA (Enter) — Discord blue |
| `.btn.btn-danger` | Destructive (Delete messages tab) — Red |
| `.btn.btn-ghost` | Secondary/neutral |
| `.btn-full` | `width: 100%` modifier |
| `.act.act-danger` | List item: Remove/Leave/Close — red border, red hover fill |
| `.act.act-warn` | List item: Delete messages — amber border, amber hover fill |
| `.act.act-neutral` | List item: Mute/Unmute — neutral, `.muted` → amber |
| `.act.act-info` | List item: Info popover trigger — ghost, blue on hover |

### List item structure

```html
<div class="li" style="--di:{index}">         <!-- stagger animation via CSS var -->
    <div class="li-av [sq] [ini]">             <!-- avatar: sq=square, ini=initials fallback -->
        <img src="..." onerror="...ini fallback...">
    </div>
    <div class="li-inf">
        <div class="li-nm">Name</div>
        <div class="li-sub"><i class="bi bi-hash"></i>ID</div>
    </div>
    <div class="li-acts">
        <button class="act act-info"    data-a="inf-f|inf-s" data-uid|data-gid="...">
        <button class="act act-danger"  data-a="rmf|lv|cdm"  ...>
        <button class="act act-warn"    data-a="dmf|dgs|ddm" ...>
        <button class="act act-neutral" data-a="mg|mdm"       ...>
    </div>
</div>
```

List items animate in with staggered delay using `--di` CSS custom property:
```css
.li { animation: fadeUp .22s ease both; animation-delay: calc(var(--di, 0) * 0.03s); }
```

---

## JavaScript Architecture

### Global state

```javascript
const API = 'https://discord.com/api/v10';
let S = null;       // { token, user } — null when logged out
let stop = false;   // cancellation flag for long operations
const friendMap = new Map();  // uid → relationship object
const serverMap = new Map();  // gid → guild object
```

### Event delegation pattern

All list item actions use `data-a` attributes, dispatched by a single click listener:

```javascript
document.addEventListener('click', async e => {
    const b = e.target.closest('[data-a]');
    if (!b || !S) return;
    switch (b.dataset.a) {
        case 'rmf':  rmFriend(uid, row);       break;
        case 'dmf':  delFriendMsgs(uid, name); break;
        case 'lv':   leaveGuild(gid, name, row); break;
        case 'dgs':  delGuildMsgs(gid, name);  break;
        case 'mg':   muteGuild(gid, b);        break;
        case 'cdm':  closeDM(cid, row);        break;
        case 'ddm':  delDMMsgs(cid, name);     break;
        case 'mdm':  muteDM(cid, b);           break;
    }
});
```

Popover triggers use `mouseenter`/`mouseleave` via capture phase:
- `data-a="inf-f"` → friend popover (uses `friendMap`)
- `data-a="inf-s"` → server popover (uses `serverMap`)

### Info popover system

A single `#pop` div is repositioned on demand:

```javascript
function showPop(trigger, html) {
    pop.innerHTML = html;
    pop.classList.add('on');
    requestAnimationFrame(() => {
        // Position right of trigger, flip left if viewport overflow
        // Clamp top to viewport bounds
    });
}
function hidePop() { /* delayed hide with clearTimeout guard */ }
```

Popover stays open when mouse moves onto it (mouseenter on `#pop` cancels hide timer).

### Progress overlay

```javascript
pShow('Title', 'Subtitle')    // show overlay
pUpd(done, total, statusText) // update progress bar (total=0 → indeterminate)
pHide()                        // hide overlay
stop = true                    // set by cancel button, checked in loops
```

### Message deletion loop

```javascript
async function delInCh(cid) {
    let del = 0, before = null, done = false;
    while (!done && !stop) {
        // fetch 100 messages, filter own, delete each with 1200ms delay
        // handle 429 with retry_after
        // paginate via before= param
    }
}
```

### Skeleton / empty / error states

```javascript
skels(n)           // returns n skeleton row HTML strings
emptyH(icon, ttl, sub)  // empty state HTML
errH(msg)          // error state HTML
```

---

## Responsive Breakpoints

- **≤ 820px**: Sidebar becomes horizontal top bar. `.sb-scroll` goes `flex-direction: row`. Profile shows only avatar + name. Nav items stack icon+label vertically. Action button text hidden (icon only).
- **≤ 480px**: List items wrap; action buttons move to second line indented under avatar.

---

## Design Decisions & Constraints

- **No `backdrop-filter` / blur / box-shadow / glow / neon** — strictly flat design.
- **No Nitro/Free/Premium references** anywhere.
- **Font:** Fira Code exclusively (monospace aesthetic).
- **Accent:** Discord blue `#5865f2` — used for active nav, primary buttons, badges.
- **Button color coding:** Blue=login, Red=destructive, Amber=mute/delete-msgs, Green=positive toasts.
- **Avatars:** Animated detection via `hash.startsWith('a_')` → use `.gif` extension.
- **XSS prevention:** All user-supplied strings passed through `esc()` before insertion into HTML.
- **No `confirm()` skipping** — destructive actions (remove friend, leave server, close DM) always `confirm()` before proceeding.
- **Token security:** Token never logged, never stored beyond `S.token` in memory.

---

## Sidebar Profile Layout

Name and handle are displayed **inline next to the avatar** (not below it), inside `.sb-av-row > .sb-id-wrap`. Bio is not shown in the sidebar. Meta grid is a **single-column flex list** (not 2-col grid) where each `.sb-mi` is a flex row with label on the left and value on the right — this ensures ID and date display without truncation.

## Diversos Tab

Three bulk-action cards at `#tp-diversos`:
- **Remover Todos os Amigos** (`btn-rm-all-f`): double-confirm, iterates `friendMap`, DELETE each, 500ms delay
- **Sair de Todos os Servidores** (`btn-lv-all-s`): double-confirm, skips `gd.owner === true`, 600ms delay
- **Silenciar Todos os Servidores** (`btn-mute-all-s`): single confirm, requires `serverMap` to be populated (load Servidores tab first), 400ms delay

All three show the progress overlay with cancellation support. The "Servidores" and "Amigos" operations require the respective tabs to have been loaded first (data lives in `friendMap`/`serverMap`).

## Info Popovers — What's Shown

| Popover | Fields shown |
|---------|-------------|
| Friend  | Name, @username, account creation date, friendship date, nickname (if set) |
| Server  | Icon, name, owner/member badge, creation date |

ID and Features were intentionally removed from both popovers per user request.

## Pending / Not Yet Built

- **Webhook tab** — placeholder "Em breve" only. Full send/manage webhook functionality not implemented.
- **Guild member counts** — `/users/@me/guilds` doesn't return counts; `/guilds/{id}?with_counts=true` would need separate per-guild calls (rate-limit-heavy). Not fetched.
- **Friend online status** — REST API doesn't expose presence; would require Gateway (WebSocket). Not implemented.
- **Bulk message delete** — Discord only allows bulk delete for bots; user tokens must delete one-by-one with 1200ms delay.
