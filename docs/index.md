# Home

Welcome! Contents hopefully coming really soon.

## Conceptual overview

To motivate hh200, let's mentally execute an HTTP server test scenario three times.
The test scenario says that a sequence of `POST /login { "username": <Person>, "password": ... }`
and `GET /report` always return Person's report. We're also interested in learning
the number of parallel users after which the system-under-test starts to respond in >1 second.

First using Postman, where units of HTTP request are organized in "Collections". Unless your
team already have a specific workflow, you can do the following:

* In example-based manner, create requests for the POST and GET endpoints under a Collection.
* **(Alternative 1)** Utilize pre-request and post-response "Scripts" to encode the necessary chaining effects (i.e. parsing access token, passing the token as variable)
* **(Alternative 2)** Utilize "Flows". To my knowledge, at the time of writing, simulating parallel users is tricky even with this premium feature.

Second, let's use [hurl](hurl.dev). <sketch of how it can look like in hurl\>

Why not a general purpose language like python, is what I'm hoping the reader to think at this point, and the third method that we're going
to sketch. I've been developing an unreleased python package that allows the following program.

    import pp200

    ...

No one method is superior most of the time. We'll leave with the following premises (as they seem to me)
of each method:

1. Postman: "how can teams collaborate on API documentation"
2. Hurl: "requests in simple text format"
3. Python (or any GPL): "the test scripts are as important as the system-under-test itself; might as well use the same main language"

and hh200, like Hurl's, but more expressive. All in all, hh200 doesn't compete with Postman; agrees with
the above python approach where complexity is preferably hidden; aspires to be more modern Hurl.


## How-to guides

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

## Reference
<hackage project url\>
