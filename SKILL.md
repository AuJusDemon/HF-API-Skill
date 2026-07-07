---
name: hf-api
description: HackForums API v2 skill — OAuth2 flow, write-action safety rules (bytes transfers, thread bumps, contract approvals), and endpoint reference for building or debugging anything that calls hackforums.net's API. Use whenever building, modifying, or reasoning about an HF API integration (bots, dashboards, trading tools, autobumpers, contract trackers).
---

# HackForums API v2

Docs: https://apidocs.hackforums.net/ · Swagger: https://swagger.hackforums.net/

## Core mechanics

Base URL `https://hackforums.net/api/v2`. `/read` and `/write`, POST only. Header `Authorization: Bearer ACCESS_TOKEN`. Body is an `asks` object: `_underscore` fields are inputs, plain fields are outputs (`true`). All response values are strings, cast explicitly. Single result = dict, multiple = list:
```python
rows = data.get("contracts")
if isinstance(rows, dict): rows = [rows]
```
Transport: form-encoded (`data={"asks": json.dumps(body)}`) and raw JSON (`json={"asks": body}`) both confirmed working (tested 2026-07-05). Required headers, missing → CF 403 on first call:
```python
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Accept": "application/json, text/plain, */*",
    "Accept-Language": "en-US,en;q=0.9",
}
```

## Write safety rules

1. Bytes send / vault deposit / vault withdraw / sigmarket `buy` — confirm amount + recipient before calling.
2. Thread bump: HF hard limit 6h minimum between bumps on a thread (site rule). Software floor 300s (`MIN_BUMP_GAP_SECS`) only guards against double-fire on restart/DB glitch/race condition, unrelated to the 6h rule. Live-read thread (`lastpost`, `lastposteruid`, `numreplies`) immediately before bumping; read failure → skip and log, never bump on stale data. Post-bump, `lastposteruid` = the Stanley bump-bot account, not the thread owner.
3. Contract `approve`/`complete`/`cancel` — confirm terms + counterparty uid first. No undo for `approve`.
4. No delete-post/thread, no prefix write — posting is permanent.
5. Rate limit ~240/hr/token via `x-rate-limit-remaining`. Background-polling floor ~20-30 remaining.
6. Reads: unattended OK. Writes: require a confirmation gate.

## OAuth2

1. `GET https://hackforums.net/api/v2/authorize?response_type=code&client_id=CLIENT_ID&state=OPTIONAL_STATE`
2. Redirect back: `https://YOUR_REDIRECT_URI/?code=CODE&state=STATE`
3. Token exchange, same `/authorize` URL:
```bash
curl -X POST 'https://hackforums.net/api/v2/authorize' \
  -H 'content-type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=authorization_code' \
  --data-urlencode 'client_id=YOUR_CLIENT_ID' \
  --data-urlencode 'client_secret=YOUR_SECRET_KEY' \
  --data-urlencode 'code=AUTHORIZATION_CODE'
```
Returns `access_token`, `expires_in`, `refresh_token` (usually), `uid` (sometimes). Call `/me` to confirm uid.
4. Refresh: same endpoint, `grant_type=refresh_token` + `client_id`, `client_secret`, `refresh_token`. Failure → mark token dead, require re-login.

App registration: `usercp.php?action=apideveloper`. Vendors auto-approved; others reviewed, notified by PM. Scope increase requires re-auth. Never expose `client_secret`/tokens client-side.

## Scopes

Basic Info: `uid`, `username`, `usergroup`, `bytes`, `vault`, etc.
Advanced Info: `unreadpms`, `invisible`, `totalpms`, `lastactive`, `warningpoints`.
Posts: forums, threads, posts, read + optional write.
Users: public info of other members.
Bytes: byte logs, read + optional write (transfers, deposits, withdrawals, bumps).
Contracts: contracts, disputes, b-ratings.

## Endpoints

Generic: `POST /read`, `POST /write`.

| Method | Path |
|---|---|
| POST | `/read/me` |
| POST | `/read/users` |
| POST | `/read/forums` |
| POST | `/read/threads` |
| POST | `/write/threads` |
| POST | `/read/posts` |
| POST | `/write/posts` |
| POST | `/read/bytes` |
| POST | `/write/bytes` |
| POST | `/write/bytes/deposit` |
| POST | `/write/bytes/withdraw` |
| POST | `/write/bytes/bump` |
| POST | `/read/contracts` |
| POST | `/read/disputes` |
| POST | `/read/bratings` |
| POST | `/read/sigmarket/market` |
| POST | `/read/sigmarket/order` |

No helper path for contract/dispute/b-rating writes. Contract writes: generic `POST /write` only (see Write: contracts below). No b-rating write endpoint exists.

## Read: me

Scope: Basic Info (Advanced for private fields)

| Field | Scope | Notes |
|---|---|---|
| `uid`, `username`, `usergroup`, `displaygroup`, `additionalgroups` | Basic | |
| `postnum`, `threadnum`, `awards` | Basic | |
| `bytes` | Basic | Can be absent on success — fallback: `users._uid` → `myps` |
| `vault` | Basic | API Client Vault balance |
| `avatar`, `avatardimensions`, `avatartype` | Basic | Relative path — prefix `https://hackforums.net/`, strip leading `./` |
| `lastvisit`, `usertitle`, `website`, `timeonline`, `reputation`, `referrals` | Basic | |
| `lastactive`, `unreadpms`, `invisible`, `totalpms`, `warningpoints` | Advanced | Absent (not zero/null) without Advanced Info scope |

`avatardimensions` → `"120|120"` (width\|height). `additionalgroups` → comma-separated. `usergroup` `"7"`/`"38"` = self-exile/self-ban.

```python
data = await hf.read({"me": {"uid": True, "username": True, "bytes": True, "vault": True, "unreadpms": True}})
```

## Read: users

Scope: Users. Input: `_uid` [ints].

Fields: `uid`, `username`, `usergroup`, `displaygroup`, `additionalgroups`, `postnum`, `threadnum`, `awards`, `myps`, `avatar`, `avatardimensions`, `avatartype`, `usertitle`, `website`, `timeonline`, `reputation`, `referrals`

`myps` = `me.bytes`. Advanced fields not available here.

```python
data = await hf.read({"users": {"_uid": [uid_int], "uid": True, "username": True, "myps": True}})
```

## Read: forums

Scope: Posts. Input: `_fid` [array]. Fields: `fid`, `name`, `description`, `type`.

`type`: `"f"` = forum (has threads), `"c"` = category (never has threads — used as `_fid` in a thread query, returns empty).

Category fids: `1, 7, 45, 88, 105, 120, 141, 151, 156, 241, 259, 444, 445, 446, 447, 448, 449, 450, 451, 452, 453, 460`

```python
data = await hf.read({"forums": {"_fid": [fid_int], "fid": True, "name": True, "type": True}})
```

## Read: threads

Scope: Posts. Inputs: `_tid` [array] / `_fid` [array] / `_uid` [array, paginated].

Fields: `tid`, `uid`, `fid`, `subject`, `closed`, `numreplies`, `views`, `dateline`, `firstpost`, `lastpost`, `lastposter`, `lastposteruid`, `prefix`, `icon`, `poll`, `username`, `sticky`, `bestpid`

`_uid` page 1 = newest first. No total-count field — page until empty, caps ~20-30 rows/page. tid lookup 200 + missing resource = inconclusive, not deletion proof. `numreplies` not reliable in every `_uid` context.

```python
data = await hf.read({"threads": {"_uid": [uid_int], "_page": 1, "_perpage": 30,
                                   "tid": True, "subject": True, "lastpost": True, "numreplies": True}})
```

## Read: posts

Scope: Posts. Inputs: `_pid` [array] / `_tid` [array, paginated] / `_uid` [array, paginated].

Fields: `pid`, `tid`, `uid`, `fid`, `dateline`, `message`, `subject`, `edituid`, `edittime`, `editreason`

Embed author: `"posts": {"_tid": [TID], "message": True, "author": {"uid": True, "username": True}}`. `_uid`: oldest-first, excludes own thread OPs — combine with `threads.firstpost` pids, last page ≈ `(postnum-threadnum)/30`. `_tid`: also oldest-first, same math via `numreplies`.

## Read: bytes

Scope: Bytes. Inputs: `_id` / `_uid` / `_from` / `_to` (arrays, ints, paginated max 30).

Fields: `id`, `amount`, `dateline`, `type`, `reason`

`amount`: float string (`"430.43"`), cast `int(float(x))`. Direction: separate `_from` (sent) / `_to` (received) calls. `from`/`to`: unconfirmed — one integration reports embedded fields never return even when requested; another reports `bytes.from`/`bytes.post` return as lists (first element raw id or nested dict). Normalize for absent/scalar/list. `_from`/`_to`/`_uid`: integer UIDs required — string UID → silent empty result.

`reason` is freeform text typed by the sender at send time, not a fixed enum — don't treat any value in it as guaranteed.

```python
sent     = await hf.read({"bytes": {"_from": [uid_int], "_page": 1, "_perpage": 30,
                                     "id": True, "amount": True, "dateline": True, "reason": True, "type": True}})
received = await hf.read({"bytes": {"_to": [uid_int], "_page": 1, "_perpage": 30,
                                     "id": True, "amount": True, "dateline": True, "reason": True, "type": True}})
```

### Bytes type codes

| Code | Dir | Description |
|---|---|---|
| `att` | OUT | Manual bytes send |
| `bla` | IN | Blackjack winner |
| `bon` | IN | Bonus |
| `bum` | OUT | Thread bump fee |
| `cfl` | OUT | Coin flips loser |
| `cfw` | IN | Coin flips winner |
| `cgp` | OUT | Crypto game coin purchase |
| `cgs` | IN | Crypto game coin sell |
| `cvr` | IN | Convo rain |
| `don` | IN/OUT | Peer-to-peer send/receive. `reason` is whatever the sender set. |
| `gce` | OUT | Bytes to game cash exchange |
| `ltb` | OUT | Lottery ticket purchase |
| `qlc` | IN | Quick love convo |
| `qlp` | IN/OUT | Quick love post |
| `sbs` | IN | Sportsbook winner |
| `sbw` | OUT | Sportsbook wager |
| `sbc` | IN | Sportsbook cancel/refund |
| `scp` | OUT | Scratch card purchase |
| `slo` | IN | Slots winner |
| `ugb` | IN | Upgrade bonus |

## Read: contracts

Scope: Contracts. Inputs: `_cid` [array] / `_uid` [array, paginated].

Fields: `cid`, `dateline`, `otherdateline`, `public`, `timeout_days`, `timeout`, `status`, `type`, `istatus`, `ostatus`, `muid`, `inituid`, `otheruid`, `iprice`, `icurrency`, `iproduct`, `oprice`, `ocurrency`, `oproduct`, `iaddress`, `oaddress`, `terms`, `tid`, `idispute`, `odispute`

`idispute`/`odispute` embedded — no separate `/disputes` call needed for tracked contracts. `iaddress`/`oaddress`: role-labeled (initiator/counterparty), absent until approval. `status` 0/1 (awaiting approval): never auto-expired by HF. Stale heuristic used in practice: ~90 days since last update. All values numeric strings.

### Contract status map — two integrations disagree, unconfirmed which is current

| Value | Label A | Label B |
|---|---|---|
| `"0"` | Awaiting Approval | — |
| `"1"` | Awaiting Approval | Awaiting approval |
| `"2"` | Cancelled | Cancelled |
| `"3"` | Unknown (likely middleman escrow) | — |
| `"4"` | Unknown (likely middleman escrow) | — |
| `"5"` | Active Deal | Active |
| `"6"` | Complete | Complete |
| `"7"` | Disputed | — |
| `"8"` | Expired | Incomplete |

`istatus`/`ostatus`: per-party ready flag (`"0"`/`"1"`), independent of `status`. `status="0"` == `status="1"` (both awaiting).

### Contract type map

| Value | Label |
|---|---|
| `"1"` | Selling |
| `"2"` | Purchasing |
| `"3"` | Exchanging |
| `"4"` | Trading |
| `"5"` | Vouch Copy |

`type` = initiator's position at creation.

### Contract value fallback

```python
def contract_value(c: dict) -> str:
    iprice, icur = c.get("iprice", "0"), c.get("icurrency", "other")
    oprice, ocur = c.get("oprice", "0"), c.get("ocurrency", "other")
    iproduct, oproduct = c.get("iproduct", ""), c.get("oproduct", "")
    if iprice != "0" and icur.lower() != "other":
        return f"{iprice} {icur}"
    if oprice != "0" and ocur.lower() != "other":
        return f"{oprice} {ocur}"
    if iproduct not in ("", "other", "n/a"):
        return iproduct
    if oproduct not in ("", "other", "n/a"):
        return oproduct
    return ""
```

Contract URL: `https://hackforums.net/contracts.php?action=view&cid=CID`

## Read: bratings

Scope: Contracts. Inputs: `_crid` / `_cid` / `_uid` / `_from` / `_to` (arrays).

Fields: `crid`, `contractid`, `fromid`, `toid`, `dateline`, `amount`, `message`, `contract` (embedded), `from`/`to` (embedded)

Read-only. `amount` = rating value (`1`/`-1`), not bytes. `_to:[uid]` = received. `_from:[uid]` = left by uid. "Needs rating": use `_from` — no row with `fromid == my_uid` for that `contractid`. Embedded `from`/`to` on paginated `_to`/`_from` call → `"Users - Maximum of 30 users allowed."` Use flat `fromid`/`toid`.

```python
data = await hf.read({"bratings": {"_to": [uid_int], "_page": 1, "_perpage": 30,
                                    "crid": True, "fromid": True, "amount": True, "message": True}})
```

## Read: disputes

Scope: Contracts. Inputs: `_cdid` / `_cid` / `_uid` / `_claimantuid` / `_defendantuid` (arrays).

Fields: `cdid`, `contractid`, `claimantuid`, `defendantuid`, `dateline`, `status`, `dispute_tid`, `claimantnotes`, `defendantnotes`, `contract`/`claimant`/`defendant`/`dispute_thread` (embedded)

```python
data = await hf.read({"disputes": {"_cid": [cid_int], "cdid": True, "status": True, "claimantnotes": True, "defendantnotes": True}})
```

## Read: sigmarket

**market** — `_type="market"`, `_uid` [array], paginated. Fields: `uid`, `user` (embedded), `price`, `duration`, `active`, `sig`, `dateadded`, `ppd`

**order** — `_type="order"`, `_smid`/`_uid`/`_seller`/`_buyer` [arrays], paginated. Fields: `smid`, `startdate`, `enddate`, `price`, `duration`, `active`

Embedded `buyer`/`seller` on paginated order calls → `"Users - Invalid User ID Supplied."` Parallel same-token calls: one integration reports 503/challenge under parallel market+order calls (ran sequentially); another parallelizes successfully with per-call exception isolation.

```python
market = await hf.read({"sigmarket": {"_type": "market", "_uid": [uid_int], "_page": 1, "_perpage": 1,
                                       "uid": True, "price": True, "active": True, "sig": True}})
orders = await hf.read({"sigmarket": {"_type": "order", "_seller": [uid_int], "_page": 1, "_perpage": 30,
                                       "smid": True, "active": True, "startdate": True, "enddate": True}})
```

## Write: posts

Scope: Posts Write
```python
{"posts": {"_tid": TID, "_message": "BBCode content"}}
```
Returns `pid`, `tid`, `uid`, `message`.

## Write: threads

Scope: Posts Write
```python
{"threads": {"_fid": FID, "_subject": "Title", "_message": "Body"}}
```
Returns `tid`, `uid`, `subject`, `dateline`, `firstpost` (`pid`, `message`). No `_prefix` param.

## Write: bytes

Scope: Bytes Write
```python
{"bytes": {"_uid": "UID", "_amount": "100", "_reason": "Payment"}}
{"bytes": {"_deposit": 500}}
{"bytes": {"_withdraw": 500}}
{"bytes": {"_bump": TID}}
```
Min deposit/withdraw: 100.

## Write: contracts

Scope: Contracts Write

| `_action` | Notes |
|---|---|
| `new` | Requires `_uid`, `_terms`, `_position` (`selling`/`buying`/`exchanging`/`trading`/`vouchcopy`). Optional: `_yourproduct`, `_yourcurrency`, `_youramount`, `_theirproduct`, `_theircurrency`, `_theiramount`, `_tid`, `_muid`, `_timeout`, `_public` |
| `undo` | Undo a contract you just created |
| `deny` | Deny as counterparty |
| `approve` | Approve as counterparty |
| `cancel` | Request cancellation — both parties must submit |
| `complete` | Mark your side complete |
| `middleman_deny` / `middleman_approve` | Middleman actions |

`new` reliability constraints known — test exact `_position`/currency combination.

```python
{"contracts": {"_action": "new", "_uid": OTHER_UID, "_terms": "Terms text",
               "_position": "selling", "_yourproduct": "Item", "_youramount": "100"}}
{"contracts": {"_action": "approve", "_cid": CID}}  # _cid as target field, not independently confirmed
```

## Write: sigmarket

```python
{"sigmarket": {"_type": "setsale", "_price": BYTES, "_duration": DAYS}}
{"sigmarket": {"_type": "removesale"}}
{"sigmarket": {"_type": "changesig", "_smid": SMID_OR_"all", "_sig": "new BBCode"}}
{"sigmarket": {"_type": "buy", "_uid": UID, "_price": MAX_PRICE}}
```

## Errors, rate limits, revocation

| Signal | Meaning |
|---|---|
| `401` | Missing/invalid/expired token |
| HTTP 200 + JSON `message` in `TOKEN_DISABLED`/`TOKEN_EXPIRED`/`TOKEN_INVALID`/`TOKEN_REVOKED`/`INVALID_TOKEN` | Treat as 401 |
| HTTP 200 + HTML body (starts `<!`) | CF challenge — check `Content-Type`/body prefix before parsing. Don't retry immediately, back off substantially or serve stale cache |
| `503` | Usually >4 endpoints per call; can also be transient. One retry after a few seconds max |
| `403`/`502` | Transient, small number of backoff retries |
| Datacenter/VPS source IP | Can be CF-blocked outright (not rate-limited) for non-relay integrations — residential proxy fixes this |

Rate limit: ~240/hr/token via `x-rate-limit-remaining`. Revocation: user-initiated via Authorized Apps page — detect via 401/token-error, prompt re-auth.

## Usergroups

```python
is_banned = usergroup in ("7", "38") or displaygroup in ("7", "38")
```

Core groups (official HF system groups):

| ID | Label |
|---|---|
| `"2"` | Registered |
| `"9"` | L33t |
| `"28"` | Ub3r |
| `"67"` | Vendor |
| `"7"` | Exiled |
| `"38"` | Banned |

Member-owned groups (community factions):

| ID | Label |
|---|---|
| `"46"` | H4CK3R$ |
| `"68"` | Brotherhood |
| `"48"` | Quantum |
| `"52"` | PinkLSZ |
| `"78"` | VIBE |
| `"70"` | Gamblers |
| `"50"` | Legends |
| `"77"` | Academy |
| `"71"` | Warriors |

## Batching

Max 4 endpoint keys per `read()` call — 5th → silent 503. Endpoint keys unique per call. `_page`/`_perpage` independent per endpoint, `_perpage` max 30 (higher returns empty). `me` always returns exactly 1 result.

```python
# Call 1 [4/4 slots]
data1 = await hf.read({
    "me":        {"uid": True, "bytes": True, "vault": True, "unreadpms": True},
    "bytes":     {"_to": [uid_int], "_page": 1, "_perpage": 30,
                  "id": True, "amount": True, "dateline": True, "reason": True, "type": True},
    "contracts": {"_uid": [uid_int], "_page": 1, "_perpage": 30,
                  "cid": True, "status": True, "type": True,
                  "iproduct": True, "oproduct": True, "iprice": True, "icurrency": True,
                  "oprice": True, "ocurrency": True, "dateline": True},
    "threads":   {"_uid": [uid_int], "_page": 1, "_perpage": 30,
                  "tid": True, "subject": True, "lastpost": True,
                  "lastposteruid": True, "numreplies": True},
})

# Call 2 [1/4 slots — "bytes" key already used in call 1]
data2 = await hf.read({
    "bytes": {"_from": [uid_int], "_page": 1, "_perpage": 30,
              "id": True, "amount": True, "dateline": True, "reason": True, "type": True},
})
```

```python
# recent post activity: posts._uid oldest-first, last_page ≈ (postnum-threadnum)/30
import math
reply_count = max(0, int(user["postnum"]) - int(user["threadnum"]))
last_page = max(1, math.ceil(reply_count / 30))
data = await hf.read({
    "posts": {"_uid": [uid], "_page": last_page, "_perpage": 30,
              "pid": True, "tid": True, "dateline": True, "message": True}
})
```

```python
# all contracts for a user, paginated
page, all_contracts = 1, []
while True:
    resp = await hf.read({"contracts": {"_uid": [uid], "_page": page, "_perpage": 30,
                                         "cid": True, "status": True, "type": True}})
    rows = resp.get("contracts", [])
    if isinstance(rows, dict): rows = [rows]
    if not rows: break
    all_contracts.extend(rows)
    if len(rows) < 30: break
    page += 1
```

```python
# resolve uids to usernames
data  = await hf.read({"users": {"_uid": uid_list, "uid": True, "username": True}})
rows  = data.get("users", [])
if isinstance(rows, dict): rows = [rows]
names = {str(r["uid"]): r["username"] for r in rows}
```

```python
# b-rating history
data = await hf.read({"bratings": {"_to": [uid], "_page": 1, "_perpage": 30,
                                    "crid": True, "amount": True, "message": True, "dateline": True}})
```

```python
# newest posts in a thread: posts._tid oldest-first, last_page via numreplies
import math
last_page = max(1, math.ceil((int(numreplies) + 1) / 30))
data = await hf.read({
    "posts": {"_tid": [tid], "_page": last_page, "_perpage": 30,
              "pid": True, "uid": True, "dateline": True, "message": True}
})
```

```python
# unread PMs, requires Advanced Info scope
data  = await hf.read({"me": {"unreadpms": True}})
count = int(data["me"].get("unreadpms") or 0)
```

Thread reply detection: `threads._uid` gives `lastpost`/`lastposteruid` free in any call. Compare `lastpost` to stored cursor; if changed and `lastposteruid != your_uid`, fetch posts via `numreplies` page math.

Rate budget (1 active user): full dashboard refresh = 2 calls. Reply detection (30 threads) = 0 extra. Fetch posts for a thread with new replies = 1-3 calls. Username batch resolution = 1 call/batch. Autobump cycle (per 4 threads) = 1 read + 1-2 writes per bump.

Reading from local DB/cache and client-side filtering costs nothing against the rate limit. Prefer batching, caching, and eliminating redundant calls over increasing polling frequency.

## Known limitations

No prefix write. No b-rating write. No delete-post/delete-thread. No reliable `contracts` `new` for every case. No guaranteed `from`/`to` counterparty resolution on `bytes`. Advanced Info scope required for `unreadpms` etc. (absent, not zero, without it). Category fids → empty on `_fid` thread queries. Contracts awaiting approval never auto-expire. `threads._uid` surfaces up to ~30 most recently active threads — quiet older threads fall out of view.

## Resources

Swagger UI: https://swagger.hackforums.net/
PHP example (Xerotic): https://github.com/xerotic/HF-API-v2
Python integration (ENKI): https://github.com/LNodesL/HF-API-Python
HFToolbox: https://github.com/AuJusDemon/HFToolbox
