# Stapel store data

The store, brand and logo snapshots the [Stapel](https://stapel-app.nl) loyalty-card wallet
downloads. Every file here is published, signed and read by installed phones, so this repository is
the live data feed and not an archive.

## Licences: three datasets, not one

**This is the part to read before reusing anything.** The repository's `LICENSE` is the ODbL, and it
does not cover the logo packs.

| Files | What they are | Licence |
|---|---|---|
| `stores.<cc>.<nnnn>.v1.signed` | Store locations, in parts per country | **ODbL 1.0**, © OpenStreetMap contributors |
| `countries.v1.signed` | The index naming every country and part | **ODbL 1.0** |
| `brands.v1.signed` | The brand catalogue: chains, their names, colours and barcode formats | **ODbL 1.0** |
| `logos.<cc>.v1.signed` | Brand logo packs, one per country | **Not ODbL. Per file**, see [`ATTRIBUTION.md`](ATTRIBUTION.md) |

The licence record covers 620 brand entries over 554 image files, because one mark serves every
brand that shares it. Of those files 443 travel in the packs published here and 111 ship inside
the app itself. Their licences fall into eight states: 493 public domain, 54
CC BY-SA, 9 CC0, 3 CC BY 4.0, 50 asserted PD-textlogo without a third party having ruled on it, and
11 trademarks included on the maintainer's own judgement. So 57 entries carry an attribution duty and
61 rest on an opinion rather than a ruling. [`ATTRIBUTION.md`](ATTRIBUTION.md) gives the licence,
author and Commons file page for each, and also lists the marks that are deliberately not bundled,
with the reason. Reusing a pack without carrying that record forward will put you in breach on the
57, and will inherit the maintainer's judgement on the other 61.

A pack holds only what a card in that country can reach, so no single pack contains every file.

Copyright and trademark are separate here. A wordmark generally sits below the threshold of
originality, so no copyright subsists in it, while the mark stays a registered trademark of its
owner. Stapel shows a logo to identify a card its owner already holds, which is nominative use, and
claims no affiliation with any chain. A chain that would rather not appear only has to ask.

## Attribution for the ODbL data

Carry this, or wording to the same effect, wherever the data or something derived from it is shown:

> Store locations © OpenStreetMap contributors, available under the
> [Open Database License](https://opendatacommons.org/licenses/odbl/) 1.0.

A derived database must itself be offered under the ODbL. Something you merely display from it is a
Produced Work under the licence: it needs the attribution above and a note of where the data can be
obtained, and it does not have to be ODbL itself.

## Where the data comes from

Store locations are derived from OpenStreetMap, taken from Geofabrik country extracts and matched to
a chain on the `brand:wikidata` tag, so a shop is attributed to a chain by its own OSM tagging rather
than by name matching. Logos are resolved from each chain's Wikidata entity via property **P154
(logo image)**, so no filename is guessed at, and the licence is then read from Commons' own file
metadata.

Snapshots are rebuilt and republished about monthly. The code that builds them is not published.

## File format

Every `.signed` file is the same envelope: one header line, then the payload.

```
stapel-store-snapshot-v1 <base64 of a DER ECDSA signature>\n
<payload bytes>
```

The signature is over SHA-256 of the payload bytes alone, using ECDSA on P-256. The header line is
at most 256 bytes and ends at the first `\n`; everything after that byte is payload. A payload is at
most 6 MiB, which is the point past which the app stops reading.

The key the app accepts:

```
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE/h/wU3KNAUYdSoYUFgoiyyVVRD0yzdHVfIHbMTidD5th8oo2XMMWNTH3I8qeJGevi+ab3tbN6B7A03N1sx+3SA==
```

To check a file for yourself:

```bash
echo "<the base64 above>" | base64 -d > pub.der
openssl ec -pubin -inform DER -in pub.der -pubout -out pub.pem

python3 - <<'PY'
import base64, pathlib
raw = pathlib.Path("brands.v1.signed").read_bytes()
end = raw.index(b"\n")
magic, signature = raw[:end].split(b" ", 1)
assert magic == b"stapel-store-snapshot-v1"
pathlib.Path("signature.der").write_bytes(base64.b64decode(signature, validate=True))
pathlib.Path("payload.bin").write_bytes(raw[end + 1:])
PY

openssl dgst -sha256 -verify pub.pem -signature signature.der payload.bin
```

The signature exists so a phone can tell a tampered snapshot from a real one. It is not there to
keep anyone out, and the key above is published for exactly that reason.

### The payloads

`stores.*`, `countries.*` and `brands.*` payloads are UTF-8 JSON. Each carries its own `v`, the
build date as `YYYYMMDD`, and a `generated` date. Different files can come from different builds.

**`countries.v1.signed`** is the index of what exists, one row per country:

```json
{"code": "AR", "format": 1, "stores": 8597, "parts": 5, "logos": 0}
```

`parts` is how many files that country's shops are split into. `logos: 0` means it has no logo pack.

**`stores.<cc>.manifest.v1.signed`** lists those parts, each with an `id`, `sha256`, `bytes`, a
`stores` count and a bounding box (`minLat`, `maxLat`, `minLon`, `maxLon`), so a reader can fetch
only the parts covering the area it cares about. `outline` holds the country border as
`lat,lon;lat,lon;...` strings.

**`stores.<cc>.<nnnn>.v1.signed`** holds `chains` and `stores`. A chain is an object:

```json
{"k": "ah", "name": "Albert Heijn", "c": "NL"}
```

A shop is a **positional array of exactly six values**, because over the whole dataset field names
cost more than the data they label:

```json
["ah", 50.77265, 6.02083, "Prins Willem Alexanderplein 5", "6291GZ", "Vaals"]
```

| 0 | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|
| chain key, joins `chains[].k` | latitude | longitude | street | postcode | town |

Latitude comes before longitude. Coordinates carry five decimals, about a metre, and are never
empty. Street, postcode and town may each be an empty string:

```json
["ah", 50.84985, 5.70557, "", "", ""]
```

**`brands.v1.signed`** holds `brands` and `renames`. A brand is the loyalty programme, which is not
the same thing as a chain: one programme can span several, which is what `chains` lists.

```json
{"key": "ah", "name": "Albert Heijn Bonuskaart", "chain": "Albert Heijn",
 "chains": ["ah", "ah_be"], "aliases": ["Bonuskaart", "AH Bonuskaart"],
 "colorHex": "#00ADE6", "icon": "local_grocery_store", "preferredFormat": "EAN_13",
 "kind": "LOYALTY", "confidence": {"color": "verified", "format": "verified"},
 "logo": "brand_ah"}
```

Every field shown is on every row except `logo`, which is on 620 of 933. `kind` is `LOYALTY`,
`MEMBERSHIP` or `NONE`. `confidence` says which of `colorHex` and `preferredFormat` were checked
against a real card rather than inferred, so a reuser can decline to trust a guess. `logo` is the
key of an entry in a logo pack index, below. `renames` records a programme that changed name, with
the build version it changed in.


**`logos.<cc>.v1.signed`** is an index and a blob, in one file:

```
<compact JSON index>\n<every logo's bytes, concatenated>
```

The index is `{"version": 1, "country": "<CC>", "bytes": <total>, "logos": [...]}`, and each row is
`{"key": "<brand key>", "sha256": "<hex of this logo>", "offset": <from the start of the blob>,
"bytes": <length>}`. Images are lossless WebP. One file serves every brand that shares a mark, such
as a chain's Dutch and Belgian cards, so brand rows outnumber images.

## Reporting a problem

Issues and pull requests are closed on this repository, because its contents are generated and a
patch here would be overwritten by the next publish. A wrong shop or a missing chain is worth
reporting through the [contact routes on the website](https://stapel-app.nl/meedoen/). A wrong shop
in OpenStreetMap is best fixed in OpenStreetMap, where the fix reaches everyone using the data and
arrives here at the next rebuild.
