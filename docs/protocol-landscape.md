# NFC in bitcoin: the protocol and specification landscape

What protocols and specifications exist for moving bitcoin value or
bitcoin signing data over NFC, on-chain and off-chain (Lightning,
LNURL, BOLT11, BOLT12, ecash, Ark, Spark), regardless of how widely
they are used, and how they relate to one another.

*Last reviewed 2026-09, from public sources only. Items marked
[unverified] rest on a single secondary source or are contradicted by a
vendor's own documentation.* The tabular summary lives in the
[README](../README.md#formats); this document is the prose version.

## Summary

There is exactly **one** widely deployed, openly specified NFC
*payment* protocol in bitcoin, and it is
[**Bolt Card**](https://github.com/boltcard/boltcard/blob/main/docs/SPEC.md):
LNURL-withdraw (LUD-03) carried in an NDEF URI record, replay-protected
by the NXP NTAG 424 DNA "SUN/SDM" hardware feature. Everything else is
one of four things:

1. **Plain URI on a tag.** A `bitcoin:` / `lightning:` URI (BIP-21,
   soon BIP-321) or a Lightning Address written into an NDEF record. No
   protocol beyond NDEF; works with anything that reads a tag.
2. **Hardware-signer transports.** NFC as a wire for PSBTs, xpubs and
   signatures (Coldcard, Coinkite tapcards, Bitkey, Portal, Satochip,
   Tangem, …). Several are openly specified; each is its own protocol.
3. **Phone emulates a tag.** Android Host Card Emulation used to hand a
   payment request over a tap, or to take a token back. This is where
   the ecash world lives (Cashu / Numo / Minibits), where the NWC over
   NFC draft lives (§3.3), and where Block's Square Register bitcoin
   tap-to-pay presumably lives, though Block has published no spec.
4. **Nothing at all.** The state of Ark / Arkade and Spark: no NFC
   protocol, no NFC implementation, no proposal found (§7). Any Ark
   tap experience has to be invented or bridged.

---

## 1. Layer map

NFC "protocols" in bitcoin stack up in four layers, and conversations
get confused when people name a layer-4 product (Bolt Card) as if it
were a layer-2 format:

| Layer | What it is | Bitcoin-relevant instances |
| --- | --- | --- |
| **Radio / tag** | [ISO/IEC 14443](https://en.wikipedia.org/wiki/ISO/IEC_14443) Type A+B (13.56 MHz, most cards), [ISO/IEC 15693](https://en.wikipedia.org/wiki/ISO/IEC_15693) a.k.a. NFC-V (vicinity, "Type 5 tag"), ISO/IEC 18092 (NFC-F / FeliCa) | NTAG21x + NTAG 424 DNA (14443-A); Coldcard Mk4/Q (NFC-V / Type 5); JavaCards (14443-A) |
| **Data / framing** | [NDEF](https://nfc-forum.org/build/specifications) records (Well-Known `U`/`T`, MIME, external types); or raw [ISO/IEC 7816-4](https://en.wikipedia.org/wiki/ISO/IEC_7816) APDUs for smartcards | URI records for `bitcoin:` / `lightning:` / `lnurlw:`; external records for PSBTs; APDUs for tapcards / Bitkey / Satochip |
| **Payment / signing protocol** | The actual semantics | LNURL-withdraw (Bolt Card), LNURL-pay, BIP-21/321, BIP-70 (dead), Cashu NUT-18, cktap, Bitkey WCA, Coldcard NFC records |
| **Product / service** | Who holds the money and who is trusted | Bolt Card services (LNbits, Bolt Card Hub, CoinCorner), OpenCryptoPay / DFX, Square, Cashu mints |

Two structural facts fall out of this map and drive everything below:

* **A card cannot push a payment.** A passive NTAG has no radio
  initiative and no key it can spend with (Bolt Card's AES keys only
  authenticate a *withdraw* request). So every card-based bitcoin
  payment is a **pull**: the merchant reads a credential and pulls funds
  from a service. That is why LNURL-**withdraw**, not LNURL-pay, is the
  basis of the card ecosystem, and why an equivalent Ark card would
  need an equivalent pull service (§7).
* **Which side emulates decides which platforms can play.** Card → POS
  works everywhere (both Android and iOS can *read*). Phone → POS needs
  Host Card Emulation, which is Android-only in practice
  ([platform-capabilities.md](./platform-capabilities.md)).

---

## 2. On-chain protocols over NFC

### 2.1 URI-on-a-tag: BIP-21 and BIP-321

The baseline and by far the most common: an NDEF **URI record** (TNF
Well-Known, type `U`) whose payload is a one-byte prefix code plus the
URI. `bitcoin:` and `lightning:` have no assigned prefix code, so they
use prefix `0x00` ("no prefix, whole URI follows"). Record-format
reference: [URI NDEF record](https://gototags.com/help/nfc/ndef/record-types/uri).

* [**BIP-21**](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki)
  is `bitcoin:<address>?amount=…&label=…`, with the de facto
  `lightning=<bolt11>` extension for unified QR/NFC payloads
  ([bitcoinqr.dev](https://bitcoinqr.dev/)).
* [**BIP-321**](https://bips.dev/321/) ([PR #1555](https://github.com/bitcoin/bips/pull/1555))
  is the modernised replacement: address-less URIs, multiple payment
  instructions in one URI, and explicitly named support for BOLT11,
  **BOLT12 offers**, silent payments and **Ark**. This is the one spec
  that gives an Ark payment instruction a standard place to live in an
  NDEF record, and the natural target for any new URI parser.
* Static tags in the wild are usually written with a consumer tool such
  as [NFC Tools](https://play.google.com/store/apps/details?id=com.wakdev.wdnfc).
  See [LightningNFC](https://github.com/theDavidCoen/LightningNFC) for
  the canonical "write `lightning:<lightning-address>` to a €0.30 tag"
  recipe used for tip jars and donation stickers.

### 2.2 BIP-70/71/72 payment requests over NDEF (legacy, dead)

Andreas Schildbach's Bitcoin Wallet carried
[BIP-70 payment requests over NDEF](https://github.com/bitcoin-wallet/bitcoin-wallet/wiki/Payment-Requests),
either a BIP-72 `bitcoin:…?r=https://…` URI record, or the serialised
`PaymentRequest` as an NDEF MIME record, plus Android Beam for
phone-to-phone taps. Historically important as the first real "tap to
pay in bitcoin" and as evidence that NDEF MIME records work fine for
this; practically dead, since
[BIP-70 is deprecated](https://bitcoinops.org/en/topics/bip70-payment-protocol/)
and Android Beam was deprecated in Android 10 and has since been
removed.

### 2.3 Coldcard NFC: PSBTs and transactions as NDEF records

The most complete open spec for moving on-chain signing data over a
tap, and unusual in that the Coldcard is a Type 5 (NFC-V / ISO 15693)
tag, not 14443-A
([nfc-coldcard.md](https://github.com/Coldcard/firmware/blob/master/docs/nfc-coldcard.md),
[NFC Tools docs](https://coldcard.com/docs/nfc-tools/),
[signing guide](https://coldcard.com/guides/using-coldcard/coldcard-nfc-signing)):

| NDEF record type | Payload |
| --- | --- |
| `urn:nfc:wkt:T` (text) | Human label: "Partly signed PSBT", "Deposit Address", "Signed Transaction" |
| `urn:nfc:ext:bitcoin.org:sha256` | 32 raw bytes, SHA-256 over the payload that follows |
| `urn:nfc:ext:bitcoin.org:psbt` | Binary PSBT per BIP-174 (`psbt\xff…`) |
| `urn:nfc:ext:bitcoin.org:txn` | Wire-format signed transaction |
| `urn:nfc:ext:bitcoin.org:txid` | 32 raw bytes |
| `application/json` | Wallet exports |

Record order is explicitly *not* fixed; the practical payload ceiling is
~8 KB, so large multisig PSBTs must fall back to microSD or animated
QR. Coldcard also accepts hex/base64 PSBTs. NFC is off by default and
the antenna is physically removable. Coldcard's own docs state that NFC
traffic is unencrypted and can be eavesdropped.

### 2.4 NFC-PushTx: broadcast by tapping a phone

A one-way mechanism that needs no app on the phone:
after signing, the signer presents an NDEF **URL** record that contains
the whole signed transaction, and the phone's OS opens it in a browser,
which broadcasts it
([nfc-pushtx.md](https://github.com/Coldcard/firmware/blob/master/docs/nfc-pushtx.md)):

```
https://coldcard.com/pushtx#t=<base64url(tx)>&c=<base64url(sha256(tx)[-8:])>&n=XTN
```

`t` = transaction, `c` = integrity check, `n` = network (omitted for
mainnet). Any service can host the endpoint;
[mempool.space](https://mempool.space) implements it natively, and
Coldcard ships a single-file static page. URLs run up to ~8 KB.

### 2.5 Smartcard signing protocols (ISO 7816-4 APDU)

These skip NDEF entirely and speak APDUs to an applet. All of them are
"tap a card, get a signature":

* [**Coinkite Tap Protocol (cktap)**](https://dev.coinkite.cards/docs/protocol.html)
  for SATSCARD / TAPSIGNER / SATSCHIP. One APDU, everything else is CBOR;
  AID `f0436f696e6b697465434152447631`; protected commands run over an
  ephemeral ECDH session so the CVC and sensitive replies are encrypted
  over the air. Reference implementations:
  [coinkite-tap-proto](https://github.com/coinkite/coinkite-tap-proto)
  (Python), [rust-cktap](https://github.com/bitcoindevkit/rust-cktap)
  (BDK). Separately, the cards expose a **dynamic NDEF URL**
  ([NFC spec](https://dev.coinkite.cards/docs/nfc-spec.html)):
  `https://getsatscard.com/start#u=<state>&o=<slot>&r=<addr-tail>&n=<nonce>&s=<sig>`
  (TAPSIGNER uses `t=1`, `c=<card-id>`), signed by the card so a phone
  with no app still lands on a page that shows real card state.
* **Bitkey (Block)**: [open firmware](https://github.com/proto-at-block/bitkey).
  NDEF is used only for firmware update, while all app ↔ device
  traffic is WCA, protobufs over APDUs. Tap-to-sign is gated on the
  device's fingerprint sensor
  ([launch post](https://block.xyz/inside/block-launches-bitkey-wallet-with-screen-automatic-bitcoin-earning-on-cash-app-and-proof-of-reserves)).
* **Satochip family**, a set of AGPL-3.0 JavaCard applets:
  [Satochip](https://satochip.io/) (BIP-32 signer),
  [Satodime](https://github.com/Toporin/Satodime-Applet) (bearer card),
  Seedkeeper (secret storage), and
  [Satocash](https://github.com/Toporin/Satocash-Applet) (an **ecash
  wallet in a JavaCard**). Dual interface: NFC or contact reader.
* [**Keycard**](https://keycard.tech/pages/keycard) is an open-source EAL6+
  NFC smartcard with non-extractable keys; it signs Bitcoin and Ethereum
  by tap. (The sibling *Keycard Shell* is an air-gapped QR device, not
  NFC.)
* [**Portal** by TwentyTwo](https://twenty-two.xyz/) is a Rust/BDK signer
  with no battery: it is powered by the phone's NFC field, so the
  tap is both the transport and the power supply. Integrated by Nunchuk
  and Bitcoin Keeper.
* **Tangem**: NFC card, app open-source, firmware closed
  ([tangem.com](https://tangem.com/en/)); the protocol is card ↔ app ↔
  chain and proprietary.
* **Cypherock X1**: the four "X1 cards" are NFC smartcards holding
  Shamir shares that talk to the vault over an encrypted NFC channel
  ([how it works](https://www.cypherock.com/how-it-works)).
* **OneKey Lite**: NFC card used purely as *seed backup* storage, no
  signing ([product page](https://onekey.so/products/onekey-lite/)).

### 2.6 Hardware wallets whose NFC story is thinner than the marketing

* **Ledger Flex / Stax**: NFC hardware is listed in third-party specs
  and retail listings
  ([review](https://coinbureau.com/review/ledger-flex-review)), but no
  Ledger-documented bitcoin NFC workflow was found. *[unverified]*
* **Trezor Safe 7**: several reviews call it "the first Trezor with
  BLE, NFC and USB"
  ([example](https://thebitcoinhole.com/hardware-wallets/trezor-safe-7)),
  but Trezor's own product page lists **only** Bluetooth 5.0+, USB-C and
  Qi2 charging, with no NFC mention
  ([trezor.io/trezor-safe-7](https://trezor.io/trezor-safe-7)). Treat
  "Safe 7 has NFC" as *[unverified/disputed]*.
* **Foundation Passport Prime**: NFC is a shipped hardware module and
  was part of the launch pitch
  ([Bitcoin Magazine](https://bitcoinmagazine.com/business/passport-prime-a-new-security-device-for-a-new-generation)),
  but Foundation's own docs list QuantumLink (BLE), QR and USB file
  transfer as the signing paths and say NFC message signing "will be
  added in a future firmware update"
  ([docs](https://docs.foundation.xyz/prime/prime-apps/wallet/)). So:
  hardware yes, documented bitcoin NFC protocol not yet.

---

## 3. Lightning: the LNURL family (the real NFC payment stack)

[LNURL](https://github.com/lnurl/luds) is the substrate. The LUDs that
matter for a tap:

| LUD | Title | Why it matters on a tap |
| --- | --- | --- |
| [01](https://github.com/lnurl/luds/blob/luds/01.md) | bech32 encoding | The `LNURL1…` / `lightning=` form found in QR codes and in OpenCryptoPay payloads |
| [03](https://github.com/lnurl/luds/blob/luds/03.md) | `withdrawRequest` | **The** card protocol: the merchant pulls funds using a credential the card produced |
| 06 | `payRequest` | Static tag on a tip jar or merchant sticker (customer pushes) |
| 08 | fast `withdrawRequest` | Fewer round trips at the till |
| 14 | `balanceCheck` | Reusable withdraw links |
| 16 | Lightning Address | `lightning:user@domain` on a cheap tag |
| [17](https://github.com/lnurl/luds/blob/luds/17.md) | scheme prefixes | `lnurlw://`, `lnurlp://`, `lnurlc://`, `keyauth://`; replaces `https://` so a tag can say which sub-protocol it is, and so the OS can route it to a wallet. This is why Bolt Cards contain `lnurlw://…` and not `https://…` |
| 19 | pay-link from withdraw-link | Lets a POS "top up" a card instead of only charging it |

**PIN support has no settled number.** The Bolt Card spec cites
"LUD-21: pinLimit for withdrawRequest" as an optional extension, but
merged LUD-21 in `lnurl/luds` is `verify` for LNURL-pay, and BTCPay
Server's notes call the same feature
"[LUD-290 (pinLimit)](https://blog.btcpayserver.org/btcpay-server-1-6-0/)".
Treat card PINs as a *de facto* extension with divergent numbering.

### 3.1 Bolt Card: the specification

[SPEC.md](https://github.com/boltcard/boltcard/blob/main/docs/SPEC.md)
builds on LUD-03 + LUD-17 + NDEF + NXP's **SUN** (Secure Unique NFC
message) / **SDM** (Secure Dynamic Messaging) feature of the
[NTAG 424 DNA](https://www.nxp.com/docs/en/application-note/AN12196.pdf)
(AES-128, ISO 14443-A, NFC Forum Type 4). The tap:

1. The card returns one NDEF URI record whose content is regenerated by
   the chip on **every** tap:

   ```
   lnurlw://card.yourdomain.com?p=A2EF40F6D46F1BB36E6EBF0114D4A464&c=F509EEA788E37E32
   ```

   `p` = AES-encrypted UID **+ monotonic tap counter**;
   `c` = AES-CMAC over that data.
2. The POS rewrites `lnurlw://` → `https://` per LUD-17 and GETs it.
3. The card service decrypts `p` with the *SDM Meta Read* key, verifies
   `c` with the *SDM File Read* key, **rejects any non-increasing
   counter** (this is the entire replay defence), optionally applies
   per-tap / per-day limits, merchant allow-lists, location checks or a
   PIN, and returns an LNURL-withdraw response.
4. The POS calls the callback with its BOLT11 invoice; the card
   service's node pays it.

Key roles when programming a card
([LNbits extension README](https://github.com/lnbits/boltcards/blob/main/README.md)):
`K0` = auth/write key, `K1` = meta key (produces `p`), `K2` = file key
(produces `c`), `K3`/`K4` set equal to `K1`/`K2` so nothing is left at
the factory-default all-zero value. SDM config: mirroring on, UID
mirroring on, counter mirroring on, meta-read access `01`, counter
retrieval key `0E`, CMAC derivation key `00`. LNbits' URL shape is
`lnurlw://<host>/boltcards/api/v1/scan/<card-external-id>?p=…&c=…`.

System architecture (card, POS, card service, two Lightning nodes) is
sketched in
[SYSTEM.md](https://github.com/boltcard/boltcard/blob/main/docs/SYSTEM.md).

Existing libraries and tools, so nobody has to reimplement the chip handling:
[boltlib](https://github.com/bitcoin-ring/boltlib) (Python read/write),
[sdm-backend](https://github.com/nfc-developer/sdm-backend) (generic
PICCData/CMAC verification),
[pylibsdm](https://pypi.org/project/pylibsdm/1.0.0a0.dev0),
[Ntag424SdmFeature](https://github.com/AndroidCrypto/Ntag424SdmFeature)
(Android/Java walkthrough of the chip),
[bolt-nfc-android-app](https://github.com/boltcard/bolt-nfc-android-app)
(the reference programmer).

**Privacy note from the spec itself:** the URL is a stable domain plus a
rotating blob, so the *issuer* sees every tap; the spec discusses
"privacy levels" for how much the merchant learns.

### 3.2 Static LNURL / Lightning-Address tags

The cheap end: write `lightning:<LNURL1…>` (LUD-01/06) or
`lightning:user@domain` (LUD-16) to a €0.30 NTAG213/215/216 and any
LNURL-capable wallet can pay it. No dynamic secret, no replay
protection, and anyone who taps it can read it. That is fine for a
receive-side sticker (tip jar, donation plaque, merchant table tent)
and unsafe for anything spendable. A static `lnurlw` withdraw link on a
tag is likewise "an offline prepaid card", explicitly described as such
in the [LNbits docs](https://github.com/lnbits/lnbits/wiki): first
tapper wins.

### 3.3 Phone-as-card (HCE) for Lightning

* [**VirtualBoltcardApp**](https://github.com/bitcoin-ring/VirtualBoltcardApp)
  is an Android HCE app that emulates a Bolt Card, importing card keys
  from an LNbits wallet by QR/URL and managing several virtual cards.
  Proof that the Bolt Card wire protocol works from a phone; Android
  only.
* **NWC over NFC** ([draft 0.1, July 2025](https://github.com/agustinkassis/nwc-nfc),
  La Crypta / LaWallet): the customer's Android phone emulates a Type 4A
  tag (AID `F005570202`) whose NDEF record is
  `nostr+walletconnect+request://relay=<url>`. The POS connects to that
  relay, pushes the BOLT11 to the phone in 200-byte APDU chunks (CLA
  `80`, INS `10`), and the phone answers with a signed NIP-47
  `pay_invoice` event in chunks (INS `20`), which the POS publishes.
  Either side can be offline: a wallet with connectivity pays directly
  and hands back the preimage, a wallet without it relies on the POS to
  publish the event. No replay protection or encryption yet; an iOS
  reader is listed as future work. This is the first phone-to-phone
  Lightning tap that needs no card service, at the price of Android on
  both sides. No public implementation beyond the authors' own apps was
  found. *[unverified]*
* **Merchant-emulates-the-tag** (the inverse, and the one that works
  with iOS *customers*): the till emulates an NFC Forum Type 4 tag
  containing the payment request, and the customer's phone reads it.
  This is exactly what [Numo](https://github.com/cashubtc/Numo) does for
  Cashu (§5) and what [Minibits](https://minibits.cash/) does for both
  ecash tokens and BOLT11 invoices.
* **Square / Block bitcoin tap-to-pay**: the buyer taps a phone with a
  Lightning wallet (Cash App, Coinbase, Wallet of Satoshi are named) on
  the customer-facing display of a Square Register 2nd gen; settlement
  is over Lightning
  ([Square](https://squareup.com/us/en/bitcoin),
  [help article](https://squareup.com/help/us/en/article/8622-accept-and-manage-bitcoin-payments),
  [Block](https://block.xyz/inside/block-launches-bitkey-wallet-with-screen-automatic-bitcoin-earning-on-cash-app-and-proof-of-reserves)).
  **No protocol specification has been published**, and none of the
  material says which side emulates. Given that iOS wallets are named,
  the terminal almost certainly emulates a tag that the phone reads,
  but that is inference, not documentation. *[unverified]*

### 3.4 OpenCryptoPay (DFX Swiss)

An "open, licence-free P2P in-person payment standard" that reuses **existing contactless terminal hardware**, backwards-compatible
with Lightning LNURL: the payment payload carries a `lightning=`
parameter holding a bech32 LNURL (LUD-01) that resolves to the till's
API URL ([opencryptopay.io](https://opencryptopay.io/),
[what-is](https://opencryptopay.io/what-is-open-cryptopay.html),
[ecosystem](https://opencryptopay.io/ecosystem.html)). Live in SPAR
Switzerland stores. Caveat: no machine-readable spec was found; the
public repo is the landing page
([waalge/OpenCryptoPay-LandingPage](https://github.com/waalge/OpenCryptoPay-LandingPage))
and the integration path documented is "use DFX's API". *[unverified:
whether an implementable spec exists outside DFX]*

### 3.5 BOLT11 and BOLT12 directly on a tag

No specification exists for either, and no implementation was found for
BOLT12. Mechanically an NDEF URI record `lightning:lnbc…` is trivial,
and this is what a merchant-emulated tag hands over in the ecash / Numo
pattern.
[**BOLT12 offers**](https://github.com/lightning/bolts/blob/master/12-offer-encoding.md)
are the natural fit for a static physical tag: reusable, no
per-payment server round trip, better receiver privacy, so a printed
sticker or engraved plaque stops being a one-shot. BIP-321 already
gives offers a URI slot. This niche is open: "BOLT12 offer on an NFC
sticker" appears to have no implementation anywhere.

---

## 4. LNURL-auth over NFC (adjacent, not payment)

`keyauth://` (LUD-04/17) on a tag turns a card into a login token; the
same trick exists for Nostr keys
([nostrnfcauth](https://github.com/blackcoffeexbt/nostrnfcauth)). Listed
only because it shares the tag plumbing.

---

## 5. Ecash over NFC (Cashu, Fedimint)

The most active new NFC work in bitcoin, and different in kind from
Bolt Card: ecash is a **bearer token**, so it can be *pushed* over
a tap, works offline on both sides, and needs no card service to hold a
key.

* [**NUT-18 payment requests**](https://github.com/cashubtc/nuts/blob/main/18.md)
  define a payment-request encoding with pluggable *transports* (nostr,
  HTTP POST). NFC is not a registered transport, so the pattern in
  practice is: the merchant emulates a Type 4 tag whose content is the
  NUT-18 payment request; the customer's wallet reads it, decodes it,
  and **writes a token back to the emulated tag**. Two-way in one tap.
* **Merchant apps:** [Numo](https://github.com/cashubtc/Numo)
  ([numopay.org](https://numopay.org/),
  [Bitcoin Magazine](https://bitcoinmagazine.com/news/numo-launches-bitcoin-tap-to-pay-app))
  is a free, open-source Android tap-to-pay POS, Cashu and Lightning
  invoices, offline support, auto-sweep to a Lightning address;
  [cashu-pos](https://github.com/babdbtc/cashu-pos).
* **Tokens on cards:** [cashu.me](https://cashu.me) can write a token
  straight onto an NFC card on Android
  ([announcement](https://x.com/callebtc/status/1868333141032649024));
  [Minibits](https://github.com/minibits-cash/minibits_wallet) reads a
  request from a tag and writes the token back, and can itself act as
  an HCE host serving a token or a BOLT11 invoice (Android-only,
  "relies on Host Card Emulation (not available on iOS)").
* **Cards / hardware:** [cashu-javacard](https://github.com/lnflash/cashu-javacard)
  (JavaCard applet for *offline* NFC ecash payments) and
  [cashu-client](https://github.com/lnflash/cashu-client), both from
  Flash; [Satocash-Applet](https://github.com/Toporin/Satocash-Applet)
  (Satochip); [Nucula](https://github.com/zeugmaster/nucula) (ESP32-C6
  ecash wallet with NFC tap-to-pay);
  [μNuts](https://github.com/Amperstrand/micronuts).
* **Fedimint** supports offline ecash spends in principle
  ([Bitcoin Design](https://bitcoin.design/guide/how-it-works/ecash/fedimint/)),
  but no NFC transport spec or shipped NFC implementation was found.
  *[unverified]*

---

## 6. What a "tap" actually is, by direction

Naming the direction removes most of the ambiguity in this space:

| Direction | Who emulates | Protocol examples | Works on iOS? |
| --- | --- | --- | --- |
| **Card → POS** (pull) | Passive chip | Bolt Card (LNURLW + SUN), static LNURLW | Yes; the POS reads NDEF, and iOS can read NTAG 424 |
| **Phone → POS** | Customer phone (HCE) | VirtualBoltcard (pull); NWC over NFC (draft) | Android only |
| **POS → phone** (push) | Merchant device (HCE) or a tag | Cashu NUT-18 via Numo / Minibits; a `lightning:` / `bitcoin:` tag; presumably Square | Yes; the phone only reads |
| **Signer ↔ phone** | Card / device applet or tag | cktap, Bitkey WCA, Coldcard NDEF, Portal, Satochip | Yes, with entitlements ([platform-capabilities.md](./platform-capabilities.md)) |

---

## 7. Ark / Arkade / Spark: the gap

**Nothing exists.** Searches across the Ark, Arkade and Spark
ecosystems turned up no NFC protocol, no NFC feature in a wallet, and no
proposal. What does exist: the
[Arkade wallet](https://github.com/arkade-os/wallet) is a **PWA**, the
[BTCPay Arkade plugin](https://github.com/ArkLabsHQ/btcpay-arkade)
covers merchant acceptance, and Arkade bridges to Lightning through
Boltz. Spark is likewise Lightning-interoperable
([spark.money](https://www.spark.money/faq)).

Options for anyone wanting an Ark tap, in increasing cost:

1. **Static tag with a BIP-321 URI** carrying an Ark payment
   instruction (plus an on-chain and/or BOLT11 fallback). Receive-side
   only, zero new protocol, works with any reader. Cheapest by far, and
   BIP-321 already reserves the slot.
2. **Merchant-emulated tag** (POS → phone) handing over an Ark address
   or an Arkade invoice, i.e. the Numo pattern with Ark instead of
   Cashu. Android-only on the *merchant* side, but customers on both
   platforms can pay. No new cryptography.
3. **Lightning bridge**: accept a Bolt Card tap and settle into Ark via
   a Boltz swap. Reuses the entire deployed card fleet, since the
   customer's card does not need to know Ark exists, at the cost of a
   swap per payment.
4. **An "Ark card"**: a genuine card-pull needs an LNURLW analogue, a
   hosted service holding VTXOs that a merchant can pull from, with SUN
   counters for replay protection. That is a new protocol *and* a new
   trusted service. Note also that a VTXO transfer needs the recipient
   online-ish for the round, which a passive card can never be, so the
   service is unavoidable.
5. A **Web NFC** path is available to the Arkade PWA specifically
   (Chrome / Android only); a cheap experiment with narrow reach.

Designing an "Ark card" protocol (option 4) is the one path that is
hard to justify today: it needs a hosted pull service and new
cryptography for a market of zero existing readers.

---

## 8. Platform capabilities

Moved to its own page: [platform-capabilities.md](./platform-capabilities.md),
covering Android reader mode / HCE / NFC-V, iOS Core NFC with its AID
entitlements and HCE restrictions, Web NFC, and desktop PC/SC.

---

## 9. Open questions

* What exactly does Square's bitcoin tap-to-pay speak? If it is a
  terminal-emulated tag with an LNURL or BOLT11 payload, any wallet that
  reads NDEF could pay at a million Square merchants. Somebody with a
  Register 2nd gen should test it. *[unverified]*
* Does OpenCryptoPay have an implementable spec outside DFX's API? If so
  it is the cheapest route to Swiss retail (SPAR).
* Does NWC over NFC get picked up outside LaWallet? It is the only
  phone-to-phone Lightning tap on the table, and it is a one-page draft
  with no security section yet.
* Is there any moving proposal for an NFC transport in NUT-18 (rather
  than the current ad hoc tag-emulation pattern)?
* Does Ledger Flex / Trezor Safe 7 NFC actually do anything
  bitcoin-facing? Vendor docs currently say no or say nothing.
* Nobody appears to have shipped **BOLT12-offer-on-a-tag**. Given that
  BIP-321 already encodes it, this is the cheapest open item on this
  list.
