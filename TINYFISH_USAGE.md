# How Scout Uses TinyFish

Scout uses three TinyFish APIs as a live search pipeline. Job data is not hardcoded: every search starts from the user's current role, location, seniority, and keywords.

## Search, Fetch, Agent

1. **TinyFish Search** receives a query built from the user's preferences. Scout accepts HTTPS results, excludes common social-media sites, and keeps up to six career-page URLs.
2. **TinyFish Fetch** reads those URLs as markdown. Scout supplies up to 3,500 characters of fetched page text to the extraction Agent.
3. **TinyFish Agent** checks up to three distinct source pages in parallel and returns structured jobs. It is instructed to include only visible, current listings and not invent details or URLs.

Scout scores and filters the extracted listings against the preferences, removes duplicate URLs, and returns up to 15 jobs (at most five per Agent-checked page). The score is a heuristic, not a guarantee of fit or availability. A listing can close or change after it is found; users should confirm details on the employer's page.

### Search data flow

```text
Role + location + seniority + keywords
  → TinyFish Search (up to 6 pages)
  → TinyFish Fetch (page text)
  → TinyFish Agent (up to 3 distinct pages, structured openings)
  → Scout match scoring + deduplication
  → Job cards with source/application links
```

Search, Fetch, and Agent failures are surfaced as errors or reflected in the stage counts; a configured API key does not guarantee a successful live search. Scout does not cache job results.

## Application preparation

The optional **Prepare application** action is a separate, consent-gated TinyFish Agent run. The user chooses which profile fields to send. The Agent is instructed to fill only visible text/email/phone/URL fields that clearly match supplied values, and never to submit, log in, upload files, or complete legal or sensitive fields.

The Agent runs in a hosted browser that Scout cannot transfer into the user's browser. Scout displays only verified field labels and values that exactly match the submitted profile, so the user can review or copy them on the employer page. The application is not submitted, and profile details are not stored by Scout. Only a recent job link returned by Scout is accepted for this action; the temporary allowlist expires after 30 minutes.

## Important limitations

- Search results depend on the live pages TinyFish can discover and read.
- An Agent extraction can be incomplete or stale; always verify the role on its original page.
- Match scores are ranking hints, not an assurance that a role meets every preference.
- LinkedIn/X share links open a draft/composer only. The user must add the public demo URL, tag TinyFish, and publish both posts.

See [README.md](./README.md) for setup and API details.
