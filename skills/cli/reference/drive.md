# Drive

Files and folders. Tree, search, get, download, upload, mkdir.

## Discover

```bash
copera drive tree --json                       # workspace root
copera drive tree --parent <folderId> --json   # subtree
copera drive tree --depth 1-10                 # default 3 (server-side)
```

JSON return shape:

```json
{
  "nodes": [
    { "id": "…", "name": "…", "type": "folder", "children": [ … ] },
    { "id": "…", "name": "report.pdf", "type": "file", "size": 1234567, "mimeType": "application/pdf" }
  ],
  "truncated": false,
  "totalItems": 42
}
```

If `truncated: true`, the tree was clipped — drill into a specific folder with `--parent` to see deeper.

```bash
copera drive search "query" --json [--sort-by createdAt|updatedAt] [--sort-order asc|desc] [--limit 1-50]
```

Returns `{ "hits": [...], "totalHits": <int> }` — `hits` may be a subset of `totalHits`.

## Get metadata

```bash
copera drive get <fileId> --json
```

Works for files and folders. Folder responses have no `size` / `mimeType`.

## Download

```bash
copera drive download <fileId>             # writes to ./<original-filename>
copera drive download <fileId> -o <path>   # custom destination
copera drive download <fileId> --dest <path>
```

Refuses to download a folder (use `tree` then download files individually). Resolves a presigned CloudFront URL server-side, validates it, then streams. Path traversal in `--dest` (`..` components) is rejected.

In interactive terminals a progress bar is shown on stderr; suppress with `--quiet` or by piping.

## Upload

```bash
copera drive upload <localPath>                       # file or directory
copera drive upload <localPath> --parent <folderId>   # target folder
copera drive upload <localPath> --chunk-size <bytes> --concurrency <n>
```

- **Single file:** multipart upload via S3 presigned URLs (start → presign parts → upload parts in parallel → finalize).
- **Directory:** walks recursively, creates folders first (preserving structure), then uploads files. Empty files are skipped with a notice.

Uploading an empty single file is a usage error.

JSON output for directory upload:

```json
{
  "files": [...],
  "fileCount": 12,
  "folderCount": 3,
  "totalBytes": 4567890
}
```

`--chunk-size` and `--concurrency` defaults are tuned for typical use; only override when uploading very large files on bandwidth-constrained connections.

## Create folder

```bash
copera drive mkdir "Folder Name" [--parent <folderId>] --json
```

Without `--parent`, the folder lands at the workspace root.

## Path safety notes for agents

- `drive download` defaults to writing in the current working directory using the server-provided filename (sanitized: `..` and path separators stripped). Confirm the destination with the user before running on shared/production directories.
- `drive upload` of a directory can create many remote folders quickly — for batches over ~50 items, summarize the planned upload and confirm before running.
