Build me a fully working PAPER-TRADING bot that trades X (Twitter) sentiment against Polymarket's crypto "Up or Down" markets. Set it up end to end, test it, then run it on a schedule until the end date below.

⚠️ PAPER ONLY. Never place real orders, never connect a wallet, never ask me for keys, funds, or a Polymarket login. Polymarket is used only as a read-only price and resolution source (public gamma API, no auth). If anything in this setup would need real money, stop and ask me first.

SETTINGS (edit these):
- Assets: btc, eth            (Polymarket slug prefixes; sol and xrp also have hourly markets)
- Timeframe: 1h               (1h or 15m. 15m needs 4x the runs and usage, and X sentiment is thin per window)
- Start bankroll: $100
- Stake per trade: 10% of current cash
- Max entry price: 0.60       (never buy above this; your win rate must beat your average entry)
- Min directional posts: 6
- Min |score|: 0.4
- Posts to read per asset per run: up to 25, from the last 60 minutes (last 15 for 15m)
- My timezone: <e.g. Asia/Tehran>
- End date: <e.g. 2026-12-08 19:00 in my timezone>

HOW IT WORKS
Every window (on the hour for 1h), for each asset: read recent X posts about the coin, tag each one bullish, bearish or neutral, compute score = (bullish − bearish) / (bullish + bearish), and if there are enough directional posts and the score is strong enough, paper-buy that side ("up" if score > 0, "down" if score < 0) of the Polymarket market for the window that just started, at the live ask. Skip if the ask is above the max entry price. Each share pays $1 if it resolves your way, $0 otherwise. Trades settle on Polymarket's own resolution.

Polymarket market lookup (read-only):
- URL: https://gamma-api.polymarket.com/events?slug={asset}-updown-{timeframe}-{ts}, where ts = window start in unix seconds, UTC (e.g. btc-updown-1h-1795471200).
- markets[0].outcomePrices = [Up, Down]; Up ask = bestAsk; Down ask = 1 − bestBid.
- Resolved when closed is true or one price is ≥ 0.99.

STEP 1: Create the ledger script
On your box, create the folder ~/polybot and save this exact file as ~/polybot/pb.py. Then create ~/polybot/config.json from my SETTINGS (keys: assets, timeframe, start_bankroll, stake_pct, max_entry, min_posts, min_score, end_date). The script creates a default config.json if it's missing.

```python
#!/usr/bin/env python3
"""Paper-trading ledger for Polymarket crypto "Up or Down" markets (PAPER ONLY, never places orders).

Commands:
  pb.py quote                                   live Up/Down ask for each asset's current window
  pb.py trade ASSET SIDE SCORE N_POSTS "NOTE"   paper-buy SIDE (up|down) in ASSET's current window
  pb.py skip  ASSET SCORE N_POSTS "NOTE"        log a no-trade check
  pb.py settle                                  settle open trades whose markets have resolved
  pb.py status                                  bankroll and stats (JSON)
Settings live in config.json next to this file (created with defaults on first run).
"""
import json, os, sys, time, datetime, urllib.request

D = os.path.dirname(os.path.abspath(__file__))
LEDGER, CONFIG = os.path.join(D, "ledger.json"), os.path.join(D, "config.json")
DEFAULTS = {
    "assets": ["btc", "eth"],   # Polymarket slug prefixes, e.g. btc-updown-1h-<ts>
    "timeframe": "1h",          # "1h" or "15m"
    "start_bankroll": 100.0,
    "stake_pct": 0.10,          # fraction of current cash staked per trade
    "max_entry": 0.60,          # never pay more than this per share
    "min_posts": 6,             # min directional (bullish+bearish) posts to trade
    "min_score": 0.4,           # min |score|, score = (bull - bear) / (bull + bear)
    "end_date": "",             # ISO time the test stops, e.g. 2026-12-01T19:00:00+03:00
}
GAMMA = "https://gamma-api.polymarket.com/events?slug="

def cfg():
    if not os.path.exists(CONFIG):
        json.dump(DEFAULTS, open(CONFIG, "w"), indent=1)
    c = dict(DEFAULTS); c.update(json.load(open(CONFIG))); return c

C = cfg()
PERIOD = {"1h": 3600, "15m": 900}[C["timeframe"]]

def now(): return datetime.datetime.now().astimezone().isoformat(timespec="seconds")

def get(url):
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})
    return json.load(urllib.request.urlopen(req, timeout=20))

def load():
    if os.path.exists(LEDGER): return json.load(open(LEDGER))
    return {"start": C["start_bankroll"], "bankroll": C["start_bankroll"], "trades": [], "skips": []}

def save(l):
    tmp = LEDGER + ".tmp"; json.dump(l, open(tmp, "w"), indent=1); os.replace(tmp, LEDGER)

def cur_ts(): return int(time.time()) // PERIOD * PERIOD   # window start, unix seconds (UTC)

def market(asset, ts):
    """Polymarket event for ASSET's window starting at ts, e.g. btc-updown-1h-1794981600."""
    ev = get(f"{GAMMA}{asset}-updown-{C['timeframe']}-{ts}")
    if not ev: return None
    m = ev[0]["markets"][0]
    return {"slug": ev[0]["slug"], "title": ev[0]["title"], "closed": bool(m.get("closed")),
            "prices": [float(x) for x in json.loads(m["outcomePrices"])],   # [Up, Down]
            "bestBid": m.get("bestBid"), "bestAsk": m.get("bestAsk")}

def asks(m):
    """Up ask = bestAsk on the Up token; Down ask = 1 - best Up bid."""
    up = float(m["bestAsk"]) if m["bestAsk"] is not None else None
    down = round(1 - float(m["bestBid"]), 4) if m["bestBid"] is not None else None
    return up, down

def quote():
    ts = cur_ts(); out = {}
    for a in C["assets"]:
        m = market(a, ts)
        if not m: out[a] = "no market found"; continue
        up, down = asks(m)
        out[a] = {"title": m["title"], "slug": m["slug"], "up_ask": up, "down_ask": down, "seconds_into_window": int(time.time()) - ts}
    print(json.dumps(out, indent=1))

def trade(asset, side, score, nposts, note=""):
    side = side.lower(); assert side in ("up", "down"), "side must be up or down"
    l = load(); ts = cur_ts(); m = market(asset, ts)
    if not m or m["closed"]: print("no open market"); return
    up, down = asks(m); price = up if side == "up" else down
    if not price or price <= 0 or price >= 0.97: print("price unusable:", price); return
    base = {"time": now(), "asset": asset, "side": side, "slug": m["slug"], "ts": ts, "entry": price,
            "score": float(score), "posts": int(nposts), "note": note}
    if price > C["max_entry"]:
        base.update(price_capped=True, note=f"PRICE CAP: ask {price} > {C['max_entry']}. {note}")
        l["skips"].append(base); save(l); print(f"skipped: ask {price} above cap {C['max_entry']}"); return
    stake = round(l["bankroll"] * C["stake_pct"], 2)
    if stake <= 0: print("bankroll empty"); return
    l["bankroll"] = round(l["bankroll"] - stake, 2)
    t = {"id": len(l["trades"]) + 1, **base, "title": m["title"], "stake": stake,
         "shares": round(stake / price, 4), "status": "open"}
    l["trades"].append(t); save(l); print(json.dumps(t, indent=1))

def skip(asset, score, nposts, note=""):
    l = load()
    l["skips"].append({"time": now(), "asset": asset, "score": float(score), "posts": int(nposts), "note": note})
    save(l); print("skip logged")

def settle():
    l = load(); settled = []
    for t in l["trades"]:
        if t["status"] != "open" or time.time() < t["ts"] + PERIOD + 60: continue
        m = market(t["asset"], t["ts"])
        if not m: continue
        up, down = m["prices"]
        if not (m["closed"] or max(up, down) >= 0.99): continue   # not resolved yet
        t["result"] = "up" if up > down else "down"
        won = t["result"] == t["side"]
        t["status"] = "won" if won else "lost"
        t["payout"] = round(t["shares"], 2) if won else 0.0     # $1 per winning share
        t["pnl"] = round(t["payout"] - t["stake"], 2)
        t["settled_at"] = now()
        l["bankroll"] = round(l["bankroll"] + t["payout"], 2); settled.append(t)
    save(l)
    print(json.dumps({"settled": [{k: t[k] for k in ("id", "asset", "side", "entry", "stake", "status", "pnl")} for t in settled]}, indent=1))
    status()

def status():
    l = load(); tr = l["trades"]; done = [t for t in tr if t["status"] != "open"]
    opn = [t for t in tr if t["status"] == "open"]; w = sum(t["status"] == "won" for t in done)
    eq = l["start"]; peak = eq; mdd = 0.0
    for t in done:
        eq += t["pnl"]; peak = max(peak, eq); mdd = max(mdd, (peak - eq) / peak)
    def grp(key):
        out = {}
        for t in done:
            g = out.setdefault(t[key], {"trades": 0, "wins": 0, "pnl": 0.0})
            g["trades"] += 1; g["wins"] += t["status"] == "won"; g["pnl"] = round(g["pnl"] + t["pnl"], 2)
        return out
    equity = round(l["bankroll"] + sum(t["stake"] for t in opn), 2)
    print(json.dumps({
        "start": l["start"], "cash": l["bankroll"], "open_trades": [{k: t[k] for k in ("id", "asset", "side", "entry", "stake", "slug")} for t in opn],
        "equity_at_cost": equity, "pct_change": round((equity / l["start"] - 1) * 100, 1),
        "trades": len(tr), "settled": len(done), "wins": w, "losses": len(done) - w,
        "win_rate": round(w / len(done), 3) if done else None,
        "avg_entry": round(sum(t["entry"] for t in done) / len(done), 3) if done else None,
        "realized_pnl": round(sum(t["pnl"] for t in done), 2),
        "peak_bankroll": round(peak, 2), "max_drawdown_pct": round(mdd * 100, 1),
        "by_asset": grp("asset"), "by_side": grp("side"),
        "skips": len(l["skips"]), "price_capped_skips": sum(1 for s in l["skips"] if s.get("price_capped")),
        "config": C}, indent=1))

if __name__ == "__main__":
    a = sys.argv[1:] or ["status"]
    cmd = a[0]
    if cmd == "quote": quote()
    elif cmd == "trade": trade(a[1], a[2], a[3], a[4], a[5] if len(a) > 5 else "")
    elif cmd == "skip": skip(a[1], a[2], a[3], a[4] if len(a) > 4 else "")
    elif cmd == "settle": settle()
    elif cmd == "status": status()
    else: print(__doc__)
```

STEP 2: Test it before scheduling anything
- Run `python3 ~/polybot/pb.py quote` and show me the live Up/Down asks for each asset. If a market isn't found, check the slug format against the gamma API and fix it (Polymarket has renamed slugs before).
- Run `python3 ~/polybot/pb.py status` and confirm the bankroll and config.
- Do one dry sentiment pass now with your X search (read-only public X reads): for each asset, show me the posts you read, how you tagged each one, and the score. Don't log a trade from this dry run.

STEP 3: Create the routine
Create a recurring routine named "X-sentiment paper trader" that runs at minute 0 of every hour (every 15 minutes if timeframe is 15m), in my timezone, with exact timing. Each run does exactly this:

1. Settle: run `python3 ~/polybot/pb.py settle`. For every trade it settles, send me one short message: asset, side, entry price, won/lost, PnL, new W–L record, win rate, bankroll vs the start.
2. Read X: for each asset, search X for posts from the last 60 minutes (15 for 15m) about that coin, e.g. "$BTC OR bitcoin" and "$ETH OR ethereum", latest first, up to 25 posts per asset. English and other languages are fine if you can read the sentiment.
3. Tag each post bullish, bearish or neutral about the coin's price over the next hour. Ignore and don't count: giveaways, airdrops, referral/promo spam, bots repeating the same text, pure news with no direction, posts about other coins, and price-alert bots with no opinion. Count each account once per run.
4. Score = (bullish − bearish) / (bullish + bearish). Directional posts = bullish + bearish.
5. Decide per asset:
   - If directional posts ≥ min_posts AND |score| ≥ min_score: run `python3 ~/polybot/pb.py trade <asset> <up|down> <score> <directional_posts> "<one-line note: counts + 2 example post links>"`. The script applies the max-entry cap itself and logs a price-capped skip if the ask is too high.
   - Otherwise: run `python3 ~/polybot/pb.py skip <asset> <score> <directional_posts> "<short reason>"`.
   Do this quickly: score first, then trade, so the entry lands as close to the window open as possible.
6. Run `python3 ~/polybot/pb.py status`.
7. Notify me only when a trade settled, a new trade opened, or something broke (market not found, X search failing, script error). Otherwise stay quiet. Never place a real order.
8. End date: on the first run at or after the end date, settle what you can, pause the routine, and send me a final summary: trades, wins/losses, win rate, average entry vs win rate, bankroll vs start (% change), peak bankroll and max drawdown, BTC vs ETH and Up vs Down breakdown, skips and price-capped skips, any trade still open. Then ask whether to keep it running.

STEP 4: Confirm
Show me the routine's schedule, its next run time in my timezone, the exact per-run instructions you saved, and the end date. Then tell me it's paper-only.

Notes:
- Public X reads have a daily cap (about 1,000 posts a day in our setup). 25 posts × 2 assets × 24 hourly runs = 1,200, so if you hit the cap, drop to about 20 posts per asset.
- Fills are at the live ask with no fees or slippage, so real results would be somewhat worse.
- This is an experiment, not financial advice. In our week-long test (Nov 17–24, 2026) it went 14–15 and turned $100 into $70.77.
