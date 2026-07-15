Vendored upstream: 7-Zip 26.02
Source version markers:
- C/7zVersion.h
- DOC/readme.txt

This directory is intended to stay as close to upstream 7-Zip as possible.
Most files should match the upstream source snapshot. Project-specific changes
must stay small, auditable, and documented here.

Current local content in this tree falls into two groups:

1. Upstream source sync
- The tree now contains the upstream 7-Zip 26.02 source snapshot.
- Official language catalogs were added under Lang/ (Lang/en.ttt and
  Lang/*.txt).
- DOC/unRarLicense.txt is the upstream license text.
- Asm/arm64/LzmaDecOpt.S and DOC/unRarLicense.txt are synced directly from the
  same upstream 26.02 snapshot.

2. Local patches

- C/Alloc.h
- C/Alloc.c
  Keep upstream 26.02 large-page controls based on z7_LargePage_Set() and
  Z7_LARGE_PAGES_FLAG_* while preserving a small macOS-specific mach_vm
  superpage allocation path. On macOS, real superpage allocations are attempted
  only when large-page mode is enabled; failures follow upstream FAIL_STOP
  semantics or fall back to upstream aligned allocation behavior.

- CPP/7zip/UI/Common/Update.h
- CPP/7zip/UI/Common/Update.cpp
- CPP/7zip/UI/Common/DirItem.h
- CPP/7zip/UI/Common/EnumDirItems.cpp
  Restore AddPathPrefix in CUpdateOptions and pass it to EnumerateItems().
  This keeps update/add flows aligned with the caller-supplied path prefix
  instead of silently dropping it.

  Add direct input-item path mapping for the Qt/native backend. CUpdateOptions
  now carries parallel physical source paths and archive-relative destination
  paths, CDirItem can override its logical name with LogName, and CDirItems::
  GetLogPath() uses that override when present. Update.cpp maps scanned disk
  items onto the requested archive paths, preserves directory subtrees, removes
  duplicate mapped logical paths so later explicit inputs override earlier
  directory-scanned entries, and uses a separate archive-path censor when
  updating an existing archive. This matches the original 7-Zip update model of
  separating physical file names from archive item names and avoids a first-party
  staging tree for AddRequest::input_items.

- CPP/7zip/UI/Common/ArchiveExtractCallback.h
- CPP/7zip/UI/Common/ArchiveExtractCallback.cpp
  Extend ZoneBuf/ZoneMode handling to macOS. Zone.Identifier payloads are
  mapped to the com.apple.quarantine xattr on extracted files and on newly
  created directories. xattr write failures are treated as best-effort so
  extraction still succeeds on filesystems that do not support xattrs.

- CPP/7zip/UI/Common/WorkDir.cpp
  On macOS, honor the "removable only" work-directory mode by checking whether
  the target volume is removable or ejectable via CoreFoundation volume
  properties.

- CPP/Common/StringConvert.cpp
  Add iconv-based multibyte decoding on non-Windows builds, explicit handling
  for common Windows code pages, and a conservative macOS-only legacy-encoding
  fallback that requires a clean UTF-8 round trip before accepting the result.

- CPP/7zip/UI/Agent/Agent.h
- CPP/7zip/UI/Agent/Agent.cpp
- CPP/7zip/UI/Agent/AgentOut.cpp
- CPP/7zip/UI/Agent/ArchiveFolderOut.cpp
- CPP/7zip/UI/Agent/UpdateCallbackAgent.cpp
  Collect non-Windows correctness fixes needed by the Qt integration:
  Win-only alternate data stream checks are guarded on non-Windows platforms,
  device detection uses POSIX metadata where needed, read-only checks use a
  portable sentinel, synthetic folder creation uses portable helpers, read-only
  directory cleanup uses SetFileAttrib_PosixHighDetect(), and update progress
  callbacks use the correct UInt64 type.

- CPP/7zip/Archive/ApfsHandler.cpp
  Preserve full-width APFS inode and parent-inode identifiers by exposing them
  as UInt64 properties instead of truncating them to UInt32. This keeps inode
  identity stable for metadata consumers and hard-link restoration.

- CPP/7zip/Archive/DmgHandler.cpp
  Prefer the canonical UTF-8 CFName emitted by modern DiskImages when naming
  partitions, while retaining the legacy Name field as a fallback for older
  images. This avoids presenting MacRoman bytes that were mis-decoded as UTF-8.

- CPP/7zip/Archive/HfsHandler.cpp
  Restore HFS+ filesystem semantics needed by native extraction: resolve file
  hard-link records to their shared private inode payload, expose link identity
  through kpidINode, hide the private hard-link metadata trees, and surface the
  public com.apple.FinderInfo value as an alternate stream. Malformed or
  ambiguous private inode mappings remain visible through the handler's header
  error state instead of silently selecting duplicate payloads.

- CPP/7zip/Archive/SquashfsHandler.cpp
  Expose each SquashFS node identifier through kpidINode and advertise inode
  availability at archive scope so extraction can preserve hard-link identity.

- CPP/7zip/Archive/XarHandler.cpp
  Decode XAR file names whose XML name element declares base64 encoding, using
  the existing strict base64 decoder. Reject malformed input, embedded NULs,
  invalid UTF-8, and unknown encodings instead of accepting unsafe path data.

Patch intent
- Keep the vendored tree close to upstream.
- Limit local deltas to macOS support and cross-platform correctness required
  by this project.
- Prefer removing a local patch when upstream gains an equivalent fix.

When refreshing upstream
- Replace this tree with a clean upstream 7-Zip 26.02-or-newer source
  snapshot.
- Keep official assets such as Lang/* and DOC/unRarLicense.txt in sync with the
  same snapshot.
- Reapply only the local patches documented above.
- Update this file if the local delta changes.
