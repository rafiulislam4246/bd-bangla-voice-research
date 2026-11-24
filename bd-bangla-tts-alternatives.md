# Bangladeshi Bangla TTS Alternatives (Nov 2026)

Research date: 2026-11-23
Baseline: `actorcritic/IndicF5_Bangladeshi`

## Verdict

**Nothing clearly beats IndicF5_Bangladeshi on all 4 axes** (BD accent + human-sounding + <350ms + commercial license).

## Worth testing (2)

### 1. Orpheus Bangla Emotional
- **URL**: https://huggingface.co/ehzawad/orpheus-bangla-emotional-tts
- **Base**: 3B Llama-based Orpheus
- **Highlights**: happy/sad/frustrated style tags, real BD speaker data
- **Pros**: more expressive than IndicF5, actual emotion control
- **Cons**: 3B model = slower. Streaming + good GPU ~200ms. CPU unusable.
- **License**: base Apache-2.0, fine-tune card needs verify

### 2. Chatterbox Bangla
- **URL**: https://huggingface.co/Banglabox/chatterbox-bangla-tts
- **Base**: Chatterbox T3 + Bangla LoRA + tokenizer extension
- **Pros**: zero-shot voice clone like IndicF5, base RTF ~0.5 (fast)
- **Cons**: unknown if BD-specific or pan-Bengali, author anonymous, preview quality

## Skip (checked, not good enough)

| Model | Reason |
|---|---|
| `EMTIAZZ/bangladeshi-bangla-tts-vits` | Author himself recommends Orpheus instead |
| `ehzawad/indicf5-bangla-tts` | Same family, worse license (Research-Only) |
| `kawshikbuet17/bengali-cosyvoice3-tts` | Likely bn-IN data |
| `facebook/mms-tts-ben` | CC-BY-NC, robotic, Indian pronunciation |
| `dheeyantra/dhee-indic-f5` | Non-commercial |
| `Saunak359/IndicF5-Multilingual` | Non-commercial, no BD specificity |
| BnTTS (NAACL 2025, XTTS-based) | Paper only, no checkpoint |
| BnVITS (DU Jan 2026) | No release |
| BUET / Giga Tech / REVE | No 2026 TTS release found |
| Piper bn-BD | Nothing credible |
| Bark Bangla | Nothing credible |
| XTTS v2 BD fine-tune | Nothing credible |

## Recommended strategy

Keep IndicF5_Bangladeshi as production default. Add Orpheus Bangla as **emotion-triggered secondary**:
- Normal lines → IndicF5 (350ms)
- Emotional/empathy lines → Orpheus (accept ~200-400ms for expression)
- Route via sentiment analysis on text

## Why this space is thin

Bangladeshi Bangla TTS hasn't attracted enough investment. Most "Bengali TTS" releases are West Bengal / Kolkata accent. The 2 real BD-specific options are both recent (Orpheus, Chatterbox) and both from single contributors. No major lab / BUET / Giga / REVE has released a 2026 BD TTS model.

## Sources

- [ehzawad/orpheus-bangla-emotional-tts](https://huggingface.co/ehzawad/orpheus-bangla-emotional-tts)
- [Banglabox/chatterbox-bangla-tts](https://huggingface.co/Banglabox/chatterbox-bangla-tts)
- [ehzawad Bangla TTS collection](https://huggingface.co/collections/ehzawad/bangla-text-to-speech-tts)
- [BnTTS paper NAACL 2025](https://arxiv.org/html/2502.05729v1)
- [DU Bangla TTS benchmark Jan 2026](https://www.banglajol.info/index.php/DUJS/article/view/84122)
