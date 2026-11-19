# Rules for Claude — TCF IRN B2 Trainer

**Read this file first in every new chat, then read `PROJECT_NOTES.md`** (project facts, changelog, roadmap).

## What this project is
A single-page web app (`index.html`) that trains learners for the **TCF IRN writing test (expression écrite), target level B2**.
It has 3 interface languages (EN / FR / RU), an offline database, an optional Gemini AI layer, and Supabase cloud sync.
The owner is in **beta with family**; the long-term goal is a **commercial product** (store / subscription).

## Working rules
1. **Before any change**: read `PROJECT_NOTES.md` → "Current state", "Known issues", "Roadmap".
2. **Versioning (mandatory)**
   - `index.html` in the root = the current live version (GitHub Pages serves it).
   - Before shipping a change, bump the version comment at the top of `index.html` (`<!-- APP VERSION: vX.Y (date) -->`).
   - After the change works, copy it to `versions/vX.Y_YYYY-MM-DD.html`. Never edit or delete old files in `versions/`.
   - Minor fixes = +0.1 (v1.1 → v1.2). Big rework (e.g. splitting into modules, login) = new major (v2.0).
3. **Log every session** in `PROJECT_NOTES.md` → "Changelog" (date, version, what changed, why), and update "Known issues" / "Roadmap".
4. **Keep it one file for the beta** (owner's decision, 2026-11-19). Split into modules only when the owner says the beta is polished (planned for v2.0).
5. **Three languages always**: every new visible text goes into `I18N.en`, `I18N.fr`, `I18N.ru` and is read with `t('key')` or `data-i18n="key"`. Never add EN-only UI text.
6. **Official exam rules are the reference** (France Éducation international, checked 2026-11-19):
   - Writing = 3 tasks in **30 minutes total**.
   - Tâche 1: **30–60 words**, short message describing a person/place/object in reply to a message.
   - Tâche 2: **40–90 words**, account of daily activities / an experience.
   - Tâche 3: **40–90 words**, opinion about a place, object or person.
   - Too few/too many words, off-topic, or a missing task ⇒ "A1 non atteint".
   Re-check https://www.france-education-international.fr/test/tcf-irn if something looks outdated.
7. **French content must be correct.** Any new word/phrase/model text: check accents, gender, agreements, and that model answers fit the word range (count with whitespace split, like the app's `countFrenchWords`).
8. **Offline first**: AI is optional. Every feature must still work without a Gemini key (fallback to the built-in bank). AI results are cached in `localStorage` (`tcf_irn_ai_bank_v1`) so the offline bank grows.
9. **Never touch real user data while testing.** Test with a copy where `DEFAULT_SUPABASE_URL`/`DEFAULT_SUPABASE_ANON_KEY` are blanked (served from the scratchpad), not the live file.
10. **GitHub**: repo `https://github.com/only4viz/tcf-irn-studio` (branch `main`). On this PC git needs `-c http.sslBackend=schannel` (or the global setting) for HTTPS. **Always ask the owner before pushing.**
11. **Secrets**: never put a Gemini key in the code or in URLs that get committed. The Supabase anon key is public by design, but see "Security" in the notes before going commercial.
12. Talk to the owner in simple English; they may ask in EN/FR/RU.
