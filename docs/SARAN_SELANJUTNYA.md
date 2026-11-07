# SARAN NEXT STEPS - Meridian Bot

**Konteks:** VPS run profitable overall, beberapa hari losing, token selection key

---

## PRIORITAS 1: SAFETY & STABILITY

### 1.1 Disable High-Risk Features ✅
- [x] Discord listener (ToS violation, risk banned)
- [x] Jupiter referral fee (keep 100% profit)
- [ ] Test dengan DRY_RUN dulu sebelum live

**Action:**
```bash
# Edit .env:
DISCORD_USER_TOKEN=
JUPITER_REFERRAL_FEE_BPS=0

# Edit user-config.json:
"useDiscordSignals": false

# Restart
npm run pm2:restart
```

### 1.2 Tighten Position Limits
**Current risk:** `maxPositions: 3`, `maxDeployAmount: 50 SOL`

**Recommendation:**
```json
{
  "maxPositions": 2,           // Reduce concurrent exposure
  "maxDeployAmount": 20,       // Cap per-position risk
  "deployAmountSol": 0.3,      // Lower base deploy
  "positionSizePct": 0.25,     // 25% of deployable balance
  "gasReserve": 0.3            // More safety buffer
}
```

### 1.3 Stricter Stop Loss
**Current:** `-50%` (sangat loose)

**Recommendation:**
```json
{
  "stopLossPct": -20,          // Cut losses faster
  "takeProfitPct": 8,          // Take profit lebih cepat
  "trailingTriggerPct": 5,     // Activate trailing earlier
  "trailingDropPct": 2.0       // Tighter trailing
}
```

---

## PRIORITAS 2: TOKEN SELECTION (KEY INSIGHT)

**User context:** Token selection believed most important factor

### 2.1 Analisis Historical Performance

**Study pool-memory.json & lessons.json:**

```bash
# Check win rates per pool/token
node cli.js performance --limit 100

# Identify patterns
cat pool-memory.json | jq '.[] | select(.win_rate > 0.6) | {name, win_rate, avg_pnl_pct}'

# Check lessons untuk token attributes
cat lessons.json | jq '.lessons[] | select(.outcome == "positive") | .context'
```

**Questions to answer:**
1. Which token attributes correlate dengan winning positions?
   - Holder count range?
   - Market cap sweet spot?
   - Organic score threshold?
   - Token age?

2. Which pools to avoid?
   - Check `pool-memory.json` untuk pools dengan multiple OOR closes
   - Check `token-blacklist.json` untuk patterns

### 2.2 Optimize Screening Thresholds

**Current defaults bisa terlalu permissive:**

```json
{
  "minOrganic": 60,            // Raise to 70-75?
  "minHolders": 500,           // Raise to 750-1000?
  "minMcap": 150000,           // Raise to 250000?
  "maxMcap": 10000000,         // Lower to 5000000?
  "minTokenFeesSol": 30,       // Raise to 50?
  "maxBotHoldersPct": 30,      // Lower to 20?
  "maxTop10Pct": 60            // Lower to 50?
}
```

**Method:**
1. Analyze your `performance` data di `lessons.json`
2. Run `node cli.js evolve` jika ada 5+ closed positions
3. Monitor win rate improvement

### 2.3 Token Age Filter

**Avoid too-new tokens (high volatility, low liquidity):**

```json
{
  "minTokenAgeHours": 6,       // Avoid <6h old tokens
  "maxTokenAgeHours": 168      // Focus on <1 week old (still fresh)
}
```

### 2.4 Launchpad Filtering

**If you notice pattern dari lessons:**

```json
{
  "blockedLaunchpads": ["pump.fun"],  // Example: jika pump.fun underperform
  "allowedLaunchpads": []              // Or whitelist only good launchpads
}
```

---

## PRIORITAS 3: RISK MANAGEMENT

### 3.1 Out-of-Range Protection

**Current:** 30 min wait before close

**Tighten:**
```json
{
  "outOfRangeWaitMinutes": 15,        // Close OOR faster
  "oorCooldownTriggerCount": 2,       // After 2 OOR (not 3)
  "oorCooldownHours": 24              // Longer cooldown (not 12h)
}
```

### 3.2 Low Yield Detection

**Current:** 7% fee/TVL threshold, 60 min age

**Adjust:**
```json
{
  "minFeePerTvl24h": 10,              // Higher yield requirement
  "minAgeBeforeYieldCheck": 120       // Wait 2h before judging (not 1h)
}
```

### 3.3 Volatility-Based Bin Range

**Current bot uses linear formula. Consider dari code:**

```javascript
// In config: volatility affects bins_below
// Higher volatility = wider range
// Check if your winning positions have specific volatility range
```

**Study:** Do your profitable positions cluster around certain volatility levels?

---

## PRIORITAS 4: MONITORING & LEARNING

### 4.1 Daily Monitoring Checklist

```bash
# Morning check
node cli.js positions
node cli.js balance

# Check recent decisions
cat decision-log.json | jq '.decisions[-10:]'

# Check lessons
node cli.js lessons | tail -20

# Performance summary
node cli.js performance --limit 20
```

### 4.2 Weekly Analysis

```bash
# Generate briefing
node cli.js briefing

# Check evolve thresholds (after 5+ closes)
node cli.js evolve

# Review pool memory
cat pool-memory.json | jq 'to_entries | map({pool: .key, win_rate: .value.win_rate, deploys: .value.total_deploys}) | sort_by(-.win_rate) | .[:10]'
```

### 4.3 Enable Detailed Logging

**In .env:**
```bash
LOG_LEVEL=debug
```

**Monitor:**
```bash
tail -f logs/agent-$(date +%Y-%m-%d).log
```

---

## PRIORITAS 5: STRATEGY TUNING

### 5.1 Active Strategy Review

**Check current:**
```bash
node cli.js strategy list
```

**Current default:** `bid_ask`

**Consider testing:**
- `spot` untuk directional plays
- Adjust `bins_below` / `bins_above` based pada analysis

### 5.2 Darwin Signal Weights

**If enabled:**
```json
{
  "darwinEnabled": true,
  "darwinRecalcEvery": 5,      // Recalc after 5 closes
  "darwinMinSamples": 15       // Need 15 samples min (not 10)
}
```

**Check weights evolution:**
```bash
cat signal-weights.json | jq '.weights'
```

---

## PRIORITAS 6: ADVANCED (Optional)

### 6.1 Chart Indicators

**Currently disabled. Consider enable jika Anda technical trader:**

```json
{
  "chartIndicators": {
    "enabled": true,
    "entryPreset": "rsi_plus_supertrend",
    "exitPreset": "supertrend_break"
  }
}
```

### 6.2 Smart Wallets Tracking

**Add KOL wallets yang Anda trust:**

```bash
node cli.js smart-wallet add \
  --address <wallet_address> \
  --name "KOL Name" \
  --type lp
```

**Effect:** Bot mendapat confidence boost saat smart wallet ada di pool.

### 6.3 HiveMind Contribution

**Current:** Auto mode

**Consider:**
```json
{
  "hiveMindPullMode": "manual"  // Disable auto-pull dari other agents
}
```

**Reason:** Other agents might have different risk profiles.

---

## TIMELINE RECOMMENDATION

### Week 1: Safety First
- [ ] Disable Discord ✅
- [ ] Disable/reduce referral fee ✅
- [ ] Tighten position limits
- [ ] Lower stop loss threshold
- [ ] Run DRY_RUN untuk test

### Week 2: Data Collection
- [ ] Run live dengan safer settings
- [ ] Collect 10+ position closes
- [ ] Run `evolve` untuk auto-tune
- [ ] Analyze `performance` data

### Week 3: Token Selection Analysis
- [ ] Deep dive `pool-memory.json`
- [ ] Identify winning token patterns
- [ ] Adjust screening thresholds
- [ ] Test new thresholds

### Week 4: Optimization
- [ ] Fine-tune based pada Week 2-3 data
- [ ] Consider chart indicators
- [ ] Add smart wallet tracking
- [ ] Establish monitoring routine

---

## EXPECTED OUTCOMES

**With tighter risk controls:**
- Lower drawdowns (smaller losing days)
- More consistent returns
- Better capital preservation

**With better token selection:**
- Higher win rate (target: 60%+)
- Fewer OOR closes
- Better fee yield

**Realistic target:**
- Win rate: 55-65%
- Avg winning position: +8-12%
- Avg losing position: -15-20% (with -20% stop)
- Net: Positive over 20+ positions

---

## CRITICAL: WHAT NOT TO DO

❌ **Jangan increase `maxDeployAmount`** saat losing streak
❌ **Jangan disable stop loss** untuk "ride it out"
❌ **Jangan chase pumps** dengan manual deploy
❌ **Jangan ignore OOR warnings**
❌ **Jangan run multiple bots** pada same wallet (race conditions)

---

## SUPPORT & RESOURCES

**Dokumentasi:**
- `README.md` - User guide
- `CLAUDE.md` - Engineering manual
- `logs/` - Audit trail

**Commands untuk troubleshooting:**
```bash
# Check bot status
node cli.js status

# Force close position
node cli.js close --position <address>

# Update config on-the-fly
node cli.js config set minOrganic 75

# Check recent decisions
node cli.js decisions
```

---

**KESIMPULAN:**

1. **Start safe:** Disable Discord + tighten limits
2. **Collect data:** Run 2-3 weeks untuk pattern recognition
3. **Analyze:** Focus pada token selection (your key insight)
4. **Iterate:** Adjust thresholds based pada data
5. **Monitor:** Daily checks + weekly deep dive

**Key metric to watch:** Win rate & average profit per winning position.

Good luck! 🚀
