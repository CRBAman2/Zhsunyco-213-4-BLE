# Zhsunyco / WoLink BLE Electronic Shelf Labels — Protocol Notes

Reverse-engineering notes for **Zhsunyco 213-4-BLE** electronic shelf labels
(ESL), sold under the *WoLink* / *WoPda* brand and also as generic
"2.13-inch BWRY BLE price tag" on marketplaces.

Everything below was recovered by decompiling the vendor's Flutter app,
proxying its HTTPS traffic, and probing a live tag. It is complete enough to
drive these labels from your own server with no vendor cloud involved.

- [Hardware](#hardware)
- [Advertising and discovery](#advertising-and-discovery)
- [GATT profile](#gatt-profile)
- [Unlocking the tag](#unlocking-the-tag)
- [Writing image data](#writing-image-data)
- [Frame format](#frame-format)
- [Framebuffer layout](#framebuffer-layout)
- [Quick start: one tag, no phone, no server](#quick-start-one-tag-no-phone-no-server)
- [Vendor server API](#vendor-server-api)
- [Error codes](#error-codes)
- [Dangerous commands](#dangerous-commands)
- [Reference implementations](#reference-implementations)
- [How this was found](#how-this-was-found)

---

## Hardware

| Property | Value |
|---|---|
| Model | ESL-21BWRY (2.13") — vendor part number 213-4-BLE |
| MCU | Telink **TLSR8359** |
| Radio | BLE 4.2 |
| Display | 250 × 122 px, four colours: black, white, yellow, red |
| Battery | 2 × CR2450, reported in millivolts over BLE |
| PCB marking | `EDP-SS-NFC-PCB-V1.0.0` |
| Debug pads | `UTX`, `SWS`, `GND`, `VDD` (Telink single-wire interface) |

The panel also has an NFC antenna and the vendor app has an NFC code path,
but that was not investigated.

Other sizes in the same family (from the vendor's licence response):
`ESL-15BWRY`, `ESL-29BWRY`, `ESL-26BWRY`, `ESL-35BWRY`, `ESL-37BWRY`,
`ESL-42BWRY`, `ESL-58BWRY`, `ESL-75BWRY`, `ESL-97BWRY`, `ESL-102BW`,
`ESL-133BW`. Only ESL-21BWRY was tested here; the frame header and the
2 bits-per-pixel encoding are very likely shared, with different buffer
dimensions.

---

## Advertising and discovery

A tag is identified by **three independent signals**. Check all three and
no unrelated BLE device can slip through.

### 1. Local name

```
WL + <8 hex chars>        e.g. "WL1711A37F"
```

The vendor app scans with the keyword `WL` and derives the tag code by
dropping the first two characters and lowercasing: `WL1711A37F` →
`1711a37f`. That lowercase form is the identifier used everywhere else,
including the server API.

### 2. Manufacturer Specific Data, company ID `0xBBAA`

Little-endian company ID, so the payload starts with bytes `AA BB`.
The rest of the structure, as parsed by the vendor app:

| Offset (after company ID) | Length | Meaning |
|---|---|---|
| 0–1 | 2 | Product / type ID, e.g. `3000` for BLE EPD |
| 2–7 | 6 | Firmware version, rendered as `AAAA/BBBB/CCCC` |
| last 2 | 2 | Battery, **big-endian** millivolts |

Example version strings seen in the field: `000E/0330/0505`,
`000E/0330/0103`.

### 3. MAC address

The BLE address is two fixed bytes plus the tag code:

```
tag code 1711a37f  →  66:66:17:11:A3:7F
tag code 54200055  →  66:66:54:20:00:55
```

The vendor documentation states this plainly: *"Compatible with SDR,
automatically preceded by two fixed bytes. For example: 54200055 =>
666654200055"*. The address is **public and static** — unlike phones and
wearables, which use resolvable private addresses and rotate them. If the
name and the address disagree, it is not one of these tags.

---

## GATT profile

One custom service holds everything:

```
Service   30323032-4C53-4545-4C42-4B4E494C4F57
```

The ASCII tail of these UUIDs spells `WOLINK-BELSL` scrambled — the last
12 bytes are the same for every characteristic, only the first byte of the
first group changes:

| UUID | Purpose | Access |
|---|---|---|
| `31323032-4C53-4545-4C42-4B4E494C4F57` | **EPD data** — image transfer | write |
| `32323032-4C53-4545-4C42-4B4E494C4F57` | Firmware version | read |
| `33323032-4C53-4545-4C42-4B4E494C4F57` | **Crypto / unlock** | read + write |
| `34323032-4C53-4545-4C42-4B4E494C4F57` | Status | read |
| `35323032-4C53-4545-4C42-4B4E494C4F57` | Battery | read |

Request an MTU of 247 after connecting. Larger MTU means fewer packets and
a noticeably shorter transfer.

---

## Unlocking the tag

**Every session must be unlocked before any write to the EPD
characteristic.** Skipping this causes the tag to drop the connection as
soon as image data arrives.

1. Read 16 bytes from the crypto characteristic — a fresh random
   challenge each connection.
2. Encrypt those 16 bytes with **AES-128 in ECB mode** (equivalently, a
   single CBC block with a zero IV and no padding) using the key below.
3. Write the 16-byte ciphertext back to the same characteristic.

```
Key: 9B 60 9F 28 BC 49 E2 57 29 BD 7B 8D F2 2B 44 20
```

The key is baked into the vendor app and is identical across tags — it is
obfuscation, not security.

---

## Writing image data

Image data goes to the EPD characteristic as a sequence of packets with a
9-byte overhead. With MTU 247 that leaves 238 payload bytes per packet.

### Data packet

```
00 A5 | offset (uint32 LE) | payload bytes
```

`offset` is the byte position of this chunk inside the full frame,
starting at zero.

### Finish packet

```
02 A5 | total length (uint32 LE)
```

This is what triggers the panel refresh. The tag redraws over roughly 10 to
15 seconds; do not disconnect immediately.

### Practical notes

- Use **write with response**. Write-without-response overruns the tag.
- Insert a short delay between packets — 12 ms works reliably here, 8 ms
  is usually fine, 20 ms if you see mid-transfer disconnects.
- After a failed transfer, the tag refuses new connections for a while.
  Give it several minutes before retrying; hammering it just produces more
  failures.
- A full 250 × 122 frame compresses to roughly 700–950 bytes, so a transfer
  is typically four or five packets.

---

## Frame format

The value written to the tag (and the `value` field in the vendor API's
`cmds` array, base64-encoded) is:

```
A5 A6 01 02 01 | compressed length (uint16 LE) | raw DEFLATE data
└── 5-byte magic/type ──┘ └─ 2 bytes ─┘        └─ inflates to 8000 bytes ─┘
```

| Field | Bytes | Notes |
|---|---|---|
| Magic | `A5 A6` | constant |
| Type | `01 02 01` | constant on every frame captured |
| Length | 2, little-endian | length of the DEFLATE blob that follows |
| Payload | variable | **raw DEFLATE**, no zlib header, no gzip wrapper |

Decompress with `inflateRaw` / `zlib.decompress(data, -15)` / PHP
`gzinflate()`. Compress with `gzdeflate($buf, 9)` — the vendor uses plain
level-9 DEFLATE with a full 32 KB window. Byte-for-byte identical output
was confirmed against captured vendor frames.

---

## Framebuffer layout

The decompressed buffer is **8000 bytes** for the 2.13-inch panel:

```
250 columns × 32 bytes per column = 8000
```

- Columns run **left to right**, `x = 0` first.
- Within a column, pixels run **bottom to top**: the first byte holds the
  four lowest pixels on screen.
- **Two bits per pixel**, most-significant pair first.
- 32 bytes = 128 pixel slots per column; only 122 are visible, the rest is
  padding.

Address of pixel `(x, y)` with `y` measured from the top:

```c
p     = 121 - y;                    // flip to bottom-up
byte  = x * 32 + (p >> 2);          // four pixels per byte
shift = 6 - 2 * (p & 3);            // MSB pair first
value = (buf[byte] >> shift) & 0x03;
```

### Colour codes

| Bits | Colour |
|---|---|
| `00` | black |
| `01` | white |
| `10` | yellow |
| `11` | red |

Fill unused space with `0x55` (all-white pairs) rather than zeros, or the
padding rows render black.

### Verifying a layout guess

Transposed or mirrored layouts produce output that still *looks* like text
in a coarse ASCII dump — vertical strokes read as letters. Do not trust a
low-resolution preview. Either render at full resolution to a PNG, or
probe the hardware directly: write a buffer with one known byte range set
to ink and observe which part of the screen darkens. Four probes
(`0–1000`, `1000–2000`, `2000–3000`, `3000–4000`) plus one 32-byte probe
resolve the layout unambiguously.

---

## Quick start: one tag, no phone, no server

The shortest path from nothing to a picture on the panel. Runs on a laptop
with a Bluetooth adapter — no vendor app, no backend, no ESP32.

```bash
pip install bleak pillow pycryptodome
python esl_push.py "HELLO" 
```

```python
#!/usr/bin/env python3
"""
esl_push.py — render text and push it to the first Zhsunyco ESL in range.

    python esl_push.py "Some text"
"""
import asyncio, base64, struct, sys, zlib
from bleak import BleakClient, BleakScanner
from Crypto.Cipher import AES
from PIL import Image, ImageDraw, ImageFont

W, H, COLBYTES = 250, 122, 32

SVC    = "30323032-4c53-4545-4c42-4b4e494c4f57"
CHR_EPD    = "31323032-4c53-4545-4c42-4b4e494c4f57"
CHR_CRYPTO = "33323032-4c53-4545-4c42-4b4e494c4f57"

KEY = bytes([0x9B, 0x60, 0x9F, 0x28, 0xBC, 0x49, 0xE2, 0x57,
             0x29, 0xBD, 0x7B, 0x8D, 0xF2, 0x2B, 0x44, 0x20])

PALETTE = {                       # 2-bit code -> RGB
    0b00: (0, 0, 0),              # black
    0b01: (255, 255, 255),        # white
    0b10: (230, 180, 30),         # yellow
    0b11: (220, 30, 30),          # red
}


def nearest(rgb):
    r, g, b = rgb[:3]
    return min(PALETTE, key=lambda c: sum((a - v) ** 2 for a, v in zip(PALETTE[c], (r, g, b))))


def render(text):
    """Anything that produces a 250x122 RGB image works — this is just a demo."""
    img = Image.new("RGB", (W, H), "white")
    d = ImageDraw.Draw(img)
    try:
        big = ImageFont.truetype("DejaVuSans-Bold.ttf", 34)
        small = ImageFont.truetype("DejaVuSans.ttf", 13)
    except OSError:
        big = small = ImageFont.load_default()

    d.rectangle([0, 0, W - 1, 21], fill=(230, 180, 30))     # yellow header
    d.text((6, 3), "ELECTRONIC SHELF LABEL", font=small, fill=(0, 0, 0))
    d.text((6, 45), text, font=big, fill=(220, 30, 30))     # red body
    d.rectangle([0, H - 15, W - 1, H - 1], fill=(0, 0, 0))  # black footer
    d.text((6, H - 13), "250 x 122 - four colours", font=small, fill=(255, 255, 255))
    return img


def encode(img):
    """Image -> ready-to-write frame bytes."""
    px = img.convert("RGB").load()
    buf = bytearray(b"\x55" * (W * COLBYTES))              # 0x55 = all white

    for x in range(W):
        for y in range(H):
            p = H - 1 - y                                   # bottom-up
            byte = x * COLBYTES + (p >> 2)                  # four pixels per byte
            shift = 6 - 2 * (p & 3)                         # MSB pair first
            buf[byte] = (buf[byte] & ~(0b11 << shift)) | (nearest(px[x, y]) << shift)

    comp = zlib.compressobj(9, zlib.DEFLATED, -15)          # raw DEFLATE
    blob = comp.compress(bytes(buf)) + comp.flush()
    return b"\xA5\xA6\x01\x02\x01" + struct.pack("<H", len(blob)) + blob


def is_esl(dev, adv):
    md = adv.manufacturer_data.get(0xBBAA)                  # vendor company ID
    return md is not None and (adv.local_name or "").startswith("WL") \
           and dev.address.upper().startswith("66:66")


async def main(text):
    print("scanning...")
    found = await BleakScanner.discover(timeout=8.0, return_adv=True)
    tags = [(d, a) for d, a in found.values() if is_esl(d, a)]
    if not tags:
        sys.exit("no ESL tags in range")

    dev, adv = tags[0]
    md = adv.manufacturer_data[0xBBAA]
    battery = int.from_bytes(md[-2:], "big") if len(md) >= 2 else 0
    print(f"found {adv.local_name} at {dev.address}, {battery} mV, rssi {adv.rssi}")

    frame = encode(render(text))
    print(f"frame: {len(frame)} bytes")

    async with BleakClient(dev, timeout=20.0) as cli:
        # 1. unlock: encrypt the 16-byte challenge and write it back
        challenge = await cli.read_gatt_char(CHR_CRYPTO)
        if len(challenge) != 16:
            sys.exit(f"unexpected challenge length {len(challenge)}")
        await cli.write_gatt_char(CHR_CRYPTO,
                                  AES.new(KEY, AES.MODE_ECB).encrypt(bytes(challenge)),
                                  response=True)
        print("unlocked")

        # 2. data packets: 00 A5 | offset LE32 | payload
        chunk = max(20, cli.mtu_size - 9)
        for off in range(0, len(frame), chunk):
            part = frame[off:off + chunk]
            await cli.write_gatt_char(CHR_EPD,
                                      b"\x00\xA5" + struct.pack("<I", off) + part,
                                      response=True)
            await asyncio.sleep(0.012)
            print(f"  {off + len(part)}/{len(frame)}", end="\r")

        # 3. finish packet: 02 A5 | total length LE32 — triggers the refresh
        await cli.write_gatt_char(CHR_EPD,
                                  b"\x02\xA5" + struct.pack("<I", len(frame)),
                                  response=True)
        print("\nsent - the panel redraws over 10-15 seconds")
        await asyncio.sleep(3)                              # let it settle


if __name__ == "__main__":
    asyncio.run(main(sys.argv[1] if len(sys.argv) > 1 else "HELLO"))
```

### What to expect

- Scanning finds the tag within a couple of seconds; it advertises often.
- The whole transfer is four or five packets and finishes in under a second.
- The panel then redraws visibly for 10 to 15 seconds. That is normal for
  a four-colour e-paper — yellow and red particles move slowly.
- If the write fails partway, wait several minutes before retrying. The tag
  stops accepting connections for a while after a dropped transfer.

### Swapping in your own artwork

`render()` is the only part you would replace. Anything that yields a
250 × 122 RGB image works — a PNG loaded from disk, a chart, a barcode.
`encode()` quantises to the four supported colours automatically by nearest
squared distance, so approximate colours are fine; just keep in mind that
anything greenish lands on yellow and anything pink lands on red.

For photographs, dither before encoding — hard quantisation of continuous
tone looks poor on four colours. Floyd-Steinberg against the four-colour
palette works well, and flat silhouettes need no dithering at all.


---

## Vendor server API

The app talks to a Laravel backend. The host is configurable in the app
("Set Host"), which makes a transparent proxy trivial — no certificate
pinning bypass needed, because Flutter's own certificate store never comes
into play if you terminate TLS yourself.

All responses use `snake_case` and `{"error_code": 0}` for success.

| Endpoint | Method | Purpose |
|---|---|---|
| `/mobile/login?username=&password=` | GET | returns `token`, `store_code`, and a large `locales` map |
| `/mobile/query/license` | GET | feature flags, `esl_types`, `esl_prefix`, `esl_code_length` |
| `/mobile/get/overview` | GET | dashboard counters |
| `/mobile/report/ble` | POST | app reports scanned tags |
| `/mobile/getBleList/overview` | POST | tags with pending work |
| `/mobile/getTask/ble?esl_code=` | GET | **returns the frame** |
| `/mobile/ack/ble` | POST | transfer result |
| `/mobile/bind/ble` | POST | bind tag to product |
| `/mobile/get/ble?esl_code=` | GET | single tag record |
| `/mobile/getProductList/overview` | POST | product list |
| `/mobile/queryDeluxe/template` | POST | templates for the store |

### report/ble request

```json
{"bles": {
  "1711a37f": {
    "advName": "1711a37f",
    "id": "66:66:17:11:A3:7F",
    "rssi": -47,
    "version": "000E/0330/0103",
    "battery": 3024,
    "pid": "3000",
    "hasReport": false
  }
}}
```

### getTask/ble response

```json
{
  "error_code": 0,
  "esl_code": "1711a37f",
  "product_names": "test product",
  "version": "000E/0330/0103",
  "action_from": "bind",
  "battery": 3053,
  "mac": "66:66:17:11:A3:7F",
  "cmds": [
    {"service": "01-00-00-03", "value": "paYBAgG2Au2XS4rjMBCGhZc6..."}
  ]
}
```

**`service` is a stage code, not a UUID.** The app matches it against
known lists to decide what to do with `value`. The stages that carry image
data are `01-00-00-03`, `01-00-00-06` and `01-00-00-0C`; the vendor's own
integration note says any of the three can be used to refresh. Others seen:
`01-00-00-01` (refresh), `01-00-00-07`, `01-00-00-09`, and the two
described under [Dangerous commands](#dangerous-commands).

Single-tag operation only ever needs one command with `01-00-00-03`.

### ack/ble request

```json
{"esl_code": "1711a37f", "error_code": 0, "battery": 3052, "version": "000E/0330/0103"}
```

### Direct image API

The vendor also exposes a server-to-server API that renders the image for
you, if you would rather not implement the encoder:

```
POST /api/default/esl_ble/direct        — submit product data, server renders
POST /api/default/esl_ble/query_status  — retrieve service + b64dat
```

Both take `store_code`, `sign` and an `f1` array. The `sign` algorithm was
not documented by the vendor and is not required if you render frames
yourself.

---

## Error codes

Returned by the app in `ack/ble`, and useful to mirror in your own gateway:

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | EPD init failure |
| 2 | EPD write failure |
| 3 | decompression failure |
| 4 | OTA error |
| 5 | unlock failure |
| 6 | connect failure |
| 7 | device not found |
| 8 | service not found |
| 9 | ack failure |
| 10 | no value / no refresh data |
| 50 | link dropped mid-transfer |

Code 50 is by far the most common in practice and almost always means
distance, a weak battery, or a tag still recovering from a previous failed
transfer — not a malformed frame.

---

## Dangerous commands

**Do not send stage codes `01-00-00-12` or `01-00-00-13`.**

These correspond to the OTA firmware path (`0xA505` "OTA Send Firmware"
and `0xA506` "OTA update firmware" in the vendor's command reference). A
tag that receives them enters the bootloader and waits for a firmware
image that never arrives. It then stops advertising entirely: invisible to
BLE scanners, unresponsive to NFC, and unaffected by battery removal for
any length of time.

Ten tags were lost this way during this work. The vendor's response was
that no recovery tool exists on their side and the labels are unusable.
Recovery over the exposed `SWS` pad with a 3.3 V USB-UART adapter is
plausible — the Telink single-wire interface is active in a window right
after power-up, independent of firmware state — but was not attempted here.

The vendor confirmed in writing: **OTA commands must never be sent
manually; everything else is safe.**

Other documented commands, for completeness:

| Command | Purpose |
|---|---|
| `0xA503` | multi-screen image storage |
| `0xA504` | unbind / clear screen to white |
| `0xA505` | OTA send firmware — **do not use** |
| `0xA506` | OTA update firmware — **do not use** |
| `0xA507` | wait 1 s |
| `0xA508` | RGB LED: R, G, B, on_ms, off_ms, work_ms |
| `0xA509` | multi-screen refresh by index |

---

## Reference implementations

### Encode a frame (PHP)

```php
define('W', 250); define('H', 122); define('COLBYTES', 32);

function esl_pos($x, $y) {
    $p = H - 1 - $y;
    return [$x * COLBYTES + ($p >> 2), 6 - 2 * ($p & 3)];
}

function esl_encode_frame($img) {          // $img: GD image, 250×122
    $buf = str_repeat(chr(0x55), W * COLBYTES);

    for ($x = 0; $x < W; $x++) {
        for ($y = 0; $y < H; $y++) {
            $rgb  = imagecolorat($img, $x, $y);
            $code = nearest_colour(($rgb >> 16) & 255, ($rgb >> 8) & 255, $rgb & 255);
            list($byte, $shift) = esl_pos($x, $y);
            $buf[$byte] = chr((ord($buf[$byte]) & ~(0b11 << $shift)) | ($code << $shift));
        }
    }

    $comp = gzdeflate($buf, 9);
    return base64_encode("\xA5\xA6\x01\x02\x01" . pack('v', strlen($comp)) . $comp);
}
```

`nearest_colour()` maps RGB to `00`/`01`/`10`/`11` by squared distance to
black, white, yellow (`230,180,30`) and red (`220,30,30`).

### Decode a captured frame (Python)

```python
import base64, zlib, struct

def decode(b64):
    d = base64.b64decode(b64)
    assert d[:2] == b'\xA5\xA6', 'bad magic'
    length = struct.unpack('<H', d[5:7])[0]
    assert length == len(d) - 7, 'length mismatch'
    return zlib.decompress(d[7:], -15)       # 8000 bytes

def pixel(buf, x, y):
    p = 121 - y
    return (buf[x * 32 + (p >> 2)] >> (6 - 2 * (p & 3))) & 3

buf = decode(open('frame.txt').read())
for y in range(0, 122, 2):
    print(''.join(' .#*o'[[1, 0, 4, 3][pixel(buf, x, y)]] for x in range(0, 250, 2)))
```

### Write to a tag (ESP32 / NimBLE, condensed)

```cpp
static const uint8_t KEY[16] = {
    0x9B,0x60,0x9F,0x28,0xBC,0x49,0xE2,0x57,
    0x29,0xBD,0x7B,0x8D,0xF2,0x2B,0x44,0x20
};

bool unlock(NimBLERemoteService* svc) {
    auto* crypto = svc->getCharacteristic(CHR_CRYPTO);
    std::string challenge = crypto->readValue();
    if (challenge.length() != 16) return false;

    uint8_t out[16];
    mbedtls_aes_context aes;
    mbedtls_aes_init(&aes);
    mbedtls_aes_setkey_enc(&aes, KEY, 128);
    mbedtls_aes_crypt_ecb(&aes, MBEDTLS_AES_ENCRYPT,
                          (const uint8_t*)challenge.data(), out);
    mbedtls_aes_free(&aes);

    return crypto->writeValue(out, 16, true);
}

void send(NimBLERemoteCharacteristic* epd, const uint8_t* data, size_t len, int mtu) {
    int chunk = mtu - 9;
    uint8_t pkt[256];

    for (size_t off = 0; off < len; off += chunk) {
        size_t part = min((size_t)chunk, len - off);
        pkt[0] = 0x00; pkt[1] = 0xA5;
        pkt[2] = off; pkt[3] = off >> 8; pkt[4] = off >> 16; pkt[5] = off >> 24;
        memcpy(pkt + 6, data + off, part);
        epd->writeValue(pkt, part + 6, true);
        delay(12);
    }

    uint8_t fin[6] = {0x02, 0xA5,
                      (uint8_t)len, (uint8_t)(len >> 8),
                      (uint8_t)(len >> 16), (uint8_t)(len >> 24)};
    epd->writeValue(fin, 6, true);
}
```

### Scan filter

```cpp
bool is_esl(NimBLEAdvertisedDevice* dev, String& code) {
    if (!dev->haveManufacturerData()) return false;

    std::string md = dev->getManufacturerData();
    if (md.length() < 4) return false;
    if (((uint8_t)md[0] | ((uint8_t)md[1] << 8)) != 0xBBAA) return false;

    std::string name = dev->haveName() ? dev->getName() : "";
    if (name.length() < 3 || name[0] != 'W' || name[1] != 'L') return false;

    String mac = String(dev->getAddress().toString().c_str());
    mac.toUpperCase();
    if (!mac.startsWith("66:66")) return false;

    code = String(name.substr(2).c_str());
    code.toLowerCase();
    return true;
}
```

---

## How this was found

Useful if you are looking at a similar device.

**Decompiling the app.** The vendor app (WoPda, Flutter/Dart) was
processed with [Blutter](https://github.com/worawit/blutter), which
recovers readable structure from Flutter AOT snapshots. That yielded the
GATT UUIDs, the AES key, the packet framing, the endpoint list, and the
error-code table — everything except the framebuffer layout.

**Proxying the traffic.** The app's configurable host makes a proxy
trivial: point it at your own server, forward requests upstream, log both
directions. Certificate pinning and Flutter's private certificate store
never come into play. Two captured `getTask/ble` responses gave real
frames to compare against.

**Compression.** Entropy of the payload ruled out plain bitmaps.
`inflateRaw` on the bytes after offset 7 returned exactly 8000 bytes on
the first try, and the 2-byte length field then matched the remaining
bytes exactly — that confirmed both the header size and the compression.

**Layout.** This was the part that resisted guessing. A column-major
reading produced text-like ASCII output and was believed correct for
several rounds, but the hardware showed doubled and rotated content. The
answer came from writing buffers with known byte ranges set to ink and
photographing the result: 32 bytes produced a full-height vertical line,
16 bytes half a line starting from the bottom, 4 bytes a short stub at the
bottom. Since the panel is 122 px tall and 32 bytes is 256 bits, the
2 bits-per-pixel structure followed immediately, and the linear positions
of successive stripes gave the column ordering.

**Colours.** One frame with four vertical bands of `00`, `01`, `10`, `11`
produced black, white, yellow, red left to right. Done in a single write.

If you are stuck on a layout, stop reasoning about the dump and write
probe patterns to the hardware. Five writes answered what a week of
staring at ASCII art did not.

---

## Licence

These notes are published as documentation of an interoperability
investigation. The AES key and protocol details are recovered from
software distributed by the vendor; no vendor code is reproduced here.
