# ADR-039: Registration processes, answers on the profile, routing by topic and language

- **Status:** Proposed for product and engineering review, 27 November 2026 (BITS GmbH, DigiSucht).
- **Decision owner:** Frank Gerhardt (product owner); proposed by Florian Wenzel and Karl Lober (BITS GmbH).
- **Amends:** [ADR-036](ADR-036-configurable-conversation-modules-and-one-question-model.md) part E (journeys), decisions 11, 18, 29, 30, 37 and 41, for the check-in of Mailberatung (`AGENCY_COUNSELLING`).
- **Issues:** epic [ORISO-Admin#1565](https://github.com/OpenResilienceInitiative/ORISO-Admin/issues/1565), [ORISO-Frontend#2333](https://github.com/OpenResilienceInitiative/ORISO-Frontend/issues/2333), [ORISO-UserService#1789](https://github.com/OpenResilienceInitiative/ORISO-UserService/issues/1789), [ORISO-Frontend#2526](https://github.com/OpenResilienceInitiative/ORISO-Frontend/issues/2526), [ORISO-Frontend#2527](https://github.com/OpenResilienceInitiative/ORISO-Frontend/issues/2527).

## Context

ADR-036 part E describes journeys with three phases (check-in, on board, arrival), routes, destinations, layers for Träger and Beratungsstelle, triggers, ratings, translation states and keywords. The storage in UserService and the Admin editor exist. Nothing of it runs for advice seekers yet:

- The registration in the Frontend does not follow the check-in (ORISO-Frontend#2333).
- Answers are not stored and never reach a counsellor (ORISO-UserService#1789).
- Answers cannot influence which Beratungsstelle is matched (ADR-036 decision 30).

So the editor configures questions whose answers are thrown away. Asking different questions per topic or per Beratungsstelle has no value until answers are kept or used. On the test instance (DigiSucht, ORISO 2.0.18) we walked through the editor with the platform admin and found three more problems:

- The travel metaphor (Reisen, Check-in, An Bord, Ankunft, Ziele) has to be translated back into plain words on every screen; admins think in "registration", "during" and "after the counselling".
- The editor sits inside a tile of Funktionszugriff, which is about permissions, not content.
- Most topics share one registration, but the editor shows them one by one, and "one question per screen" (decision 41) makes even a short registration a long wizard.

## Decision

### Naming and place

1. **Plain words.** In the Admin a journey's check-in is a **process** (German **Ablauf**). Topics and links are **entry points** (Einstiege). On board and arrival are "during" and "after the counselling" and stay hidden until something consumes them. The storage model (journeys) and its API are unchanged.
2. **Own place.** The processes get their own settings tab **Registrierung**, next to Themen and Begrüßung. The journeys block leaves the Funktionszugriff tiles.

### Processes and pages

3. **A process is a list of pages; a page holds blocks.** Blocks are platform steps (topic choice, postcode, Beratungsstelle, username and password, consent, e-mail, add-ons such as age) and own questions. The admin decides how many pages there are, their headings, and which blocks sit on which page. Blocks move between pages by drag and drop. This **replaces decision 41** (one question per screen) for the check-in: a page shows all its blocks at once.
4. **Pages are stored beside the steps.** A journey gets an optional `pages` list: `[{ id, title, stepIds }]`. The order of `phases.departure` always follows the pages, so the existing server rules keep holding unchanged: the topic choice comes first, the postcode before the Beratungsstelle. A journey without `pages` shows every check-in step as a page of its own, so stored journeys stay valid.
5. **Topics belong to a process.** In `/registrierung` each topic runs exactly one process, as before; most topics share the default process. Topics are assigned on the process page. A topic in no process is link-only. A link names its own topics and its own process, so the same topic can run different processes through different links.
6. **Platform level first, nothing layered.** Only the platform admin edits processes in this cut. Layers of Träger and Beratungsstelle (decisions 29 and 37) are not used for the check-in. Letting Träger and Beratungsstelle admins create their own links (with a process of their own) is the next step and needs a scope check in UserService.

### Answers

7. **Answers are kept on the advice seeker's profile** (amends decisions 11 and 18). When the registration completes, UserService stores one record per answered question: question id and version, the question text and the chosen answer **in all languages as they were at that moment**, the date, the topic and the process. Later edits to a question never change a stored record.
8. **Counsellors read them in the case.** The profile of the advice seeker shows a block "Angaben aus der Registrierung", one entry per registration or follow-up enquiry, newest first, to everyone who may see the case today. No new role, no new permission.
9. **Lifecycle.** The records are deleted with the account. They are not part of statistics in this cut. Data protection stays the operator's responsibility as in decision 11; the Admin shows a short notice that answers are stored on the profile.

### Routing

10. **Routing uses data that exists.** Matching stays topic, then postcode, then the chosen Beratungsstelle. A new platform block **Sprache** narrows it by language:
    - The languages of a Beratungsstelle are the languages of its counsellors (`consultant_language` in UserService). Nobody maintains a second list.
    - The block offers only languages that a Beratungsstelle for the chosen topic covers.
    - If no Beratungsstelle near the postcode covers the language, the person is matched without language and told so. (Open for product: fall back nationwide instead.)
11. **No free routing rules yet.** Own questions do not route in this cut. If a real case needs more than topic and language, Beratungsstellen get **Merkmale** (a key and values, for example "Angebot: Glücksspiel") and a choice option may require one. That is a separate decision.

## Consequences

- The Admin part exists on the BITS test instance (branch `feat/registration-view` of BITS-GmbH/ORISO-Admin, image `harbor.mybits.dev/digisucht/admin:2.0.18-reg-d3d2b0e`). It reuses the journey model, store and rules of ADR-036; the journeys editor in Funktionszugriff goes away.
- The Frontend renders a process page by page (ORISO-Frontend#2333), not one question per screen. `JourneyQuestionPager` (ORISO-Frontend#2417) can stay for the in-chat surveys.
- UserService gains an answers table and an endpoint for the counsellor view, plus language-aware matching. The journey storage needs no migration: `pages` is an extra field of the stored JSON.
- The ADR-036 parts that nothing consumes (layers for the check-in, triggers, ratings, translation states, keywords, block profiles) stay in the code but are not shown in the registration editor.
- DigiSucht's own registration override (gender, counselling relation, age) can move into a process once answers are stored; until then it coexists.

## How we check it

- Unit tests in the Admin cover the page model: grouping, order rules, copies with ids the server accepts.
- On the test instance: the platform admin builds a process with three pages, assigns two topics, saves, reloads and sees the same pages.
- Once #2333 and the answers store exist: a new advice seeker registers through that process; the counsellor of the matched Beratungsstelle sees the answers in the profile; a later edit of a question leaves the stored answer unchanged.
- Routing: with two Beratungsstellen near one postcode, one with a counsellor who speaks Turkish, choosing Turkish matches that one; choosing a language nobody covers matches without language and shows the note.
