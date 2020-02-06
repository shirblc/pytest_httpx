<h2 align="center">pytest fixture for HTTPX</h2>

<p align="center">
<a href="https://pypi.org/project/pytest-httpx/"><img alt="pypi version" src="https://img.shields.io/pypi/v/pytest_httpx"></a>
<a href="https://travis-ci.com/Colin-b/pytest_httpx"><img alt="Build status" src="https://api.travis-ci.com/Colin-b/pytest_httpx.svg?branch=master"></a>
<a href="https://travis-ci.com/Colin-b/pytest_httpx"><img alt="Coverage" src="https://img.shields.io/badge/coverage-100%25-brightgreen"></a>
<a href="https://github.com/psf/black"><img alt="Code style: black" src="https://img.shields.io/badge/code%20style-black-000000.svg"></a>
<a href="https://travis-ci.com/Colin-b/pytest_httpx"><img alt="Number of tests" src="https://img.shields.io/badge/tests-34 passed-blue"></a>
<a href="https://pypi.org/project/pytest-httpx/"><img alt="Number of downloads" src="https://img.shields.io/pypi/dm/pytest_httpx"></a>
</p>

Notice: This module is still under development, versions prior to 1.0.0 are subject to breaking changes without notice.

Use `pytest_httpx.httpx_mock` [`pytest`](https://docs.pytest.org/en/latest/) fixture to mock [`httpx`](https://www.python-httpx.org) requests.

- [Add responses](#add-responses)
  - [JSON body](#add-json-response)
  - [Custom body](#reply-with-custom-body)
  - [Multipart body (files, ...)](#add-multipart-response)
  - [HTTP method](#add-non-get-response)
  - [HTTP status code](#add-non-200-response)
  - [HTTP headers](#reply-with-custom-headers)
  - [HTTP/2.0](#add-http/2.0-response)
  - [Callback and exception throwing](#callback)
- [Check requests](#check-sent-requests)

## Add responses

You can register responses for both sync and async `httpx` requests.

```python
import pytest
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_something(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url")

    with httpx.Client() as client:
        response = client.get("http://test_url")


@pytest.mark.asyncio
async def test_something_async(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url")

    async with httpx.AsyncClient() as client:
        response = await client.get("http://test_url")
```

In case more than one request is sent to the same URL, the responses will be sent in the registration order.

First response will be sent as response of the first request and so on.

If the number of responses is lower than the number of requests on an URL, the last response will be used to reply to all subsequent requests on this URL.

If all responses are not sent back during test execution, the test case will fail at teardown.

Default response is a 200 (OK) without any body for a GET request on the provided URL using HTTP/1.1 protocol version.

Default matching is performed on the full URL, query parameters included.

### Add JSON response

Use `json` parameter to add a JSON response using python values.

```python
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_something(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", json=[{"key1": "value1", "key2": "value2"}])

    with httpx.Client() as client:
        assert client.get("http://test_url").json() == [{"key1": "value1", "key2": "value2"}]
    
```

### Reply with custom body

Use `data` parameter to reply with a custom body by providing bytes or UTF-8 encoded string.

```python
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_str_body(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", data="This is my UTF-8 content")

    with httpx.Client() as client:
        assert client.get("http://test_url").text == "This is my UTF-8 content"


def test_bytes_body(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", data=b"This is my bytes content")

    with httpx.Client() as client:
        assert client.get("http://test_url").content == b"This is my bytes content"
    
```

### Add multipart response

Use `data` parameter as a dictionary or `files` parameter (or both) to send multipart response.

You can specify `boundary` parameter to specify the multipart boundary to use.

```python
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_multipart_body(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", data={"key1": "value1"}, files={"file1": "content of file 1"}, boundary=b"2256d3a36d2a61a1eba35a22bee5c74a")

    with httpx.Client() as client:
        assert client.get("http://test_url").text == '''--2256d3a36d2a61a1eba35a22bee5c74a\r
Content-Disposition: form-data; name="key1"\r
\r
value1\r
--2256d3a36d2a61a1eba35a22bee5c74a\r
Content-Disposition: form-data; name="file1"; filename="upload"\r
Content-Type: application/octet-stream\r
\r
content of file 1\r
--2256d3a36d2a61a1eba35a22bee5c74a--\r
'''
    
```

### Add non GET response

Use `method` parameter to specify the HTTP method (POST, PUT, DELETE, PATCH, HEAD) to reply to on provided URL.

```python
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_post(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", method="POST")

    with httpx.Client() as client:
        response = client.post("http://test_url")


def test_put(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", method="PUT")

    with httpx.Client() as client:
        response = client.put("http://test_url")


def test_delete(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", method="DELETE")

    with httpx.Client() as client:
        response = client.delete("http://test_url")


def test_patch(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", method="PATCH")

    with httpx.Client() as client:
        response = client.patch("http://test_url")


def test_head(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", method="HEAD")

    with httpx.Client() as client:
        response = client.head("http://test_url")
    
```

### Add non 200 response

Use `status_code` parameter to specify the HTTP status code of the response.

```python
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_something(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", status_code=404)

    with httpx.Client() as client:
        assert client.get("http://test_url").status_code == 404

```

### Reply with custom headers

Use `headers` parameter to specify the extra headers of the response.

```python
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_something(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", headers={"X-Header1": "Test value"})

    with httpx.Client() as client:
        assert client.get("http://test_url").headers["x-header1"] == "Test value"

```

### Add HTTP/2.0 response

Use `http_version` parameter to specify the HTTP protocol version of the response.

```python
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_something(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url", http_version="HTTP/2.0")

    with httpx.Client() as client:
        assert client.get("http://test_url").http_version == "HTTP/2.0"

```

### Callback

You can perform custom manipulation upon request reception by registering callbacks.

Callback should expect at least two parameters:
 * request: The received request.
 * timeout: The timeout linked to the request.

Callback should return a httpx.Response instance.

```python
from typing import Optional

import httpx
from httpx import content_streams
from pytest_httpx import httpx_mock, HTTPXMock


def test_something(httpx_mock: HTTPXMock):
    def custom_response(request: httpx.Request, timeout: Optional[httpx.Timeout]) -> httpx.Response:
        return httpx.Response(
            status_code=200,
            http_version="HTTP/1.1",
            headers=[],
            stream=content_streams.JSONStream({"url": str(request.url)}),
            request=request,
        )

    httpx_mock.add_callback(custom_response, "http://test_url")

    with httpx.Client() as client:
        response = client.get("http://test_url")
        assert response.json() == {"url": "http://test_url"}

```

You can simulate httpx exception throwing by raising an exception in your callback.

This can be useful if you want to assert that your code handles httpx exceptions properly.

```python
from typing import Optional

import httpx
import pytest
from pytest_httpx import httpx_mock, HTTPXMock


def test_something(httpx_mock: HTTPXMock):
    def raise_timeout(request: httpx.Request, timeout: Optional[httpx.Timeout]) -> httpx.Response:
        raise httpx.exceptions.TimeoutException()

    httpx_mock.add_callback(raise_timeout, "http://test_url")
    
    with httpx.Client() as client:
        with pytest.raises(httpx.exceptions.TimeoutException):
            client.get("http://test_url")

```

In case more than one request is sent to the same URL, the callbacks will be called in the registration order.

First callback will be called upon reception of the first request and so on.

If the number of callbacks is lower than the number of requests on an URL, the last callback will be called for each subsequent requests on this URL.

If all callbacks are not executed during test execution, the test case will fail at teardown.

Default callback is for a GET request on the provided URL.

Default matching is performed on the full URL, query parameters included.

## Check sent requests

```python
import httpx
from pytest_httpx import httpx_mock, HTTPXMock


def test_something(httpx_mock: HTTPXMock):
    httpx_mock.add_response("http://test_url")

    with httpx.Client() as client:
        response = client.get("http://test_url")

    request = httpx_mock.get_request("http://test_url")
```

A request can only be retrieved once per test case. 

Calling order is preserved, so in case more than one request is sent to the same URL, the first one will be returned first.
