# api-contract-testing

![API tests](https://github.com/SalvadorGuerra777/api-contract-testing/actions/workflows/api-tests.yml/badge.svg)

Functional and contract tests for [Restful-Booker](https://restful-booker.herokuapp.com/apidoc/index.html), a public REST API built for practicing API testing. The tests live in a Postman collection and run from the command line with Newman, locally and in GitHub Actions.

## Tech stack

- Postman (collection v2.1, `pm.*` test scripts, JSON schema validation)
- Newman and newman-reporter-htmlextra
- Node.js 20+
- GitHub Actions (on push, on pull requests and nightly)

## Project structure

```
├── collections/restful-booker.postman_collection.json   # Requests and tests
├── environments/restful-booker.postman_environment.json # Base URL and API credentials
├── data/bookings.json                                    # Test data, one iteration per booking
├── package.json                                          # Newman command
└── .github/workflows/api-tests.yml
```

## Setup

```bash
git clone https://github.com/SalvadorGuerra777/api-contract-testing.git
cd api-contract-testing
npm install
```

To use Postman instead, import the collection and the environment, then run the collection with `data/bookings.json` as the data file.

## Running the tests

```bash
npm test
```

The CLI prints each request and assertion. An HTML report is written to `reports/report.html`; in CI it is uploaded as the `newman-report` artifact.

The environment file holds the API's public demo credentials (`admin` / `password123`, published in the Restful-Booker docs). To point at another environment, override values from the command line, for example `npx newman run ... --env-var baseUrl=https://my-env`.

## What is covered

Each run creates a booking per row in `data/bookings.json` and follows it through its full lifecycle.

| Folder | Request | Checks |
|---|---|---|
| Health | Ping | API is up |
| Auth | Create token | 200, token returned and stored for later requests |
| | Wrong password | No token, `Bad credentials` reason |
| Booking lifecycle | Create booking | 200, numeric id, response schema, data echoed back |
| | Get by id | 200, schema, stored data matches |
| | Search by name | New booking appears in the results |
| | List bookings | 200, every item has a numeric id, new booking is listed |
| | Update without token | 403 |
| | Update with invalid token | 403 |
| | Full update with missing fields | 400 |
| | Full update | 200, schema, every field replaced |
| | Partial update | 200, only the patched field changes |
| | Delete without token | 403 |
| | Delete | 201 |
| | Get deleted booking | 404 |
| Negative cases | Non-existent id | 404 |
| | Missing required fields | Request is rejected |

Every request also checks that the response time is under 5 seconds.

Test data covers a regular booking, boundary values (total price `0`, same-day check-in and check-out, empty additional needs) and names with accents, hyphens and apostrophes.

Last local run: 3 iterations, 51 requests, 147 assertions, 0 failures.

## Defects found

| Request | Expected | Actual |
|---|---|---|
| `POST /auth` with a wrong password | `401 Unauthorized` | `200 OK` with `{"reason": "Bad credentials"}` |
| `POST /booking` with missing required fields | `400 Bad Request` | `500 Internal Server Error` |

Why these expected results:

- **401 for wrong credentials.** RFC 9110 (section 15.5.2) defines 401 for a request without valid authentication credentials. A 200 tells clients the request succeeded, so a client that only checks the status code continues without a token.
- **400 for missing fields.** A 5xx status means the server failed; RFC 9110 (section 15.5.1) defines 400 for a request the server can't process because of a client error. The same API already does this on `PUT /booking/{id}`: the same incomplete body returns 400 there (see "Full update with missing fields"), so the 500 on `POST` is inconsistent with its own behaviour.

The collection asserts the current behaviour and labels it, so a fix in the API shows up as a failing test that needs updating.
