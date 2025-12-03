# Download File

`hh200` can download files and save them to disk.

## Example

This example demonstrates how to request a file and save it using the `[Options]` section.

```
GET http://example.com/report?format=xls
[Options]
output: report.xls
HTTP 200
```
