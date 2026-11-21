# Test Results — Expected vs Actual

Collection: `collections/api-testing-practice.postman_collection.json`
Environment: `environments/public-apis.postman_environment.json`
Last run: 33 requests, 79 assertions, 0 failures — see [`reports/newman-run.txt`](../reports/newman-run.txt) and [`reports/newman-junit.xml`](../reports/newman-junit.xml)

APIs under test:
- **JSONPlaceholder** (`https://jsonplaceholder.typicode.com`) — fake REST API, writes are simulated
- **ReqRes** (`https://reqres.in`) — login/register validation, pagination, API-key auth
- **GitHub REST API** (`https://api.github.com`) — a production API, used as the reference for correct 401 behaviour

Method coverage: GET 17 · POST 9 · DELETE 3 · PUT 2 · PATCH 2

Result legend: ✅ matched expectation · ⚠️ observation (API behaves differently from REST convention; assertion locks in actual behaviour so the run stays green, gap is logged below) · 📝 note (differs from what I expected, but defensible, so not counted as an observation)

## Posts (JSONPlaceholder)

| # | Request | Type | Expected | Actual | Result |
|---|---|---|---|---|---|
| 1 | GET /posts | Happy path | 200, array of 100, keys `id,userId,title,body` | 200, 100 items, keys match | ✅ |
| 2 | GET /posts/1 | Boundary (min id) | 200, `id: 1` | 200, `id: 1` | ✅ |
| 3 | GET /posts/100 | Boundary (max id) | 200, `id: 100` | 200 | ✅ |
| 4 | GET /posts/0 | Boundary (below min) | 404, `{}` | 404, `{}` | ✅ |
| 5 | GET /posts/101 | Boundary (above max) | 404 | 404 | ✅ |
| 6 | GET /posts/abc | Invalid ID (non-numeric) | 404, no stack trace | 404, `{}` | ✅ |
| 7 | GET /posts?userId=1 | Filter | 200, 10 posts all `userId: 1` | 200, 10 posts | ✅ |
| 8 | GET /posts?userId=9999 | Filter no match | 200, `[]` (not 404) | 200, `[]` | ✅ |
| 9 | POST /posts (valid) | Create | 201, `id: 101`, body echoed | 201, `id: 101` | ✅ |
| 10 | POST /posts `{}` | Missing required fields | **400** with validation error | **201**, `{ id: 101 }` | ⚠️ OBS-01 |
| 11 | POST /posts wrong types | Type validation | **400** | **201**, values echoed unchanged (`userId: "one"`) | ⚠️ OBS-02 |
| 12 | POST /posts malformed JSON | Parser error | **400 Bad Request** | **500 Internal Server Error** | ⚠️ OBS-05 |
| 13 | PUT /posts/1 | Full update | 200, updated fields returned | 200 | ✅ |
| 14 | PUT /posts/9999 | Update missing resource | 404 | 404, `{}` | ✅ |
| 15 | PATCH /posts/1 `{title}` | Partial update | 200, `title` changed, `body`/`userId` preserved | 200, preserved | ✅ |
| 16 | PATCH /posts/9999 | Partial update of missing resource | **404** (same as PUT #14) | **200**, `{ title }` echoed, no `id` | ⚠️ OBS-03 |
| 17 | DELETE /posts/1 | Delete | 200 (or 204), `{}` | 200, `{}` | ✅ |
| 18 | DELETE /posts/9999 | Delete missing resource | 404 | 200, `{}` | 📝 N-1 |

## Comments & Users (JSONPlaceholder)

| # | Request | Type | Expected | Actual | Result |
|---|---|---|---|---|---|
| 19 | GET /posts/1/comments | Nested resource | 200, 5 comments, all `postId: 1`, valid emails | 200, 5 | ✅ |
| 20 | GET /posts/9999/comments | Nested on missing parent | 200, `[]` | 200, `[]` | ✅ |
| 21 | GET /users/-1 | Negative ID | 404 | 404 | ✅ |

## ReqRes (pagination & validation)

| # | Request | Type | Expected | Actual | Result |
|---|---|---|---|---|---|
| 22 | GET /api/users?page=2 | Pagination | `page: 2`, `data.length == per_page`, `total_pages == ceil(total/per_page)` | Matches | ✅ |
| 23 | GET /api/users?page=999 | Page out of range | 200, `data: []` | 200, `[]` | ✅ |
| 24 | GET /api/users/23 | Not found | 404, `{}` | 404, `{}` | ✅ |
| 25 | POST /api/login (valid) | Auth happy path | 200, `token` string (saved to a collection variable) | 200, token | ✅ |
| 26 | POST /api/login (no password) | Missing field | 400, `"Missing password"` | 400 | ✅ |
| 27 | POST /api/register (no email) | Missing field | 400, `"Missing email or username"` | 400, exact message | ✅ |
| 28 | DELETE /api/users/2 | 204 contract | 204, empty body | 204, empty | ✅ |

## Auth failures

| # | Request | Type | Expected | Actual | Result |
|---|---|---|---|---|---|
| 29 | POST /api/login (unknown email) | Invalid credentials | 401 | 400, `"user not found"` | 📝 N-2 |
| 30 | POST /api/login (known email, wrong password) | Invalid credentials | **401**, no token | **200**, same token as the valid login (#25) | ⚠️ OBS-04 |
| 31 | GET /api/users/2 with unknown `x-api-key` | Invalid API key | 401 | 403, `error: invalid_api_key`, no user data | 📝 N-3 |
| 32 | GitHub GET /user, no token | Missing credentials | 401, `"Requires authentication"` | 401, exact message | ✅ |
| 33 | GitHub GET /user, invalid bearer token | Invalid credentials | 401, `"Bad credentials"` | 401, exact message | ✅ |

## Observations (5)

Places where the response breaks HTTP/REST convention in a way a client cannot work around. JSONPlaceholder and ReqRes are mock APIs, so this is behaviour by design, but on a real project each would be raised as a defect.

| ID | Endpoint | Expected | Actual | Why it matters |
|---|---|---|---|---|
| OBS-01 | POST /posts with `{}` | 400 + field errors | 201 | No server-side validation — a client bug that drops fields would silently create empty records |
| OBS-02 | POST /posts with wrong types | 400 | 201, types echoed back | `userId: "one"` accepted; downstream joins would break |
| OBS-03 | PATCH /posts/9999 | 404 | 200 | PUT and PATCH disagree on the same missing id (404 vs 200); a client gets "success" for an update that touched nothing |
| OBS-04 | POST /api/login, wrong password | 401, no token | 200 + token | Password is not checked at all for a known email — the auth failure case is never reported |
| OBS-05 | POST /posts malformed JSON | 400 | 500 | Parser exceptions leak as 5xx; should be caught and reported as a client error |

## Notes (not counted)

Differences from my first expectation that are defensible, so I did not flag them:

| ID | Endpoint | Expected | Actual | Why not flagged |
|---|---|---|---|---|
| N-1 | DELETE /posts/9999 | 404 | 200 | Treating DELETE as idempotent and returning success for a missing resource is a common, documented design choice |
| N-2 | POST /api/login, unknown email | 401 | 400 | Many login endpoints return 400 for bad credentials in the body; the error message is clear |
| N-3 | Unknown `x-api-key` | 401 | 403 | Strictly 401 fits "credentials invalid", but 403 for a revoked/unknown key is widespread and the body explains it |

## Changes since the first run (Sep 2026)

- `PUT /posts/9999` used to return **500**; it now returns **404**. The old OBS-03 no longer reproduces, so test #14 was changed to assert 404 and the observation was replaced by the PATCH inconsistency (#16).
- Added real auth-failure tests (#29–#33). Previously "auth" only meant ReqRes login validation errors.
- Dropped five redundant GETs (all users, user by id, user todos, `?postId=` equivalence, 3 s `delay` check) to keep the collection at 33 requests.

## Lessons

- Always assert on body shape, not just status code — #10 and #11 returned 201 and *looked* fine until I checked the payload.
- Test the boundaries on both sides (0 / 1 / 100 / 101), not just "a valid id" and "a huge id".
- A missing-resource test is needed for **every** write verb — PUT, PATCH and DELETE on the same missing id gave three different answers (#14, #16, #18).
- For auth, test "wrong password for a real user", not just "unknown user" — only the first one exposed OBS-04.
- Public APIs change: re-running the suite found that #14 had changed behaviour since it was written.
