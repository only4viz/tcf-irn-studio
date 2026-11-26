# TCF IRN B2 Trainer — Project Notes

> Living document. Update it at the end of every work session (see `CLAUDE.md`).

## 1. Product summary
- **Goal**: help adults pass the **TCF IRN** (Intégration, Résidence, Nationalité) writing test at **level B2** (B2 is required for French naturalisation).
- **Users now**: owner + close relatives (beta). **Later**: commercial app (store / subscription).
- **Interface languages**: English, French, Russian.
- **Live version**: `index.html` (root) → GitHub repo `only4viz/tcf-irn-studio`, branch `main` → GitHub Pages: https://only4viz.github.io/tcf-irn-studio/ (updates ~1 min after a push).
- Commits are signed `only4viz <357567804+only4viz@users.noreply.github.com>` (GitHub private noreply address, keeps the personal email out of the public repo).

### Modes
1. **Words drill** – spelling, accents (é è ê à ç…), gender (le/la). Score /20, mistakes saved.
2. **Phrases & skeletons** – reusable B2 sentences per task + grammar traps.
3. **Exam simulator** – official tasks, word counter, timer, local checker + optional Gemini examiner. **Full exam mode**: 3 random subjects, one 30-min clock.
4. **Find the errors** (v1.6) – a TCF-style text with 6 planted mistakes; tap a word, type the fix, check. Score /20, missed errors saved as rule notes.
5. **Mistake bank & progress** – per-profile mistakes, streak (2 correct in a row = mastered), history, XP.
6. **B2 grammar memo** – cheat sheet.

### Tech
- One HTML file, Tailwind (CDN), Supabase JS (CDN), Gemini REST API (browser `fetch`).
- Storage keys (localStorage): `tcf_irn_studio_state_v1` (profiles), `tcf_irn_cloud_config_v1` (keys), `tcf_irn_ai_bank_v1` (AI-generated words/phrases/themes/errorTexts cached offline), `tcf_irn_recent_items_v1` (anti-repetition), `tcf_irn_exam_drafts_v1` (essay drafts), `tcf_irn_exam_results_v1` (last graded results, kept until "Reset results"), `tcf_irn_lang_v1` (UI language).
- AI calls: 45 s limit per attempt (60 s for essay grading), 2 min max per call, "thinking" limited (`thinkingLevel: low` on Gemini 3.x, `thinkingBudget: 1024` on 2.5; retried without it if a model refuses).
- Supabase table `tcf_irn_users (id text pk, name, data jsonb, updated_at)`, realtime enabled.
- Gemini models: **auto-detected** from Google's model list for the user's key (newest stable `flash` first, then `flash-lite`, previews last), cached 24 h in `tcf_irn_gemini_models_v1`; retired models (404) are dropped automatically. Fallback list if detection fails: `gemini-3.5-flash`, `gemini-3.5-flash-lite`, `gemini-flash-latest`, `gemini-flash-lite-latest`, `gemini-2.5-flash`. Key sent **only** in the `x-goog-api-key` header.

## 2. Official exam reference (France Éducation international, checked 2026-11-19)
| Task | Words | What to write |
|---|---|---|
| Tâche 1 | 30–60 | Short message replying to someone: describe a person / group / place / object |
| Tâche 2 | 40–90 | Account of daily activities or an experience (récit / compte rendu) |
| Tâche 3 | 40–90 | Give and justify an opinion about a place / object / person |

- 30 minutes for all 3 tasks. Whole TCF IRN = 1 h 35.
- "A1 non atteint" if: word count not respected, off-topic, a task missing, illegible.
- Re-corrections of writing/speaking suspended from 1 Sept 2026 (back autumn 2027).

## 3. Current state (v1.8, 2026-11-26)
- Offline bank: **112 words (16–24 per category), 50 phrases (10–14 per category), 21 exam subjects (7 per task), 21 error-hunt texts (6 mistakes each)** (+ everything Gemini generates, cached per device).
- Exam ranges now follow the official table above (v1.0 had Tâche 3 at 90–110 words = would be "A1 non atteint").

## 4. Changelog
### v1.8 — 2026-11-26
- Owner hit "HTTP 429 … Generate content requests per day limit exceeded on Free Tier (limit 20)". Free Gemini keys allow ~20 requests **per model per day**; a full exam used 3, each retry / AI text 1 more.
- **Full exam = 1 AI request** for all written tasks (`gradeEssaysTogether`, JSON keyed by tache1/2/3, 90 s per attempt, 3 min max). A task missing from the answer gets the yellow box + Retry (single-task request). Single "Grade" and Retry still use `gradeEssayText` (1 request). Prompt pieces shared (`AI_TASK_SPEC`, `aiReportJsonShape`, `localEssayReport`, `applyAiGrade`).
- `callGeminiJson`: a **daily** 429 ("per day" / "retry in NNh") no longer waits; goes straight to the next model and shows a translated message (`aiDailyLimit`: resets ~9:00 France time). New key `aiIncomplete`. The yellow box shows up to 360 characters.
- Note for owner: a Google AI Pro (Gemini app) subscription does NOT raise API limits — the API is billed separately (Google AI Studio → enable billing on the key's project).
- Tested on a blanked copy with mocked Gemini: 1 request for 3 tasks; partial answer → task 3 offline + Retry → AI; daily 429 in RU → 2 quick calls (one per model), RU message, no waiting.

### v1.7 — 2026-11-26
- Bug report (owner, full exam in RU): "no grammar errors found" although there were some. Cause: not RU and not the new mode — when the AI grading failed (most likely the free per-minute quota, 429, with 3 texts at once), the app silently fell back to the offline checker, which only knows a few traps and then said "🎉 No major errors". The AI error was only a short toast.
- Fixes:
  - The report now shows a clear ⚠️ box when AI grading failed (with the reason) and a **🔁 Retry with AI** button (re-grades that task from the saved text).
  - Offline checker no longer says "No errors 🎉": it says it found none of the few traps it knows, which does NOT mean the text is correct.
  - 429 handling: wait the delay Google asks for (max 15 s) and retry the same model once before trying the next one.
  - Full exam: the 3 AI calls start 1.5 s apart instead of in the same second.
- Tested in a Supabase-blanked copy with mocked 429 replies (RU): banner + retry button shown; retry with a working reply replaces the offline report.

### v1.6 — 2026-11-25
- New tab **🔍 4. Find the Errors** (Mistake bank is now tab 5). Owner idea: a correction exercise.
  - Offline: the 21 B2 model answers with 6 planted learner mistakes each (`ERROR_HUNT_BANK`: correct form, wrong form, type, short French rule). Types: accent, gender, agreement, être/avoir, verb form, preposition, spelling, subjunctive, word form (au/du/nouvel).
  - Tap a word → edit box (accent keyboard works) → Enter. "Check": fixed = 1 pt, spotted but wrong fix = ½, changing a correct word = −½; score /20, +20 XP, history line "🔍 Error hunt". Missed mistakes → Mistake bank as rule notes (not used in spelling drills). "Show solution" gives no score.
  - ✨ AI text: Gemini writes a new text on a random subject of the chosen task with 6 mistakes; the app checks every "correct form" really exists in the text (bad ones dropped, needs ≥3) and saves it (max 40) in the AI bank → grows the offline set.
  - EN/FR/RU for all new texts. All 126 planted mistakes checked by script (each correct form found in its model text).
- Tested in a Supabase-blanked copy: scoring (3 fixed + 1 spotted + 1 false alarm = 10/20), mistakes saved, AI path with a mocked reply (invalid item dropped), RU/FR labels, phone width (no horizontal scroll). Not yet tested with the real Gemini key.

### v1.5 — 2026-11-24
- "🔄 Reset results" now also clears the texts that were graded (all 3 after a full exam, or the graded task), so the next attempt starts with an empty box. It asks for confirmation first (EN/FR/RU). Owner request: old text stayed in the box after reset.
- Tested in a Supabase-blanked copy: cancel keeps text + results; confirm empties textarea, drafts, results, word counter.

### v1.4 — 2026-11-24
- 3 new exam subjects requested by the owner (FR prompt + EN/RU translations, required points, B2 model answer counted inside the official range):
  - T1 `t1_nouvelle_boulangerie` — reply to a friend: describe the new bakery nearby + your opinion (model 57/60 words).
  - T2 `t2_soiree_parc` — tell a friend about an evening with friends in the park (model 76/90).
  - T3 `t3_seul_ou_equipe` — opinion: working alone or in a team, pros and cons (model 71/90).
- Tested in a Supabase-blanked copy: subjects listed per task, prompt + model shown, RU titles OK, no console errors.

### v1.3 — 2026-11-21
Owner feedback after testing v1.2 (5 notes + "grading sometimes takes 20+ min").
- **Slow grading fixed**: v1.2 had no time limit on Google requests (a busy server could hold one for minutes), tried up to 4 models one after another, graded the 3 texts one after another, and let "thinking" models think without limit. Now: time limit per attempt + per call, the 3 texts are graded **in parallel**, short thinking, live label "Analyzing… 1/3 (model)". If the AI is too slow, the offline result is shown with a clear message.
- **Results stay on screen**: after grading (full exam or one task), clicking T1/T2/T3 shows each text with its own correction; the graded subject stays selected; results survive a page refresh. A **🔄 Reset results** button clears them. Changing the subject shows "graded on another subject" instead of deleting the result.
- **Topic filters really filter**: v1.2 mixed ~1/3 of *all* active mistakes into every batch (even whole phrases in the Words drill, whatever the category). Now mistakes remember their kind (word/phrase/essay) and category; only matching ones come back. Phrase mistakes now come back in the Phrases drill (same category). Old mistakes are classified automatically. AI batches made in "All" are filed in the right category (v1.2 put all phrases under Tâche 3).
- **Synonyms accepted**: with a Gemini key, answers that don't match the expected text are checked by the AI once per batch: "✅ correct too" (20/20), "👍 works, but better: X" (17/20, not saved as a mistake) or explained errors + your sentence corrected (phrases). 25 s limit; offline the app explains that a different wording may still be right. AI prompts now ask for synonyms in `alt`.
- **Bigger offline bank**: +72 words, +35 phrases, +6 subjects (all model answers counted inside the official ranges). Two duplicate words replaced (*à cause de*, *par conséquent*).
- Rules display *italic* / `code` formatting instead of raw stars; user text escaped in feedback.
- Tested with simulated Google responses (timeout → next model, parallel grading, synonym verdicts, reset, reload, RU, offline). **Not yet tested with the owner's real key.**

### v1.2 — 2026-11-19
- Owner's Test showed: `gemini-2.5-flash-lite: HTTP 404 — no longer available to new users… use gemini-3.5-flash-lite`. The hard-coded model names were retired for new keys.
- Fix: the app now asks Google which models the key can use (ListModels) and picks the best `flash` model automatically; retired models are skipped and forgotten; cache 24 h; Test button refreshes the list and shows which model answered.
- Newer "thinking" models: thought parts are ignored, only the answer text is parsed.
- When all models fail, the error lists every model tried (e.g. 429 quota vs 404 retired).
- Tested with simulated Google responses, then **confirmed by the owner with the real `AQ.` key: "✅ OK — gemini-3.6-flash"** (2026-11-19).

### v1.1 — 2026-11-19
**AI (the "AI Generate gives the same words" problem)**
- Root cause: the key was sent twice (`?key=` + header). New `AQ.` keys from Google AI Studio reject that, and every error was silently replaced by the local bank.
- Now: key sent once; real Google error shown in a toast; **🔌 Test** button in ☁️ Settings; model fallback list updated.
- More variety: random theme + grammar focus per batch, "do not repeat" list (recent + known items), higher temperature, learner's weak points injected.
- Every AI batch (words, phrases, subjects) is **saved to the offline bank** on the device.
- AI examiner prompt rewritten: official task rules, 4 criteria (/5 each), "to reach B2" tips, exact quotes, corrected version within the word range.

**Bug fixes**
- "Show solutions" used to grade the filled answers as 20/20 (free XP, mistakes marked "mastered"). Now: shows answers, 0 points, counts as mistakes.
- Pressing Enter / Check several times added duplicate history + XP. Now graded once per batch; Enter moves to the next field.
- Supabase: every save pushed **all** profiles → a device with an old copy could overwrite another family member's progress. Now only the active profile is pushed.
- Offline checker: regexes failed after accented letters (`\b` after é) → "j'ai allé" wasn't detected. Fixed; more accent words; essay accent errors become real spelling drills.
- Essay generic rule corrections are stored as "📝 Rule note" (not as spelling drills).
- Mistake meanings stored in all 3 languages; history capped at 200 per profile.
- Language choice remembered; typed answers kept when switching language.
- AI text rendered with HTML escaping.

**Exam simulator**
- Official word ranges (T1 30–60, T2/T3 40–90) + optional "60–90 B2 zone".
- **Full exam mode** (3 random subjects, 30:00, switch tasks freely, grade all 3 with average).
- Drafts autosaved per task (survive refresh); clear asks for confirmation.
- 10-min timer for single-task practice; model answer hidden when subject changes.
- Wrong word count ⇒ "A1 non atteint" warning (local + AI).
- 6 new subjects matching the official task types; cheat sheet updated.

**UI**
- Header not sticky on phones; accent bar folded by default on phones; engine pill hidden on phones.
- New UI texts translated in EN/FR/RU.

**Repo**
- `versions/` folder created (v1.0 original + v1.1). `CLAUDE.md` (rules) + this file.

### v1.0 — 2026-11-18
- Original beta uploaded to GitHub ("Add files via upload").

## 5. Known issues / to check
- Gemini `AQ.` keys: some Google accounts report 401 `ACCESS_TOKEN_TYPE_UNSUPPORTED` even with correct usage. If the **Test** button shows that, create a key in another Google Cloud project or ask Google (forum threads, 2026). Not fixable on our side.
- Settings modal, some toasts and the cheat sheet are still partly English-only.
- Supabase URL/key typed in Settings are ignored while `DEFAULT_SUPABASE_*` constants are set in the code (constants win).
- Deleting a profile locally does not delete it in Supabase → it comes back on next load.
- Realtime update replaces a profile without comparing `updated_at` (last write wins).
- Phrases "traps" category still small (2 items offline).
- Offline essay checker is rule-based (a few traps only); scores without AI are indicative.
- Offline Words/Phrases check can't recognise synonyms that are not in `alt` (AI check needed). Some phrases are given in one gender only (alt lists the other where it matters).
- Free Gemini quota: ~20 requests per model per day (and a per-minute limit). Since v1.8 a full exam uses 1 request; heavy testing can still use up the free day. Paid tier (billing on) removes this in practice.

## 6. Security (MUST fix before going commercial — owner chose "later", 2026-11-19)
- Supabase policy `for all using (true)` + anon key inside `index.html` ⇒ anyone can read/modify/delete **all** users' data.
  → Use Supabase Auth (email/magic link), `user_id uuid` column, RLS `auth.uid() = user_id`.
- "Copy phone link" puts the Gemini key in the URL (browser history, chat apps). → Remove; log in instead.
- Gemini key lives in the browser. For a paid product → call AI through a small backend (Supabase Edge Function) with our key + per-user quota; users never see a key.

## 7. Roadmap
### Short term (beta)
- [x] Owner tests the AI with the **Test** button → works with `gemini-3.6-flash` (2026-11-19).
- [ ] Grow the offline bank (target: 300 words, 120 phrases, 30 subjects per task) — can be done with AI then reviewed.
- [ ] Translate remaining EN-only UI (settings modal, cheat sheet).
- [ ] Real spaced repetition (review dates: 1 d / 3 d / 7 d / 21 d) instead of "2 correct in a row".
- [ ] Progress chart (score per task over time) and "weak points" summary.
- [x] "Correct the text" exercise → v1.6 Find the Errors.
- [ ] Find the Errors: more offline texts (a 2nd set of mistakes per subject), optional "hint" (show the line with an error).

### Ideas discussed (2026-11-21)
- **Local/offline LLM instead of Gemini?** Not for now: a model good enough to grade French at B2 is several GB, slow on phones, and weaker than Gemini at catching accent/agreement errors. Training one ourselves is far beyond the budget. Better path: keep the offline bank (rules + data) for drills, use an online AI for grading; for the commercial version, call it through our own backend (key hidden, quotas).
- **Expression orale (speaking)**: needs an online model with speech (e.g. Gemini Live / audio input) — good premium feature for v2.x. Prototype: record answer → transcript + examiner feedback.

### Commercial plan discussed (2026-11-25) — keep for later, after polishing
- Rough running costs (estimates, re-check prices before launch): GitHub Pages free; Supabase free → ~$25/month (Pro) with hundreds of active users; domain ~$10–15/year; Apple $99/year + Google $25 once, stores take 15–30%; Stripe ~3%/sale.
- AI: text grading with a Flash model ≈ $0.002–0.005 per essay (< $0.50/month for a heavy user). Speaking: recorded answer + grading = a few cents; live voice conversation ≈ $0.05–0.15/min → premium only.
- Price idea: €5–10/month or a €15–30 "exam pass".
- Marketing: small/medium YouTubers (5k–100k) teaching French / naturalisation / TCF prep: free premium + viewer discount code with commission (affiliate); also Facebook/Reddit/Telegram expat groups. Only after the app is polished, with a landing page.
- Never present the app as official (TCF = France Éducation international trademark): "TCF IRN prep".

### v2.0 (when beta is polished — commercial prep)
- [ ] Split into modules (data/, js/, css/, i18n/) with a build step (Vite). Tailwind compiled (no CDN).
- [ ] Supabase Auth + secure RLS; migrate existing profiles.
- [ ] AI behind a backend (Edge Function), usage limits per plan; optional other free providers as fallback.
- [ ] PWA (installable, offline cache) → later wrap for stores (Capacitor) if needed.
- [ ] Payments (Stripe / store billing), free tier vs premium (AI grading, unlimited subjects).
- [ ] Legal: privacy policy (RGPD), terms, no use of FEI trademarks/logos in a misleading way ("unofficial practice tool").
- [ ] Analytics + error reporting; automated tests for grading functions.
