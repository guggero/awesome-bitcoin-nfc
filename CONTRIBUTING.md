# Contributing

Pull requests are welcome. Please keep the following in mind so the list
stays useful to the people it is written for: developers deciding what
to implement and what to buy in order to test it.

## Scope

- **Bitcoin only.** On-chain, Lightning, ecash, Ark, Spark and the NFC
  plumbing underneath them. No fiat card-network products, even if they
  hold a bitcoin balance and even if they are marketed to bitcoiners.
- **NFC has to do something.** A wallet that merely runs on a phone
  with an NFC chip does not belong here. A wallet that reads a Bolt
  Card, writes a tag, emulates a tag, or talks to a signer over NFC
  does.
- Discontinued products stay in the list, marked as such, as long as
  the hardware or the protocol is still out there to be encountered.

## Entries

- One entry per row. Put the canonical product page as the row's main
  link and the source repository in the **Source** column, or the
  words `Closed` or `n/a` if there is none or it is unknown.
- Fill every capability column with one of `✅`, `⚠️`, `❌` or `n/a` as
  defined in the README legend. Use `✅` only for behaviour the vendor
  documents or the source code shows; anything resting on a review, a
  forum post or a marketing claim is `⚠️` with a link to where the
  claim comes from.
- Say which side does what. "Supports NFC" is not enough. Reading a
  card, writing a tag, emulating a tag, and exchanging APDUs with a signer
  are four different things with different platform constraints.
- Name the payload format (Bolt Card, BIP-21 URI, Cashu token, cktap,
  Coldcard NDEF, …) so the row can be cross-referenced from the
  Formats section.
- Keep notes short and factual. No superlatives, no pricing unless it
  is stable and matters (for example "cents each" for blank tags).
- If a vendor's own documentation contradicts a third-party claim, the
  vendor wins and the row says so.

## Removing or correcting

If you know from first-hand testing that a row is wrong, open a PR that
fixes it and say in the description what you tested with (device,
firmware or app version, platform). First-hand evidence beats the
secondary sources most rows are built on.

## Style

- Markdown tables, one line per row, no HTML.
- Wrap prose at roughly 72 columns; table rows are exempt.
- Links point at primary sources where they exist: the spec, the
  repository, the vendor page. Coverage articles are fine as a second
  link when they contain something the primary source does not.
- Long-form explanations go into `docs/`, not into table cells.
