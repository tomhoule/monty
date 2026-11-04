# Temporary files with `tempfile`

!!! note "Not implemented"

    The `tempfile` module is not implemented.
    The mounts and interpreter snapshot APIs used below already exist.

Python's `tempfile` module creates scratch files and directories in storage supplied by the host.
Use `TemporaryDirectory` to remove a directory when a `with` block exits, or `mkdtemp` to keep it until you delete it.
Use `NamedTemporaryFile` with an explicit mode to create a file with a pathname and cleanup on close or context exit.
Files inside temporary directories work with `open()` and `pathlib`.
Anonymous files and spooling are [not supported](limitations/tempfile.md#unsupported-functions).

## Supplying storage

Temporary files use the same [filesystem mounts](filesystem.md#mounts) as ordinary files.
There is no new SDK storage helper, storage dump format or automatic filesystem fallback.
No temporary storage is available by default.
Mount a private host directory at `/tmp`, or use `dir=` to select a directory in another writable mount.

```python test="skip"
import tempfile

from pydantic_monty import Monty, MountDir

code = """
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as directory:
    report = Path(directory) / 'report.txt'
    report.write_text('total: 42')
    text = report.read_text()

text
"""

with tempfile.TemporaryDirectory() as host_directory:
    with (
        MountDir(
            host_path=host_directory, virtual_path='/tmp', mode='read-write'
        ) as scratch,
        Monty() as pool,
    ):
        with pool.checkout() as session:
            print(session.feed_run(code, mount=scratch))
            #> total: 42
```

The outer `tempfile` is host Python's module and creates the backing directory.
The sandbox's `with` block removes its temporary directory and contents.
The host's `with` block removes the backing directory and anything left in it.
Mounting a private directory at `/tmp` does not expose the host's shared temporary directory.
Sandbox paths are virtual POSIX paths on every host platform.

Pass a list to `mount=` to combine scratch storage with application mounts, for example `mount=[data, scratch]`.
Mounts must have distinct virtual paths and disjoint host directories.
A separate scratch mount lets the host choose where temporary files are written without changing application storage.
A path such as `/tmp/report.txt` is a sandbox path; host tools need an explicit mapping to the backing storage.

Use `mode='read-write'` when files must survive successive snippets.
The default mount mode, `overlay`, discards writes when a feed ends.
A read-only mount rejects creation.
Without a mount or an OS callback that handles the operation, creation raises `PermissionError`.
The interpreter does not search for another writable location or create storage automatically.

Mount write limits and memory limits follow the existing [mount options](filesystem.md#options).
The write budget applies per feed; it is not a quota on all retained files.
The host manages disk capacity and eventual disposal of the backing directory.

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
Creation reserves each candidate exclusively, without first checking whether it exists.
Directory creation requests mode `0o700`; named-file creation requests `0o600`, subject to the backing filesystem.
It tries up to 20 candidate names, then raises `FileExistsError` if all already exist.
Other filesystem errors propagate immediately.
Creating temporary directories and files requires exclusive creation even though ordinary `open(..., 'x')` remains
unsupported.
See [path restrictions](limitations/tempfile.md#paths-and-directory-selection) and
[random state](limitations/tempfile.md#independent-random-state).

`TemporaryDirectory` has two additional cleanup options:

- `delete=False` keeps the directory when the `with` block exits.
    An explicit `.cleanup()` call still removes it.
- `ignore_cleanup_errors=True` ignores filesystem `OSError` exceptions during cleanup.
    It does not suppress resource limits or exceptions from the body of the `with` block.

Calling `.cleanup()` repeatedly succeeds if the directory is already absent.
Cleanup must respect mount confinement and must not follow symlinks outside the temporary directory.
See [cleanup limitations](limitations/tempfile.md#cleanup-requires-execution) for permission errors and paused execution.

## Using named temporary files

Pass an explicit `mode` to `NamedTemporaryFile`.
The supported modes are `w`, `wb`, `a` and `ab`, including equivalent spellings accepted by `open()`.
The default `w+b` raises `ValueError` because Monty does not support read/write update modes.
Use `w` for text or `wb` for bytes:

```python test="skip"
import tempfile

from pydantic_monty import Monty, MountDir

code = """
import tempfile
from pathlib import Path

with tempfile.NamedTemporaryFile(mode='w', delete=False) as report:
    report.write('total: 42')
    name = report.name

text = Path(name).read_text()
Path(name).unlink()
text
"""

with tempfile.TemporaryDirectory() as host_directory:
    with (
        MountDir(
            host_path=host_directory, virtual_path='/tmp', mode='read-write'
        ) as scratch,
        Monty() as pool,
    ):
        with pool.checkout() as session:
            print(session.feed_run(code, mount=scratch))
            #> total: 42
```

The file exposes its absolute virtual pathname as `.name` and uses the existing file I/O implementation.
It reserves a new name exclusively; selecting an append mode does not open an existing candidate.
`suffix`, `prefix` and `dir` follow the directory creation rules above.
Other file options follow the existing [open() restrictions](limitations/open.md#open-arguments).

With the default `delete=True, delete_on_close=True`, closing the file removes it.
With `delete=True, delete_on_close=False`, closing the file keeps it until context exit.
With `delete=False`, the caller removes the file and `delete_on_close` has no effect.
There is no finalizer cleanup.
See [named-file limitations](limitations/tempfile.md#namedtemporaryfile).

## Reusing files across snippets

A feed is one submitted snippet, such as `session.feed_run()` or `session.feed_start()`.
Supply the mount on every feed that needs the files.
With a read-write mount, files remain in the backing directory after a feed or session ends, until explicitly deleted.
Closing a mount does not delete the directory.
A host `TemporaryDirectory` context does delete it when the context exits.

`feed_start` retains the mounts throughout a suspended feed, so `resume_auto()` uses the same mounts.
An explicit `resume(...)` supplies the result directly and bypasses automatic filesystem handling.
Later feeds still require `mount=`.

## Checkpoint and restore

Interpreter dumps do not include mounted files.
There is no combined checkpoint API or filesystem export/import API in this feature.
To restore temporary files, retain the backing directory or save its contents using your host storage system.
Reattach the matching storage when restoring the interpreter.
Temporary files still needed by the program must remain available after restoration.

Capture interpreter state and files while the session is idle or suspended, with no host callback or filesystem operation
running.
Do not resume execution or modify the files between captures.
Save them as a matching pair so a failed save cannot pair new interpreter state with old files.
An interpreter dump alone does not make disk files durable or copy them to another host.

Between feeds, use `session.dump()` and `session.load_session(data)`, then supply the mounts on the next feed.
For suspended execution, use `snapshot.dump()` and restore with `session.load_snapshot(data, mount=scratch)`.
The restored snapshot's `resume_auto()` uses that mount.
See [interpreter snapshots](snapshots.md#storing-and-restoring) for the complete restore API and snapshot trust contract.
Combined interpreter and filesystem checkpointing is deferred.

The host owns the backing storage's lifetime independently of the interpreter.
Do not let a host `TemporaryDirectory` context exit if you intend to restore from its files later, unless you have saved
them elsewhere first.
A paused or terminated interpreter cannot run Python cleanup; the host must dispose of abandoned storage.
