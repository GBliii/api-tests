# API tests — Wikipedia API & Restful Booker

[![API tests](https://github.com/GBliii/api-tests/actions/workflows/api-tests.yml/badge.svg)](https://github.com/GBliii/api-tests/actions/workflows/api-tests.yml)

API tests for two public APIs, written as [Postman](https://www.postman.com/) collections and run from the command line and in CI with [Newman](https://github.com/postmanlabs/newman):

- **Wikipedia API** — the REST API (page summaries, featured feed) and the MediaWiki Action API (search). Read-only, GET requests only. The same product is covered on the UI level in [maestro-mobile-e2e-tests](https://github.com/GBliii/maestro-mobile-e2e-tests), and several requests here mirror those UI tests.
- **Restful Booker** — a public practice API with authentication and full CRUD on bookings: POST, GET, PUT, PATCH, DELETE, token handling, negative cases.

A portfolio project: request chaining, JSON schema validation, negative testing, dynamic test data, documented API quirks, CI with reports.

## What is covered

### Wikipedia API — 11 requests, 43 checks

| # | Request | Checks | Type |
|---|---------|--------|------|
| W01 | `GET /page/summary/Amelia_Earhart` | 200, JSON, schema, `type=standard`, title, language, extract, description, desktop URL | positive |
| W02 | `GET /page/summary/<nonsense>` | 404, problem body (`not_found`, title, detail) | negative |
| W03 | `GET /page/summary/USA` (redirect) | 302 followed → 200, canonical title `United_States` | positive |
| W04 | `GET /page/summary/Alien_(film)` | 200, parentheses preserved in the title | positive, special characters |
| W05 | `GET /feed/featured/{today}` | 200, `tfa` (featured article) present and well-formed, `onthisday` not empty | positive, dynamic data |
| W06 | `GET /feed/featured/2026/13/45` | 4xx client error with a JSON problem body | negative |
| W07 | `action=query&list=search&srsearch=Earhart` | 200, `batchcomplete`, hits > 0, exactly `srlimit` results, result shape, "Amelia Earhart" present | positive |
| W08 | `action=query&list=search` with a nonsense query | 200, `totalhits=0`, empty list | negative |
| W09 | `action=opensearch&search=Amelia Ea` | 200, array of four arrays, ≤ limit suggestions, URLs on en.wikipedia.org | positive |
| W10 | `action=nonsense` | 200 with an `error` object (`badvalue`), message names the parameter | negative, error handling |
| W11 | `action=query&list=search` without `srsearch` | 200 with an `error` object naming `srsearch` | negative, error handling |

### Restful Booker — 14 requests, 43 checks

| # | Request | Checks | Type |
|---|---------|--------|------|
| B01 | `GET /ping` | 201 (documented), body `Created` | health |
| B02 | `POST /auth` valid credentials | 200, non-empty token → saved for later requests | positive, auth |
| B03 | `POST /auth` wrong password | 200 (documented), `reason: Bad credentials`, no token | negative, auth |
| B04 | `POST /booking` | 200, numeric `bookingid`, echoed booking equals the payload → id saved | positive, create |
| B05 | `GET /booking/{id}` | 200, JSON schema, equals the created booking | positive, read |
| B06 | `GET /booking?firstname=&lastname=` | 200, list contains the new id | positive, filter |
| B07 | `PUT /booking/{id}` + token | 200, response equals the full update | positive, update |
| B08 | `PATCH /booking/{id}` + token | 200, sent fields changed, other fields unchanged | positive, partial update |
| B09 | `DELETE /booking/{id}` + token | 201 (documented), body `Created` | positive, delete |
| B10 | `GET /booking/{id}` after delete | 404, body `Not Found` | positive, verify delete |
| B11 | `GET /booking/999999999` | 404 | negative |
| B12 | `PUT /booking/1` without token | 403, body `Forbidden` | negative, auth |
| B13 | `DELETE /booking/1` with an invalid token | 403 | negative, auth |
| B14 | `POST /booking` with an incomplete body | 500 (observed), no id returned | negative, validation |

Test cases with priorities, observations and risks: [TEST_PLAN.md](TEST_PLAN.md).

## How the tests are built

- **Every request asserts, not just calls.** Status code, content type, JSON schema (`pm.response.to.have.jsonSchema`), exact field values, and what must *not* be there (no token on a failed login, no id after a bad create).
- **Request chaining.** The auth token and the new booking id are passed through collection variables, so the Restful Booker collection is one scenario: create → read → find → update → patch → delete → verify the 404.
- **Payload built once, compared everywhere.** The booking to create is built in a pre-request script and stored; the tests compare the API's answers with exactly what was sent instead of re-typing the expected values.
- **Dynamic data.** The featured-feed request computes today's date in a pre-request script (UTC) — the same idea as the date script in the Maestro project.
- **Configuration outside the tests.** Base URL and credentials live in an environment file; the collection itself has no hard-coded host.
- **Collection-level scripts.** Headers that every request needs (`Api-User-Agent` for Wikimedia, `Accept: application/json` for Restful Booker) and a response-time check are set once for the whole collection.
- **Self-contained data.** The tests create the booking they work on and delete it at the end; nothing depends on records that other users of the shared sandbox may change.
- **Quirks are asserted on purpose.** Where an API deliberately deviates from HTTP conventions (`201` on ping and delete, `200` on a failed login) the tests assert the documented behaviour and the test names say so; see Observations below.

## Project structure

```
postman/
  wikipedia-api.postman_collection.json       # 3 folders, 11 requests
  restful-booker.postman_collection.json      # 4 folders, 14 requests
  restful-booker.postman_environment.json     # base URL + public demo credentials
.github/workflows/api-tests.yml               # CI: both collections on every push
package.json                                  # newman + HTML reporter, npm scripts
TEST_PLAN.md                                  # test cases, observations, risks
```

## How to run

### In Postman

Import the two collections and the environment (File → Import → drop the files from `postman/`), select the *Restful Booker - public sandbox* environment, open a collection and click **Run**. The Wikipedia collection needs no environment: its base URLs are collection variables.

### From the command line (Newman)

Requirements: Node.js 18+.

```bash
npm ci                      # installs newman and the HTML reporter from package-lock.json
npm test                    # both collections
npm run test:wikipedia-api  # one collection
npm run test:restful-booker
```

Each run prints the results to the terminal and writes two reports to `reports/`: an HTML report (`newman-reporter-htmlextra`) and a JUnit XML file for CI systems.

Without installing anything:

```bash
npx newman run postman/wikipedia-api.postman_collection.json
npx newman run postman/restful-booker.postman_collection.json -e postman/restful-booker.postman_environment.json
```

### CI (GitHub Actions)

[`api-tests.yml`](.github/workflows/api-tests.yml) runs both collections as parallel matrix jobs on every push to `main` and on every pull request, and can be started by hand from the Actions tab. Each job uploads its HTML and JUnit reports as a build artifact.

## Observations (API behaviour worth knowing)

Restful Booker is a practice API and some of its answers are deliberately unconventional; the tests assert the documented behaviour and the test plan records what a production API would be expected to do instead:

- `GET /ping` answers **201 Created** instead of 200.
- A failed login (`POST /auth` with wrong credentials) answers **200** with `{"reason": "Bad credentials"}` instead of 401.
- `POST /booking` answers **200**, not 201; `DELETE /booking/{id}` answers **201**, not 200/204.
- An invalid booking payload answers **500 Internal Server Error** instead of 400 with a validation message.
- The database is shared with everyone and is reset periodically, so ids and records change under you; tests must own their data.

Wikipedia:

- The MediaWiki **Action API reports errors with HTTP 200** and an `error` object in the body; a client that only checks the status code will miss them.
- The REST API answers **404 with a problem+json body** (`type`, `title`, `detail`) for unknown pages, and **302** for redirect pages (clients normally follow it).
- Wikimedia asks API clients to identify themselves; the collection sends an `Api-User-Agent` header from a collection-level pre-request script.

## Roadmap

- The same two suites in **TypeScript with Playwright's API testing** (`request` fixture), side by side with the Postman version.
- Data-driven search tests (Newman iteration data) for a list of queries.
- Contract checks of the REST API against its published OpenAPI specification.

## Author

Gerta — QA Engineer · [GitHub](https://github.com/GBliii)
