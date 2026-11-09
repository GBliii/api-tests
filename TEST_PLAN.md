# Test plan — API tests (Wikipedia API & Restful Booker)

## Scope

**Wikipedia API (read-only):** REST API page summaries and the featured feed; Action API full-text search and OpenSearch suggestions; error handling of both APIs.
Out of scope: authenticated or write operations (editing, watchlists), other language editions, media endpoints, rate-limit behaviour.

**Restful Booker:** health check, token authentication, the full life cycle of a booking (create, read, filter, full and partial update, delete), authorization and input-validation negatives.
Out of scope: Basic-auth variant of the protected endpoints, XML responses, concurrency, performance.

## Environment

| | Wikipedia API | Restful Booker |
|---|---|---|
| Base URL | `https://en.wikipedia.org/api/rest_v1`, `https://en.wikipedia.org/w/api.php` | `https://restful-booker.herokuapp.com` |
| Auth | none; `Api-User-Agent` header identifies the client | token from `POST /auth` sent as `Cookie: token=…`; demo credentials `admin` / `password123` (public, documented) |
| Data | live, changes daily (feed) and over time (descriptions) | shared public sandbox, reset periodically |
| Tooling | Postman desktop app, Newman 6.2, Node 22 | same |
| CI | GitHub Actions, `ubuntu-latest`, matrix of the two collections | same |

## Test cases — Wikipedia API

| ID | Title | Priority | Type | Request | Expected result |
|----|-------|----------|------|---------|-----------------|
| TC-W01 | Summary of an existing article | High | Positive | `GET /page/summary/Amelia_Earhart` | 200, JSON; schema (type, title, pageid, lang, extract, content_urls); `type=standard`, `title=Amelia Earhart`, `lang=en`; extract mentions Earhart; description starts with "American aviation pioneer"; desktop URL is the article page |
| TC-W02 | Summary of a non-existent article | High | Negative | `GET /page/summary/Blahblahchanandolerbong` | 404; JSON problem body with `type` containing `not_found`, a `title` saying not found and a `detail` |
| TC-W03 | Redirect page | Medium | Positive | `GET /page/summary/USA` | 302 followed by the client → 200; `title=United States`, `titles.canonical=United_States` |
| TC-W04 | Title with parentheses | Medium | Positive, special characters | `GET /page/summary/Alien_(film)` | 200; `title=Alien (film)`; description mentions a film |
| TC-W05 | Today's featured feed | High | Positive, dynamic data | `GET /feed/featured/{yyyy}/{mm}/{dd}` (today, UTC) | 200; `tfa` with non-empty title, normalizedtitle, extract, `type=standard`; `onthisday` non-empty with `text` and `year` |
| TC-W06 | Invalid date in the feed path | Medium | Negative | `GET /feed/featured/2026/13/45` | 4xx client error (400 or 404), JSON problem body with `type` and `title` |
| TC-W07 | Full-text search finds the article | High | Positive | `action=query&list=search&srsearch=Earhart&srlimit=5` | 200; `batchcomplete=true`; `totalhits>0`; exactly 5 results; each has numeric `pageid`, `title`, `snippet`; "Amelia Earhart" among titles |
| TC-W08 | Search with no results | Medium | Negative | `srsearch=blahblahchanandolerbong` | 200; `totalhits=0`; `search=[]` |
| TC-W09 | OpenSearch suggestions | Medium | Positive | `action=opensearch&search=Amelia Ea&limit=3` | 200; array of 4 arrays; first element is the query; ≤3 titles including "Amelia Earhart"; URLs on en.wikipedia.org |
| TC-W10 | Unknown action | Medium | Negative, error handling | `action=nonsense` | 200; `error.code` is `badvalue` (or `unknown_action`); `error.info` names `action` |
| TC-W11 | Missing required parameter | Medium | Negative, error handling | `action=query&list=search` without `srsearch` | 200; `error.code` is `missingparam` (or `nosrsearch`); `error.info` names `srsearch` |

## Test cases — Restful Booker

| ID | Title | Priority | Type | Request | Expected result |
|----|-------|----------|------|---------|-----------------|
| TC-B01 | Health check | High | Health | `GET /ping` | 201 (documented), body `Created` |
| TC-B02 | Create token | High | Positive, auth | `POST /auth` with valid credentials | 200; non-empty `token` |
| TC-B03 | Create token with a wrong password | High | Negative, auth | `POST /auth` | 200 (documented); `reason=Bad credentials`; no `token` |
| TC-B04 | Create a booking | High | Positive | `POST /booking` with a full payload | 200; numeric `bookingid`; `booking` equals the payload |
| TC-B05 | Read the booking | High | Positive | `GET /booking/{id}` | 200; schema (strings, number, boolean, ISO dates); equals the created booking |
| TC-B06 | Filter bookings by guest name | Medium | Positive | `GET /booking?firstname=Regina&lastname=Phalange` | 200; list of `{bookingid}` contains the new id |
| TC-B07 | Full update | High | Positive | `PUT /booking/{id}` + token | 200; response equals the new payload |
| TC-B08 | Partial update | High | Positive | `PATCH /booking/{id}` + token, two fields | 200; the two fields changed; all other fields unchanged |
| TC-B09 | Delete the booking | High | Positive | `DELETE /booking/{id}` + token | 201 (documented), body `Created` |
| TC-B10 | Deleted booking is gone | High | Positive | `GET /booking/{id}` | 404, body `Not Found` |
| TC-B11 | Unknown booking id | Medium | Negative | `GET /booking/999999999` | 404 |
| TC-B12 | Update without a token | High | Negative, auth | `PUT /booking/1` without Cookie | 403, body `Forbidden` |
| TC-B13 | Delete with an invalid token | High | Negative, auth | `DELETE /booking/1`, `Cookie: token=invalid-token` | 403 |
| TC-B14 | Create with an incomplete body | Medium | Negative, validation | `POST /booking` with only `firstname` | 500 (observed behaviour), no `bookingid` in the response |

Execution order matters for TC-B02 → TC-B10: they form one chain and share the token and the booking id through collection variables. Newman runs folders and requests in the order they are listed.

## Candidates for the next iteration

- Wikipedia: `/page/related`, `/page/media-list`, language variants (`de.wikipedia.org`), response-header checks (`cache-control`, `etag`), data-driven search over a list of queries.
- Restful Booker: Basic-auth variant of PUT/PATCH/DELETE, filtering by `checkin`/`checkout` dates, boundary values for `totalprice`, concurrent updates of one booking.
- Both: the same suites in TypeScript with Playwright's API testing; schema files stored next to the collections instead of inline.

## Observations

- **Restful Booker uses unconventional status codes on purpose.** `GET /ping` → 201; `DELETE` → 201; `POST /booking` → 200; a failed login → 200 with a `reason`. The tests assert the documented behaviour, and the test names say "documented behaviour" so that nobody reads them as an endorsement. In a production API these would be 200, 200/204, 201 and 401.
- **Restful Booker answers 500 to invalid input** (TC-B14). A 500 means the server crashed on bad data instead of validating it; a production API should answer 400 with a message naming the invalid or missing fields. Recorded here rather than reported: the API is a training sandbox and this is known behaviour.
- **The Action API reports errors with HTTP 200.** Checking only the status code would mark a broken request as passed; the tests check for the `error` object. The REST API, by contrast, uses real status codes and a problem+json body.
- **Redirects are invisible to a client that follows them.** `GET /page/summary/USA` looks like a plain 200 in Postman; the 302 shows only with redirects disabled. Worth knowing when a test "passes" for a title that is actually a redirect.

## Risks and known limitations

- **Live data (Wikipedia).** The featured feed changes every day and the short description of an article comes from Wikidata and can be edited at any time; TC-W01 asserts only the beginning of the description, TC-W05 asserts structure, not content.
- **Shared sandbox (Restful Booker).** Other users create and delete bookings at the same time and the database is reset periodically. The tests create their own booking and never assert on the contents of pre-existing records (TC-B12/TC-B13 use id 1 only because authorization is checked before the record is looked up).
- **Network and third parties.** Both APIs are public services; an outage or slow response fails the run for reasons unrelated to the tests. The response-time check (8 s) is generous for that reason.
- **Credentials in the repository.** The environment file contains the public demo credentials of Restful Booker because they are documented and shared by everyone. Real credentials would be injected from CI secrets (`--env-var password=$SECRET`) and never committed.
- **Error codes of the Action API** are asserted as "one of" two historical values (`badvalue`/`unknown_action`, `missingparam`/`nosrsearch`) because MediaWiki has renamed them between versions.
