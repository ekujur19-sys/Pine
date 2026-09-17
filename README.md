# Pine — Nifty 50 / Indian Market Strategies

Five self-contained Pine Script **v6** strategies for NIFTY 50, written for NSE
session hours (IST) with ATR-based risk, session square-off, and
lot-rounded position sizing.

| File | Setup | Best timeframe | Trades/day | Market it needs |
|---|---|---|---|---|
| `strategies/nifty50_opening_range_breakout.pine` | Opening range breakout | 5m | 0–2 | Gap / directional open |
| `strategies/nifty50_vwap_pullback.pine` | VWAP trend pullback | 5m–15m | 0–3 | Trending day |
| `strategies/nifty50_supertrend_momentum.pine` | Supertrend flip out of a squeeze | 5m–15m | 0–3 | Range → expansion |
| `strategies/nifty50_trend_strategy.pine` | EMA cross + ADX, ATR trail | 15m–1h / daily | Few | Sustained trend |
| `strategies/nifty50_5ema_power_of_stocks.pine` | 5 EMA fade (Power of Stocks) | 15m–1h / daily | 0–2 | Overextended move |

They are deliberately different in *character*, not just in indicator. Running
them all on the same day is not diversification — ORB and Supertrend will often
fire on the same move. Pick the one that matches the day type you actually trade.

---

## 1. Opening Range Breakout (ORB)

**Idea.** NIFTY rarely opens flat. The first 15–30 minutes set the day's value
area; the first clean break of that range tends to run because the trapped side
has to cover.

**Rules.**
- Mark high/low of `0915-0930` (or `0915-0945` for fewer, cleaner signals).
- Skip the day if the range is too tight (noise) or too wide (the move already
  happened) — `minRangePct` / `maxRangePct`.
- Enter on a **close** beyond the edge plus a buffer of `buffTicks × range width`.
  The buffer is what kills most false breaks; do not set it to zero.
- Stop at the opposite range edge (default), the midpoint (tighter, more stops),
  or ATR. Target `rrTarget × R`, trailing after 1R.

**Tuning notes.** The opposite-edge stop is wide, so risk-% sizing will hand you
a small quantity — that is correct behaviour, not a bug. If you want size, use
the midpoint stop and accept the lower hit rate. `useBias` (previous day's close)
roughly halves trade count and is worth testing.

---

## 2. VWAP Trend Pullback

**Idea.** Intraday institutional flow anchors to session VWAP. On a trending day
price holds one side of it and retests it repeatedly. Trading the VWAP *cross*
whipsaws; trading the *retest* does not.

**Rules.**
- Regime: price on one side of VWAP **and** VWAP sloping that way by at least
  `minSlope` over `slopeLen` bars, optionally confirmed by a 20 EMA.
- Wait for price to pull back into a band of `touchMul × ATR` around VWAP.
- Enter on a rejection bar back in the trend direction, within `lookback` bars
  of the touch, and only if price is not already `maxStretch × ATR` away from
  VWAP (no chasing).
- Stop beyond the pullback extreme, target `rrTarget × R`, optional hard exit if
  price closes back through VWAP.

**Tuning notes.** `minSlope` is the single most important input — it is what
keeps you out of flat chop days where VWAP is horizontal and every touch fails.
Raise it until the equity curve stops bleeding on range days.

---

## 3. Supertrend Momentum with Squeeze Release

**Idea.** NIFTY compresses, then makes one or two real legs. Sit out the
compression, take the expansion.

**Rules.**
- Squeeze = Bollinger Bands inside Keltner Channels. Wait for the **release**
  (squeeze ends), then accept a Supertrend flip within `sqzGrace` bars.
- RSI must agree with the direction but not be exhausted (`rsiMaxLong` /
  `rsiMinShort` refuse late entries).
- The Supertrend line itself is the stop and the trail; fixed R target on top.

**Tuning notes.** On 5m, `stFactor` 2.0–2.5 is reasonable; below 2.0 you will
flip constantly. Turning off `useSqz` roughly triples trade count and, in most
tests, lowers expectancy.

---

## 4. 5 EMA Strategy (Power of Stocks)

Subhasish Pani's setup, implemented as taught, with the common variations
exposed as toggles rather than baked in.

**Rules.**
- **Sell:** a candle whose **low is entirely above** the 5 EMA. Sell stop at
  that candle's **low**, SL at its **high**.
- **Buy:** a candle whose **high is entirely below** the 5 EMA. Buy stop at
  that candle's **high**, SL at its **low**.
- While the setup lives, each new qualifying candle **replaces** the reference —
  trigger and stop both shift (`shiftRef`, on by default).
- The moment a candle touches the 5 EMA, the setup is dead and the resting
  order is cancelled.
- Default exit is partial at 2R with the runner trailed on the 5 EMA.

**The two settings that change everything.**
- `breakMode` — *Wick* rests a real stop order at the level and fills intrabar,
  which is how it is taught. *Candle close beyond level* waits for confirmation.
  These backtest as two different strategies: wick triggers far more often and
  stops out more; close misses the fast moves entirely. Test both before you
  decide which one you are actually trading.
- `useTrendFilter` — **off by default, and off is the original strategy.** In
  pure form this setup is *counter-trend*: a run of candles fully above the 5
  EMA is by definition a strong uptrend, and the rules short into it. That is
  the source of both the outsized winners and the losing streaks. The filter
  lets you measure the "only trade with the higher timeframe" variant against
  the real rules instead of assuming it helps.

**Other notes.** `lotSize` defaults to 1 for cash stocks — set it to 75 for
NIFTY futures. The reference candle's range *is* your risk, so risk-% sizing
hands you a small quantity after a wide candle; that is the model working, not
a bug. Below 15m the 5 EMA gets touched almost every candle and very few setups
survive to trigger.

---

## 5. EMA + ADX Trend (higher timeframe)

Not really an intraday scalper — this is the 15m/1h/daily positional variant.
EMA 20/50 cross, filtered by a 200 EMA regime and ADX ≥ 20, with an ATR initial
stop that ratchets into a chandelier trail. Intraday square-off is optional and
auto-disables on daily charts.

---

## Common mechanics (all four)

- **Sizing.** `Risk %` mode risks a fixed % of equity across the stop distance,
  then rounds *down* to whole lots. Default lot size is **75** for the NIFTY
  strategies (**1** for the 5 EMA one, which is stock-oriented) — update
  `lotSize` if the NSE contract spec changes. 1 index point = ₹1 per unit.
- **Session.** Entry windows default to `0930-1430`; square-off `1515-1525`.
  All session logic is timezone-pinned to `Asia/Kolkata`, so it is correct
  regardless of your chart's timezone setting.
- **Costs.** Pre-set to ₹25/order plus 2 ticks slippage. This is a placeholder —
  put your real brokerage, STT, exchange charges and GST in before you believe
  any backtest number. Intraday strategies live or die on this input.
- **Bar confirmation.** `process_orders_on_close = true` and `barstate.isconfirmed`
  guards mean signals evaluate on closed bars only, so backtest and live
  behaviour agree.

## Before you trade any of this

- Backtest on the **instrument you will actually trade** (NIFTY futures, not the
  spot index) — spot has no spread, no slippage, and no roll.
- TradingView's intraday backtests use bar-level fills. A stop and a target
  inside the same bar are resolved pessimistically, but sequence is still a
  guess. Treat the equity curve as an upper bound.
- Options buyers: these are *directional signals*, not options strategies.
  Premium decay, IV crush and strike selection will dominate your P&L and none
  of that is modelled here.
- Walk-forward the parameters. Every input above can be curve-fit to a beautiful
  backtest on one year of NIFTY and fail on the next.

Nothing here is investment advice; it is backtesting code. Size accordingly.
