# `tempfile` module

!!! note "Not implemented"

    The `tempfile` module and the `TempStorage` SDK helper are not implemented.

`TemporaryDirectory`, `mkdtemp`, `gettempdir` and `gettempprefix` are available.
Use [temporary storage](../temporary-storage.md) to supply a filesystem and save it alongside interpreter state.

## Unsupported functions

`NamedTemporaryFile`, `TemporaryFile`, `mkstemp`, `SpooledTemporaryFile`, `mktemp`, `gettempdirb` and `gettempprefixb`
are absent from the module.
The module has no writable `tempdir` setting.
Use `dir=` to select another directory.

`mktemp()` has been deprecated in Python since 2.3.
It returns a pathname without creating or reserving a file.
Another caller can create that path before it is used, so Monty excludes the function.

### `NamedTemporaryFile`

`NamedTemporaryFile` is not implemented, even with a mode other than its default `w+b`.
That default creates a binary file for both reading and writing.
Monty's [file objects](open.md#mode-strings) reject read/write update modes.
Reads use a cached buffer, while writes replace or append whole-file content.
They do not support overwriting bytes at the current seek position and keeping the read buffer consistent afterward.
Create an ordinary file inside `TemporaryDirectory` instead.
Its file operations follow the existing `open()` and `pathlib` limitations.

### Descriptors and spooling

Monty file objects have no native descriptor, and `fileno()` is unavailable.
CPython's POSIX `TemporaryFile` can unlink a file and continue using its open descriptor.
Monty reopens files by path for host I/O, so unlinking the file would break later operations.
`mkstemp()` also requires a native descriptor as part of its result.

`SpooledTemporaryFile` is not implemented.
It requires an in-memory file that can switch to filesystem storage while preserving its contents,
position and mode.
Memory-backed storage for a temporary directory does not provide that transition for individual file objects.

## Paths and directory selection

`gettempdir()` returns `/tmp` on every host, even when no storage is attached there.
`gettempprefix()` returns `tmp`.
Monty does not inspect `TMPDIR`, `TEMP`, `TMP`, host directories or the working directory to choose temporary storage.
CPython searches those environment settings and platform directories, tests which directory is writable, and caches
the result in `tempfile.tempdir`.
Assigning `tempfile.tempdir` overrides the default for later calls throughout the process.
Monty keeps the default fixed and accepts per-call `dir=` arguments instead.
Importing the module does not grant filesystem access.
Creating a directory without an accessible filesystem raises `PermissionError`.

Temporary directory creation and cleanup support only the SDK's `TempStorage` filesystem.
`dir=` must resolve to `/tmp` or a directory below it in that storage; other locations raise `ValueError`.
A disk-backed mount at `/tmp` does not provide `tempfile` support.

Bytes paths raise `TypeError`.
Prefixes and suffixes containing `/` or a null byte raise `ValueError`.
Monty treats backslashes as ordinary filename characters under its POSIX path rules.
See [using temporary directories](../temporary-storage.md#using-temporary-directories) for signatures, defaults and
creation behavior.

## Independent random state

Temporary naming uses a private MT19937 stream initialized on its first candidate name.
It follows the session's `random_start` policy:

- A configured seed derives a separate naming stream without consuming the module-level `random` stream or the stream
    that initializes unseeded `Random()` instances.
- `system` seeds it from 2,496 bytes of worker OS entropy.
    Entropy failure ends the run with the same uncatchable error as unseeded `random` initialization.
- `call_host` requests 2,496 bytes through `os.urandom`.
    A reply of the wrong type or length raises `RuntimeError`, as for `random` initialization.

Calling `random.seed()` does not reset temporary naming, and creating temporary directories does not change the other
random streams.
The private state persists across imports, feeds and interpreter dumps.
An import or `gettempdir()` call does not initialize it.
Restoring a dump repeats candidate names from that point, so exclusive creation and collision retries still apply.
Names are reproducible under a session seed, but their exact sequence is not a compatibility guarantee across releases.
This stream is not a source of cryptographic secrets.
See [entropy configuration](../security.md#entropy) for the host policy and [random](random.md#entropy) for unanswered
entropy requests.

## Cleanup requires execution

Use a `with` block or an explicit `.cleanup()` call to remove a temporary directory.
The [cleanup options](../temporary-storage.md#using-temporary-directories) control removal on context exit and handling
of filesystem errors.

### Permission errors

CPython's cleanup can change file or directory permissions and retry removal after a permission error.
Monty does not repair permissions or retry deletion through `chmod`.
With `ignore_cleanup_errors=False`, a cleanup permission error propagates.
With `ignore_cleanup_errors=True`, cleanup ignores the error and can leave files behind.
It does not report those remaining files as deleted.

### No finalizer cleanup

CPython attempts cleanup when a `TemporaryDirectory` object becomes unreachable.
Monty has no finalizer cleanup.
Garbage collection and interpreter shutdown do not remove temporary directories.
Use a `with` block or call `cleanup()` explicitly.

A paused interpreter cannot run context-manager exit code or explicit cleanup, and a terminated worker cannot run it
later.
Cleanup cannot finish unless the interpreter resumes.
The host must dispose of storage whose interpreter never resumes.

An interpreter dump retains temporary directory objects and pending cleanup, but does not contain host filesystem
contents.
Save the matching files through [checkpoint and restore](../temporary-storage.md#checkpoint-and-restore).
Temporary files survive successive feeds when the host supplies the same storage.
Existing per-feed overlays discard writes and do not provide that lifetime.
