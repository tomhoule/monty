# Temporary files with `tempfile`

!!! note "Not implemented"

    The `tempfile` module and the `TempStorage` SDK helper are not implemented.

Python's `tempfile` module creates scratch directories for files used during execution.
Use `TemporaryDirectory` to remove a directory when a `with` block exits, or `mkdtemp` to keep it until you delete it.
Files inside these directories work with `open()` and `pathlib`.
`NamedTemporaryFile` is not supported.
Create an ordinary file inside `TemporaryDirectory` instead.
See the [read/write-mode limitation](limitations/tempfile.md#namedtemporaryfile).
Anonymous files and spooling are also [not supported](limitations/tempfile.md#unsupported-functions).

The host supplies storage before running sandboxed code.
No temporary storage is available by default.
The Python SDK's `TempStorage` provides a private, in-memory filesystem at `/tmp`:

```python test="skip"
from pydantic_monty import Monty, TempStorage

code = """
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as directory:
    report = Path(directory) / 'report.txt'
    report.write_text('total: 42')
    text = report.read_text()

text
"""

with TempStorage() as storage, Monty() as pool:
    with pool.checkout() as session:
        print(session.feed_run(code, os=storage))
        #> total: 42
```

The sandbox's `with` block removes the temporary directory and its contents.
The host's `with TempStorage()` block closes the storage and releases any remaining files.

## Using temporary directories

`TemporaryDirectory(suffix=None, prefix=None, dir=None, ignore_cleanup_errors=False, *, delete=True)` creates a
directory and exposes its path as `.name`.
Entering its `with` block returns that path as a string.
Exiting the block removes the directory and its contents, including when the body raises an exception.

`mkdtemp(suffix=None, prefix=None, dir=None)` creates a directory and returns its path without arranging cleanup.
Both functions return absolute virtual paths, including when `dir` is relative.

| Argument | Default | Meaning                                                            |
| -------- | ------- | ------------------------------------------------------------------ |
| `suffix` | `None`  | String appended to the generated name; `None` means no suffix      |
| `prefix` | `None`  | String prepended to the generated name; `None` uses `tmp`          |
| `dir`    | `None`  | Parent directory as a string or `pathlib.Path`; `None` uses `/tmp` |

Each name contains eight random lowercase letters, digits or underscores between the prefix and suffix.
The host creates the directory exclusively with mode `0o700`, without first checking whether it exists.
Creation tries up to 20 candidate names, then raises `FileExistsError` if all already exist.
Other filesystem errors propagate immediately.
See [path restrictions](limitations/tempfile.md#paths-and-directory-selection) and
[random state](limitations/tempfile.md#independent-random-state).

`TemporaryDirectory` has two additional cleanup options:

- `delete=False` keeps the directory when the `with` block exits.
    An explicit `.cleanup()` call still removes it.
- `ignore_cleanup_errors=True` ignores filesystem `OSError` exceptions during cleanup.
    It does not suppress resource limits or exceptions from the body of the `with` block.

Calling `.cleanup()` repeatedly succeeds if the directory is already absent.
Cleanup stays within `TempStorage`, which contains no symlinks or nested mounts.
See [cleanup limitations](limitations/tempfile.md#cleanup-requires-execution) for permission errors and paused execution.

## Configuring storage

`TempStorage` takes keyword-only arguments:

| Argument               | Default      | Meaning                                                 |
| ---------------------- | ------------ | ------------------------------------------------------- |
| `max_total_file_bytes` | `50_000_000` | Maximum combined size of file contents, in bytes        |
| `max_entries`          | `10_000`     | Maximum number of files and directories below `/tmp`    |
| `fallback`             | `None`       | An `os` callback for operations outside this filesystem |

Limits are non-negative integers and cannot be disabled.
Wrong argument types raise `TypeError` at construction.
Negative limits raise `ValueError`.
An operation that exceeds either limit raises `OSError` in the sandbox before changing the filesystem.
Paths longer than 4,096 UTF-8 bytes, including `/tmp`, raise `OSError`.
Deleting files releases their quota.
The worker's memory limit is separate because these files occupy host memory.

The storage contains only directories and regular files.
It does not create host files, follow symlinks or expose the host's shared temporary directory.
Paths are virtual POSIX paths on every host platform.
Files have mode `0o600` and directories have mode `0o700`, regardless of a requested creation mode.
`stat()` reports zero timestamps.

Pass the storage as `os=storage`.
It handles filesystem calls under `/tmp`, including `open()` and `pathlib` operations.
Missing files under `/tmp` raise the corresponding filesystem error without consulting `fallback`.
Other calls go to `fallback`, or return `NOT_HANDLED` when no fallback is configured.
This includes environment queries and host entropy requests.
Storage dumps do not contain the fallback callback.

You can combine `TempStorage` with mounts outside `/tmp`.
A mount at `/tmp`, above it, or below it is rejected before execution so it cannot shadow the storage.
`tempfile` directory creation and cleanup require `TempStorage`.
Disk-backed mounts do not support these operations, including when mounted at `/tmp`.
Use [filesystem mounts](filesystem.md#mounts) for ordinary file I/O outside the temporary filesystem.
`TempStorage` has no disk backend.

## Reusing files across snippets

A feed is one submitted snippet, such as `session.feed_run()` or `session.feed_start()`.
Temporary files survive the end of a feed.
Pass the same storage to each feed that uses them:

```python test="skip"
from pydantic_monty import Monty, TempStorage

with TempStorage() as storage, Monty() as pool:
    with pool.checkout() as session:
        session.feed_run(
            "import tempfile\nfrom pathlib import Path\n"
            "directory = tempfile.mkdtemp()\n"
            "report = Path(directory) / 'report.txt'\n"
            "report.write_text('total: 42')",
            os=storage,
        )
        print(session.feed_run('report.read_text()', os=storage))
        #> total: 42
```

`feed_start` retains the handler throughout a suspended feed, so `resume_auto()` uses the same storage.
An explicit `resume(...)` supplies the result directly and bypasses the handler, as for other OS callbacks.
Storage does not attach itself to later feeds.
Omitting `os=storage` leaves filesystem calls to the other handlers supplied for that feed.

A `TempStorage` object can be attached to only one live session at a time.
Using it from another session raises `RuntimeError` until the first session closes.
Closing the session detaches the storage and keeps its files.
Existing overlay mounts discard writes when a feed ends.

## Saving files separately

`storage.dump()` returns opaque bytes containing the files and directories below `/tmp`.
`TempStorage.load(data, *, max_total_file_bytes=50_000_000, max_entries=10_000, fallback=None)` returns new storage.
Load enforces the limits you pass, regardless of the limits used when saving.
Every load creates an independent writable copy.

Storage dumps preserve paths, file bytes and directory structure, including empty directories.
They contain no host paths, callbacks or session attachment.
They use a versioned format without compression or pickle.
Dump and load reject payloads larger than 256 MiB with `ValueError`.
Load also rejects duplicate paths, traversal, paths outside `/tmp`, unsupported entry types and quota violations with
`ValueError` before exposing any files.

You can dump storage between feeds or while the interpreter is suspended, provided no host callback is running.
Dumping or closing storage during a running feed or callback raises `RuntimeError`.
A storage dump does not contain interpreter state.
The caller must pair that data with the matching interpreter dump.

## Checkpoint and restore

Interpreter state and temporary files have separate dump/load APIs.
There is no combined checkpoint operation.
Take both dumps while the session is idle or suspended, with no host callback or filesystem operation running.
Do not resume execution or modify the files between dumps.

```python test="skip"
from pydantic_monty import Monty, TempStorage

with TempStorage() as storage, Monty() as pool:
    with pool.checkout() as session:
        session.feed_run(
            "import tempfile\nfrom pathlib import Path\n"
            "directory = tempfile.mkdtemp()\n"
            "report = Path(directory) / 'report.txt'\n"
            "report.write_text('total: 42')",
            os=storage,
        )
        interpreter_state = session.dump()
        storage_state = storage.dump()

# The original interpreter and storage have closed.
# Save both dumps together to resume in another process.

with TempStorage.load(storage_state) as storage, Monty() as pool:
    with pool.checkout() as session:
        session.load_session(interpreter_state)
        print(session.feed_run('report.read_text()', os=storage))
        #> total: 42
```

For suspended execution, use `snapshot.dump()` instead of `session.dump()`.
Restore the storage first, then call `session.load_snapshot(interpreter_state, os=storage)` on a fresh session.
The restored snapshot's `resume_auto()` uses that storage.
Later feeds still require `os=storage`.
See [interpreter snapshots](snapshots.md#storing-and-restoring) for answering the paused call.

Use a `with` block for the loaded storage so it closes even if interpreter loading fails.
Restoration preserves virtual paths and creates an independent writable copy of the files.
Storage dumps contain no other mounts or files outside `/tmp`.
After restoring interpreter state, supply live host objects and callbacks again.
See [host objects and snapshots](host-objects.md#snapshots).
Neither dump captures external side effects outside the temporary filesystem.
Only load interpreter state from a trusted, compatible producer under the existing
[snapshot trust contract](snapshots.md#snapshot-trust).

Save the two dumps as a matching pair.
Use your storage system's transaction or atomic replacement mechanism so a failed save cannot pair new interpreter
state with old files.
The dump methods return bytes and do not perform durable writes.

## Closing storage

`storage.close()` releases all files and is idempotent.
Other calls using closed storage raise `RuntimeError`.
Closing a paused session's storage does not run Python cleanup.
The session can no longer access those files on resume.
Keep the storage open while a session still needs it, or save both dumps before closing it.
See [cleanup limitations](limitations/tempfile.md#cleanup-requires-execution).

The helper is part of the Python SDK.
Its `os` handler works with local sync and async sessions, and with remote sessions whose filesystem calls are
serviced by the client.
There is no JavaScript helper.
All execution paths save storage separately from interpreter state.
