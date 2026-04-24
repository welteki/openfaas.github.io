---
title: "Automate OpenFaaS from Python with the new SDK"
description: "Introducing the OpenFaaS Python SDK, manage functions, namespaces, secrets, and build container images from Python."
date: 2026-04-24
author_staff_member: han
categories:
  - python
  - sdk
  - automation
  - integration
dark_background: true
image: "/images/2026-04-python-sdk/background.png"
hide_header_image: true
---

In this post we'll introduce the OpenFaaS Python SDK and show how to use it to manage functions, namespaces, secrets, stream logs, and build container images with the Function Builder.

## Introduction

We're announcing the [OpenFaaS Python SDK](https://github.com/openfaas/python-sdk), a new client library that gives you full, typed access to the [OpenFaaS REST API](https://docs.openfaas.com/reference/rest-api/) from Python. Whether you're writing automation scripts, building CI/CD pipelines, or integrating OpenFaaS into a larger platform, the SDK covers the complete API surface: functions, namespaces, secrets, logs, invocations, and the [Function Builder](https://docs.openfaas.com/openfaas-pro/builder/).

All request and response types use Pydantic v2 models, so you get IDE auto-complete, type checking, and clear error messages out of the box.

The SDK also has first-class support for [OpenFaaS IAM](https://docs.openfaas.com/openfaas-pro/iam/overview/), including token exchange, per-function scoped tokens, and a pluggable `TokenSource` protocol for any identity provider.

## Installation

```bash
pip install openfaas-sdk
```

The SDK requires Python 3.10+ and has two dependencies: [requests](https://requests.readthedocs.io) for HTTP communication with the gateway, and [pydantic](https://docs.pydantic.dev) v2 for validating and serialising all API request and response types.

## Getting started

Create a client with basic auth and start interacting with the gateway:

```python
from openfaas import Client, BasicAuth

with Client(
    gateway_url="https://gateway.example.com",
    auth=BasicAuth("admin", "password"),
) as client:
    # Get system info
    info = client.get_info()
    print(f"Gateway version: {info.version.release}")

    # List functions
    for fn in client.get_functions("openfaas-fn"):
        print(f"Function: {fn.name}  Replicas: {fn.replicas}")
```

The client is a context manager that closes the underlying `requests.Session` when done, but you can also call `client.close()` manually.

## Authentication

### Basic authentication

The examples above use `BasicAuth`. The password can also be read from a mounted secret file:

```python
from openfaas import BasicAuth

with open("/var/secrets/basic-auth-password") as f:
    password = f.read().strip()

auth = BasicAuth(username="admin", password=password)
```

### Zero trust with OpenFaaS IAM

When [OpenFaaS IAM](https://docs.openfaas.com/openfaas-pro/iam/overview/) is enabled, the SDK supports token-based authentication through `TokenAuth` with pluggable token sources. The SDK ships with two built-in token sources:

* `ServiceAccountTokenSource`, reads a Kubernetes projected service account token from disk. Ideal for workloads running inside the same cluster as OpenFaaS. See: [How to authenticate to the OpenFaaS API using Kubernetes JWT tokens](/blog/kubernetes-tokens-openfaas-api/).
* `ClientCredentialsTokenSource`, fetches tokens from an external identity provider using the OAuth 2.0 `client_credentials` grant. Suited for services running outside the cluster.

You can also implement the `TokenSource` protocol yourself to integrate any identity provider, see [Custom token sources](#custom-token-sources) below.

Here is an example using `ClientCredentialsTokenSource` with `TokenAuth`. `TokenAuth` exchanges the id_token from your provider for an OpenFaaS gateway access token via the gateway's `/oauth/token` endpoint:

```python
from openfaas import Client, TokenAuth, ClientCredentialsTokenSource

ts = ClientCredentialsTokenSource(
    client_id="my-app",
    client_secret="secret",
    token_url="https://idp.example.com/oauth/token",
    scope="openid",
)
auth = TokenAuth(
    token_url="https://gateway.example.com/oauth/token",
    token_source=ts,
)

with Client("https://gateway.example.com", auth=auth) as client:
    functions = client.get_functions("openfaas-fn")
```

The OAuth 2.0 `client_credentials` grant is designed for server-to-server authentication where no user interaction is involved. Most identity providers support it, including Auth0, Okta, Keycloak, Microsoft Entra ID, and Google Cloud. You register an application (or service account) with the provider and receive a client ID and secret that can be exchanged for an OIDC token. `ClientCredentialsTokenSource` takes four parameters:

* `client_id` and `client_secret`, the credentials issued by your identity provider when you registered the application.
* `token_url`, the token endpoint of your identity provider (e.g. `https://idp.example.com/oauth/token`).
* `scope`, the OAuth scope to request. `openid` is required so the provider returns an OIDC id_token that OpenFaaS can verify.

The id_token from your identity provider and the OpenFaaS gateway access token are both cached internally and refreshed automatically before they expire.

### Custom token sources

If the built-in token sources don't fit your setup, you can implement your own. The `TokenSource` protocol requires a single method, `token() -> str`, that returns a raw OIDC JWT. The implementation can contain any logic you need, for example reading a token from a file, calling an external API, or signing a JWT locally:

```python
class FileTokenSource:
    """Read a token from a file on each call."""

    def __init__(self, path: str) -> None:
        self._path = path

    def token(self) -> str:
        with open(self._path) as f:
            return f.read().strip()
```

Wire it up with `TokenAuth` the same way as any other token source:

```python
ts = FileTokenSource("/var/run/secrets/oidc-token")
auth = TokenAuth(
    token_url="https://gateway.example.com/oauth/token",
    token_source=ts,
)
```

See the [SDK README](https://github.com/openfaas/python-sdk#authentication) and the [IAM documentation](https://docs.openfaas.com/openfaas-pro/iam/overview/) for details on all supported authentication strategies.

## Error handling

API errors are raised as typed exceptions, so you can handle them explicitly. To keep the examples in this post short, we omit error handling in the rest of the examples, but in practice you would wrap calls like this:

```python
from openfaas.exceptions import NotFoundError, UnauthorizedError, ForbiddenError, APIConnectionError

try:
    fn = client.get_function("env", "openfaas-fn")
except NotFoundError:
    print("Function does not exist")
except UnauthorizedError:
    print("Invalid credentials")
except ForbiddenError:
    print("Insufficient permissions")
except APIConnectionError:
    print("Could not reach the gateway")
```

## Functions

The SDK covers the full function lifecycle: deploy, update, and delete. To build function container images from source code, see the [Function Builder](#function-builder) section below.

```python
from openfaas import Client, BasicAuth
from openfaas.models import FunctionDeployment

with Client("https://gateway.example.com", auth=BasicAuth("admin", "password")) as client:

    # Deploy the printer function from the function store,
    # it pretty-prints incoming HTTP requests to the function logs.
    spec = FunctionDeployment(
        service="printer",
        image="ghcr.io/openfaas/printer:latest",
        namespace="openfaas-fn",
    )
    client.deploy(spec)

    # Update the function with autoscaling labels
    spec.labels = {
        "com.openfaas.scale.type": "rps",
        "com.openfaas.scale.target": "50",
        "com.openfaas.scale.max": "10",
        "com.openfaas.scale.zero": "true",
        "com.openfaas.scale.zero-duration": "10m",
    }
    client.update(spec)

    # Get function details
    fn = client.get_function("printer", "openfaas-fn")
    print(f"Replicas: {fn.replicas}  Image: {fn.image}")
```

After deploying a function, you can poll `get_function` to wait until replicas are available:

```python
import time

while True:
    fn = client.get_function("printer", "openfaas-fn")
    if fn.available_replicas >= 1:
        break
    time.sleep(1)

print("Function is ready")
```

To remove a function from the cluster:

```python
client.delete_function("printer", "openfaas-fn")
```

## Invoking functions

Once a function is deployed, you can call it through the SDK. The SDK provides two methods: `invoke_function` for synchronous calls and `invoke_function_async` for queued, asynchronous calls.

### Synchronous invocation

Synchronous invocations are the simplest way to integrate with a function. The caller will block until a result is returned from the function. These are ideal for ingesting data from a webhook, user-facing operations, or when a web-page is served from a function. We'll cover asynchronous invocations in the next section.

`invoke_function` calls the function directly and returns the raw [`requests.Response`](https://requests.readthedocs.io/en/latest/api/#requests.Response). You specify the HTTP method for each call:

```python
# POST with a string or bytes payload
resp = client.invoke_function("printer", namespace="openfaas-fn", method="POST", payload="hello from the SDK")
print(resp.status_code, resp.text)

# GET request
resp = client.invoke_function("printer", namespace="openfaas-fn", method="GET")
```

You can also pass extra headers and query parameters:

```python
resp = client.invoke_function(
    "printer",
    namespace="openfaas-fn",
    method="POST",
    payload="hello",
    headers={"Content-Type": "text/plain"},
    query_params={"verbose": "1"},
)
```

Unlike operations such as deploy or delete, function invocations do not raise exceptions on non-2xx status codes. The response is returned as-is, so you can handle application-level errors from your function directly:

```python
resp = client.invoke_function("process-image", namespace="openfaas-fn", method="POST", payload=image_data)

if resp.ok:
    print("Processed:", resp.text)
elif resp.status_code == 429:
    print("Rate limited, retry later")
else:
    print(f"Unexpected status: {resp.status_code} - {resp.text}")
```

### Asynchronous invocations

Asynchronous invocations are event-driven, get retried automatically and are best for long-running or fire and forget operations like converting files, ingesting data, and scheduled jobs.

`invoke_function_async` queues the invocation via the gateway's [async endpoint](https://docs.openfaas.com/reference/async/). The gateway returns `202 Accepted` immediately and the function is executed in the background via the queue-worker. Async invocations always use POST internally.

```python
# Fire-and-forget
client.invoke_function_async("process-order", payload=order_json)

# With a callback URL to receive the result
client.invoke_function_async(
    "process-order",
    payload=order_json,
    callback_url="https://api.example.com/hooks/order-complete",
)
```

Headers and query parameters can be passed the same way as with synchronous invocations.

## Namespaces

[Namespaces](https://docs.openfaas.com/reference/namespaces/) lets you organise functions into logical groups for tenants, teams, or projects. OpenFaaS namespaces map to Kubernetes namespaces, so policies like LimitRanges, network isolation, and dedicated nodepools can also be applied. The SDK supports the full CRUD lifecycle:

```python
from openfaas.models import FunctionNamespace

# List all namespaces
namespaces = client.get_namespaces()
print(namespaces)  # ["openfaas-fn", "staging"]

# Create a namespace
client.create_namespace(FunctionNamespace(name="staging", labels={"team": "backend"}))

# Get namespace details
ns = client.get_namespace("staging")
print(ns.labels)

# Update a namespace
client.update_namespace(FunctionNamespace(name="staging", annotations={"owner": "alice"}))

# Delete a namespace
client.delete_namespace("staging")
```

## Secrets

[Secrets](https://docs.openfaas.com/reference/secrets/) can be created, updated, listed, and deleted per namespace. The value is write-only, it is never returned by the API after creation:

```python
from openfaas.models import Secret

# Create a secret
client.create_secret(Secret(name="db-password", namespace="openfaas-fn", value="s3cr3t"))

# List secrets in a namespace
for s in client.get_secrets("openfaas-fn"):
    print(s.name)

# Update a secret
client.update_secret(Secret(name="db-password", namespace="openfaas-fn", value="n3w-s3cr3t"))

# Delete a secret
client.delete_secret("db-password", namespace="openfaas-fn")
```

## Logs

OpenFaaS provides [streaming access to function logs](https://docs.openfaas.com/cli/logs/) via the gateway API. `get_logs` returns a lazy iterator that streams log lines from the gateway. You can tail a fixed number of lines, follow logs in real time, or filter by time:

```python
# Get the last 100 lines
for msg in client.get_logs("printer", "openfaas-fn", tail=100):
    print(f"[{msg.timestamp}] {msg.instance}: {msg.text}")

# Follow (stream) logs as they arrive
for msg in client.get_logs("printer", "openfaas-fn", follow=True):
    print(msg.text)
```

Filter by time:

```python
from datetime import datetime, timezone

since = datetime(2026, 4, 1, tzinfo=timezone.utc)
for msg in client.get_logs("printer", namespace="openfaas-fn", since=since):
    print(msg.text)
```

## Function Builder

The [Function Builder API](https://docs.openfaas.com/openfaas-pro/builder/) can be used to publish new OCI images from source code. This component, when combined with the other APIs can be used to extend an existing platform with custom functionality and scripts from customers and staff.

The workflow has three steps:

1. Assemble a build context from an OpenFaaS template and your function handler
2. Pack the context into a tar archive along with the build configuration
3. Send the tar to the Function Builder API, which builds and pushes the container image

For a detailed overview of the Function Builder API, including how templates work, registry authentication, and multi-arch builds, see: [How to build functions from source code with the Function Builder API](/blog/how-to-build-via-api/).

### Assemble a build context

`create_build_context` prepares a Docker build context on disk from a template and a handler directory, then `make_tar` packages it into a tar archive that can be sent to the Function Builder API. Templates can be pulled with `faas-cli template store pull <lang>` or fetched from any other source.

```python
from openfaas.builder import create_build_context, BuildConfig, make_tar, FunctionBuilder

# 1. Assemble the build context from template + handler
context_path = create_build_context(
    function_name="hello-world",
    handler="./hello-world",       # directory containing your function code
    language="python3-http",
    template_dir="./template",     # directory containing pulled templates
    build_dir="./build",
)

# 2. Pack the context into a tar archive with the build configuration
config = BuildConfig(
    image="ttl.sh/hello-world:1h",
    platforms=["linux/amd64"],
)
make_tar("/tmp/req.tar", context_path, config)
```

### Build

Create a `FunctionBuilder` client with the URL of the builder and the HMAC secret. The `hmac_secret` must match the `payload-secret` configured in the builder deployment. You can obtain it with:

```bash
kubectl get secret -n openfaas payload-secret \
    -o jsonpath='{.data.payload-secret}' | base64 --decode
```

```python
with open("/var/secrets/payload-secret") as f:
    hmac_secret = f.read().strip()

builder = FunctionBuilder(
    "https://builder.example.com",
    hmac_secret=hmac_secret,
)
```

`build_stream()` yields `BuildResult` objects as lines arrive, so you can display log output in real time:

```python
for result in builder.build_stream("/tmp/req.tar"):
    for line in result.log:
        print(line)
    if result.status in ("success", "failed"):
        print(f"Final status: {result.status}")
        print(f"Image: {result.image}")
```

`build()` blocks until the builder returns a single complete result:

```python
result = builder.build("/tmp/req.tar")
print(result.status)   # "success" / "failed"
print(result.image)    # fully-qualified image name
for line in result.log:
    print(line)
```

## End-to-end example: source to URL in a custom namespace

So far we've looked at each part of the SDK in isolation. This section brings everything together in a single script that goes from source code to a running, authenticated function — creating the namespace and secret along the way.

The full source code for this example is available at [welteki/openfaas-python-sdk-example](https://github.com/welteki/openfaas-python-sdk-example).

### The greeter function

The function we'll build and deploy is a simple API key validator. It reads an API key from an OpenFaaS secret mounted at `/var/openfaas/secrets/api-key`, then checks every incoming request for an `Authorization: Bearer <token>` header. A matching token gets a `200` response; anything else gets a `401`.

The function is initialised with `faas-cli` using the `python3-http` template:

```bash
faas-cli template store pull python3-http
faas-cli new greeter --lang python3-http
```

The handler in `greeter/handler.py`:

```python
import json


def handle(event, context):
    secret_path = "/var/openfaas/secrets/api-key"
    with open(secret_path) as f:
        api_key = f.read().strip()

    auth_header = event.headers.get("Authorization", "")
    if not auth_header.startswith("Bearer "):
        return {
            "statusCode": 401,
            "body": json.dumps({"error": "Unauthorized"}),
            "headers": {"Content-Type": "application/json"},
        }

    token = auth_header[len("Bearer "):]
    if token != api_key:
        return {
            "statusCode": 401,
            "body": json.dumps({"error": "Unauthorized"}),
            "headers": {"Content-Type": "application/json"},
        }

    return {
        "statusCode": 200,
        "body": json.dumps({"message": "Hello from OpenFaaS!"}),
        "headers": {"Content-Type": "application/json"},
    }
```

### Create the namespace and secret

We create a `tenant1` namespace to isolate the function, then generate a random UUID as the API key and store it as an OpenFaaS secret in that namespace. API errors are raised as typed exceptions, so a single `try/except` block around the client session is enough to handle auth and connection failures in one place:

```python
import uuid
from openfaas import BasicAuth, Client
from openfaas.exceptions import APIConnectionError, ForbiddenError, UnauthorizedError
from openfaas.models import FunctionNamespace, Secret

try:
    with Client(gateway_url="https://gateway.example.com", auth=BasicAuth("admin", password)) as client:

        # Create a dedicated namespace for this tenant
        client.create_namespace(
            FunctionNamespace(
                name="tenant1",
                labels={"managed-by": "openfaas-python-sdk-example"},
            )
        )

        # Generate an API key and store it as a secret
        api_key = str(uuid.uuid4())
        client.create_secret(Secret(name="api-key", namespace="tenant1", value=api_key))

except UnauthorizedError:
    print("Error: unauthorized. Check your credentials.")
except ForbiddenError:
    print("Error: insufficient permissions.")
except APIConnectionError as e:
    print(f"Error: could not reach the gateway: {e}")
```

The secret value is write-only — once created it is never returned by the API. Only the function that has it mounted can read it.

### Build from source

With the namespace and secret in place, we build the container image from source using the Function Builder. `create_build_context` assembles the Docker build context from the template and handler directory, `make_tar` packs it into a tar archive, and `build_stream` sends it to the builder and streams the log output line by line:

```python
from openfaas.builder import BuildConfig, FunctionBuilder, create_build_context, make_tar

# Assemble the build context from the template and handler
context_path = create_build_context(
    function_name="greeter",
    handler="./greeter",
    language="python3-http",
    template_dir="./template",
    build_dir="./build",
)

# Pack the context into a tar with the build config
config = BuildConfig(image="ttl.sh/greeter:1h", platforms=["linux/amd64"])
make_tar("/tmp/greeter-build.tar", context_path, config)

# Stream the build logs from the Function Builder
with open("/var/secrets/payload-secret") as f:
    hmac_secret = f.read().strip()

builder = FunctionBuilder("https://builder.example.com", hmac_secret=hmac_secret)
for result in builder.build_stream("/tmp/greeter-build.tar"):
    for line in result.log:
        print(line)
    if result.status in ("success", "failed"):
        print(f"Build {result.status}: {result.image}")
```

### Deploy into the namespace

Once the image is pushed, deploy the function into the `tenant1` namespace and wire in the `api-key` secret so the function can read it at runtime:

```python
from openfaas.models import FunctionDeployment

spec = FunctionDeployment(
    service="greeter",
    image="ttl.sh/greeter:1h",
    namespace="tenant1",
    secrets=["api-key"],
)
client.deploy(spec)
```

### Invoke and verify

With the function running, call it with the API key as a Bearer token to get the `200` response:

```python
resp = client.invoke_function(
    "greeter",
    namespace="tenant1",
    method="GET",
    headers={"Authorization": f"Bearer {api_key}"},
)
print(resp.status_code, resp.text)
# 200 {"message": "Hello from OpenFaaS!"}
```

The complete script — including the ready-poll loop, log streaming, and cleanup — is in [`e2e.py`](https://github.com/welteki/openfaas-python-sdk-example/blob/main/e2e.py) in the example repository.

## Wrapping up

The [OpenFaaS Python SDK](https://github.com/openfaas/python-sdk) gives you typed, validated access to the OpenFaaS API from Python, including the Function Builder for building and pushing container images from source.

To get started:

* Install the SDK: `pip install openfaas-sdk`
* Read the [SDK README](https://github.com/openfaas/python-sdk) for the full API reference
* Learn more about [OpenFaaS IAM](https://docs.openfaas.com/openfaas-pro/iam/overview/)
* See the [Function Builder documentation](https://docs.openfaas.com/openfaas-pro/builder/)

The SDK is a good fit for automation scripts, CI/CD pipelines, and platform integrations. If you are building a multi-tenant platform where customers provide their own code, you can combine the Function Builder with the functions and namespaces APIs to go from source code to a running function in seconds. See also: [Build a Function Editor for Your Customer Dashboard](/blog/build-a-function-editor/).

If you have questions, suggestions, or want to report a bug, feel free to open an issue on the [openfaas/python-sdk](https://github.com/openfaas/python-sdk) repository. You can also [reach out to us](https://openfaas.com/pricing) to discuss your use case.
