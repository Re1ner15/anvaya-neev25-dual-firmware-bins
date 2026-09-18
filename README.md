# anvaya-neev25-dual-firmware-bins

Public OTA firmware images for the **Neev v2.5-dual** metering board
(ESP32-S3 + ATM90E32 + ADS131M08). Source: `neev_dev` →
`firmware/pzem/Final/anvaya_neev25_dual_firmware`.

Images are published as **GitHub Release assets** (one release per version),
the same convention as `anvaya-pzem-firmware-bins` / `anvaya-heatpump-firmware-bins`.
Each asset is the **app-only** `.ino.bin` (never the merged bin — that bricks OTA).

## Triggering an OTA

The device subscribes to `neev/{MAC}/cmd/ota` and expects:

```json
{ "action": "update",
  "url": "https://github.com/Re1ner15/anvaya-neev25-dual-firmware-bins/releases/download/<tag>/<asset>.ino.bin",
  "version": "0.6.5-neev25dual",
  "sha256": "<64-hex, mandatory>" }
```

- `version` **must differ** from the running version or the device replies `skipped`.
- `sha256` is **mandatory** (64 hex); the image is verified before flashing.
- Use `neev_dev/scripts/ota_push.sh <MAC> <version> <release-download-url>` — it
  downloads the asset, computes the SHA-256, and publishes the command.
- Watch: `mosquitto_sub -t "neev/{MAC}/status/ota" -v`
  (`downloading → verifying → success → verified`).

## TLS pin caveat (read before large rollouts)

The firmware **pins 5 root CAs** (ISRG X1, USERTrust ECC/RSA, DigiCert Global
Root CA/G2) rather than trusting the full CA set. The `releases/download/...`
URL 302-redirects to GitHub's `release-assets.githubusercontent.com` CDN, whose
edge certs rotate — so an individual OTA attempt can **intermittently** fail TLS
(status `failed`, reason `http_code`) if it lands on an edge whose chain isn't in
the pinned set.

- This is **safe**: a failed download never touches the running slot — the device
  stays online on its current image. **Just re-fire the OTA.**
- Deterministic alternative: commit the bin under `bins/` and point the OTA `url`
  at `https://raw.githubusercontent.com/Re1ner15/anvaya-neev25-dual-firmware-bins/main/bins/<asset>.ino.bin`
  (raw host chains to the pinned ISRG X1, no CDN redirect).
- If intermittent misses become a problem at fleet scale, broaden the firmware's
  trust to the full ESP32 x509 CA bundle instead of the 5 pinned roots — a
  firmware change, not a hosting change.

Auto-rollback is enabled from `0.6.4` (a new image that can't sustain WiFi+MQTT
for 30 s within 5 min reverts to the previous slot automatically).
