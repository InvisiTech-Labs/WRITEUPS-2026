## Solution walkthrough

### 1. Establish the noisy baseline

Open `exfil.pcap` in Wireshark. Start with the HTTP traffic:

```wireshark
http
```

The capture contains many requests to `public-storage.inm.bank` (`10.7.15.30`) for JPEG-like media objects. It also contains music-stream/CDN traffic, update checks, and periodic telemetry requests. These are environmental noise and help make the capture less obvious.

The recurring workstation is **`10.7.15.50`**. Telemetry identifies it by the hostname **`FIN-WS-17`**.


### 2. Identify the cloud retrieval

Filter for the cloud host:

```wireshark
http.host == "cloud.inm.bank"
```

The client requests:

```http
GET / HTTP/1.1
Host: cloud.inm.bank
```

then:

```http
GET /objects/archive-index.json HTTP/1.1
Host: cloud.inm.bank
```

and finally:

```http
GET /objects/recovery_bundle.zip HTTP/1.1
Host: cloud.inm.bank
```

The cloud server is `10.7.15.13`. The ZIP download is an HTTP response with `Content-Type: application/zip` and a content length of 1781 bytes.

### 3. Recover the ZIP from the PCAP

In Wireshark:

1. Select **File → Export Objects → HTTP**.
2. Find the object requested from `cloud.inm.bank` at `/objects/recovery_bundle.zip`.
3. Save it as `recovery_bundle.zip`.

Alternatively, right-click the relevant HTTP response and use **Export Packet Bytes** if needed. Ensure the saved file begins with the ZIP signature `PK` and can be opened as a ZIP archive.

### 4. Inspect the archive

List the archive contents:

```bash
unzip -l recovery_bundle.zip
```

The archive contains these useful files:

```text
recovery_bundle/README.txt
recovery_bundle/operations/recovery-notes.txt
recovery_bundle/recovery/archive-index.txt
```

Extract and read them:

```bash
unzip recovery_bundle.zip -d extracted
cat extracted/recovery_bundle/README.txt
cat extracted/recovery_bundle/operations/recovery-notes.txt
cat extracted/recovery_bundle/recovery/archive-index.txt
```

The notes reference:

- `LEDGER-RECOVERY-2026-09`
- workstation `FIN-WS-17`
- correlating the cloud retrieval with workstation telemetry
- the final verification value being absent from the repository archive

This is the pivot to the telemetry service.

### 5. Find the recovery-verification telemetry event

Filter:

```wireshark
http.host == "telemetry.inm.bank"
```

Inspect the HTTP POST requests to `/api/v1/events`. Most contain ordinary `heartbeat` events for `FIN-WS-17`. One event near the cloud ZIP retrieval is different:

```json
{
  "host": "FIN-WS-17",
  "application": "digital-banking-client",
  "event": "recovery_verification",
  "reference": "LEDGER-RECOVERY-2026-09",
  "verification": "ZmxhZ3tpbm1fdGhlX25vaXNlX3dhc190aGVfY2x1ZX0=",
  "timestamp": "2026-09-25T05:50:25"
}
```

The timestamp aligns with the archive retrieval. The `reference` also matches the reference in the ZIP's recovery notes. This correlation confirms that the event is relevant rather than routine background telemetry.

### 6. Decode the verification value

The `verification` field is Base64. Decode it:

```bash
printf '%s' 'ZmxhZ3tpbm1fdGhlX25vaXNlX3dhc190aGVfY2x1ZX0=' | base64 -d
```

Expected output:

```text
flag{inm_the_noise_was_the_clue}
```

## Final answer

`flag{inm_the_noise_was_the_clue}`

## Evidence chain summary

1. Identify the recurring client `10.7.15.50`, named `FIN-WS-17` in telemetry.
2. Find the download of `/objects/recovery_bundle.zip` from `cloud.inm.bank` (`10.7.15.13`).
3. Export and inspect the ZIP.
4. Follow the archive's instruction to correlate with telemetry.
5. Find the `recovery_verification` event for `LEDGER-RECOVERY-2026-09` at approximately `2026-09-25 05:50:25`.
6. Base64-decode the `verification` value to obtain the flag.

