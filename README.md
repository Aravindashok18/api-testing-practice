# API Testing Practice

API testing practice with **Postman** and **Newman**. A collection of **33 requests / 79 assertions** run against public REST APIs (JSONPlaceholder, ReqRes, GitHub), covering **GET, POST, PUT, PATCH and DELETE**.

Focus is on negative and edge cases: boundary and invalid IDs, missing fields, wrong data types, malformed JSON, pagination limits and auth failures (401/403). Every request has assertions in its Tests tab, and every result is written up as expected-vs-actual in [`docs/test-results.md`](docs/test-results.md).

**5 non-standard responses flagged** — see [Observations](docs/test-results.md#observations-5).

## Contents

```
collections/    api-testing-practice.postman_collection.json   (33 requests, 4 folders)
environments/   public-apis.postman_environment.json           (base URLs + dummy invalid credentials)
docs/           test-results.md                                (expected vs actual for all 33, observations, notes)
reports/        newman-run.txt, newman-junit.xml               (latest CLI run)
```

| Folder | Requests | What it covers |
|---|---|---|
| Posts | 18 | Full CRUD, boundary IDs (0/1/100/101), non-numeric ID, filters, missing fields, wrong types, malformed JSON, write verbs on a missing ID |
| Comments & Users | 3 | Nested resources, missing parent, negative ID |
| ReqRes (pagination & validation) | 7 | Pagination maths, page out of range, 404, login/register missing fields, 204 contract |
| Auth failures | 5 | Unknown user, wrong password, invalid API key, missing token, invalid bearer token |

Methods: GET 17 · POST 9 · DELETE 3 · PUT 2 · PATCH 2

## APIs used

| API | Used for |
|---|---|
| [JSONPlaceholder](https://jsonplaceholder.typicode.com) | CRUD on posts, comments, users; nested resources; filters |
| [ReqRes](https://reqres.in) | Login/register validation, pagination, 204 contract, API-key auth |
| [GitHub REST API](https://docs.github.com/rest) | Reference for correct 401 responses (no token / bad token) |

No real credentials are used. The environment only holds deliberately invalid values (`invalidApiKey`, `invalidGithubToken`, `wrongPassword`), plus the public demo login that ReqRes documents.

## Running it

**Postman:** import the collection and environment, select the "Public APIs" environment, then *Run collection*.

**CLI (Newman):**

```bash
npm install
npm test
```

`npm run test:report` also writes a JUnit XML report to `reports/newman-junit.xml`.

Latest run (21 Nov 2026): **33/33 requests, 79/79 assertions passing**, about 8 s total.

## What I found

Five responses that break REST convention (details and impact in [docs/test-results.md](docs/test-results.md#observations-5)):

| ID | Request | Expected | Actual |
|---|---|---|---|
| OBS-01 | `POST /posts` with `{}` | 400 | 201 |
| OBS-02 | `POST /posts` with wrong types | 400 | 201 |
| OBS-03 | `PATCH /posts/9999` | 404 (PUT returns 404) | 200 |
| OBS-04 | `POST /api/login` with wrong password | 401 | 200 + token |
| OBS-05 | `POST /posts` with malformed JSON | 400 | 500 |

Three more differences (DELETE on a missing ID → 200, unknown user → 400, invalid API key → 403) are logged as notes, not defects, because they are common, defensible designs.

Tests that hit a known issue assert the *actual* behaviour and print a `console.warn` with the observation ID. That way the suite stays green, and it fails if the API changes. Re-running it did catch a change: `PUT /posts/9999` used to return 500 and now returns 404.

## License

MIT
