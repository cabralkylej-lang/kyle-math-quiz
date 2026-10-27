# Daily Quiz (math + English + brain & feelings) – roadmap & how to extend

Goal: Lillian and Christian get *advanced* at math (and later any topic) through daily repetition + practice.
Improve slowly, as we see the need.

## How it works today (iteration 3)
- **Registry of topics** (`topics.js`): math = `algebra`, `numsense`, `facts`, `fractions`; English = `english` (9 skills); psychology = `psych` ("How my brain & feelings work", 8 skills).
  Each topic has levels (5; algebra 12), per-skill kinds, `generate(level, seed)` and `explain(q)` (the "Great try!" steps). Every question has `topic`, `kind` (skill id) and `skill` (readable name).
  The taxonomy is mirrored in the DB table `skill_catalog` (34 rows). Word/item banks live in `banks/` (`en_vocab`, `en_grammar`, `en_reading`, `psych`) and are inlined into `index.html` by `sh build.sh`.
- **Answer format**: `q.mode` is `tf` (True/False buttons) or `choice2` (two big buttons, `q.choices=[first,second]`). The stored answer is always a boolean:
  `answer=true` means the FIRST choice (or "True"), `truth=true` means the first choice is the correct one. So all old code/data (`correct = answer === truth`) works unchanged.
  `Topics.textOf(q)` gives the plain-text form of a question (statement, choices, passage) for history/reminders. Reading questions carry `q.passage={t,x}`.
- **Adaptive engine** (`engine.js`): per-skill memory (attempts, recent results, last miss, spaced-review step 1/3/7 days, mastery, and per-level tallies `l`) and per-topic working level.
  Daily AND practice answers update the same memory.
- **Daily mix (10 questions, score stays out of 10)**: slots = 4 algebra + 1 other math (numsense/facts/fractions, chosen by weakness) + 3 English + 1 brain&feelings + 1 flex.
  The flex slot is drawn from numsense/facts/fractions/English (English x1.3)/psychology (x0.6). Result: math 5-6, English 3-4, brain & feelings 1-2. Within a topic the skill is picked by
  weakness (recent misses, due for review, not yet mastered). The very first quiz ever is all algebra (nothing to adapt to). Plans are seeded by the date, so a reload gives the same quiz.
  The mix is described in the parent corner (`#mixnote`).
- **Practice** (kid picks the topic from the picker grouped Math / English / How my brain works; rounds of 5 or 10): 2 right in a row = level up, a miss = one level down. Never changes the daily score.
  Stored per question in `practice_questions`; summary views in SQL.
- **Birth month/year**: asked once, at the first sign-in after iteration 3 (required, month + year dropdowns, no day). Stored server-side in `kid_profile` (kid-sync action `set_profile`);
  age is derived (8-13 accepted, else `bad_age`), estimated grade = age on Sept 1 minus 5 (clamped 1-9). The client keeps only `{age, grade}` and uses the grade to choose the starting level of new topics.
  Reports only carry the age band (`kid_profile_summary.age_band`). To correct a mistake: ask Kyle's assistant to run `delete from kid_profile where kid='<name>'` (the kid is asked again at the next sign-in).
  A PIN reset deletes the kid row and cascades to the profile.
- **Parent-only view** (parent corner, behind a soft gate - a short arithmetic question, NOT security; a PIN-based parent mode is a future option): per skill and per topic, an estimate vs grade 4 and grade 5:
  `approaching` / `on track` / `ahead` / `collecting data`. There is deliberately no "behind" label and nothing is ever compared with other children.
  Kid screens show only growth messages (personal bests, improvement vs own past, effort, next steps).
- **Grade estimate mapping (assumption, honest)**: every topic has `gradeEq` = the grade-level equivalent of each level (english [3.5,4,5,5.5,6]; fractions [3.5,4,4.5,5,5.5]; numsense/facts [3,3.5,4,4.5,5];
  algebra [3.5,4,4.5,5,5.5,6,6,6.5,6.5,6.5,7,7]; psychology has none: it is not graded). For grade g the target level T(g) is the lowest level with gradeEq >= g.
  Evidence = answers at levels >= T(g). Fewer than 5 answers -> collecting data; >= 75% right -> on track (ahead if 5+ answers at levels ABOVE T(g) are also >= 75%); otherwise approaching.
  It is an estimate from this app's own questions, not a state test. Massachusetts (MCAS, 2017 ELA/Literacy Framework) and Florida (B.E.S.T. standards, FAST progress monitoring) frame what grade 4-5 expectations look like.

## How to add a topic (30 min)
1. In `topics.js` add a topic to the registry: `{id,name,short,emoji,maxLevel,kinds:[...]}` and a kind entry (label, skill, tip, `generate(level, rnd, wantTrue)` returning `{stmt, truth, sections, ...}`).
2. Add node tests in `tests/topics.test.js` (truth is verified independently, explanations are consistent).
3. `node tools/gen-catalog-sql.js > db/migrations/00N_catalog_<topic>.sql`, then apply it as a named migration
   (the edge function reads `skill_catalog`, so new kinds are rejected until the catalog row exists).
4. `sh build.sh`, `node test.js && node tests/topics.test.js && node tests/engine.test.js && node tests/ui-sync.js`, deploy.
   Nothing else in the UI needs to change (picker, stats, daily mix and review pages are generic).

## "Show your work" written-problem mode (NOT built yet – hook is in place)
- Hook: `ANSWER_MODES` in `app.src.html`. Today `tf` (tap True/False) and `choice2` (two big buttons). Add a `work` mode that renders a text area (steps) + a final-answer
  box and calls `answer(val, {work: text})`; set `q.answerMode = 'work'` on written-problem kinds. Extend `answer(val, extra)` to store `extra.work` with the answer.
- Storage: add `work text` to `practice_questions` and a `work` field on each daily question (edge function `cleanResult` / `cleanPractice`
  currently drop unknown fields – add it with a length cap, e.g. 1000 chars).
- Marking: start with the final answer only (auto), show the kid a worked solution (`explain`) next to their work; a parent review view later.
- Idea: grading hints ("you wrote the right first step!") and an optional photo of paper work.

## Ideas backlog
- **Math Bank rewards (Kyle's idea)**: kids earn "bank" points/coins for finished daily quizzes, personal bests, mastered skills and practice minutes (never for beating a sibling); Kyle sets what the points can be traded for.
  Needs a `kid_rewards` ledger table (event-sourced: +/- rows), a small "bank" screen for kids and an approval/spend screen in the parent view.
- **Show-your-work mode**: see the section above.
- **Writing prompts (later)**: short, kind prompts ("Describe your favorite place using three describing words") with a checklist the kid ticks (capital letters, end punctuation, details),
  stored privately and readable by a parent; no automatic scoring of creativity.
- More English: dictionary/syllables, commonly misspelled words by rule, editing a short paragraph, poetry terms; more psychology: sleep and focus experiments, "how do I learn best".
- More topics: geometry (area/perimeter), decimals, measurement/time, patterns/sequences, negative numbers, word problems, logic puzzles; later any subject (science vocab, spelling, geography) using the same True/False + registry.
- Other question formats: multiple choice, type-in answer, drag/sort, fill the blank.
- Parent dashboard page (read-only, behind a parent PIN) for the SQL views: weekly progress, weakest skills, practice minutes, streak calendar.
- Goals & rewards: weekly practice goal, badges per mastered skill, "boss level" every Friday.
- Timed fluency rounds for times tables (separate from the daily score).
- Sibling challenges (compare streaks, never scores shown as rankings).
- Weekly email/Slack summary to the parent (needs explicit approval to set up).
- Resumable practice sessions across reloads; sync of the adaptive memory as a log (merge by event, not last-write-wins).
- Teacher/parent-written custom questions.

## Known limits
- Adaptive memory is a compact blob in `kid_state.state` (last-write-wins with a per-skill max-attempts merge across devices); raw per-question rows
  (`quiz_results.questions`, `practice_questions`) are the source of truth and the blob can be rebuilt from them.
- Offline cache is per device. A PIN reset deletes the kid row and cascades to that kid's data.

## Skill map and standards (inspiration only - no official affiliation, no official scoring)
- English skills (vocabulary in context, parts of speech, verbs, commonly confused words, punctuation and capitals, spelling patterns, short-passage reading, figurative language, word roots/prefixes/suffixes) follow the spirit of
  - Massachusetts: 2017 ELA/Literacy Curriculum Framework - https://www.doe.mass.edu/frameworks/ela/2017-06.pdf
  - Florida: B.E.S.T. ELA Standards (2025 edition) - https://flrules.org/gateway/readRefFile.asp?filename=B.E.S.T.+ELA+Standards+2025_Clean+6A-1.09401.pdf&refId=18865
- Brain & feelings skills (growth mindset, memory & sleep, focus, feelings & calm-down, empathy, thinking tricks, habits, teamwork) follow the spirit of the CASEL SEL framework - https://casel.org/fundamentals-of-sel/what-is-the-casel-framework/
  They are educational and kid-friendly, not diagnostic or clinical.

## Public statewide context (for the PARENT only; never shown to kids; never used to rank a child)
These are statewide shares of ALL students on a real state test. They are not comparable with this app's quiz results and say nothing about how Lillian or Christian rank.
- Massachusetts, MCAS ELA, % of students "meeting or exceeding expectations" (Massachusetts DESE school/district profiles, state totals):
  - Spring 2026 (data updated Sep 22, 2026): grade 4 = 37% (5% exceeding + 33% meeting, rounded); grade 5 = 38% (4% exceeding + 34% meeting); grades 3-8 combined = 40%.
  - Spring 2025: grade 4 = 40%; grade 5 = 38%.
  - (Math for reference: 2026 grade 4 = 43%, grade 5 = 39%; 2025 grade 4 = 43%, grade 5 = 40%.)
  - https://profiles.doe.mass.edu/mcas/achievement_level.aspx?fycode=2026&orgcode=00000000
  - https://profiles.doe.mass.edu/mcas/achievement_level.aspx?fycode=2025&orgcode=00000000
  - https://www.doe.mass.edu/mcas/2025/results/summary.pdf  and  https://profiles.doe.mass.edu/statereport/mcas.aspx
- Florida, FAST ELA Reading (Florida Department of Education, statewide PM3 results), % of students at Achievement Level 3 or above ("on grade level"):
  - 2025-26: grade 3 = 60%; grade 4 = 60%; grade 5 = 61%; grades 3-10 overall = 61%.
  - 2024-25: grade 4 = 56%; grade 5 = 56%.
  - https://cdn.fldoe.org/file/5668/84ELAandMathResults26.pdf  (official FLDOE results PDF)
  - https://www.fldoe.org/accountability/assessments/k-12-student-assessment/results/  and  .../results/2025.stml (statewide result pages)
  - Florida math grade 4/5 rates were not extracted. Check the same PDF if needed.
- Takeaway: on both state tests roughly 4 in 10 (MA) to 6 in 10 (FL) fourth and fifth graders reach the "meets expectations" level. The two states use different tests and cut scores, so MA and FL numbers should not be compared with each other.
- There is no public peer dataset of individual children's results for this kind of app, so there is no percentile or ranking anywhere in the product. If a real comparison is ever wanted, the honest route is a
  nationally normed assessment (for example NWEA MAP Growth or a state test report) taken at school, and kept parent-only.

## Parent gate and security notes
- The gate in the parent corner is a soft gate (a multiplication/addition question) to keep young kids out of the parent-only view; it is not authentication. A PIN-based parent mode is a future option.
- Birth month/year are never shown on the kid screen after saving, never in public pages, and not in the dashboard views (only the age band and estimated grade).
- Tables have RLS on with no policies (only the edge function, using the server connection, can read/write). Views are `security_invoker` and revoked from anon/authenticated.
