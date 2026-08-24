# Stable-release data-safety fixes (2026-08-24) — DONE

All items implemented, reviewed, and tested (270/270 tests green ×2, clippy clean).

## Phase A — self-contained files
- [x] keys.rs: bootstrap/rotate only on NotFound; other read errors abort loudly
- [x] keys.rs: fsync temp + parent dir; 0600 at creation; `.maxio-keys.json.bak` before rotate; unique temp name
- [x] main.rs: SIGTERM graceful shutdown (verified live with kill -TERM)
- [x] crypto.rs: FrameDecryptor rejects EOF at frame boundary when plaintext remains

## Phase B — filesystem.rs
- [x] fsync data + sidecar before rename; parent-dir fsync after (sync_file / sync_parent_dir / write_file_atomic)
- [x] publish: atomic rename-over for flat files; EC dirs use named `.maxio-bak-<name>` backup (mtime-stamped) with housekeeping crash-restore
- [x] housekeeping: age-gated recursive temp sweep (24 h), backup restore (1 h), multipart staleness vetoed by recent part activity
- [x] sharded per-key Mutex + per-bucket RwLock; wired into put/delete/tagging/complete/version ops, get meta+open, delete_bucket, `.bucket.json` RMW (consolidated into update_bucket_meta)
- [x] is_versioned errors propagate (no more unwrap_or(false))
- [x] upload_part: fresh random nonce prefix per attempt (GCM nonce reuse fixed); temp+rename part + atomic sidecar; fsync
- [x] complete: InvalidPartOrder + duplicate rejection; per-part size+MD5 re-verification (verify_part_bytes); best-effort upload-dir cleanup after publish
- [x] versioned PUT: version snapshot from temp BEFORE publish; rollback_failed_versioned_publish on error
- [x] null-version preservation on overwrite/delete + restore via update_current_version; corrupt version sidecars skipped, not fatal
- [x] flaky test test_sse_s3_cross_version_ciphertext_swap_rejected fixed (accepts transport-level rejection)
- [x] stale EC/flat payload cleanup on format change; ensure_bucket_exists inside publish critical section

## Adversarial review (subagent) — all findings fixed
- [x] backup age gate: rename preserves mtime → touch_now() stamps backups at publish
- [x] failed versioned PUT after null-archive → rollback restores current
- [x] flat meta-rename-after-payload failure → loud tracing::error (unrecoverable by design; fs-failure only)
- [x] stale `.ec` backup restore guarded against shadowing newer flat object
- [x] null archive writes sidecar before moving data (crash-recoverable)

## Verification
- [x] cargo test --release: 272 passed / 0 failed
- [x] cargo clippy --all-targets --release: clean
- [x] Live smoke test with real `mc` client (2026-08-24): bucket ops, put/get,
      nested keys, 10MB + 70MB multipart round-trips (byte-identical), recursive
      listing, versioning + null-version protect/restore, delete markers,
      version deletion via raw SigV4 API, SSE-S3 (ciphertext on disk verified),
      keyring rotate (.bak written, old objects decrypt after restart),
      SIGTERM graceful drain, kill -9 mid-overwrite (committed object intact),
      restart persistence, rm/rb cleanup.
- [x] Smoke test found + fixed 2 bugs: empty `delimiter=` treated as set
      (broke `mc ls -r`); crash-leftover temps blocked DeleteBucket.
      Both have regression tests now.
- [ ] kill -9 soak under sustained load + power-loss simulation (dm-flakey):
      recommended before tagging stable
