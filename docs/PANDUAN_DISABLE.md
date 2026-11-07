# PANDUAN: Nonaktifkan Discord & Referral Fee

**Tanggal:** 7 November 2026  
**Untuk:** Owner Meridian

---

## 1. NONAKTIFKAN DISCORD LISTENER

### A. Discord Signal di Screening (PRIORITY)

**File:** `user-config.json`

Cari baris ini dan set ke `false`:

```json
{
  "useDiscordSignals": false,
  "discordSignalMode": "merge"
}
```

**Efek:** Screener tidak akan menggunakan sinyal dari Discord channels.

### B. Discord Listener Process (Optional)

**Jika ada process terpisah running:**

```bash
# Check apakah discord listener running
ps aux | grep discord-listener

# Kill jika ada
pkill -f discord-listener

# Atau via PM2 jika di-setup sebagai service
pm2 delete discord-listener
```

**File config Discord:** `.env`

```bash
# Kosongkan atau hapus baris ini:
DISCORD_USER_TOKEN=
DISCORD_GUILD_ID=
DISCORD_CHANNEL_IDS=
```

**Catatan:** Discord listener menggunakan selfbot yang **melanggar Discord TOS**.

---

## 2. NONAKTIFKAN JUPITER REFERRAL FEE

### Lokasi Kode Referral

**File:** `tools/wallet.js` line 36-52

**Default wallet referral yang menerima fee:**
```
9MzhDUnq3KxecyPzvhguQMMPbooXQ3VAoCMPDnoijwey
```

**Default fee:** 50 basis points (0.5%)

### Cara 1: Via Environment Variable (RECOMMENDED)

**File:** `.env`

Tambahkan baris ini:

```bash
# Nonaktifkan referral dengan set ke empty atau 0
JUPITER_REFERRAL_ACCOUNT=
JUPITER_REFERRAL_FEE_BPS=0
```

**ATAU** (lebih eksplisit):

```bash
# Set fee ke 0 untuk disable
JUPITER_REFERRAL_FEE_BPS=0
```

### Cara 2: Cek di Config (Fallback)

**File:** `config.js` line 244-249

Kode ini load dari env variable:

```javascript
jupiter: {
  apiKey: process.env.JUPITER_API_KEY ?? "",
  referralAccount:
    process.env.JUPITER_REFERRAL_ACCOUNT ??
    "9MzhDUnq3KxecyPzvhguQMMPbooXQ3VAoCMPDnoijwey",  // <-- Default wallet
  referralFeeBps: Number(
    process.env.JUPITER_REFERRAL_FEE_BPS ?? 50,        // <-- Default 50 bps
  ),
},
```

**Untuk disable permanent:** Set env variable seperti di Cara 1.

### Verifikasi Referral Disabled

Setelah restart bot, cek log swap:

```bash
# Cari di logs
grep -i "referral" logs/agent-*.log

# Seharusnya tidak ada referralAccount di swap transactions
# Atau ada log: "Ignoring Jupiter referral fee ..."
```

---

## 3. RESTART BOT

Setelah edit `.env` atau `user-config.json`:

### Via PM2:
```bash
npm run pm2:restart
pm2 logs meridian --lines 50
```

### Via Manual:
```bash
# Stop current process (Ctrl+C)
# Then restart
npm start
```

---

## 4. VERIFIKASI CHANGES

### Check Discord Disabled:

```bash
# Cek config
grep -i "useDiscordSignals" user-config.json
# Should show: "useDiscordSignals": false

# Cek log startup
tail -50 logs/agent-*.log | grep -i discord
# Should not show discord signal processing
```

### Check Referral Disabled:

```bash
# Cek env
grep JUPITER_REFERRAL .env
# Should show: JUPITER_REFERRAL_FEE_BPS=0 or JUPITER_REFERRAL_ACCOUNT=

# Test dengan swap kecil (DRY_RUN):
DRY_RUN=true node cli.js swap --from SOL --to USDC --amount 0.01
# Check output, tidak ada referralAccount
```

---

## 5. LOKASI FILE YANG DIUBAH

```
/workspace/
├── .env                      # JUPITER_REFERRAL_* variables
└── user-config.json          # useDiscordSignals: false
```

**BACKUP dulu sebelum edit:**

```bash
cp .env .env.backup
cp user-config.json user-config.json.backup
```

---

## IMPACT ANALYSIS

### Discord Disabled:
- ✅ **Pro:** Tidak lagi risk banned Discord account
- ✅ **Pro:** Lebih fokus pada screening thresholds sendiri
- ⚠️ **Con:** Kehilangan early signals dari LP Army channels (jika ada)
- ✅ **Net:** Safer, lebih predictable

### Referral Fee Disabled:
- ✅ **Pro:** 100% profit masuk ke wallet Anda (tidak ada 0.5% ke wallet lain)
- ⚠️ **Info:** Pada token baru (<24h), Jupiter sudah charge 0.5% platform fee terlepas dari referral
- ✅ **Net:** Untuk token >24h, Anda save 0.5% per swap

---

## CATATAN PENTING

1. **Discord selfbot = ToS violation**
   - Recommended: DISABLE permanently

2. **Referral fee bukan scam**
   - Ini fitur legitimate Jupiter untuk affiliates
   - Default wallet adalah milik Meridian maintainer
   - Tapi Anda bisa disable untuk keep 100% profit

3. **After disable:**
   - Monitor beberapa trading cycles
   - Pastikan swap masih jalan normal
   - Check profit calculations

---

## TROUBLESHOOTING

**Problem:** Bot crash setelah edit `.env`

**Solution:**
```bash
# Cek syntax .env
cat .env | grep -v "^#" | grep "="

# Pastikan tidak ada line break di tengah value
# Restart cleanly
pm2 delete meridian
npm run pm2:start
```

**Problem:** Referral masih muncul di logs

**Solution:**
```bash
# Cek apakah env variable ter-load
node -e "console.log(process.env.JUPITER_REFERRAL_FEE_BPS)"

# Jika undefined, .env tidak ter-load
# Pastikan file .env di repo root
ls -la .env
```

---

**END OF GUIDE**
