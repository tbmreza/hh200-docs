# Basic Usage

This guide explains how to write and run basic tests with `hh200`.

## Writing a Test

hh200 tests are written in a simple text format. A test file consists of a sequence of HTTP requests, along with optional assertions and captures.

Create a file named `test.hh` with the following content:

```
POST http://staging.example.com/login
Content-Type: application/json
{
  "username": "bob",
  "password": "loremipsum"
}
HTTP 200
[Captures]
AUTH_TOKEN: jsonpath "$.token"

GET http://staging.example.com/status
Authorization: Bearer {{AUTH_TOKEN}}
HTTP 200
[Asserts]
jsonpath "$.username" == "bob"
duration < 1000
```

### Breakdown

- **Request**: The first part specifies the HTTP method (e.g., `POST`, `GET`) and the URL.
- **Headers**: Key-value pairs following the request line define HTTP headers.
- **Body**: The request body follows the headers.
- **Expected Response**: `HTTP 200` asserts that the response status code should be 200.
- **Captures**: The `[Captures]` section allows you to extract data from the response (e.g., using JSONPath) and store it in variables (like `AUTH_TOKEN`) for use in subsequent requests.
- **Asserts**: The `[Asserts]` section allows you to verify properties of the response, such as checking JSON fields or response time (`duration`).

## Running the Test

To run the test, use the `hh200` command followed by the path to your test file:

```sh
hh200 test.hh
```

This will execute the requests in order and report any failures.
