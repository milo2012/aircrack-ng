# airodump-ng AUTH-EAP modes (patched)

The patched `airodump-ng` extends the old `AUTH` column (`MGT` for all
Enterprise) into `AUTH-EAP`, showing which **outer** EAP methods were
actually observed on the air for each AP. Inner methods (MSCHAPv2/GTC
inside the tunnel) stay opaque — same limit as any passive sniffer.

Source: `airodump-eap.patch` (`eap_outer_tag()` in
`src/airodump-ng/airodump-ng.c`, CSV suffixes in `dump_write.c`).

## Live samples (lab SSID `test`, channel 6)

AP up, no EAP exchange seen yet — plain `MGT`:

```text
 CH  6 ][ Elapsed: 18 s ][ 2026-10-01 11:04 ][ paused output
 BSSID              PWR RXQ  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH-EAP ESSID
 00:C0:CA:96:45:2F  -26  78      160        0    0   6   54   WPA2 CCMP   MGT      test
```

After a TEAP client session — `MGT+TEAP`:

```text
 CH  6 ][ Elapsed: 3 mins ][ 2026-10-01 11:06 ][ paused output
 BSSID              PWR RXQ  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH-EAP ESSID
 00:C0:CA:96:45:2F  -32  47     1462      456   72   6   54   WPA2 CCMP   MGT+TEAP test
```

## All modes

| AUTH-EAP   | Meaning                              | Pentester read                              |
|------------|--------------------------------------|---------------------------------------------|
| `MGT`      | Enterprise, no EAP seen yet          | Wait for clients / deauth to force EAP      |
| `MGT+TLS`  | Outer EAP-TLS (type 13) observed     | hostapd-mana (`mana_eaptls`) + portal if fleet validation is lax/clickable |
| `MGT+PEAP` | Outer PEAP (type 25) observed        | eaphammer hash capture (MSCHAPv2/GTC inners); portal only post-crack |
| `MGT+TTLS` | Outer TTLS (type 21) observed        | eaphammer hash capture, same as PEAP        |
| `MGT+TEAP` | Outer TEAP (type 55) observed        | Custom twin (`teap-twin rogue-auto`); expect MSCHAPv2 hashes or dual-TLS |
| `MGT+EAP`  | Some other EAP type (MD5/GTC/…)      | Check CSV / capture; likely password-based, eaphammer candidate |
| `MGT+T/P`  | TEAP + PEAP clients on one SSID      | Mixed fleet — run both tool paths           |
| `MGT+T/TLS`| TEAP + EAP-TLS clients               | Mixed fleet — cert path (mana) + TEAP path  |
| `MGT+P/TTLS`| PEAP + TTLS clients                 | eaphammer covers both                       |
| `MGT+2types` | Two outer methods, other combo     | Mixed fleet                                 |
| `MGT+3types` / `MGT+4types` | Three/four outer methods | Heterogeneous fleet, target per client  |

Console shows the compact tag above. CSV/TXT output instead appends each
method literally, e.g. `MGT`, `MGT+TEAP`, `MGT+TLS+PEAP+TEAP`.

## Caveats

- **Passive evidence only.** `MGT` alone does not mean "no EAP" — it means
  no EAP exchange was captured yet. Small Identity/NAK frames (< ~50 bytes
  on air) are skipped by an upstream length check; full method flights
  (TLS/PEAP/TEAP handshakes) are large enough to register.
- **Outers only.** NAK-offered types are recorded as offered, not
  negotiated — the client may settle on a different method.
- **Sticky per run.** The mask accumulates until airodump-ng restarts;
  `MGT+T/P` can reflect two clients hours apart, not simultaneous support.
