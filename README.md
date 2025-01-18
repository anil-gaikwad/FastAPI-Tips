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
