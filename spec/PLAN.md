# OPFS binding plan (frontend/opfs)

## Goal
- Provide a general-purpose OPFS (Origin Private File System) binding for MoonBit JS target.
- Keep API ergonomic for path-based usage while exposing handle-based APIs for full coverage.
- Remain independent from DuckDB-specific code.

## Non-goals (for first milestone)
- No browser-specific polyfills or fallback storage (e.g., IndexedDB).
- No Node.js fs support.
- No automatic permission prompts or UI integration.

## Reference
- MDN OPFS page (Origin Private File System).

## Public API design (draft)

### Types (opaque JS handles)
- `type OpfsDirHandle`
- `type OpfsFileHandle`
- `type OpfsSyncAccessHandle`
- `struct OpfsEntry { name : String, kind : String }` // kind: "file" | "directory"
- `struct OpfsFileInfo { size : Int, last_modified : Int64 }`

### Support / root
- `check_support(on_done~ : (Result[Bool, String]) -> Unit)`
- `get_root(on_done~ : (Result[OpfsDirHandle, String]) -> Unit)`

### Directory operations (handle-based)
- `get_directory_handle(dir : OpfsDirHandle, name : String, create? : Bool = false, on_done~ : (Result[OpfsDirHandle, String]) -> Unit)`
- `get_file_handle(dir : OpfsDirHandle, name : String, create? : Bool = false, on_done~ : (Result[OpfsFileHandle, String]) -> Unit)`
- `remove_entry(dir : OpfsDirHandle, name : String, recursive? : Bool = false, on_done~ : (Result[Unit, String]) -> Unit)`
- `list_entries(dir : OpfsDirHandle, on_done~ : (Result[Array[OpfsEntry], String]) -> Unit)`

### File operations (handle-based, async)
- `read_file(file : OpfsFileHandle, on_done~ : (Result[Bytes, String]) -> Unit)`
- `write_file(file : OpfsFileHandle, data : Bytes, on_done~ : (Result[Unit, String]) -> Unit)`
- `get_file_info(file : OpfsFileHandle, on_done~ : (Result[OpfsFileInfo, String]) -> Unit)`

### Sync access handle (worker-only)
- `open_sync_access_handle(file : OpfsFileHandle, on_done~ : (Result[OpfsSyncAccessHandle, String]) -> Unit)`
- `sync_get_size(h : OpfsSyncAccessHandle) -> Int`
- `sync_read(h : OpfsSyncAccessHandle, buffer : Bytes, offset? : Int = 0) -> Int`
- `sync_write(h : OpfsSyncAccessHandle, buffer : Bytes, offset? : Int = 0) -> Int`
- `sync_truncate(h : OpfsSyncAccessHandle, size : Int) -> Unit`
- `sync_flush(h : OpfsSyncAccessHandle) -> Unit`
- `sync_close(h : OpfsSyncAccessHandle) -> Unit`

### Path-based convenience
- `open_file(path : String, create? : Bool = false, on_done~ : (Result[OpfsFileHandle, String]) -> Unit)`
- `create_dir(path : String, on_done~ : (Result[OpfsDirHandle, String]) -> Unit)`
- `list_files(path : String, on_done~ : (Result[Array[OpfsEntry], String]) -> Unit)`
- `delete_path(path : String, recursive? : Bool = false, on_done~ : (Result[Unit, String]) -> Unit)`

## Implementation notes
- Use `navigator.storage.getDirectory()` as the root handle entry point.
- Implement path traversal by splitting `/` and walking `getDirectoryHandle`.
- Convert JS `File` -> `Bytes` via `arrayBuffer` + `Uint8Array`.
- Convert `Bytes` -> JS buffer for `write_file` using `Uint8Array`.
- `createWritable()` should be used for async writes; call `close()` to commit.
- `createSyncAccessHandle()` is only available in dedicated workers; document this.

## Files and structure (planned)
- `opfs/types.mbt` (opaque handle types + structs)
- `opfs/support.mbt` (check_support, get_root)
- `opfs/dir.mbt` (directory handle functions)
- `opfs/file_async.mbt` (read/write/info)
- `opfs/file_sync.mbt` (sync access handle APIs)
- `opfs/path.mbt` (path-based helpers)
- Remove sample `fib/sum` from `opfs.mbt` and split per file.

## Tests and docs
- README: add usage examples for path-based and handle-based APIs (`mbt nocheck`).
- Tests: minimal smoke tests guarded by runtime checks; avoid CI failures in non-browser environments.
- Run `moon check` and `moon info` to generate `pkg.generated.mbti` before publish.

## Publish checklist
- Update `moon.mod.json`: repository, description, keywords, version.
- Ensure public API is stable; generate `pkg.generated.mbti`.
- Verify JS target usage in README.
