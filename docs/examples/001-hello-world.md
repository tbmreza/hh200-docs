# Hello World

This is the simplest example of an `hh200` test. It sends a single GET request to a URL and asserts that the response status is 200.

Create a file named `hello.hh`:

```
GET http://example.com/
HTTP 200
```

Run it:

```sh
hh200 hello.hh
```
