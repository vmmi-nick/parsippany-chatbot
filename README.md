# Parsippany-Troy Hills Constituent Chatbot

Bilingual (English/Spanish) constituent chatbot for the Township of
Parsippany-Troy Hills, NJ, duplicating the North Bergen chatbot architecture.

- Index-first: answers come from a verified local topic index
  (`data/index.json`, 28 topics) and a verified council meeting schedule
  (`data/meetings.json`, 24 meetings in 2026).
- Backup: questions the index does not cover go to the Meta Model API only
  when `MODEL_API_KEY` is set, instructed to answer only from the approved
  trusted sources (`data/trusted-sources.json`) and otherwise return the
  Mayor's Action Center fallback verbatim.
- Emergencies get a 911 redirect. No personal information is collected.
- Answer style: one direct sentence first, then supporting details, then
  phone numbers / addresses / links in a Contact block at the end.

## Design
Parsippany's own colors from parsippany.net: dark teal `#0C2830` / `#024751`
with gold/ochre `#C8822A`. Layout mirrors the North Bergen front end
(slim top bar, hero, topic cards, "Try asking" chips, sticky mobile composer).

## Run locally
```
npm install
node server.js
# http://localhost:3000  — health check at /health
```

## Deploy (Render)
Blueprint in `render.yaml` (one Starter web service, `parsippany-chatbot`).
Set `MODEL_API_KEY` in the Render dashboard Environment tab. Never commit it.
Deployment under Nick's accounts (GitHub `vmmi-nick`, new repo
`parsippany-chatbot`) comes after local QA passes.

## Verified basics (checked 2026-11-21 on parsippany.net)
- Town Hall: 1001 Parsippany Boulevard, Parsippany, NJ 07054, main 973-263-4350,
  Mon-Fri 9:00 AM-5:00 PM
- Mayor: Pulkit Desai. Mayor's Action Center (help line): 973-263-4262
- Township Clerk / Registrar: 973-263-4351
- Council meetings: Tuesdays 7:00 PM, Municipal Building (Agenda + Regular)

## Open items for production (not build blockers)
- Nick's prototype testing
- Township Clerk sign-off contact
- Site-admin embed paste on parsippany.net
- Possible transfer to township-owned hosting / AI accounts
- Decisions pending: add pthreconline.com / mcmua.com to trusted sources?
  (currently flagged NOT approved, discussion only)

## Question logging

Every `/api/ask` request is logged so unanswered questions can be reviewed
and used to improve the verified index.

What IS stored:

- Timestamp and language of the question.
- For questions the index matched: the matched topic id only (counts of
  which topics get asked about). The matched question text is NOT stored.
- For questions the index did NOT match (Mayor's Action Center fallback or
  model fallback): the question text, and whether the model fallback was
  used. This is the list that shows where the index has gaps.

What is NOT stored: names, contact details, or any personal information
(residents are never asked for any), and matched question text.

Where the log lives:

- Each entry is written to `logs/questions.jsonl` in the service directory.
  Render's filesystem is ephemeral, so that file only lasts while the
  instance lives and resets on redeploys and restarts.
- Each entry is ALSO emitted as a structured console line prefixed
  `CHATLOG`, which lands in the Render service logs (dashboard > service >
  Logs) and can be searched and exported from there. Treat the Render logs
  as the durable copy.

Review endpoint:

- `GET /api/admin/unanswered` returns the unmatched questions, grouped so
  near-duplicates (same words, different case or punctuation) show as one
  entry with a count, sorted by count.
- It is enabled ONLY when the `ADMIN_TOKEN` environment variable is set.
  Set it in the Render dashboard: service > Environment > Add Environment
  Variable, key `ADMIN_TOKEN`, value a long random string of your choosing.
  The service redeploys automatically.
- Call it with the header `x-admin-token: <your ADMIN_TOKEN>`. Without the
  env var set the endpoint returns 404; with a missing or wrong token it
  returns 401.
