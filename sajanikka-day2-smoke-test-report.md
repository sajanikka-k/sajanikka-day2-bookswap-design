# BookSwap — Mock and Smoke Test Report

## Day 2 — Deliverable 4



* **OpenAPI file:** `sajanikka-day2-bookswap-openapi.yaml`
* **Mock server:** Stoplight Prism (`@stoplight/prism-cli`)
* **Mock server address:** Prism listened on `0.0.0.0:4010`; requests were sent to `http://127.0.0.1:4010` (curl) and `http://0.0.0.0:4010` (Postman)



## 1\. Setup



I used Stoplight Prism to run a mock API from the BookSwap OpenAPI specification. I used `curl.exe` in PowerShell to send the smoke-test requests and check the HTTP status codes and response bodies. I also used Postman to send one request to each of the seven API operations (see Section 6).

**Command used to start the mock server**

```powershell
npx.cmd @stoplight/prism-cli mock "$env:USERPROFILE\\\\OneDrive - BISTEC Global\\\\Documents\\\\sajanikka-day2-bookswap-openapi.yaml" --port 4010 -h 0.0.0.0 -d
```

The `-d` option runs Prism in dynamic mode, so it generates random response values that match the schemas. The `-h 0.0.0.0` option makes Prism listen on all network addresses.

**Tools used**

* **Stoplight Prism:** to create a mock API from the OpenAPI specification.
* **PowerShell:** to run the commands.
* **curl.exe:** to send HTTP requests and inspect responses. (`curl.exe` is used instead of `curl`, because in Windows PowerShell `curl` is an alias for a different command.)
* **JSON files:** to provide request bodies for the book creation tests. Sending the JSON from a file avoids PowerShell quoting problems.
* **Postman:** to send a request to every operation and capture screenshots.

The test results below are from the mock server, not a live production API.





## 2\. Results Summary



|Metric|Target|Actual result|Status|
|-|-|-|-|
|Smoke tests executed|5|5|Pass|
|Tests returning the expected HTTP status|5|5|Pass|
|Tests covering negative scenarios|At least 2|2|Pass|
|API operations with documented error responses|At least 4|7 of 7|Pass|

All five smoke tests returned the expected HTTP status codes. The two negative tests checked a missing required field (Test 3) and a missing Authorization header (Test 5).





## 3\. Test Cases and Results



### Test 1: List books

**Request:** `GET /books?page=1\\\&pageSize=20`

**Purpose:** Check whether the API can return a list of books.

**Command:**

```powershell
curl.exe -i -H "Authorization: Bearer test" "http://127.0.0.1:4010/books?page=1\\\&pageSize=20"
```

**Expected result:** `200 OK`
**Actual result:** `200 OK`
**Status:** PASS

**Observation:** The mock API returned a page of books that matched the `PageOfBooks` schema. However, the generated pagination values did not match the request. The response contained `"page": 9776541` and `"pageSize": 4`, although the request used `page=1` and `pageSize=20`. The values stayed within the schema limits (`pageSize` between 1 and 100, no negative numbers), but a mock cannot link response values to request values.

This shows that generated mock data does not demonstrate correct pagination behaviour.



### Test 2: Create a book with valid data

**Request:** `POST /books`

**Purpose:** Check whether the API accepts a book request containing the required fields.

The request body was saved in `book.json`:

```json
{
  "title": "The Hobbit",
  "author": "J.R.R. Tolkien",
  "isbn": "9780547928227",
  "condition": "good"
}
```

**Command:**

```powershell
curl.exe -i -X POST "http://127.0.0.1:4010/books" -H "Authorization: Bearer test" -H "Content-Type: application/json" -d "@book.json"
```

**Expected result:** `201 Created`
**Actual result:** `201 Created`
**Status:** PASS

**Observation:** The mock API returned a `Book` object, including the server-generated fields `id`, `ownerId`, `availability` and `createdAt`. However, the generated `title`, `author` and `isbn` did not match the submitted values.

This is a limitation of a generated mock response. It does not prove that a real implementation saves the submitted data correctly.



### Test 3: Create a book without the required title (negative test)

**Request:** `POST /books`

**Purpose:** Check whether the API rejects a book request that is missing a required field.

The request body was saved in `bad-book.json`:

```json
{
  "author": "J.R.R. Tolkien",
  "isbn": "9780547928227",
  "condition": "good"
}
```

**Command:**

```powershell
curl.exe -i -X POST "http://127.0.0.1:4010/books" -H "Authorization: Bearer test" -H "Content-Type: application/json" -d "@bad-book.json"
```

**Expected result:** `400 Bad Request` or `422 Unprocessable Entity` (both are documented for this operation)
**Actual result:** `422 Unprocessable Entity`
**Status:** PASS

**Observation:** Prism's `sl-violations` response header reported that the request body must have the required property `title`. This confirmed that the OpenAPI schema defines `title` as a required field. The response body used the documented `ErrorResponse` format (`{"error": {"code": ..., "message": ...}}`).



### Test 4: Submit a borrow request

**Request:** `POST /books/999/borrow-requests`

**Purpose:** Check whether the borrow-request endpoint returns the expected response.

**Command:**

```powershell
curl.exe -i -X POST "http://127.0.0.1:4010/books/999/borrow-requests" -H "Authorization: Bearer test"
```

**Expected result:** `201 Created`
**Actual result:** `201 Created`
**Status:** PASS

**Observation:** The mock API returned a `BorrowRequest` object with `"status": "pending"`. The specification requires a newly created borrow request to be `pending`, and the mock followed this rule. The specification does not define a request body for this endpoint, so the request was sent without one.

The test confirms that the mock returns the documented success response. It does not prove that the book with ID `999` exists or that the borrowing rules are enforced. Any book ID is accepted by the mock.

**Authentication limitation:** `Bearer test` was a placeholder token used for this mock test. It was not a real JWT, so this test does not verify real token validity or borrower permissions.



### Test 5: List books without an Authorization header (negative test)

**Request:** `GET /books?page=1\\\&pageSize=20`

**Purpose:** Check the response when the Authorization header is missing.

**Command:**

```powershell
curl.exe -i "http://127.0.0.1:4010/books?page=1\\\&pageSize=20"
```

**Expected result:** `401 Unauthorized`
**Actual result:** `401 Unauthorized`
**Status:** PASS

**Observation:** Prism returned `401 Unauthorized`, and the `sl-violations` header reported "Invalid security scheme used". This confirms that the mock applies the bearer-token security requirement defined in the specification.

It does not prove that a real API correctly validates JWT signatures, expiry or permissions.



## 4\. Findings





### Finding 1: Required-field validation works in the mock

The book creation request without `title` returned `422 Unprocessable Entity`, and Prism identified the missing required property.

**Why it matters:** Required fields help prevent incomplete book records from being accepted.



### Finding 2: The authentication requirement is enforced by the mock

The request without an Authorization header returned `401 Unauthorized`.

**Why it matters:** Protected endpoints should reject requests that do not provide authentication credentials. Real token validation still needs to be tested against an implemented API.



### Finding 3: Generated mock responses do not reflect request data

The book creation response contained generated values instead of the submitted book details. The book-list response also returned pagination values different from those requested.

**Why it matters:** A mock is useful for checking response shapes and documented status codes, but these results do not prove that a real API correctly processes, stores or returns data.



### Finding 4: The borrow-request endpoint has no request body

The `POST /books/{bookId}/borrow-requests` operation defines no request body and returned `201 Created` in the mock.

**Why it matters:** This may be correct if the server identifies the borrower from the authenticated user's token and needs no other input. The specification should state this clearly. If clients must submit additional information, a request body should be defined.



### Finding 5: Some success responses can contradict the request

In Postman (Section 6), `PATCH /loans/{loanId}` was sent with `"status": "returned"`, but the `200 OK` response contained `"status": "overdue"` and `"returnedAt": null`. The `200` response of this operation uses the general `Loan` schema, which allows any loan status. The same applies to `PATCH /borrow-requests/{requestId}`: its `200` response allows `pending`, `accepted` or `declined`, whatever decision was sent.

**Why it matters:** A client reading the specification cannot rely on the response confirming the change it requested. The specification should state what a successful response looks like for each operation.



## 5\. Recommended OpenAPI Specification Changes





Line numbers refer to the current version of `sajanikka-day2-bookswap-openapi.yaml` (533 lines). They can change if the file is edited, so the location names are also given.



### Change 1: Make the error object required

**Location:** `components.schemas.ErrorResponse` (line 493)

The schema already has `additionalProperties: false`, but it does not list `error` as required. A response of `{}` would therefore be valid.

**Recommended change:**

```yaml
ErrorResponse:
  type: object
  additionalProperties: false
  required:
    - error
  properties:
    error:
      $ref: "#/components/schemas/Error"
```

**Reason:** This makes the error object mandatory in every error response and keeps error responses consistent.





### Change 2: Clarify borrow-request input

**Location:** `paths./books/{bookId}/borrow-requests.post` (line 119)

The operation currently has no `requestBody`.

**Recommended action:** If the borrower is identified from the authenticated user's token and no other input is needed, keep the request body omitted and document this in the operation description. For example:

```yaml
description: >
  Creates a borrow request for the specified book.
  The borrower is identified from the authenticated user's token.
  No request body is required.
```

Only use this description if it matches the intended design. If the client must send additional information, define a request body instead.

**Reason:** Clear documentation helps API consumers understand how to submit a valid borrow request.



### Change 3: Add pagination examples

**Locations:**

* `paths./books.get.parameters`: `page` (line 47) and `pageSize` (line 53)
* `paths./books/{bookId}/loans.get.parameters`: `page` (line 255) and `pageSize` (line 261)
* `components.schemas.PageOfBooks` (line 433) and `components.schemas.PageOfLoans` (line 457)

**Recommended action:** Add realistic `example` values, for example `page: 1` and `pageSize: 20` for a first-page request. The range limits (`minimum` and `maximum`) are already defined.

**Reason:** Meaningful examples make the API easier to understand and help developers tell sample mock data apart from expected real behaviour. Examples alone do not enforce correct pagination logic.





### Change 4: Constrain the "mark as returned" response

**Location:** `paths./loans/{loanId}.patch`, `200` response (line 227)

**Recommended change:** State that a successful response always has `status: returned` and a non-null `returnedAt`:

```yaml
"200":
  description: Loan marked as returned
  content:
    application/json:
      schema:
        allOf:
          - $ref: "#/components/schemas/Loan"
          - required:
              - returnedAt
            properties:
              status:
                const: returned
              returnedAt:
                type: string
                format: date-time
```

For `PATCH /borrow-requests/{requestId}` (line 178), the response status depends on the decision sent, so it cannot be fixed with a single `const`. Instead, add to the description that the returned `status` matches the submitted `decision`.

**Reason:** Clients should be able to rely on a successful response confirming the requested change.





## 6\. Postman Evidence



In addition to the curl smoke tests, I sent one request to each of the seven operations in Postman, using a bearer token and the mock server on port 4010. Screenshots are in the `screeshorts` folder.





|Screenshot|Request|Result|
|-|-|-|
|Screenshot 2026-10-27 095958|`GET /books` with query parameters|200 OK|
|Screenshot 2026-10-27 100233|`POST /books` with The Hobbit body|201 Created|
|Screenshot 2026-10-27 100630|`GET /books/{bookId}`|200 OK|
|Screenshot 2026-10-27 100755|`POST /books/{bookId}/borrow-requests`|201 Created, `status: pending`|
|Screenshot 2026-10-27 100909|`PATCH /borrow-requests/{requestId}` with `decision: accepted`|200 OK|
|Screenshot 2026-10-27 101012|`PATCH /loans/{loanId}` with `status: returned`|200 OK, but response `status: overdue` (Finding 5)|
|Screenshot 2026-10-27 101113|`GET /books/{bookId}/loans`|200 OK|

All seven operations returned their documented success status. The same mock-data limitations from Finding 3 apply. For example, the borrowing-history response contained a loan with `status: out` and a `returnedAt` date earlier than its `borrowedAt` date, which a real API should never return.

