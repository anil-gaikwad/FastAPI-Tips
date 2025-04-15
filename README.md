# FastAPI Tips

This repository contains trips and tricks for FastAPI. If you have any tip that you believe is useful, feel free
to open an issue or a pull request.


> [!TIP]
    Remember to **watch this repository** to receive notifications about new tips.

## 1. Install `uvloop` and `httptools`

By default, [Uvicorn][uvicorn] doesn't comes with `uvloop` and `httptools` which are faster than the default
asyncio event loop and HTTP parser. You can install them using the following command:

```bash
pip install uvloop httptools
```

[Uvicorn][uvicorn] will automatically use them if they are installed in your environment.

> [!WARNING]
> `uvloop` can't be installed on Windows. If you use Windows locally, but Linux on production, you can use
> an [environment marker](https://peps.python.org/pep-0496/) to not install `uvloop` on Windows
> e.g. `uvloop; sys_platform != 'win32'`.

## 2. Be careful with non-async functions

There's a performance penalty when you use non-async functions in FastAPI. So, always prefer to use async functions.
The penalty comes from the fact that FastAPI will call [`run_in_threadpool`][run_in_threadpool], which will run the
function using a thread pool.

> [!NOTE]
    Internally, [`run_in_threadpool`][run_in_threadpool] will use [`anyio.to_thread.run_sync`][run_sync] to run the
    function in a thread pool.

> [!TIP]
    There are only 40 threads available in the thread pool. If you use all of them, your application will be blocked.


To change the number of threads available, you can use the following code:

```py
import anyio
from contextlib import asynccontextmanager
from typing import Iterator

from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI) -> Iterator[None]:
    limiter = anyio.to_thread.current_default_thread_limiter()
    limiter.total_tokens = 100
    yield

app = FastAPI(lifespan=lifespan)
```

You can read more about it on [AnyIO's documentation][increase-threadpool].


## 3. Use `async for` instead of `while True` on WebSocket

Most of the examples you will find on the internet use `while True` to read messages from the WebSocket.

I believe the uglier notation is used mainly because the Starlette documentation didn't show the `async for` notation for a long time.

Instead of using the `while True`:

```py
from fastapi import FastAPI
from starlette.websockets import WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket) -> None:
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Message text was: {data}")
```

You can use the `async for` notation:

```py
from fastapi import FastAPI
from starlette.websockets import WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket) -> None:
    await websocket.accept()
    async for data in websocket.iter_text():
        await websocket.send_text(f"Message text was: {data}")
```

You can read more about it on the [Starlette documentation][websockets-iter-data].


## 4. Ignore the `WebSocketDisconnect` exception

If you are using the `while True` notation, you will need to catch the `WebSocketDisconnect`.
The `async for` notation will catch it for you.

```py
from fastapi import FastAPI
from starlette.websockets import WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket) -> None:
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Message text was: {data}")
    except WebSocketDisconnect:
        pass
```

If you need to release resources when the WebSocket is disconnected, you can use that exception to do it.

If you are using an older FastAPI version, only the `receive` methods will raise the `WebSocketDisconnect` exception.
The `send` methods will not raise it. In the latest versions, all methods will raise it.
In that case, you'll need to add the `send` methods inside the `try` block.


## 6. Use Lifespan State instead of `app.state`

Since not long ago, FastAPI supports the [lifespan state], which defines a standard way to manage objects that need to be created at
startup, and need to be used in the request-response cycle.

The `app.state` is not recommended to be used anymore. You should use the [lifespan state] instead.

Using the `app.state`, you'd do something like this:

```py
from contextlib import asynccontextmanager
from typing import AsyncIterator

from fastapi import FastAPI, Request
from httpx import AsyncClient


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    async with AsyncClient(app=app) as client:
        app.state.client = client
        yield


app = FastAPI(lifespan=lifespan)


@app.get("/")
async def read_root(request: Request):
    client = request.app.state.client
    response = await client.get("/")
    return response.json()
```

Using the lifespan state, you'd do something like this:


