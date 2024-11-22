# Llama Stack Client Python API library

[![PyPI version](https://img.shields.io/pypi/v/llama_stack_client.svg)](https://pypi.org/project/llama_stack_client/) [![PyPI - Downloads](https://img.shields.io/pypi/dm/llama-stack-client)](https://pypi.org/project/llama-stack-client/)
[![Discord](https://img.shields.io/discord/1257833999603335178)](https://discord.gg/llama-stack)

The Llama Stack Client Python library provides convenient access to the Llama Stack Client REST API from any Python 3.7+
application. The library includes type definitions for all request params and response fields,
and offers both synchronous and asynchronous clients powered by [httpx](https://github.com/encode/httpx).

It is generated with [Stainless](https://www.stainlessapi.com/).

## Documentation

For starting up a Llama Stack server, please checkout our guides in our [llama-stack](https://github.com/meta-llama/llama-stack/blob/main/docs/resources/llama-stack-spec.html) repo.

The REST API documentation can be found on our [llama-stack OpenAPI spec](https://github.com/meta-llama/llama-stack/blob/main/docs/resources/llama-stack-spec.html). The full API of this library can be found in [api.md](api.md).

You can find more example apps with client SDKs to talk with the Llama Stack server in our [llama-stack-apps](https://github.com/meta-llama/llama-stack-apps/tree/main) repo.

## Installation

```sh
pip install llama-stack-client
```

## Usage

The full API of this library can be found in [api.md](api.md). You may find basic client examples in our [llama-stack-apps](https://github.com/meta-llama/llama-stack-apps/tree/main) repo.

```python
from llama_stack_client import LlamaStackClient
from llama_stack_client.types import UserMessage

client = LlamaStackClient(
    base_url=f"http://{host}:{port}",
)

response = client.inference.chat_completion(
    messages=[
        UserMessage(
            content="hello world, write me a 2 sentence poem about the moon",
            role="user",
        ),
    ],
    model="meta-llama/Llama-3.2-3B-Instruct",
    stream=False,
)
print(response)
```

After installing the `llama-stack-client` package, you can also use the [`llama-stack-client` CLI](https://github.com/meta-llama/llama-stack/tree/main/llama-stack-client) to interact with the Llama Stack server.
```bash
llama-stack-client inference chat-completion --message "hello, what model are you"
```

```python
ChatCompletionResponse(
    completion_message=CompletionMessage(
        content="Hello! I'm an AI model, and I'm based on a large language model architecture. My knowledge cutoff is December 2023, which means I was trained on a dataset that was current up to that point in time.\n\nI don't have a specific model name, but I'm similar to other 
conversational AI models like LLaMA, Bard, or ChatGPT. My primary function is to understand and respond to human language, generating human-like text based on the input I receive.\n\nI'm designed to be helpful and informative, and I can assist with a wide range of topics and tasks, 
from answering questions and providing information to generating text and completing tasks. How can I help you today?",
        role='assistant',
        stop_reason='end_of_turn',
        tool_calls=[]
    ),
    logprobs=None
)
```

## Async usage

Simply import `AsyncLlamaStackClient` instead of `LlamaStackClient` and use `await` with each API call:

```python
import asyncio
from llama_stack_client import AsyncLlamaStackClient

client = AsyncLlamaStackClient(
    # defaults to "production".
    environment="sandbox",
)


async def main() -> None:
    session = await client.agents.sessions.create(
        agent_id="agent_id",
        session_name="session_name",
    )
    print(session.session_id)


asyncio.run(main())
```

Functionality between the synchronous and asynchronous clients is otherwise identical.

## Using types

Nested request parameters are [TypedDicts](https://docs.python.org/3/library/typing.html#typing.TypedDict). Responses are [Pydantic models](https://docs.pydantic.dev) which also provide helper methods for things like:

- Serializing back into JSON, `model.to_json()`
- Converting to a dictionary, `model.to_dict()`

Typed requests and responses provide autocomplete and documentation within your editor. If you would like to see type errors in VS Code to help catch bugs earlier, set `python.analysis.typeCheckingMode` to `basic`.

## Handling errors

When the library is unable to connect to the API (for example, due to network connection problems or a timeout), a subclass of `llama_stack_client.APIConnectionError` is raised.

When the API returns a non-success status code (that is, 4xx or 5xx
response), a subclass of `llama_stack_client.APIStatusError` is raised, containing `status_code` and `response` properties.

All errors inherit from `llama_stack_client.APIError`.

```python
import llama_stack_client
from llama_stack_client import LlamaStackClient

client = LlamaStackClient()

try:
    client.agents.sessions.create(
        agent_id="agent_id",
        session_name="session_name",
    )
except llama_stack_client.APIConnectionError as e:
    print("The server could not be reached")
    print(e.__cause__)  # an underlying Exception, likely raised within httpx.
except llama_stack_client.RateLimitError as e:
    print("A 429 status code was received; we should back off a bit.")
except llama_stack_client.APIStatusError as e:
    print("Another non-200-range status code was received")
    print(e.status_code)
    print(e.response)
```

Error codes are as followed:

| Status Code | Error Type                 |
| ----------- | -------------------------- |
| 400         | `BadRequestError`          |
| 401         | `AuthenticationError`      |
| 403         | `PermissionDeniedError`    |
| 404         | `NotFoundError`            |
| 422         | `UnprocessableEntityError` |
| 429         | `RateLimitError`           |
| >=500       | `InternalServerError`      |
| N/A         | `APIConnectionError`       |

### Retries

Certain errors are automatically retried 2 times by default, with a short exponential backoff.
Connection errors (for example, due to a network connectivity problem), 408 Request Timeout, 409 Conflict,
429 Rate Limit, and >=500 Internal errors are all retried by default.

You can use the `max_retries` option to configure or disable retry settings:

```python
from llama_stack_client import LlamaStackClient

# Configure the default for all requests:
client = LlamaStackClient(
    # default is 2
    max_retries=0,
)

# Or, configure per-request:
client.with_options(max_retries=5).agents.sessions.create(
    agent_id="agent_id",
    session_name="session_name",
)
```

### Timeouts

By default requests time out after 1 minute. You can configure this with a `timeout` option,
which accepts a float or an [`httpx.Timeout`](https://www.python-httpx.org/advanced/#fine-tuning-the-configuration) object:

```python
from llama_stack_client import LlamaStackClient

# Configure the default for all requests:
client = LlamaStackClient(
    # 20 seconds (default is 1 minute)
    timeout=20.0,
)

# More granular control:
client = LlamaStackClient(
    timeout=httpx.Timeout(60.0, read=5.0, write=10.0, connect=2.0),
)

# Override per-request:
client.with_options(timeout=5.0).agents.sessions.create(
    agent_id="agent_id",
    session_name="session_name",
)
```

On timeout, an `APITimeoutError` is thrown.

Note that requests that time out are [retried twice by default](#retries).

## Advanced

### Logging

We use the standard library [`logging`](https://docs.python.org/3/library/logging.html) module.

You can enable logging by setting the environment variable `LLAMA_STACK_CLIENT_LOG` to `debug`.

```shell
$ export LLAMA_STACK_CLIENT_LOG=debug
```

### How to tell whether `None` means `null` or missing

In an API response, a field may be explicitly `null`, or missing entirely; in either case, its value is `None` in this library. You can differentiate the two cases with `.model_fields_set`:

```py
if response.my_field is None:
  if 'my_field' not in response.model_fields_set:
    print('Got json like {}, without a "my_field" key present at all.')
  else:
    print('Got json like {"my_field": null}.')
```

### Accessing raw response data (e.g. headers)

The "raw" Response object can be accessed by prefixing `.with_raw_response.` to any HTTP method call, e.g.,

```py
from llama_stack_client import LlamaStackClient

client = LlamaStackClient()
response = client.agents.sessions.with_raw_response.create(
    agent_id="agent_id",
    session_name="session_name",
)
print(response.headers.get('X-My-Header'))

session = response.parse()  # get the object that `agents.sessions.create()` would have returned
print(session.session_id)
```

These methods return an [`APIResponse`](https://github.com/stainless-sdks/llama-stack-python/tree/main/src/llama_stack_client/_response.py) object.

The async client returns an [`AsyncAPIResponse`](https://github.com/stainless-sdks/llama-stack-python/tree/main/src/llama_stack_client/_response.py) with the same structure, the only difference being `await`able methods for reading the response content.

#### `.with_streaming_response`

The above interface eagerly reads the full response body when you make the request, which may not always be what you want.

To stream the response body, use `.with_streaming_response` instead, which requires a context manager and only reads the response body once you call `.read()`, `.text()`, `.json()`, `.iter_bytes()`, `.iter_text()`, `.iter_lines()` or `.parse()`. In the async client, these are async methods.

```python
with client.agents.sessions.with_streaming_response.create(
    agent_id="agent_id",
    session_name="session_name",
) as response:
    print(response.headers.get("X-My-Header"))

    for line in response.iter_lines():
        print(line)
```

The context manager is required so that the response will reliably be closed.

### Making custom/undocumented requests

This library is typed for convenient access to the documented API.

If you need to access undocumented endpoints, params, or response properties, the library can still be used.

#### Undocumented endpoints

To make requests to undocumented endpoints, you can make requests using `client.get`, `client.post`, and other
http verbs. Options on the client will be respected (such as retries) will be respected when making this
request.

```py
import httpx

response = client.post(
    "/foo",
    cast_to=httpx.Response,
    body={"my_param": True},
)

print(response.headers.get("x-foo"))
```

#### Undocumented request params

If you want to explicitly send an extra param, you can do so with the `extra_query`, `extra_body`, and `extra_headers` request
options.

#### Undocumented response properties

To access undocumented response properties, you can access the extra fields like `response.unknown_prop`. You
can also get all the extra fields on the Pydantic model as a dict with
[`response.model_extra`](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_extra).

### Configuring the HTTP client

You can directly override the [httpx client](https://www.python-httpx.org/api/#client) to customize it for your use case, including:

- Support for proxies
- Custom transports
- Additional [advanced](https://www.python-httpx.org/advanced/clients/) functionality

```python
from llama_stack_client import LlamaStackClient, DefaultHttpxClient

client = LlamaStackClient(
    # Or use the `LLAMA_STACK_CLIENT_BASE_URL` env var
    base_url="http://my.test.server.example.com:8083",
    http_client=DefaultHttpxClient(
        proxies="http://my.test.proxy.example.com",
        transport=httpx.HTTPTransport(local_address="0.0.0.0"),
    ),
)
```

You can also customize the client on a per-request basis by using `with_options()`:

```python
client.with_options(http_client=DefaultHttpxClient(...))
```

### Managing HTTP resources

By default the library closes underlying HTTP connections whenever the client is [garbage collected](https://docs.python.org/3/reference/datamodel.html#object.__del__). You can manually close the client using the `.close()` method if desired, or with a context manager that closes when exiting.

## Versioning

This package generally follows [SemVer](https://semver.org/spec/v2.0.0.html) conventions, though certain backwards-incompatible changes may be released as minor versions:

1. Changes that only affect static types, without breaking runtime behavior.
2. Changes to library internals which are technically public but not intended or documented for external use. _(Please open a GitHub issue to let us know if you are relying on such internals)_.
3. Changes that we do not expect to impact the vast majority of users in practice.

We take backwards-compatibility seriously and work hard to ensure you can rely on a smooth upgrade experience.

We are keen for your feedback; please open an [issue](https://www.github.com/stainless-sdks/llama-stack-python/issues) with questions, bugs, or suggestions.

### Determining the installed version

If you've upgraded to the latest version but aren't seeing any new features you were expecting then your python environment is likely still using an older version.

You can determine the version that is being used at runtime with:

```py
import llama_stack_client
print(llama_stack_client.__version__)
```

## Requirements

Python 3.7 or higher.
