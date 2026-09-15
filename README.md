# Awesome Bitcoin NFC [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> A curated list of apps, devices, protocols and formats for moving
> bitcoin over NFC: payments as well as signing data.

Written for developers who ask themselves: *which NFC formats would I
have to support, and which apps or devices can I install or buy to test
against them?* Bitcoin only: on-chain, Lightning, ecash, and the (still
empty) Ark and Spark corners. Fiat card-network products are excluded
even when they hold a bitcoin balance, because nothing bitcoin-specific
happens at the terminal (see [Out of scope](#out-of-scope)).

The space splits into three populations that barely overlap:

- **Bolt Card and its clones.** One open protocol (LNURL-withdraw in an
  NDEF URI, replay-protected by the NTAG 424 DNA chip) covers almost the
  whole consumer payment story: dozens of card vendors, around 15 POS
  apps, a few purpose-built terminals, a few hosting services, rings and
  implants. All of it interoperates because all of it speaks
  `lnurlw://…?p=…&c=…`.
- **NFC hardware signers.** A crowded and growing field (Coldcard,
  Coinkite tapcards, Bitkey, Portal, Satochip, Keycard, Tangem,
  Cypherock, Arculus, OneKey Lite). Every vendor has its own protocol,
  and the tap replaces USB or QR rather than a payment terminal.
- **Ecash tap-to-pay.** New since 2025, mostly Android, and the only
  group where the phone is the terminal and the payment is a bearer
  token pushed over the tap (Numo, Minibits, cashu.me, JavaCard
  applets).

A fourth group is only starting: phone-to-phone Lightning payments with
no card service in between. So far that is one draft spec, NWC over NFC,
from the LaWallet and La Crypta community in Argentina.

Two facts explain most of what follows. A passive card cannot push a
payment, so every card-based bitcoin payment is a pull: the merchant
reads a credential and pulls funds from a service that holds them. That
is why the card ecosystem is built on LNURL-withdraw and not on
LNURL-pay. The second fact is that whichever side emulates a tag decides
which platforms can take part. Reading works on Android and iOS;
emulating (Host Card Emulation) is Android-only in practice. See
[docs/platform-capabilities.md](docs/platform-capabilities.md).

## Contents

- [What to support, what to test with](#what-to-support-what-to-test-with)
- [Apps](#apps)
  - [Consumer wallets](#consumer-wallets)
  - [Merchant and POS apps](#merchant-and-pos-apps)
  - [Signer companion apps](#signer-companion-apps)
  - [Tag programming tools](#tag-programming-tools)
- [Devices](#devices)
  - [Cards, rings and implants](#cards-rings-and-implants)
  - [Hardware signers and key cards](#hardware-signers-and-key-cards)
  - [Payment terminals](#payment-terminals)
  - [ATMs and vending machines](#atms-and-vending-machines)
- [Formats](#formats)
  - [Radio and tag types](#radio-and-tag-types)
  - [Framing](#framing)
  - [Payment and signing protocols](#payment-and-signing-protocols)
- [Card issuing and hosting services](#card-issuing-and-hosting-services)
- [Libraries and developer tools](#libraries-and-developer-tools)
- [Documents](#documents)
- [Out of scope](#out-of-scope)
- [Contributing](#contributing)

### Legend

| Mark | Meaning |
| --- | --- |
| ✅ | supported, per the vendor's own documentation or source code |
| ⚠️ | claimed by a single secondary source, contradicted by the vendor, or otherwise unverified |
| ❌ | not supported |
| n/a | not applicable |

Column meanings in the app tables: **Read** = reads a tag or card,
**Write** = writes to a tag or card (NDEF write or APDU commands),
**Emulate** = the phone itself acts as a tag or card (Host Card
Emulation). **Custody** describes who holds the funds the tap moves.

Everything here was collected from public sources (vendor sites, source
repositories, aggregators such as [boltcard.org](https://www.boltcard.org/)
and [Coincharge](https://coincharge.io/en/bolt-card-the-lightning-card/)).
Nothing was bought or tapped to confirm it. Corrections are welcome;
see [Contributing](#contributing).

## What to support, what to test with

The whole list condensed by developer goal. Each entry links into the
detailed tables below.

| If you want to… | Implement | Test against (software) | Test against (hardware) |
| --- | --- | --- | --- |
| **Accept taps from the deployed card fleet** (you are the till) | [Bolt Card](#payment-and-signing-protocols): read one NDEF URI record, rewrite `lnurlw://` → `https://` (LUD-17), run an LNURL-withdraw client (LUD-03), optionally the pinLimit extension | Issue yourself a card with the [LNbits Boltcards extension](#card-issuing-and-hosting-services) or [Bolt Card Hub](#card-issuing-and-hosting-services); compare behaviour with [Bolt Card PoS](#merchant-and-pos-apps), [Swiss Bitcoin Pay](#merchant-and-pos-apps) or Wallet of Satoshi; use [VirtualBoltcardApp](#consumer-wallets) as a phone-side card | Any blank NTAG 424 DNA card programmed with the [NFC Card Creator](#tag-programming-tools); a [BoltRing](#cards-rings-and-implants); a CoinCorner card |
| **Emulate a Bolt Card on the customer's phone** | Bolt Card SUN/SDM response served over Host Card Emulation; Android only | [VirtualBoltcardApp](#consumer-wallets) is the reference implementation | Read it with [Bolt Card PoS](#merchant-and-pos-apps) or any terminal below |
| **Phone-to-phone Lightning with no card service** | [NWC over NFC](#payment-and-signing-protocols): the customer phone emulates a Type 4A tag over HCE, the POS pushes the BOLT11 over APDUs, the wallet answers with a signed NIP-47 `pay_invoice` event. Draft; Android on both sides | No public implementation confirmed yet ⚠️; the LaWallet POS apps are where to look | Two Android phones |
| **Read static payment tags** (tip jars, stickers, table tents) | NDEF URI record carrying [BIP-21 / BIP-321](#payment-and-signing-protocols) `bitcoin:`, `lightning:` + BOLT11, bech32 LNURL (LUD-01) or a Lightning Address (LUD-16) | Write test tags with [NFC Tools](#tag-programming-tools); the [LightningNFC](https://github.com/theDavidCoen/LightningNFC) recipe | NTAG213/215/216 tags (cents each) |
| **Write static tags** | NDEF write: Android `Ndef`, iOS `NFCNDEFReaderSession`, Chrome-on-Android Web NFC | Verify with NFC Tools or NXP TagInfo | Same tags |
| **Talk to a hardware signer by tap** (PSBT in, signature out) | [Coldcard NDEF record set](#payment-and-signing-protocols) over NFC-V; [cktap](#payment-and-signing-protocols) APDU/CBOR; Portal; Bitkey WCA; Satochip / Keycard applets. On iOS declare every AID up front | [Nunchuk](#signer-companion-apps), [Bitcoin Keeper](#signer-companion-apps), [Cove](#signer-companion-apps) as reference clients | [Coldcard Mk4 / Q](#hardware-signers-and-key-cards), TAPSIGNER, SATSCARD, Portal, Bitkey, a DIY Satochip JavaCard |
| **Broadcast a signed transaction with no app on the phone** | [NFC-PushTx](#payment-and-signing-protocols) URL record | [mempool.space](https://mempool.space) implements the endpoint; Coldcard ships a static page | Coldcard Mk4 / Q |
| **Ecash tap-to-pay, phone to phone, offline-capable** | Merchant emulates an NFC Forum Type 4 tag holding a [NUT-18](#payment-and-signing-protocols) payment request; customer writes a Cashu token back | [Numo](#merchant-and-pos-apps) as merchant, [Minibits](#consumer-wallets) as customer, [cashu.me](#consumer-wallets) for token-on-a-card | [Nucula](#payment-terminals), [Cashu Pixel](#payment-terminals); [cashu-javacard](#payment-and-signing-protocols) |
| **Ark / Arkade / Spark over NFC** | Nothing exists to be compatible with. Cheapest paths: a BIP-321 URI on a static tag, a merchant-emulated tag, or a Lightning bridge in front of a Bolt Card tap; see [docs/protocol-landscape.md](docs/protocol-landscape.md#7-ark--arkade--spark-the-gap) | n/a | n/a |

**A minimal bench.** One Android phone with NFC (reader mode *and* HCE,
so it can play both sides), one iPhone (reader only; Core NFC needs AIDs
in the entitlements for smartcards), a strip of NTAG213 tags for static
URIs, a few NTAG 424 DNA cards for Bolt Card work, a self-hosted
[LNbits](https://github.com/lnbits/lnbits) with the Boltcards and TPoS
extensions, and a PC/SC reader such as the ACR122U or ACR1252U for
desktop-side tooling. Add a Coldcard or a TAPSIGNER when signer transport
matters.

## Apps

### Consumer wallets

The customer side: apps that pay, receive or hold funds and touch NFC
while doing so. Apps that only *talk to a hardware signer* over NFC
are in [Signer companion apps](#signer-companion-apps).

| App | Platform | Read | Write | Emulate | Payloads | Custody | Source | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [Wallet of Satoshi](https://walletofsatoshi.com/) ([Android](https://play.google.com/store/apps/details?id=com.livingroomofsatoshi.wallet), [iOS](https://apps.apple.com/us/app/wallet-of-satoshi/id1438599608)) | Android, iOS | ✅ | ❌ | ❌ | Bolt Card (as merchant); Square tap-to-pay (as customer) ⚠️ | Custodial | Closed | Advertises "tap an NFC card" to take payments; widely reported as the most reliable card reader. Named by Square as a wallet that can tap at a Register 2nd gen; that mechanism is undocumented. |
| [Blink](https://www.blink.sv/) ([Android](https://play.google.com/store/apps/details?id=com.galoyapp), [iOS](https://apps.apple.com/us/app/blink-bitcoin-wallet/id1531383905)) | Android, iOS | ✅ | ❌ | ❌ | Bolt Card | Custodial | [GitHub](https://github.com/GaloyMoney/blink-mobile) | Receive via QR, NFC, LNURL, Lightning Address or voucher. |
| [Breez](https://breez.technology/) ([Android](https://play.google.com/store/apps/details?id=com.breez.client), [iOS](https://apps.apple.com/us/app/breez-lightning-client-pos/id1463604142)) | Android, iOS | ✅ Android only | ❌ | ❌ | Bolt Card | Self-custodial | [GitHub](https://github.com/breez/breezmobile) | Built-in cash-register (POS) mode; card reading works on Android only. |
| [Bolt Card Wallet](https://www.boltcard.org/) ([Android](https://play.google.com/store/apps/details?id=com.boltcard.boltcard), [iOS](https://apps.apple.com/us/app/bolt-card-wallet/id6446301845)) | Android, iOS | ⚠️ | ❌ | ❌ | Bolt Card | Custodial (Bolt Card Hub / LNDHub; can point at your own node) | [GitHub](https://github.com/boltcard/boltcard-wallet/) | BlueWallet fork that links and unlinks a physical Bolt Card per wallet. Card programming is delegated to the [NFC Card Creator](#tag-programming-tools). |
| [VirtualBoltcardApp](https://github.com/bitcoin-ring/VirtualBoltcardApp) | Android (sideload) | ❌ | ❌ | ✅ | Bolt Card | Whatever backs the imported card (LNbits) | [GitHub](https://github.com/bitcoin-ring/VirtualBoltcardApp) | The phone *is* the Bolt Card: imports card keys from LNbits by QR/URL and emulates the NTAG 424 SUN response via HCE. Manages several virtual cards. |
| [CoinCorner](https://www.coincorner.com/) | Android, iOS | ⚠️ | ❌ | ❌ | Bolt Card | Custodial | Closed | Issuer of the original commercial [Bolt Card](https://support.coincorner.com/hc/en-us/articles/5600580995346-The-Bolt-Card); merchant reading via CoinCorner Checkout. |
| [Bitkit](https://bitkit.to/) | Android, iOS | ⚠️ | ❌ | ❌ | Bolt Card | Self-custodial | [GitHub](https://github.com/synonymdev/bitkit) | Listed as Bolt Card-capable on boltcard.org; not verified individually. |
| [Blixt](https://blixtwallet.github.io/) | Android, iOS | ⚠️ | ❌ | ❌ | Bolt Card, LNURL | Self-custodial | [GitHub](https://github.com/hsjoberg/blixt-wallet) | Listed as Bolt Card-capable on boltcard.org; not verified individually. |
| [Zeus](https://zeusln.com/) | Android, iOS | ⚠️ | ❌ | ❌ | LNURL, invoices; Bolt Card partial | Self-custodial | [GitHub](https://github.com/ZeusLN/zeus) | NFC support exists but card reading is [reported flaky](https://ereignishorizont.xyz/en/boltcard_en/). Ships pre-loaded on the Bitcoinize terminal. |
| [Edge](https://edge.app/) | Android | ⚠️ | ⚠️ | ❌ | `bitcoin:` URI | Self-custodial | [GitHub](https://github.com/EdgeApp/edge-react-gui) | [Send/receive over NFC](https://edge.app/blog/market-updates/bitcoin-nfc-tap-to-pay/) dates from the Airbitz era; current status unverified. |
| [Minibits](https://www.minibits.cash/) ([Android](https://play.google.com/store/apps/details?id=com.minibits_wallet), [iOS](https://apps.apple.com/mx/app/minibits/id6744454479)) | Android, iOS (NFC on Android only) | ✅ | ✅ | ✅ | Cashu token, NUT-18 request, BOLT11 | Ecash (mint trust) | [GitHub](https://github.com/minibits-cash/minibits_wallet) | Reads a payment request from a tag and writes the ecash token back; can itself act as an HCE host serving a token or a BOLT11 invoice. States its HCE features are "Android-only, as it relies on Host Card Emulation". |
| [cashu.me](https://cashu.me) | Web (Chrome on Android, Web NFC) | ⚠️ | ✅ | ❌ | Cashu token | Ecash (mint trust) | [GitHub](https://github.com/cashubtc/cashu.me) | Writes an ecash token straight onto an NFC card from the browser ([demo](https://x.com/callebtc/status/1868333141032649024)). |
| [eNuts](https://enuts.cash/) | Android, iOS | ⚠️ | ❌ | ❌ | Cashu | Ecash (mint trust) | [GitHub](https://github.com/cashubtc/eNuts) | Listed as tap-to-pay compatible with Numo by [Coincharge](https://coincharge.io/en/numo-numopay-accept-bitcoin-tap-to-pay-via-nfc-in-stores/); not verified individually. |
| [Macadamia](https://macadamia.cash/) | iOS | ⚠️ | ❌ | ❌ | Cashu | Ecash (mint trust) | [GitHub](https://github.com/zeugmaster/macadamia) | Listed as tap-to-pay compatible with Numo; not verified individually. Noteworthy as an iOS customer wallet in the ecash-tap pattern (iOS can read an emulated tag). |
| Sovran | n/a | ⚠️ | ❌ | ❌ | Cashu | Ecash (mint trust) | n/a | Listed as tap-to-pay compatible with Numo; not verified individually. |
| [Cash App](https://cash.app/) | Android, iOS | ⚠️ | ❌ | ❌ | Square bitcoin tap-to-pay | Custodial | Closed | Named by Square as a wallet that can [tap to pay bitcoin](https://squareup.com/help/us/en/article/8622-accept-and-manage-bitcoin-payments) at a Register 2nd gen. Mechanism unpublished. |
| [Coinbase](https://www.coinbase.com/) | Android, iOS | ⚠️ | ❌ | ❌ | Square bitcoin tap-to-pay | Custodial | Closed | As above. |

### Merchant and POS apps

The merchant side: the till reads the customer's card, or (ecash
pattern) emulates a tag the customer's phone reads. Almost all card
reading happens on Android in practice because merchants buy cheap
Android terminals; see [Payment terminals](#payment-terminals).
[BTCMap](https://btcmap.org/) can filter merchants that accept NFC.

| App | Platform | Read | Write | Emulate | Payloads | Custody | Source | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [Bolt Card PoS](https://play.google.com/store/apps/details?id=org.boltcard.boltcardpos) | Android | ✅ | ❌ | ❌ | Bolt Card | Merchant's own wallet / hub | [GitHub](https://github.com/boltcard/bolt-card-pos) | Reference POS from the Bolt Card project. |
| [Boltcard Tools Terminal](https://apps.apple.com/us/app/boltcard-tools-terminal/id6475311437) | Android, iOS | ✅ | ❌ | ❌ | Bolt Card → any BOLT11; on-chain via swap | Non-custodial (pays a scanned invoice node-to-node) | [GitHub](https://github.com/SwissBitcoinPay/boltcard-tools-terminal) | By Swiss Bitcoin Pay. Scan any Lightning invoice, tap a Bolt Card to pay it. On-chain via a Swiss Bitcoin Pay swap ([release notes](https://www.nobsbitcoin.com/boltcard-terminal-tools-app-v1-0-2/)). |
| [Swiss Bitcoin Pay](https://swiss-bitcoin-pay.ch/) ([iOS](https://apps.apple.com/mx/app/swiss-bitcoin-pay/id6444370155), [F-Droid](https://f-droid.org/en/packages/ch.swissbitcoinpay.checkout/)) | Android, iOS, web | ✅ | ❌ | ❌ | Bolt Card | Non-custodial payout to the merchant | [GitHub](https://github.com/SwissBitcoinPay/app) | Merchant checkout with Bolt Card tap; also [sells cards and terminals](https://swiss-bitcoin-pay.ch/cards). |
| [BTCPay Server](https://btcpayserver.org/) POS + LNURL NFC plugin | Web (Web NFC) | ✅ | ❌ | ❌ | Bolt Card / LNURL-withdraw | Self-hosted | [GitHub](https://github.com/btcpayserver/btcpayserver) | "Pay by NFC & LNURL-Withdraw" button since 1.5.1 via the LNURL NFC Support plugin ([plugin directory](https://plugin-builder.btcpayserver.org/public/plugins), [demo](https://www.youtube.com/watch?v=4m-FQoUAs50)). |
| [LNbits TPoS](https://github.com/lnbits/tpos) | Web (Web NFC) | ✅ | ❌ | ❌ | Bolt Card / LNURL-withdraw | Self-hosted | [GitHub](https://github.com/lnbits/tpos) | Standard shop till for LNbits; card taps supported. |
| [LaWallet POS](https://lawallet.io/) | Android (Ciontek CS30Pro and ZCS Z92 terminals), web | ✅ | ❌ | ❌ | Bolt Card (LaWallet cards follow the Bolt Card standard and LUD-03) | LaWallet instance, connected over NWC | [mobile-pos](https://github.com/lawalletio/mobile-pos), [flutter-pos](https://github.com/lawalletio/flutter-pos) | From the La Crypta community in Argentina; the same group wrote the NWC over NFC draft. |
| [bitPOS](https://bitpos.app/) | Web, plus the posBOX terminal | ✅ | ❌ | ❌ | Bolt Card (LNURL-withdraw) | Non-custodial; the merchant connects their own wallet over NIP-47 | [GitHub](https://github.com/bitPOS-app/bitpos) | Hosted settlement layer. The same repo holds an Android NTAG 424 DNA writer and the ESP32 firmware for [posBOX](#payment-terminals). |
| [Lightning PoS](https://lnpos.online/) | Web (PWA; NFC through Web NFC, so Chrome on Android) | ✅ | ❌ | ❌ | LNURL (LUD-16, LUD-21); NFC via the Web NFC API | Merchant's own Lightning Address or NWC wallet | [GitHub](https://github.com/unllamas/lightning-pos) | Next.js POS; listed in [awesome-nwc](https://github.com/getAlby/awesome-nwc) as supporting NFC payments. |
| [Numo](https://numopay.org/) | Android | ✅ | ❌ | ✅ | Cashu (NUT-18), BOLT11 | Ecash (mint trust), auto-sweep to a Lightning address | [GitHub](https://github.com/cashubtc/Numo) | The merchant phone emulates the tag; the customer's wallet reads the request and writes the token back. Inventory, tips, offline mode ([Bitcoin Magazine](https://bitcoinmagazine.com/news/numo-launches-bitcoin-tap-to-pay-app)). |
| [cashu-pos](https://github.com/babdbtc/cashu-pos) | Android (Expo) | ✅ | ❌ | ✅ | Cashu | Ecash (mint trust) | [GitHub](https://github.com/babdbtc/cashu-pos) | Self-hostable ecash POS with NFC tap-to-pay. |
| [LifPay](https://lifpay.me/) ([iOS](https://apps.apple.com/mx/app/lifpay/id1645840182)) | Android, iOS | ✅ | ❌ | ❌ | Bolt Card | Custodial | Closed | Card issuing plus POS ([what is a Bolt Card](https://blog.lifpay.me/edu/what-is-bolt-card/)). |
| [Flash](https://play.google.com/store/apps/details?id=com.lnflash) | Android | ✅ | ❌ | ❌ | Bolt Card ("Flashcard"), Cashu (in progress) | Custodial | [GitHub](https://github.com/lnflash) | Jamaica / Caribbean card programme; also developing an offline ecash JavaCard (see [Formats](#payment-and-signing-protocols)). |
| Tinkl.it | Android | ✅ | ❌ | ❌ | Bolt Card | n/a | Closed | EU-only per boltcard.org. |
| [OpenCryptoPay](https://opencryptopay.io/) | Existing contactless terminals | ✅ | ❌ | ❌ | LNURL (LUD-01 payload) | DFX settlement | [Landing page only](https://github.com/waalge/OpenCryptoPay-LandingPage) | DFX Swiss' in-person standard, live in SPAR Switzerland. Reuses existing terminal hardware; no machine-readable spec published ⚠️. |
| VoltPay, [ZEBEDEE](https://zebedee.io/), CoinCorner Checkout, Lipa (discontinued) | Mixed | ⚠️ | ❌ | ❌ | Bolt Card | Custodial | Closed | Listed as Bolt Card-capable on boltcard.org; not verified individually. |

Wallets with a merchant mode that also read cards: Wallet of Satoshi,
Blink, Breez (see [Consumer wallets](#consumer-wallets)).

### Signer companion apps

Wallet apps whose NFC use is a two-way exchange with a hardware signer
or key card. "Read" and "Write" here mean the app both receives from and
sends to the device (NDEF read+write, or APDU command/response).

| App | Platform | Read | Write | Emulate | Devices / protocols | Source | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| [Nunchuk](https://nunchuk.io/) | Android, iOS | ✅ | ✅ | ❌ | TAPSIGNER, SATSCARD (cktap); Portal; Coldcard Mk4 / Q (NDEF over NFC-V) | [GitHub](https://github.com/nunchuk-io) | Multisig-oriented; the broadest NFC signer coverage among the wallets listed here ([TAPSIGNER integrations](https://tapsigner.com/faq)). |
| [Bitcoin Keeper](https://bitcoinkeeper.app/) | Android, iOS | ✅ | ✅ | ❌ | TAPSIGNER (cktap); Portal; Coldcard (NFC) | [GitHub](https://github.com/bithyve/bitcoin-keeper) | |
| [Cove](https://covebitcoinwallet.com/) | Android, iOS | ✅ | ✅ | ❌ | TAPSIGNER (cktap) | [GitHub](https://github.com/bitcoinppl/cove) | |
| [Bitkey app](https://bitkey.world/) | Android, iOS | ✅ | ✅ | ❌ | Bitkey (WCA: protobuf over APDU) | [GitHub](https://github.com/proto-at-block/bitkey) | Tap-to-sign gated on the device's fingerprint sensor. |
| [Tangem](https://tangem.com/en/) | Android, iOS | ✅ | ✅ | ❌ | Tangem cards (proprietary card protocol) | [Android](https://github.com/tangem/tangem-app-android), [iOS](https://github.com/tangem/tangem-app-ios), [SDK](https://github.com/tangem/tangem-sdk-android) | App and SDK open source; card firmware closed. |
| [Arculus](https://www.getarculus.com/) | Android, iOS | ✅ | ✅ | ❌ | Arculus card (proprietary) | Closed | Three-factor tap (card + PIN + biometrics). |
| [OneKey](https://onekey.so/) | Android, iOS | ✅ | ✅ | ❌ | OneKey Lite | [GitHub](https://github.com/OneKeyHQ/app-monorepo) | Seed backup and restore only; no signing over NFC. |

Desktop clients that reach an NFC card through a PC/SC reader rather
than a phone: [Electrum](https://electrum.org/) and
[Sparrow](https://sparrowwallet.com/) for Satochip; Cypherock's
[cySync](https://www.cypherock.com/) (the NFC link there is between the
X1 vault and its cards, not the computer).

### Tag programming tools

| App | Platform | Read | Write | Chips / payloads | Source | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| [Bolt Card NFC Card Creator](https://play.google.com/store/apps/details?id=com.lightningnfcapp) | Android | ✅ | ✅ | NTAG 424 DNA: keys K0–K4, SDM config, `lnurlw://` URL | [GitHub](https://github.com/boltcard/bolt-nfc-android-app) | The reference programmer. Card services (LNbits, Bolt Card Hub) hand it the key material via a deep link or QR. Also wipes/resets cards. |
| [Boltcard NFC Programmer](https://apps.apple.com/us/app/boltcard-nfc-programmer/id6450968873) | iOS | ✅ | ✅ | NTAG 424 DNA | [GitHub](https://github.com/boltcard/bolt-card-programmer) | iOS counterpart of the Card Creator; the linked repository is the updated, TypeScript-based programmer. |
| [LaWallet Boltcard Installer](https://github.com/lawalletio/card-installer) | Android | ✅ | ✅ | NTAG 424 DNA: keys and `lnurlw`; tap a programmed card to wipe it | [GitHub](https://github.com/lawalletio/card-installer) | Fork of the reference Card Creator for LaWallet and Bolt Card servers. The companion [Card Manager](https://github.com/lawalletio/card-manager) activates cards on a ZCS Z92 terminal and prints the activation QR; [Wiper](https://github.com/lawalletio/wiper) resets them. |
| bitPOS card-writer | Android | ✅ | ✅ | NTAG 424 DNA | [GitHub](https://github.com/bitPOS-app/bitpos) (`artifacts/card-writer`) | Programmer for cards used with bitPOS. |
| [NFC Tools](https://www.wakdev.com/en/apps/nfc-tools.html) ([Android](https://play.google.com/store/apps/details?id=com.wakdev.wdnfc)) | Android, iOS | ✅ | ✅ | Any NDEF record on any tag type | Closed | The usual way to write a static `bitcoin:` / `lightning:` URI or Lightning Address to a cheap tag ([LightningNFC recipe](https://github.com/theDavidCoen/LightningNFC)). |
| [NXP TagWriter](https://play.google.com/store/apps/details?id=com.nxp.nfc.tagwriter) | Android | ✅ | ✅ | Any NDEF; NXP chip features | Closed | Vendor tool; handy for URI records and for checking what a chip supports. |
| [NXP TagInfo](https://play.google.com/store/apps/details?id=com.nxp.taginfolite) | Android | ✅ | ❌ | Chip identification, memory dump | Closed | Tells you whether a card really is an NTAG 424 DNA and shows the SDM-mirrored URL as read. |

For scripted or desktop programming see
[Libraries and developer tools](#libraries-and-developer-tools); for a
standalone hardware writer see Bolty under
[Payment terminals](#payment-terminals).

## Devices

### Cards, rings and implants

Bearer credentials on the payment side. Electrically they are all the
same thing: an NXP NTAG 424 DNA (AES-128, ISO 14443-A, NFC Forum
Type 4) programmed with an `lnurlw://` URL plus SUN/SDM keys. Only the
form factor differs, so any of them is a valid test target for a Bolt
Card reader.

| Item | Form | Chip | Protocol | Program it yourself | Availability | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| [Bolt Card](https://www.boltcard.org/) (DIY) | Card | NTAG 424 DNA | Bolt Card | ✅ | Buy blanks, program with the [NFC Card Creator](#tag-programming-tools) | The open reference; [spec](https://github.com/boltcard/boltcard/blob/main/docs/SPEC.md), [system overview](https://github.com/boltcard/boltcard/blob/main/docs/SYSTEM.md). |
| Blank NTAG 424 DNA cards, stickers, fobs | Card / sticker / fob | NTAG 424 DNA | Any | ✅ | Laser Eyes Cards, PlebTag, ZipNFC, shopNFC, Identiv, NFC.CARDS, Yanabu (KR); [vendor list](https://www.boltcard.org/), [LNbits shop](https://shop.lnbits.com/product/lnbits-boltcards) | A sticker or key fob behaves identically to a card. [PlebTag](https://plebtag.com/nfc-news/) tracks community builds. |
| [CoinCorner Bolt Card](https://support.coincorner.com/hc/en-us/articles/5600580995346-The-Bolt-Card) | Card | NTAG 424 DNA | Bolt Card | ❌ (issued and hosted by CoinCorner) | CoinCorner account | The original commercial issuance ([launch coverage](https://bitcoinmagazine.com/business/coincorner-releases-the-bolt-card-for-bitcoin)). |
| [The Bolt Card](https://theboltcard.com/) | Card + hosting | NTAG 424 DNA | Bolt Card | ❌ | Retail | Card plus a hosting front-end for the open project. |
| [BoltRing](https://bitcoin-ring.com/) | Ceramic ring | NTAG 424 DNA | Bolt Card | ✅ (self-hosted service or CoinCorner) | Retail | [Docs](https://docs.bolt-ring.com/). Same team publishes [boltlib](#libraries-and-developer-tools) and VirtualBoltcardApp. |
| [EB NFC Lightning Card](https://europeanbitcoiners.com/ebnfc/) | Card | NTAG 424 DNA | Bolt Card | ✅ | Community | European Bitcoiners. |
| [Tiankii Boltcard](https://www.tiankii.com/) | Card | NTAG 424 DNA | Bolt Card | ❌ | Tiankii account | LatAm-focused card plus POS. |
| Flash "Flashcard" | Card | NTAG 424 DNA | Bolt Card | ❌ | [Flash](https://play.google.com/store/apps/details?id=com.lnflash) account | Caribbean programme on Flash's Lightning infrastructure. |
| NFC implants | Bioglass / flex implant | [xSIID](https://forum.dangerousthings.com/t/xsiid-has-been-released/5585) (NTAG I²C + LED), [FlexSecure](https://dangerousthings.com/product/flexsecure/) (JavaCard) | Bolt Card ⚠️ | ✅ | Dangerous Things | [LightningPaw](https://github.com/f418me/LightningPaw) is the build guide ([coverage](https://cointelegraph.com/news/not-medical-advice-bitcoiner-implants-lightning-chip-to-make-btc-payments-by-hand)). Whether an implant can do SUN/SDM depends on its chip: an NTAG I²C can only hold a static URI. |
| [SATSCARD](https://getsatscard.com) | Card | Coinkite secure element | cktap + dynamic NDEF URL | ❌ (sealed at the factory; you unseal slots) | Retail | Bearer *savings* card, not tap-to-pay: ten sequential key slots, "tap to see the address". |
| [Satodime](https://github.com/Toporin/Satodime-Applet) | JavaCard | Any JavaCard | Satodime applet (AGPL) | ✅ (DIY) | [Satochip](https://satochip.io/) or DIY | Bearer card whose private key is never revealed until unsealed; hand it over like cash. |

### Hardware signers and key cards

NFC as a wire for PSBTs, xpubs and signatures. Every vendor speaks its
own protocol; the payload formats are in
[Payment and signing protocols](#payment-and-signing-protocols).

| Device | Form | Radio / tag type | Protocol | NFC carries | Open firmware | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| [Coldcard Mk4 / Q](https://coldcard.com/) | Full signer | **NFC-V** (ISO 15693), NFC Forum Type 5 | Coldcard NDEF record set; NFC-PushTx | PSBTs in and out, signed transactions, addresses, wallet exports; ~8 KB practical ceiling | [Source-viewable](https://github.com/Coldcard/firmware) | NFC is off by default and the antenna is physically removable. Traffic is unencrypted by design ([NFC tools](https://coldcard.com/docs/nfc-tools/), [signing guide](https://coldcard.com/guides/using-coldcard/coldcard-nfc-signing)). The only mainstream signer that is not 14443-A. |
| [TAPSIGNER](https://tapsigner.com/) | Card | ISO 14443-A | cktap | BIP-32 signing, PIN (CVC) over an ECDH-encrypted session | Protocol open, applet closed | Coinkite. Integrated by Nunchuk, Keeper, Cove and others ([FAQ](https://tapsigner.com/faq)). |
| [SATSCARD](https://getsatscard.com) / [SATSCHIP](https://satschip.com) | Card / embedded chip | ISO 14443-A | cktap + [dynamic NDEF URL](https://dev.coinkite.cards/docs/nfc-spec.html) | Bearer key slots; SATSCHIP proves artwork authenticity | Protocol open, applet closed | [Protocol](https://dev.coinkite.cards/docs/protocol.html). |
| [Portal](https://twenty-two.xyz/) (TwentyTwo) | Card-format, **no battery** | ISO 14443-A | Portal protocol (Rust / BDK) | Full signer flow, powered entirely by the phone's NFC field | [Open](https://github.com/TwentyTwoHW/portal-software) | The tap is both transport and power supply. [Store](https://store.twenty-two.xyz/). Integrated by Nunchuk and Keeper. |
| [Bitkey](https://bitkey.world/) (Block) | Fob with fingerprint sensor | ISO 14443-A | WCA: protobuf over APDU; NDEF for firmware update only | Signing, gated on the fingerprint | [Open](https://github.com/proto-at-block/bitkey) | 2-of-3 with phone and server keys ([launch](https://block.xyz/inside/block-launches-bitkey-wallet-with-screen-automatic-bitcoin-earning-on-cash-app-and-proof-of-reserves)). |
| [Satochip](https://satochip.io/) | JavaCard, dual interface | ISO 14443-A or contact | Satochip applet | BIP-32 signing | [AGPL](https://github.com/Toporin/SatochipApplet) | Works with Electrum and Sparrow through a PC/SC reader; [DIY on blank JavaCards](https://github.com/3rdIteration/Satochip-DIY). |
| Seedkeeper | JavaCard | ISO 14443-A or contact | Seedkeeper applet | Secret / seed storage | AGPL | Same family as Satochip. |
| [Keycard](https://keycard.tech/pages/keycard) | Card, EAL6+ secure element | ISO 14443-A | Keycard applet | Non-extractable keys; BTC and ETH signing | [Open](https://github.com/keycard-tech/status-keycard) | The sibling *Keycard Shell* is a QR air-gap device, not NFC. |
| [Tangem](https://tangem.com/en/) | Set of 2–3 cards | ISO 14443-A | Tangem card protocol | Seedless multi-card key storage and signing | Firmware closed; app and SDK open | |
| [Cypherock X1](https://www.cypherock.com/how-it-works) | Vault + 4 NFC cards | ISO 14443-A | Encrypted vault ↔ card link | Shamir shares | [Open](https://github.com/Cypherock/x1_wallet_firmware) | NFC is between the vault and its cards, not between a phone and the device. |
| Arculus | Metal card | ISO 14443-A | Proprietary | Signing with card + PIN + biometrics | Closed | [Third-party review](https://www.cypherock.com/blogs/arculus-cold-storage-wallet-review-is-it-the-right-choice-for-long-term-holders). |
| [OneKey Lite](https://onekey.so/products/onekey-lite/) | Card, EAL6+ | ISO 14443-A | Proprietary | **Seed backup only**, no signing | Closed | |
| Ledger Flex / Stax | Full signer | NFC hardware per retail specs | n/a | ⚠️ No Ledger-documented bitcoin NFC flow found | Closed | [Review](https://coinbureau.com/review/ledger-flex-review) mentions the chip; treat as unusable for NFC testing until documented. |
| [Trezor Safe 7](https://trezor.io/trezor-safe-7) | Full signer | ⚠️ Disputed | n/a | n/a | Open | Reviews call it "the first Trezor with NFC"; Trezor's own page lists only Bluetooth 5.0+, USB-C and Qi2. |
| [Passport Prime](https://docs.foundation.xyz/prime/prime-apps/wallet/) | Full signer | NFC module shipped | n/a | ⚠️ NFC signing announced for a future firmware; today QR, BLE (QuantumLink) and USB | Open | [Launch coverage](https://bitcoinmagazine.com/business/passport-prime-a-new-security-device-for-a-new-generation). |

### Payment terminals

| Device | Type | NFC role | Payloads | Open | Availability | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| [Bitcoinize POS v2.1](https://bitcoinize.com/product/bitcoinize-point-of-sale-machine/) | Android 14 (de-Googled) handheld, 6", thermal printer, 4G | Reads Bolt Cards through the installed apps; **NFC disabled in offline mode** | Bolt Card; on-chain, Lightning and Liquid invoices by QR | Hardware closed | Retail, ~USD 221 | Ships pre-loaded with Wallet of Satoshi POS, Breez, Blink, Bitkit, Aqua, Zeus and Swiss Bitcoin Pay. |
| [Opago POS](https://www.opago.com/en/) | Dedicated Lightning terminal | Customer taps a supported card | Bolt Card | Closed | Retail | Works online and offline ([write-up](https://coincharge.io/en/opago-terminal/)). |
| Generic Android POS terminals (Sunmi V2, ZCS Z92, Ciontek CS30Pro and similar) | Bare Android terminal | Whatever the installed app does | Depends on the app | n/a | Retail ([example](https://shop.bitcoinbazis.hu/en/termek/bitcoin-android-14-4g-pos-terminal/)) | The hardware most "bitcoin terminals" actually are; buy bare and install a [POS app](#merchant-and-pos-apps). LaWallet targets the ZCS Z92 and the Ciontek CS30Pro. |
| [LNPoS](https://github.com/lnbits/lnpos) | LilyGo / ESP32 DIY terminal | QR-first; NFC via add-on reader | Offline LNURL vouchers, online invoices, LNURL-withdraw ATM mode, on-chain via xpub | Open | Kit from the [LNbits shop](https://shop.lnbits.com/product/bitcoin-lnpos-device) | [Overview](https://coincharge.io/en/lnpos-hardware-lightning-pos-terminal/). |
| [bitcoinSwitch](https://github.com/lnbits/bitcoinswitch) / [offlineLNSwitch](https://github.com/AxelHamburch/offlineLNSwitch) | ESP32 "pay to switch a relay" (taps, vending, arcade) | LNURL / QR; NFC in some builds | LNURL | Open | DIY | |
| [posBTC-NFC](https://github.com/Fabio-Menjivar/posBTC-NFC) | Cheap NFC reader + LNbits backend | Card → POS | Bolt Card | Open | DIY | Non-custodial. |
| posBOX (bitPOS) | ESP32 + PN532 | Reads Bolt Cards | Bolt Card via the bitPOS service | Open | DIY; firmware in the [bitPOS repo](https://github.com/bitPOS-app/bitpos) (`artifacts/esp32-pos`) | Stores Wi-Fi credentials and a bearer token for the hosted service, no keys. |
| [RapiPOX](https://github.com/RapiPOX/client) | DIY hardware POS with a PN532 NFC module | Card reader | Lightning and Nostr | Open | DIY | Work in progress. |
| [ESP32 Lightning POS](https://github.com/Libertariamemes/BTC-Lightning-POS-esp32) | Touchscreen DIY POS | QR, NFC variants | Lightning | Open | DIY | |
| [Square Register (2nd gen)](https://squareup.com/us/en/bitcoin) | Mainstream fiat terminal | Buyer taps a phone on the customer display; **protocol unpublished** ⚠️ | Lightning settlement | Closed | Retail; ~1M US merchants enabled | Cash App, Coinbase and Wallet of Satoshi are named as compatible wallets ([help article](https://squareup.com/help/us/en/article/8622-accept-and-manage-bitcoin-payments), [Block](https://block.xyz/inside/block-launches-bitkey-wallet-with-screen-automatic-bitcoin-earning-on-cash-app-and-proof-of-reserves)). Which side emulates is not documented. |
| [Nucula](https://github.com/zeugmaster/nucula) | ESP32-C6 Cashu wallet / terminal | Ecash tap-to-pay | Cashu | Open | DIY | |
| [Cashu Pixel](https://github.com/swedishfrenchpress/cashu-pixel-rs) | Raspberry Pi offline ecash device with touchscreen | Ecash | Cashu | Open | DIY | |
| Bolty | ESP32 + PN532 | Card **writer** for issuing Bolt Cards on your own hardware | NTAG 424 DNA programming | Open | DIY | Via the [LNbits tooling wiki](https://github.com/lnbits/lnbits/wiki/Tooling-&-Building-with-LNbits). |

### ATMs and vending machines

| Device | What it does | NFC role | Open | Link |
| --- | --- | --- | --- | --- |
| The B.A.T. (Bitcoin Auto Teller) | Portable ESP32 Lightning ATM: cash in, sats out via Bolt Card, LNURL or Lightning Address; thermal receipts, gift-voucher mode, LNbits-funded | Reads a Bolt Card as the withdrawal target; claims to be the first ATM doing NFC withdrawals | Open | [GitHub](https://github.com/cos-it-is/TheBAT) |
| BitVend | Retrofit module that puts Lightning payments into existing vending machines | Pay by tapping a Bolt Card | n/a | [bitvend.twentyuno.net](https://bitvend.twentyuno.net/) |
| House of Satoshi vending | OpenCryptoPay-connected vending machine | QR-based; NFC only insofar as the OpenCryptoPay terminal provides it ⚠️ | n/a | [OpenCryptoPay ecosystem](https://opencryptopay.io/ecosystem.html) |
| General Bytes BATM | Mainstream ATM line | NFC card issuance mentioned in passing; no Bolt Card support documented ⚠️ | Closed | [generalbytes.com](https://www.generalbytes.com/en/products/batmthree) |

## Formats

NFC "protocols" in bitcoin stack up in layers, and conversations get
confused when a product (Bolt Card) is named as if it were an encoding
(NDEF URI). The three tables below go from the radio up. The long-form
version, with the Bolt Card tap walked through step by step, is
[docs/protocol-landscape.md](docs/protocol-landscape.md).

### Radio and tag types

| Format | What it is | Where it appears | Spec | Test with |
| --- | --- | --- | --- | --- |
| **ISO/IEC 14443** Type A / B | 13.56 MHz proximity cards; Type A is what nearly everything in this list uses | NTAG 21x, NTAG 424 DNA, every JavaCard and secure-element card, HCE | [Wikipedia](https://en.wikipedia.org/wiki/ISO/IEC_14443) | Any card here except the Coldcard |
| **ISO/IEC 15693** (NFC-V, "vicinity") | Longer-range 13.56 MHz standard; NFC Forum **Type 5** tag | Coldcard Mk4 / Q | [Wikipedia](https://en.wikipedia.org/wiki/ISO/IEC_15693) | Coldcard; Android `NfcV`, iOS `NFCISO15693Tag` |
| **NFC Forum Type 2 tag** | Cheap static NDEF memory, no cryptography | NTAG213 / 215 / 216 stickers: tip jars, donation plaques, table tents; xSIID implant | [NFC Forum](https://nfc-forum.org/build/specifications) | A cent-priced tag written with NFC Tools |
| **NFC Forum Type 4 tag** | NDEF over an ISO 7816 file system; contents can be dynamic; the type HCE emulates | NTAG 424 DNA (Bolt Card); the emulated tags of Numo and Minibits | [NFC Forum](https://nfc-forum.org/build/specifications) | Bolt Card; Numo |
| **NTAG 424 DNA SUN / SDM** | Chip feature: on every read the tag mirrors an AES-encrypted UID + tap counter and a CMAC into the NDEF URL | The entirety of Bolt Card's replay protection | [NXP AN12196](https://www.nxp.com/docs/en/application-note/AN12196.pdf) | Blank card + NFC Card Creator; [Ntag424SdmFeature](https://github.com/AndroidCrypto/Ntag424SdmFeature) |
| **Host Card Emulation** | A phone app answering as a card / Type 4 tag | VirtualBoltcardApp (as a Bolt Card), Numo and Minibits (as a payment-request tag) | [Android](https://developer.android.com/develop/connectivity/nfc/hce); [Apple's restrictions](https://developer.apple.com/support/hce-transactions-in-apps) | Any Android phone |
| **JavaCard** | Programmable smartcard platform behind most open key cards | Satochip / Satodime / Seedkeeper / Satocash, Keycard, cashu-javacard, FlexSecure implant | [Oracle](https://www.oracle.com/java/java-card/) | Blank JavaCards from any smartcard vendor + the AGPL applets |
| ISO/IEC 18092 / NFC-F (FeliCa) | Sony's tag family | No bitcoin use found | n/a | n/a |

### Framing

| Record / framing | Encoding | Used for | Reference |
| --- | --- | --- | --- |
| **NDEF URI record** (TNF Well-Known, type `U`) | One-byte prefix code plus the URI. `bitcoin:`, `lightning:` and `lnurlw:` have no assigned code, so the record uses `0x00` ("whole URI follows") | Static tags, Bolt Card, NFC-PushTx, Coinkite dynamic URLs, OS-level URL routing | [URI record format](https://gototags.com/help/nfc/ndef/record-types/uri) |
| **NDEF Text record** (type `T`) | Language code + UTF-8 | Human labels next to binary records ("Partly signed PSBT", "Deposit Address") | Coldcard |
| **NDEF MIME record** | Media type + bytes | `application/json` wallet exports (Coldcard); a serialised BIP-70 `PaymentRequest` (legacy) | [BIP-70 over NDEF](https://github.com/bitcoin-wallet/bitcoin-wallet/wiki/Payment-Requests) |
| **NDEF External record** (`urn:nfc:ext:`) | Vendor-namespaced binary | `bitcoin.org:psbt`, `bitcoin.org:txn`, `bitcoin.org:txid`, `bitcoin.org:sha256` | [Coldcard](https://github.com/Coldcard/firmware/blob/master/docs/nfc-coldcard.md) |
| **ISO/IEC 7816-4 APDU** | Command / response frames to an applet selected by AID; skips NDEF entirely | cktap, Bitkey, Satochip, Keycard, Portal, Tangem, ecash JavaCards | [Wikipedia](https://en.wikipedia.org/wiki/ISO/IEC_7816) |
| CBOR over APDU | One APDU, everything else CBOR | cktap | [Coinkite](https://dev.coinkite.cards/docs/protocol.html) |
| Protobuf over APDU | "WCA" | Bitkey | [Bitkey firmware](https://github.com/proto-at-block/bitkey) |
| URL fragment encodings | State packed into `#k=v&…` so a browser can act on it with no app | NFC-PushTx (`t`, `c`, `n`); Coinkite card URLs (`u`, `o`, `r`, `n`, `s`) | [nfc-pushtx](https://github.com/Coldcard/firmware/blob/master/docs/nfc-pushtx.md), [Coinkite NFC spec](https://dev.coinkite.cards/docs/nfc-spec.html) |

### Payment and signing protocols

| Protocol | Carrier | Direction | Payload | Spec | Status | Test with |
| --- | --- | --- | --- | --- | --- | --- |
| **BIP-21** `bitcoin:` URI | NDEF URI | Tag → phone (receive side) | Address, `amount`, `label`, de facto `lightning=<bolt11>` | [BIP-21](https://github.com/bitcoin/bips/blob/master/bip-0021.mediawiki), [bitcoinqr.dev](https://bitcoinqr.dev/) | Universal | NFC Tools + any wallet that reads NDEF |
| **BIP-321** URI | NDEF URI | Tag → phone | Address-less URIs, several payment instructions in one: BOLT11, **BOLT12 offers**, silent payments, **Ark** | [BIP-321](https://bips.dev/321/), [PR #1555](https://github.com/bitcoin/bips/pull/1555) | Emerging; the only standard slot for an Ark instruction on a tag | Write one with NFC Tools; wallet support still sparse |
| BIP-70 / 71 / 72 over NDEF | NDEF URI (`?r=`) or MIME `PaymentRequest`; Android Beam for phone-to-phone | Tag / phone → phone | Signed payment request | [Bitcoin Wallet wiki](https://github.com/bitcoin-wallet/bitcoin-wallet/wiki/Payment-Requests) | Dead: [BIP-70 deprecated](https://bitcoinops.org/en/topics/bip70-payment-protocol/), Beam removed | Historical only |
| **Bolt Card** | NDEF URI `lnurlw://host?p=…&c=…` regenerated by NTAG 424 SUN on every tap | Card → POS (pull) | `p` = AES-encrypted UID + monotonic counter, `c` = CMAC; the service rejects non-increasing counters, then answers a LUD-03 withdraw request | [SPEC.md](https://github.com/boltcard/boltcard/blob/main/docs/SPEC.md), [SYSTEM.md](https://github.com/boltcard/boltcard/blob/main/docs/SYSTEM.md) | The one widely deployed, openly specified NFC payment protocol in bitcoin | LNbits Boltcards + any [POS app](#merchant-and-pos-apps) |
| **LUD-17** scheme prefixes | URI scheme | n/a | `lnurlw://`, `lnurlp://`, `lnurlc://`, `keyauth://` replace `https://` so a tag says which sub-protocol it carries and the OS can route it to a wallet | [LUD-17](https://github.com/lnurl/luds/blob/luds/17.md) | Deployed | Every Bolt Card |
| **LUD-03** withdrawRequest, LUD-08 fast withdraw, LUD-14 balanceCheck, LUD-19 pay-link-from-withdraw | HTTPS after the tap | POS ↔ card service | Withdraw callback; LUD-19 lets a till top a card up instead of only charging it | [LUD-03](https://github.com/lnurl/luds/blob/luds/03.md), [luds](https://github.com/lnurl/luds) | Deployed | LNbits, Bolt Card Hub |
| LUD-01 bech32 LNURL, **LUD-06** payRequest, **LUD-16** Lightning Address | NDEF URI `lightning:LNURL1…` or `lightning:user@domain` | Tag → phone (push) | Static receive link | [luds](https://github.com/lnurl/luds) | Deployed; the cheap tip-jar tag | NTAG213 + NFC Tools ([recipe](https://github.com/theDavidCoen/LightningNFC)) |
| Card PIN: `pinLimit` | LNURL response field | POS ↔ card service | Require a PIN above an amount | Cited as "LUD-21" by Bolt Card, as "[LUD-290](https://blog.btcpayserver.org/btcpay-server-1-6-0/)" by BTCPay; merged LUD-21 is something else | De facto extension with divergent numbering | BTCPay Server |
| Static LNURL-withdraw tag | NDEF URI | Tag → POS | A fixed withdraw link: "an offline prepaid card", first tapper wins, no replay protection | [LNbits wiki](https://github.com/lnbits/lnbits/wiki) | Vouchers only | LNbits withdraw extension |
| LNURL-auth `keyauth://` on a tag; Nostr equivalent | NDEF URI | Tag → app | Login credential | [LUD-04](https://github.com/lnurl/luds/blob/luds/04.md), [nostrnfcauth](https://github.com/blackcoffeexbt/nostrnfcauth) | Niche; shares the tag plumbing | n/a |
| **OpenCryptoPay** (DFX) | Existing contactless terminals; a `lightning=` bech32 LNURL (LUD-01) in the payload resolves to the till's API | Terminal → phone | LNURL | [opencryptopay.io](https://opencryptopay.io/), [what is it](https://opencryptopay.io/what-is-open-cryptopay.html) | Live in SPAR Switzerland; no machine-readable spec published ⚠️ | A SPAR till |
| BOLT11 invoice on a tag | NDEF URI `lightning:lnbc…` | Tag / emulated tag → phone | One-shot invoice | No spec | Used by Minibits and Numo emulated tags | Minibits HCE host |
| **BOLT12 offer on a tag** | NDEF URI (BIP-321 gives it a slot) | Tag → phone | Reusable static offer: no per-payment server round trip, better receiver privacy | [BOLT 12](https://github.com/lightning/bolts/blob/master/12-offer-encoding.md) | No implementation found; an open niche | n/a |
| Square bitcoin tap-to-pay | Unknown | Phone ↔ Register 2nd gen | Lightning settlement | Unpublished ⚠️ | Live at ~1M US merchants | A Square Register |
| **NWC over NFC** | Emulated Type 4A tag (Android HCE), AID `F005570202`. NDEF record `nostr+walletconnect+request://relay=<url>`, then APDUs: `80 10` carries the BOLT11 from POS to wallet in 200-byte chunks, `80 20` carries the signed NIP-47 event back | Customer phone → POS; settlement through a Nostr relay | A signed `pay_invoice` event (kind 23194) that the POS publishes. Either side may be offline: a wallet with connectivity pays directly and hands back the preimage | [Draft 0.1, July 2025](https://github.com/agustinkassis/nwc-nfc) (La Crypta) | Draft. No replay protection or encryption yet; an iOS reader is listed as future work | Two Android phones; no public implementation confirmed ⚠️ |
| **Coldcard NDEF record set** | NDEF text / external / MIME records over NFC-V | Phone ↔ signer | `bitcoin.org:psbt` (BIP-174 binary), `:txn`, `:txid`, `:sha256`, JSON exports; record order not fixed; ~8 KB ceiling, larger PSBTs fall back to microSD or QR | [nfc-coldcard.md](https://github.com/Coldcard/firmware/blob/master/docs/nfc-coldcard.md), [docs](https://coldcard.com/docs/nfc-tools/) | Deployed | Coldcard + Nunchuk / Keeper |
| **NFC-PushTx** | NDEF URI with a URL fragment | Signer → any phone browser, no app | `https://…/pushtx#t=<base64url tx>&c=<base64url sha256(tx)[-8:]>&n=<network>`; ~8 KB | [nfc-pushtx.md](https://github.com/Coldcard/firmware/blob/master/docs/nfc-pushtx.md) | Deployed; [mempool.space](https://mempool.space) hosts the endpoint natively | Coldcard + any phone |
| **cktap** (Coinkite Tap Protocol) | APDU + CBOR, AID `f0436f696e6b697465434152447631`; protected commands over an ephemeral ECDH session | Phone ↔ card | Status, derive, sign, unseal, CVC | [protocol.html](https://dev.coinkite.cards/docs/protocol.html) | Deployed | TAPSIGNER / SATSCARD; [coinkite-tap-proto](https://github.com/coinkite/coinkite-tap-proto), [rust-cktap](https://github.com/bitcoindevkit/rust-cktap) |
| Coinkite dynamic NDEF URL | NDEF URI | Card → phone with no app | `https://getsatscard.com/start#u=<state>&o=<slot>&r=<addr tail>&n=<nonce>&s=<sig>` (TAPSIGNER: `t=1`, `c=<card id>`), signed by the card | [nfc-spec.html](https://dev.coinkite.cards/docs/nfc-spec.html) | Deployed | SATSCARD |
| **Bitkey WCA** | Protobuf over APDU; NDEF for firmware update only | Phone ↔ fob | Signing, fingerprint-gated | [Firmware](https://github.com/proto-at-block/bitkey) | Deployed | Bitkey |
| Satochip applet protocol | APDU | Host ↔ JavaCard | BIP-32 signing (Satochip), bearer key (Satodime), secrets (Seedkeeper) | [SatochipApplet](https://github.com/Toporin/SatochipApplet), [Satodime](https://github.com/Toporin/Satodime-Applet) | Deployed, AGPL | DIY JavaCard |
| Keycard applet protocol | APDU with secure channel | Host ↔ card | Signing with non-extractable keys | [status-keycard](https://github.com/keycard-tech/status-keycard) | Deployed | Keycard |
| Portal | APDU-based; the phone's field powers the device | Phone ↔ card | Full signer flow | [portal-software](https://github.com/TwentyTwoHW/portal-software) | Deployed | Portal + Nunchuk / Keeper |
| Tangem card protocol | APDU | Phone ↔ card | Signing across a card set | [SDK](https://github.com/tangem/tangem-sdk-android) (firmware closed) | Deployed | Tangem |
| Cypherock X1 card link | Encrypted NFC between vault and cards | Vault ↔ card | Shamir shares | [Firmware](https://github.com/Cypherock/x1_wallet_firmware) | Deployed | Cypherock X1 |
| **Cashu NUT-18 over an emulated tag** | NDEF on an HCE Type 4 tag; the customer writes the token back | POS → phone, then phone → POS, in one tap | NUT-18 payment request; NFC is not a registered NUT-18 transport, so this is an ad hoc pattern | [NUT-18](https://github.com/cashubtc/nuts/blob/main/18.md) | Deployed (Numo, Minibits); works offline on both sides | Numo + Minibits |
| Cashu token on a tag | NDEF | Card → phone | Bearer token; first reader wins | [cashu.me demo](https://x.com/callebtc/status/1868333141032649024) | Experimental | cashu.me + any tag |
| Ecash JavaCard applets | APDU | Card ↔ terminal | Offline ecash payments from a card | [cashu-javacard](https://github.com/lnflash/cashu-javacard), [Satocash](https://github.com/Toporin/Satocash-Applet), [μNuts](https://github.com/Amperstrand/micronuts) | Experimental | DIY JavaCard |
| Fedimint | n/a | n/a | Offline spends possible in principle ([Bitcoin Design](https://bitcoin.design/guide/how-it-works/ecash/fedimint/)) | n/a | No NFC transport or implementation found ⚠️ | n/a |
| **Ark / Arkade / Spark** | n/a | n/a | n/a | n/a | **Nothing exists**: no protocol, no implementation, no proposal. Options are laid out in [docs/protocol-landscape.md](docs/protocol-landscape.md#7-ark--arkade--spark-the-gap) | n/a |

## Card issuing and hosting services

A Bolt Card is worthless without a service that holds the AES keys and
the funds. Self-hosted options first; they are also what you want on a
test bench, since you control the keys and can watch the counter.

| Service | Model | Backend | Source | Notes |
| --- | --- | --- | --- | --- |
| [boltcard](https://github.com/boltcard/boltcard) (reference service) | Self-hosted | LND | [GitHub](https://github.com/boltcard/boltcard) | Issues and verifies cards; the spec's own implementation. |
| [Bolt Card Hub](https://github.com/boltcard/hub) | Self-hosted, multi-user | LND (LNDHub fork); "Phoenix Edition" variant | [boltcard-lndhub](https://github.com/boltcard/boltcard-lndhub), [docker](https://github.com/boltcard/boltcard-lndhub-docker) | Several users share one node; pairs with Bolt Card Wallet. |
| [LNbits Boltcards extension](https://github.com/lnbits/boltcards) | Self-hosted or hosted LNbits | Any LNbits funding source | [GitHub](https://github.com/lnbits/boltcards) | Per-card limits, fresh URL per tap, deep link into the NFC Card Creator. The [README](https://github.com/lnbits/boltcards/blob/main/README.md) documents the key roles (K0–K4) and SDM settings. URL shape `lnurlw://<host>/boltcards/api/v1/scan/<id>?p=…&c=…`. |
| [TagID](https://github.com/AxelHamburch/tagid_extension) | LNbits extension | Any LNbits funding source | [GitHub](https://github.com/AxelHamburch/tagid_extension) | Fork of the Boltcards extension with a per-card PIN threshold, and a fix so that failed payments no longer consume the daily limit. |
| [LaWallet NWC](https://github.com/lawalletio/lawallet-nwc) | Self-hosted (Umbrel and StartOS packages) | Any NWC wallet | [GitHub](https://github.com/lawalletio/lawallet-nwc), [card module](https://github.com/lawalletio/card) | Lightning Address platform with Bolt Card fleet management: card designs, NTAG 424 cards, activation. Cards follow the Bolt Card standard and LUD-03. |
| [Boltcard + NWC](https://github.com/jpgaviria2/boltcard-nwc) | Self-hosted | The user's own NWC wallet | [GitHub](https://github.com/jpgaviria2/boltcard-nwc) | Create, manage and serve Bolt Cards paid from a Nostr Wallet Connect connection. |
| [Nostr PHP Boltcard Server](https://github.com/dsbaars/nostr-php-nwc-boltcard) | Self-hosted | Any NWC wallet | [GitHub](https://github.com/dsbaars/nostr-php-nwc-boltcard) | FlightPHP showcase for the nostr-php-nwc library; the author says it is not production code. |
| [CoinCorner](https://support.coincorner.com/hc/en-us/articles/5600580995346-The-Bolt-Card), [LifPay](https://blog.lifpay.me/edu/what-is-bolt-card/), Tiankii, Flash | Custodial issuers | Their own | Closed | Card comes tied to an account. |

Also relevant: [BTCMap](https://btcmap.org/) filters merchants by NFC
acceptance, which is the practical way to find out whether taps are
actually used in a given city.

## Libraries and developer tools

| Tool | Language | What it does | Link |
| --- | --- | --- | --- |
| boltlib | Python | Read and write Bolt Cards (NTAG 424 DNA) from a PC/SC reader | [GitHub](https://github.com/bitcoin-ring/boltlib) |
| sdm-backend | Python | Generic NTAG 424 SUN/SDM verification: decrypt PICCData, check the CMAC | [GitHub](https://github.com/nfc-developer/sdm-backend) |
| pylibsdm | Python | SDM library | [PyPI](https://pypi.org/project/pylibsdm/1.0.0a0.dev0) |
| Ntag424SdmFeature | Java / Android | Walkthrough of the NTAG 424 DNA chip features with working code | [GitHub](https://github.com/AndroidCrypto/Ntag424SdmFeature) |
| bolt-nfc-android-app | Kotlin / Android | Source of the reference card programmer | [GitHub](https://github.com/boltcard/bolt-nfc-android-app) |
| coinkite-tap-proto | Python | Reference cktap client for SATSCARD / TAPSIGNER | [GitHub](https://github.com/coinkite/coinkite-tap-proto) |
| rust-cktap | Rust | BDK's cktap implementation | [GitHub](https://github.com/bitcoindevkit/rust-cktap) |
| portal-software | Rust | Portal firmware and SDK | [GitHub](https://github.com/TwentyTwoHW/portal-software) |
| Bitkey firmware | C / Rust / Kotlin / Swift | Firmware, WCA protobufs, mobile app | [GitHub](https://github.com/proto-at-block/bitkey) |
| Coldcard firmware docs | n/a | NDEF record set and NFC-PushTx reference | [nfc-coldcard.md](https://github.com/Coldcard/firmware/blob/master/docs/nfc-coldcard.md), [nfc-pushtx.md](https://github.com/Coldcard/firmware/blob/master/docs/nfc-pushtx.md) |
| SatochipApplet, Satodime-Applet, Satocash-Applet | JavaCard | AGPL applets you can load onto blank cards | [Satochip](https://github.com/Toporin/SatochipApplet), [Satodime](https://github.com/Toporin/Satodime-Applet), [Satocash](https://github.com/Toporin/Satocash-Applet) |
| status-keycard | JavaCard | Keycard applet | [GitHub](https://github.com/keycard-tech/status-keycard) |
| cashu-javacard, cashu-client | JavaCard | Flash's offline ecash card applet and client | [applet](https://github.com/lnflash/cashu-javacard), [client](https://github.com/lnflash/cashu-client) |
| μNuts | n/a | Minimal ecash-over-NFC experiments | [GitHub](https://github.com/Amperstrand/micronuts) |
| LightningNFC | n/a | The "write a Lightning Address to a €0.30 tag" recipe | [GitHub](https://github.com/theDavidCoen/LightningNFC) |
| Tangem SDK | Kotlin / Swift | Talk to Tangem cards | [Android](https://github.com/tangem/tangem-sdk-android) |

## Documents

Long-form material in this repository:

- [docs/protocol-landscape.md](docs/protocol-landscape.md): the layer
  map, every protocol with its spec walked through (including the Bolt
  Card tap step by step and the key roles when programming a card),
  what a tap is by direction, and why Ark / Spark have nothing yet and
  what the options are.
- [docs/platform-capabilities.md](docs/platform-capabilities.md): what
  Android, iOS, the web and the desktop each allow. Reader mode, NDEF
  write, raw APDUs, NFC-V, Host Card Emulation and Apple's entitlement
  rules.

External references to read in full:

- [Bolt Card SPEC.md](https://github.com/boltcard/boltcard/blob/main/docs/SPEC.md)
  and [SYSTEM.md](https://github.com/boltcard/boltcard/blob/main/docs/SYSTEM.md).
- [NXP AN12196](https://www.nxp.com/docs/en/application-note/AN12196.pdf),
  the NTAG 424 DNA application note that defines SUN/SDM.
- [LNURL LUDs](https://github.com/lnurl/luds). Numbers 01, 03, 06, 08,
  14, 16, 17 and 19 are the ones a tap touches.
- [Coldcard NFC docs](https://coldcard.com/docs/nfc-tools/).
- [Coinkite Tap Protocol](https://dev.coinkite.cards/docs/protocol.html)
  and [NFC spec](https://dev.coinkite.cards/docs/nfc-spec.html).
- [Cashu NUT-18](https://github.com/cashubtc/nuts/blob/main/18.md).
- [NWC over NFC](https://github.com/agustinkassis/nwc-nfc), the draft for
  phone-to-phone Lightning taps over Host Card Emulation and NIP-47.
- [Coincharge: Bolt Card](https://coincharge.io/en/bolt-card-the-lightning-card/)
  and [Numo tap-to-pay](https://coincharge.io/en/numo-numopay-accept-bitcoin-tap-to-pay-via-nfc-in-stores/)
  are the most complete aggregator write-ups.
- [Bolt Card field-test notes](https://ereignishorizont.xyz/en/boltcard_en/)
  on which wallets actually read cards reliably.
- [Apple: HCE-based transactions in apps](https://developer.apple.com/support/hce-transactions-in-apps),
  [Core NFC](https://developer.apple.com/documentation/corenfc),
  [Android NFC](https://developer.android.com/develop/connectivity/nfc),
  [Web NFC](https://developer.mozilla.org/en-US/docs/Web/API/Web_NFC_API).

## Out of scope

Fiat card products that happen to hold bitcoin balances are excluded:
the various "crypto Visa / Mastercard" debit cards, Tangem Pay, and the
Fold, Moon or Spendl style of rewards card. The tap is a card-network
transaction and nothing bitcoin-specific happens at the terminal, so
there is nothing for a bitcoin developer to support or test.

Also excluded: NFC hardware in a wallet or signer that has no documented
bitcoin-facing use (it is listed once, as ⚠️, so nobody buys it for
testing by mistake), and generic NFC tooling with no bitcoin angle
beyond the few programming apps above.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Corrections from first-hand
testing are the most valuable kind of PR: most rows here rest on vendor
documentation and aggregator sites, not on a tap.

## License

[CC0 1.0](LICENSE), public domain.
