# Bundled font provenance

All bundled faces are licensed under the SIL Open Font License 1.1
(`LICENSE-OFL.txt`).

## Noto Sans CJK (Simplified / Traditional), Regular

- Files: `noto-sans-cjk-sc-regular.otf`, `noto-sans-cjk-tc-regular.otf`
- Upstream: `notofonts/noto-cjk` (Noto Sans CJK v2.004 OTF builds)
- Commit: `165c01b46ea533872e002e0785ff17e44f6d97d8`
- SHA-256:
  - `noto-sans-cjk-sc-regular.otf`: `2c76254f6fc379fddfce0a7e84fb5385bb135d3e399294f6eeb6680d0365b74b`
  - `noto-sans-cjk-tc-regular.otf`: `dce08bd4fd91aa8aa76ed8fea4b694c2dfb8550f67871e326843212ddbeb88b4`
- License: SIL Open Font License 1.1, included in `LICENSE-OFL.txt`
- Loaded only with the `cjk` cargo feature as a glyph-level fallback for
  code points the primary face lacks (CJK ideographs, kana, fullwidth
  punctuation). Regular weight only; no synthetic bolding.

## Liberation Sans / Serif / Mono (Regular, Bold, Oblique, Bold Oblique)

- Upstream: Liberation fonts project (`liberationfonts.org`), metric
  compatible with Arial/Times New Courier.
- SHA-256:
  - `liberation-sans.ttf`: `36efda825bcd7d1de846204c3adad51ced9d9b5cb7df571fb33edc21207e324a`
  - `liberation-sans-bold.ttf`: `c2d334cafbd28522007cc16970db8db4a75256123594a43fe07f89cfceb5bacc`
  - `liberation-sans-oblique.ttf`: `f71f02009bd7320571db05b7459c7db92a1f5f68ac619009a2f637e18c7a93f9`
  - `liberation-sans-boldoblique.ttf`: `2e5f6a48571d50f277598c533dea4f659abce53335fb475aa2bc45bd2527f1d0`
  - `liberation-serif.ttf`: `8d6951ea5fc4a9656df4802227292c7943186364929f00b97629e70daf228439`
  - `liberation-serif-bold.ttf`: `cfbe3d2edd46ae844b7eb95012f9274776801873e27b2230bbf751645f011888`
  - `liberation-serif-oblique.ttf`: `2d826fdde37518f7393052764703fc8ec120ef1e19ba49b50fcc6b6186b8aa78`
  - `liberation-serif-boldoblique.ttf`: `23d013c7c9708b09be7517ddc3bff8322c5dfa16c629a68f6be2a860e2693698`
  - `liberation-mono.ttf`: `babd36837e644f08e6ba09b69feb770fbeba9e4611a619e5471fb8dbc132d42c`
  - `liberation-mono-bold.ttf`: `410af69b6297b7cf6d20dd004004bd1bdb46460a60f9fef69b04169f4ff09660`
  - `liberation-mono-oblique.ttf`: `1b09e2a1d70adf339048943b21037462c56108411817bf6988c86cb88289fc34`
  - `liberation-mono-boldoblique.ttf`: `a2ab7fe6f5bd7c2bc59ca7f5f4212f165dc174cbbab366cab171df4e73939aa2`
- License: SIL Open Font License 1.1, included in `LICENSE-OFL.txt`

## DejaVu Sans (Regular, Bold)

- Upstream: DejaVu fonts project (`dejavu-fonts.org`).
- SHA-256:
  - `dejavu-sans.ttf`: `690243adfefe0ce154b547db6205794bd30ac4277275179517a90994f4980648`
  - `dejavu-sans-bold.ttf`: `5c1247acef7f2b8522a31742c76d6adcb5569bacc0be7ceaa4dc39dd252ce895`
- License: SIL Open Font License 1.1, included in `LICENSE-OFL.txt`

## Noto Color Emoji

- File: `noto-color-emoji.ttf`
- Upstream: `googlefonts/noto-emoji`
- Version: 2.051 (Unicode 17.0)
- Commit: `8998f5dd683424a73e2314a8c1f1e359c19e8742`
- SHA-256: `72a635cb3d2f3524c51620cdde406b217204e8a6a06c6a096ff8ed4b5fd6e27b`
- License: SIL Open Font License 1.1, included in `LICENSE-NOTO-COLOR-EMOJI.txt` (this file predates `LICENSE-OFL.txt` and is retained as upstream-published)
