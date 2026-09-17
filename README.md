# API Testing Practice

Self-directed API testing practice (2025). A Postman collection of **33 requests / 79 assertions** run against public APIs, covering GET, POST, PUT, PATCH and DELETE with a focus on edge cases: boundary IDs, invalid and non-numeric IDs, missing fields, wrong data types, malformed JSON, pagination limits and auth failures.

Every request has assertions in its Tests tab, and every result is written up as expected-vs-actual in [`docs/test-results.md`](docs/test-results.md).

## Contents

```
collections/    api-testing-practice.postman_collection.json   (33 requests, 4 folders)
environments/   public-apis.postman_environment.json           (base URLs)
docs/           test-results.md                                (expected vs actual, 5 observations)
reports/        newman-run.txt                                 (latest CLI run output)
```

## APIs used

| API | Used for |
|---|---|
| [JSONPlaceholder](https://jsonplaceholder.typicode.com) | CRUD on posts / comments / users, nested resources, filters |
| [ReqRes](https://reqres.in) | Login / register validation errors, pagination, 204 contract, delayed responses |

## Running it

**Postman:** import the collection and environment files, select the "Public APIs" environment, and use *Run collection*.

**CLI (Newman):**

```bash
npx newman run collections/api-testing-practice.postman_collection.json \
  -e environments/public-apis.postman_environment.json
```

Latest run: 33/33 requests, 79/79 assertions passing.

## What I found

Five cases where the API behaviour differs from what a REST client would expect (e.g. `PUT` on a missing resource returns **500** not 404, malformed JSON returns **500** not 400, empty `POST` body is accepted with **201**). Each is logged with impact in the observations table of [docs/test-results.md](docs/test-results.md#observations).
