# Test Results — Expected vs Actual

Collection: `collections/api-testing-practice.postman_collection.json`
Environment: `environments/public-apis.postman_environment.json`
Last run: 33 requests, 79 assertions — see [`reports/newman-run.txt`](../reports/newman-run.txt)

APIs under test:
- **JSONPlaceholder** (`https://jsonplaceholder.typicode.com`) — fake REST API, writes are simulated
- **ReqRes** (`https://reqres.in`) — auth, pagination and delayed responses

Result legend: ✅ matched expectation · ⚠️ observation (API behaves differently from REST convention; assertion updated to lock in actual behaviour and the gap is logged below)

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
| 14 | PUT /posts/9999 | Update missing resource | **404** | **500** | ⚠️ OBS-03 |
| 15 | PATCH /posts/1 `{title}` | Partial update | 200, `title` changed, `body`/`userId` preserved | 200, preserved | ✅ |
| 16 | DELETE /posts/1 | Delete | 200 (or 204), `{}` | 200, `{}` | ✅ |
| 17 | DELETE /posts/9999 | Delete missing resource | **404** | **200**, `{}` | ⚠️ OBS-04 |

## Comments (JSONPlaceholder)

| # | Request | Type | Expected | Actual | Result |
|---|---|---|---|---|---|
| 18 | GET /posts/1/comments | Nested resource | 200, 5 comments, all `postId: 1`, valid emails | 200, 5 | ✅ |
| 19 | GET /posts/9999/comments | Nested on missing parent | 200, `[]` | 200, `[]` | ✅ |
| 20 | GET /comments?postId=1 | Equivalence with #18 | Same 5 comments | Same | ✅ |

## Users (JSONPlaceholder)

| # | Request | Type | Expected | Actual | Result |
|---|---|---|---|---|---|
| 21 | GET /users | Nested schema | 200, 10 users, `address.geo.{lat,lng}` | 200 | ✅ |
| 22 | GET /users/1 | Known record | `username: "Bret"` | `Bret` | ✅ |
| 23 | GET /users/-1 | Negative ID | 404 | 404 | ✅ |
| 24 | GET /users/1/todos | Type check | 200, 20 todos, `completed` is boolean | 200 | ✅ |

## ReqRes (auth & pagination)

| # | Request | Type | Expected | Actual | Result |
|---|---|---|---|---|---|
| 25 | GET /api/users?page=2 | Pagination | `page: 2`, `data.length == per_page`, `total_pages == ceil(total/per_page)` | Matches | ✅ |
| 26 | GET /api/users?page=999 | Page out of range | 200, `data: []` | 200, `[]` | ✅ |
| 27 | GET /api/users/23 | Not found | 404, `{}` | 404, `{}` | ✅ |
| 28 | POST /api/login (valid) | Auth happy path | 200, `token` string | 200, token | ✅ |
| 29 | POST /api/login (no password) | Missing field | 400, `"Missing password"` | 400, exact message | ✅ |
| 30 | POST /api/login (unknown user) | Invalid credentials | 400, `"user not found"` | 400 | ✅ |
| 31 | POST /api/register (no email) | Missing field | 400, `"Missing email or username"` | 400 | ✅ |
| 32 | DELETE /api/users/2 | 204 contract | 204, empty body | 204, empty | ✅ |
| 33 | GET /api/users?delay=3 | Latency | 200, response time ≥ 3s | 200, ~3.1s | ✅ |

## Observations

These are places where the public API does not follow the REST behaviour I expected. Because they are fake/mock APIs the behaviour is by design, but on a real project each of these would be raised as a defect.

| ID | Endpoint | Expected | Actual | Why it matters |
|---|---|---|---|---|
| OBS-01 | POST /posts with `{}` | 400 + field errors | 201 | No server-side validation — a client bug that drops fields would silently create empty records |
| OBS-02 | POST /posts with wrong types | 400 | 201, types echoed back | `userId: "one"` accepted; downstream joins would break |
| OBS-03 | PUT /posts/9999 | 404 | 500 | Client can't distinguish "not found" from "server broken"; retries would be wasted |
| OBS-04 | DELETE /posts/9999 | 404 | 200 | Idempotent delete is defensible, but the docs don't say so; callers can't detect a typo'd id |
| OBS-05 | POST /posts malformed JSON | 400 | 500 | Parser exceptions leak as 5xx; should be caught and reported as a client error |

## Lessons

- Always assert on body shape, not just status code — #10 and #11 returned 201 and *looked* fine until I checked the payload.
- Test the boundaries on both sides (0 / 1 / 100 / 101), not just "a valid id" and "a huge id".
- A missing-resource test is needed for **every** write verb, not just GET — PUT and DELETE behaved differently from each other (#14 vs #17).
- Response-time assertions catch config regressions (`delay` param) that status codes never would.
