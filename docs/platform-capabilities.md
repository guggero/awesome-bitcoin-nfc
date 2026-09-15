# Platform capabilities and hard limits

What each platform lets an app do with NFC decides which of the flows
in this list are buildable at all. The asymmetries are large enough to
determine product design, so they are collected here in one place.

*Last reviewed 2026-09.*

## The two questions that matter

1. **Who reads and who is read?** Reading a tag or card works on every
   platform. *Being* a tag or card (Host Card Emulation, HCE) is, in
   practice, Android-only.
2. **NDEF or raw?** Anything that fits in NDEF records (URIs, Coldcard's
   PSBT records, an emulated Type 4 tag) is reachable from every
   platform's high-level API. Smartcard applets (cktap, Bitkey, Satochip,
   Keycard, Portal, Tangem) need raw ISO 7816 APDUs, and iOS gates those
   behind per-app entitlements.

## Android

Reference: [NFC overview](https://developer.android.com/develop/connectivity/nfc),
[HCE](https://developer.android.com/develop/connectivity/nfc/hce).

- **Reader mode** (`NfcAdapter.enableReaderMode`) with NDEF read *and*
  write.
- **Raw ISO-DEP / APDU** via `IsoDep`, so every smartcard signer is
  reachable.
- **NFC-V** (`NfcV`) for ISO 15693 tags, which is what a Coldcard Mk4
  or Q presents.
- **Host Card Emulation**: full card emulation from an app, no secure
  element needed. This is what VirtualBoltcardApp (phone as Bolt Card),
  Numo (merchant phone as payment-request tag) and Minibits (token or
  invoice host) rely on.
- Android Beam (phone-to-phone NDEF push) was deprecated in Android 10
  and later removed; the BIP-70-over-Beam flows of the Bitcoin Wallet
  era no longer work.

Everything in this repository is possible on Android.

## iOS

Reference: [Core NFC](https://developer.apple.com/documentation/corenfc).

- `NFCNDEFReaderSession` reads **and writes** NDEF. Reading a Bolt Card
  (NTAG 424 DNA) or a static `bitcoin:` / `lightning:` tag works.
- `NFCTagReaderSession` (iOS 13+) covers ISO 7816, ISO 15693, MIFARE and
  FeliCa, so tapcards, Satochip, Keycard, Portal and Coldcard (NFC-V)
  are all reachable.
- **ISO 7816 needs every AID declared up front** in
  [`com.apple.developer.nfc.readersession.iso7816.select-identifiers`](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.nfc.readersession.iso7816.select-identifiers).
  The session SELECTs them in order; payment-industry AIDs are blocked.
  Supporting a new smartcard signer is therefore an Info.plist change
  plus an App Store submission, not a runtime decision.
- Sessions are foreground, user-initiated, and show a system sheet for
  roughly 60 seconds. Background tag reading exists on recent iPhones
  but only opens a URL from an NDEF record (which is enough for
  NFC-PushTx and for Coinkite's dynamic card URLs).
- **HCE is unavailable for most apps.** `CardSession` requires iOS 17.4+,
  an Apple-granted HCE entitlement, an EEA-based Apple ID *and* the
  device physically located in the EEA
  ([Apple](https://developer.apple.com/support/hce-transactions-in-apps)).
  Any flow that needs the *customer's* phone to emulate a card excludes
  most iPhone users. Minibits states this plainly: its HCE features are
  "Android-only, as it relies on Host Card Emulation (not available on
  iOS)".

## Web

Reference: [Web NFC](https://developer.mozilla.org/en-US/docs/Web/API/Web_NFC_API),
[explainer](https://github.com/w3c-cg/web-nfc/blob/gh-pages/EXPLAINER.md).

NDEF read and write from a page in a secure context, **Chrome on
Android only**, with a visible page and a user gesture. Enough for a PWA
point of sale (BTCPay Server's NFC plugin, LNbits TPoS) or a browser
tag writer (cashu.me writing a token onto a card); nothing on iOS, no
raw APDUs, no emulation.

## Desktop

No mobile-style API. Use a PC/SC reader (ACR122U, ACR1252U and similar)
with the platform's smartcard stack or `libnfc`; this is how the Python
tooling in the README (boltlib, coinkite-tap-proto, sdm-backend)
programs and verifies cards, and how Electrum and Sparrow reach a
Satochip.

## What a tap is, by direction

| Direction | Who emulates | Examples | Works with an iPhone? |
| --- | --- | --- | --- |
| **Card → POS** (pull) | Passive chip | Bolt Card (LNURL-withdraw + SUN), static LNURL-withdraw tag | Yes; the POS reads, and iOS can read NTAG 424 |
| **Phone → POS** | Customer phone (HCE) | VirtualBoltcardApp (pull); NWC over NFC (draft) | Android only |
| **POS → phone** (push) | Merchant device (HCE) or a static tag | Cashu NUT-18 via Numo / Minibits; a `lightning:` or `bitcoin:` tag; presumably Square Register | Yes; the phone only reads |
| **Signer ↔ phone** | Card or device applet, or a tag | cktap, Bitkey WCA, Coldcard NDEF, Portal, Satochip | Yes, with the AID entitlement for APDU-based devices |

Consequences:

- A merchant app that only *reads* cards is portable to iOS, but
  merchants buy cheap Android terminals anyway, which is why the POS
  app table in the README is Android-heavy.
- An ecash-style "merchant emulates, customer reads" design is the only
  phone-to-phone tap that works for iPhone customers without a hosted
  card service.
- A "phone emulates a Bolt Card" design excludes iPhone customers for
  the foreseeable future.
