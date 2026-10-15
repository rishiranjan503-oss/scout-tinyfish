# Scout — Workshop Submission Status

## Implemented

- [x] Preference form for role, location, seniority, and keywords.
- [x] TinyFish Search → Fetch → Agent job-discovery pipeline.
- [x] Structured job cards with source links, match ranking, and URL deduplication.
- [x] Consent-gated application preparation. The Agent is instructed not to submit, log in, upload files, or complete sensitive/legal fields.
- [x] Scout does not persist application profile details. The TinyFish-hosted browser session is not shown in the user's browser; Scout displays verified profile values for review/copy instead.
- [x] Keyless local checks: homepage returned HTTP 200; missing-key search returned 503; invalid consent and invalid JSON returned 400.

## Still required before submission

- [ ] Add a valid TinyFish API key and run a live search end to end. Live Search, Fetch, and Agent results have not been verified in this session.
- [ ] Test application preparation with consent on a safe, recent Scout result. Confirm values and that no form is submitted.
- [ ] Publish a public demo and include its URL.
- [ ] Publish a LinkedIn post tagging TinyFish.
- [ ] Publish an X post tagging @TinyFishAI.
- [ ] Record links to both published posts.

## Points and review

Scout integrates three meaningful TinyFish endpoints (Search, Fetch, and Agent), corresponding to the workshop's 200-point endpoint tier. This is a target based on the implementation, not an award or approval guarantee; the live demo and other submission requirements still need completion and review.

## Local run

1. Copy `.env.example` to `.env` and set `TINYFISH_API_KEY`.
2. Run `node --env-file=.env server.mjs`.
3. Open `http://localhost:3000`.

See [README.md](./README.md) and [TINYFISH_USAGE.md](./TINYFISH_USAGE.md) for setup and implementation details.
