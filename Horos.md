## Operations Checklist

> [!summary] Verification status (2026-04-06)
> **Confirmed working:** `FindObject execute:"Select"` (AS + RPC), `FindObject execute:"Open"` (AS + RPC), `DisplayStudy` (RPC — returns metadata, selects row), `modality ==` NSPredicate (AS), `patientID + date` NSPredicate (RPC, server.py), `PathToFrontDCM` (RPC — returns `currentDCMPath`), `System Events close w` (silent, no activation), `tell application "Horos" to activate` (AS), `open POSIX file` import + poll/click, `FindObject execute:"Delete"` (AS).
>
> **All close methods activate Horos.** Only `click button 1 of w` (AS System Events) and RPC `CloseAllWindows` actually close viewers — both activate Horos. `System Events close w` and `AXClose` do NOT close viewers. Pattern: close + `_restore_frontmost()` required.
>
> **Confirmed broken:** `System Events close w` / `AXClose` — do not close viewer, activate Horos. HTTP GET format — HTTP 500, POST only. `horos://closeallwindows` — activates Horos, does not close (URL scheme disabled).
>
> **Not confirmed:** `Close2DViewerWith*` (URL + RPC), `DownloadURL` (AS + RPC), `horos://opendir`, HTTP RPC `FindObject execute:"Delete"`, `DisplayStudy` open-viewer behavior (RPC).
>
> **Not yet tested (need PACS node):** `Retrieve`, `CMove`.

Three access layers tested per operation:
- **URL** — `horos://` URL scheme (requires Preferences → Listener → URL Scheme enabled; same-machine only; fire-and-forget, no response)
- **AS** — AppleScript `tell application "Horos" to invoke XMLRPC method ...` (no HTTP server needed; returns response)
- **RPC** — HTTP XMLRPC server (`xmlrpc.client` / curl; requires port enabled in Preferences → Listener)

---

### 1. Find study by acc no — select DB row

| Layer | Command | Status |
|---|---|---|
| URL | ❌ no accNo command in URL scheme | n/a |
| AS | `FindObject request:"accessionNumber == 'X'" execute:"Select"` | - [x] confirmed 2026-03-27 |
| AS | `DisplayStudy accessionNumber:"X"` | - [x] confirmed (selects row) |
| RPC | `FindObject` same predicate | - [x] confirmed (server.py) |
| RPC | `DisplayStudy accessionNumber:"X"` | - [x] confirmed 2026-04-06 — error:0, returns full metadata; selects row in DB browser |

### 2. Find study by acc no — open viewer

| Layer | Command | Status |
|---|---|---|
| URL | `horos://displaystudy?studyUID=X` | - [ ] not tested (needs studyUID, not accNo) |
| AS | `DisplayStudy accessionNumber:"X"` then `OpenViewerForSelected` | - [x] confirmed |
| AS | `FindObject request:"accessionNumber == 'X'" execute:"Open"` | - [x] confirmed 2026-04-06 — error:0, viewer opened |
| RPC | `FindObject execute:"Open"` | - [x] confirmed (server.py) |

### 3. Find study by chart no (± date ± modality) — select row / open viewer

NSPredicate field names: `patientID`, `date` (NSDate), `modality` (string, e.g. `"CT"`), `studyName`, `name` (patient name).

| Layer | Predicate | Status |
|---|---|---|
| URL | `horos://displaystudylistbypatientid?patientID=X` | - [ ] not tested — filters list but does not select a row |
| AS / RPC | `patientID == 'X'` | - [x] confirmed (server.py) |
| AS / RPC | `date >= CAST('Y-M-D 00:00:00 +0000', 'NSDate') AND date < CAST(...)` | - [x] confirmed (server.py) |
| AS / RPC | `patientID == 'X' AND modality == 'CT'` | - [x] confirmed 2026-04-06 — AS, returns matching CT studies |
| AS / RPC | `patientID == 'X' AND modality == 'CT' AND date >= CAST(...) AND date < CAST(...)` | - [ ] not tested (full compound) |

### 4. Close viewers

| Layer | Command | Status |
|---|---|---|
| URL | `horos://closeallwindows` | - [x] confirmed 2026-04-06 — activates Horos ❌, does NOT close viewers ❌ (URL scheme disabled; command ignored) |
| URL | `horos://close2dviewerwithstudyuid?studyUID=X` | - [ ] not tested (likely same — disabled URL scheme) |
| AS | `System Events close w` | - [x] confirmed 2026-04-06 — **does NOT close viewer ❌**, activates Horos ❌ — broken |
| AS | `System Events perform action "AXClose" of button 1 of w` | - [x] confirmed 2026-04-06 — **does NOT close viewer ❌** |
| AS | `System Events click button 1 of w` | - [x] confirmed 2026-04-06 — closes viewer ✅, activates Horos ❌ — use with `_restore_frontmost()` |
| RPC | `CloseAllWindows` | - [x] confirmed 2026-04-06 — closes viewer ✅, activates Horos ❌ |
| RPC | `Close2DViewerWithStudyUID uid:"X"` | - [ ] not tested |

> [!important] All working close methods activate Horos. Pattern: `click button 1 of w` (or RPC `CloseAllWindows`) + `_restore_frontmost()` — save caller's app before close, re-activate after. `close w` and `AXClose` do not close viewers at all.

### 5. Activate Horos (bring to front / show viewer)

| Layer | Command | Status |
|---|---|---|
| URL | any `horos://` command | - [x] confirmed 2026-04-06 — activates Horos as side effect |
| AS | `tell application "Horos" to activate` | - [x] confirmed (server.py) |
| RPC | no activate method — must use AS alongside RPC | n/a |

### 6. Delete study by acc no

#### DB-only delete (files on disk untouched)

| Layer | Command | Status |
|---|---|---|
| URL | ❌ no delete URL command | n/a |
| AS | `FindObject request:"accessionNumber == 'X'" execute:"Delete"` | - [x] confirmed 2026-03-27 |
| RPC | `FindObject execute:"Delete"` | - [ ] not tested separately — proposed: same curl as FindObject with `execute:"Delete"` — should work (same code path) |

> [!warning] Post-delete crash — re-anchor required
> After delete, call `FindObject execute:"Select"` on any remaining study; without it Horos crashes on next activation.

#### Delete real files (filesystem)

| Layer | Command | Status |
|---|---|---|
| Any | Horos has no method to delete files on disk. Locate path via `PathToFrontDCM` (RPC) or DB path, then `rm` from shell. | - [x] `PathToFrontDCM` confirmed 2026-04-06 — RPC POST returns `currentDCMPath` of frontmost DICOM file |

### 7. Import DICOM into Horos

#### By local path (symlink, no copy)

| Layer | Command | Status |
|---|---|---|
| AS | `tell application "Horos" to open POSIX file "/path"` (background) + poll + click "Copy Links" dialog | - [x] confirmed 2026-03-27 |
| URL | `horos://opendir?dir=/path` | - [ ] not tested — proposed: `open 'horos://opendir?dir=/path/to/dir'` — does it open the import dialog? |

#### By URL (remote DICOM / WADO)

| Layer | Command | Status |
|---|---|---|
| AS | `invoke XMLRPC method "DownloadURL" with parameters {{URL:"http://..."}}` | - [ ] not tested — proposed: serve a `.dcm` file locally (`python3 -m http.server`) and call `DownloadURL` |
| RPC | `DownloadURL URL:"http://..." Display:true` | - [ ] not tested |

---

### Network operations (DICOM Q/R, C-MOVE) — reference only, not yet tested

These require a PACS/DICOM node configured in Horos (Preferences → Locations).

| Operation | Method | Reference |
|---|---|---|
| Query (C-FIND) | RPC `Retrieve` with `filterKey`/`filterValue`; or AS `invoke XMLRPC method "Retrieve"` | OsiriX RIS integration docs |
| Retrieve (C-MOVE) | RPC / AS `CMove` — confirmed in binary; params: PACS node config | OsiriX RIS integration docs |
| Retrieve if not local | `Retrieve` with `retrieveOnlyIfNeeded:true` | OsiriX RIS integration docs |
| Retrieve then display | `Retrieve` with `then:"display"` | OsiriX RIS integration docs |

> All network operations: - [ ] not yet tested on this machine.
> Proposed test: configure a test DICOM node in Horos Preferences → Locations, then call `Retrieve serverName:"TestNode" filterKey:"AccessionNumber" filterValue:"X"` via curl.

---

## URL Scheme (`horos://` / `osirix://`)

> Enable in Horos: **Preferences → Listener → URL Scheme** (disabled by default)
> Binary warning when off: *"Horos URL scheme [horos:// , osirix://] is currently not activated!"*

Both `horos://` and `osirix://` are registered and interchangeable.

### Commands

| URL | Parameters | Description |
|-----|-----------|-------------|
| `horos://displaystudy` | `studyUID=` | Open study by Study Instance UID |
| `horos://displayseries` | `seriesUID=` | Open specific series |
| `horos://displaystudylistbypatientid` | `patientID=` | Filter DB browser by patient ID |
| `horos://displaystudylistbypatientname` | `patientName=` | Filter DB browser by patient name |
| `horos://close2dviewerwithstudyuid` | `studyUID=` | Close 2D viewer for a study |
| `horos://close2dviewerwithseriesuid` | `seriesUID=` | Close 2D viewer for a series |
| `horos://closeallwindows` | — | Close all viewer windows |
| `horos://opendb` | — | Open database browser |
| `horos://opendir` | `dir=` | Open a directory |

### Known query parameters

`studyUID`, `seriesUID`, `objectUID`, `patientID`, `patientName`, `accessionNumber`

### Example

```bash
open 'horos://displaystudy?studyUID=1.2.410.200010.886.1231050017.111503160373'
```

> [!warning] accessionNumber has no direct URL command
> The URL scheme has no `displaystudybyaccession` equivalent.
> Use **AppleScript** instead (confirmed working, v4.0.1):
>
> ```applescript
> tell application "Horos"
>   invoke XMLRPC method "DisplayStudy" with parameters {{accessionNumber:"11503160373"}}
>   OpenViewerForSelected
> end tell
> ```
>
> Shell one-liner:
> ```bash
> osascript -e 'tell application "Horos"
>   invoke XMLRPC method "DisplayStudy" with parameters {{accessionNumber:"ACC_NO"}}
>   OpenViewerForSelected
> end tell'
> ```

---

## AppleScript — `invoke XMLRPC method`

Access via AppleScript (no URL scheme needed, no HTTP server needed):

```applescript
tell application "Horos"
  set r to invoke XMLRPC method "MethodName" with parameters {{key:"value", key2:"value2"}}
end tell
```

### Access layers comparison

| Feature | `horos://` URL scheme | AppleScript XMLRPC | HTTP XMLRPC server |
|---|---|---|---|
| Requires enabling | Yes (Preferences → Listener) | No | Yes (set port in Preferences) |
| accessionNumber lookup | ❌ no direct command | ✅ via `DisplayStudy` | ✅ |
| SQL predicate query | ❌ | ✅ `FindObject` | ✅ |
| DICOM retrieve | ❌ | ✅ `Retrieve`, `CMove` | ✅ |
| Get open viewers list | ❌ | ✅ `GetDisplayed2DViewerSeries` | ✅ |

### Methods confirmed in Horos 4.0.1 binary (`XMLRPCInterface` class)

These 4 are the only ones natively wired to AppleScript `invoke XMLRPC method`:

| Method | Parameters | Returns | Notes |
|--------|-----------|---------|-------|
| `DisplayStudy` | `accessionNumber`, `patientID`, `studyInstanceUID`, `studyID` | study metadata record | **Confirmed working** — selects study in DB browser |
| `FindObject` | `request` (NSPredicate string), `table` (`Study`/`Series`/`Image`), `execute` (`Select`/`Open`/`Delete`/`Nothing`) | matching elements | **Confirmed working** (2026-03-27) — `Select` returns full metadata; `Delete` removes from DB only (files on disk untouched); post-delete crash fix required (see below) |
| `Retrieve` | PACS node config, optional `then`, `retrieveOnlyIfNeeded` | — | DICOM Q&R retrieve |
| `DownloadURL` | `URL`, `Display` (optional) | — | Download `.dcm`/`.zip`/`.osirixzip` into DB |

> [!note] Other methods exist but via HTTP XMLRPC only
> The binary contains selectors for `CloseAllWindows`, `GetDisplayed2DViewerSeries`, `GetDisplayed2DViewerStudies`, `SelectAlbum`, `PathToFrontDCM`, `CMove`, `KillOsiriX`, `OpenDB`, `SwitchToDefaultDBIfNeeded` — but these are **not** registered in `XMLRPCInterface` and are unreachable via AppleScript `invoke XMLRPC method`. They require the HTTP XMLRPC server to be enabled.

---

## HTTP XMLRPC Server

Enable: Horos → Preferences → Listener → set XMLRPC port (e.g. 8080)

Accepts **POST only** (standard XML-RPC envelope). GET requests return HTTP 500 — Horos tries to parse the request body as XML; an empty body causes `NSXMLDocument initWithData:options:error: nil argument`. The OsiriX docs claim GET support but it does not work in Horos 4.0.1.

```bash
curl -s -X POST http://localhost:3334/RPC2 \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?><methodCall>
    <methodName>CloseAllWindows</methodName>
    <params></params></methodCall>'
```

### Full method list

All methods confirmed present in Horos binary (`strings` on `Horos.app/Contents/MacOS/Horos`):

| Method | Parameters | Description | In Horos binary |
|--------|-----------|-------------|-----------------|
| `DisplayStudy` | `accessionNumber` / `patientID` / `studyInstanceUID` / `studyID` | Find + select study in DB | ✅ |
| `DisplaySeries` | `patientID`, `seriesInstanceUID` | Select specific series | ✅ |
| `DisplayStudyListByPatientID` | `patientID` | Filter DB list | ✅ |
| `DisplayStudyListByPatientName` | `patientName` | Filter DB list | ✅ |
| `FindObject` | `request`, `table`, `execute` | NSPredicate query on DB | ✅ (also aliased as `DBWindowFind`) |
| `Retrieve` | PACS node params, `then`, `retrieveOnlyIfNeeded` | DICOM Q&R retrieve | ✅ |
| `CMove` | PACS node params | DICOM C-MOVE | ✅ |
| `DownloadURL` | `URL`, `Display` | Import from URL | ✅ |
| `GetDisplayed2DViewerSeries` | — | Array of open series | ✅ |
| `GetDisplayed2DViewerStudies` | — | Array of open studies | ✅ |
| `CloseAllWindows` | — | Close all viewers | ✅ |
| `Close2DViewerWithStudyUID` | `uid` | Close viewer by study UID | ✅ |
| `Close2DViewerWithSeriesUID` | `uid` | Close viewer by series UID | ✅ |
| `SelectAlbum` | `name` | Select DB album by name | ✅ |
| `OpenDB` | `path` | Open/create database | ✅ |
| `SwitchToDefaultDBIfNeeded` | — | Switch to default database | ✅ |
| `PathToFrontDCM` | — | Path of currently displayed DICOM file | ✅ |
| `KillOsiriX` | — | Quit Horos | ✅ |

> [!note] OsiriX name: `DBWindowFind`
> OsiriX documentation calls this method `DBWindowFind`. In Horos both names are registered — `DBWindowFind` and `FindObject` are aliases in the binary. OsiriX docs also describe the `request` parameter as "SQL" but the actual query syntax is **NSPredicate** (not SQL).

### FindObject — correct syntax (NSPredicate)

```applescript
-- Find by accession number
tell application "Horos"
  invoke XMLRPC method "FindObject" with parameters {{request:"accessionNumber == '11503160373'", table:"Study", execute:"Select"}}
end tell

-- Find by patient ID
tell application "Horos"
  invoke XMLRPC method "FindObject" with parameters {{request:"patientID == '0002092064'", table:"Study", execute:"Open"}}
end tell
```

```bash
# Via HTTP XMLRPC
curl -s -X POST http://localhost:8080/RPC2 \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?><methodCall>
    <methodName>FindObject</methodName>
    <params><param><value><struct>
      <member><name>request</name><value><string>accessionNumber == '"'"'11503160373'"'"'</string></value></member>
      <member><name>table</name><value><string>Study</string></value></member>
      <member><name>execute</name><value><string>Select</string></value></member>
    </struct></value></param></params></methodCall>'
```

> [!success] `FindObject` confirmed working (2026-03-27)
> `execute:"Select"` — returns full study metadata, `error:0`. `execute:"Delete"` — DB-only delete, files on disk untouched, `error:0`; `error:-1` if accession not found.
> **Crash fix:** after delete, Horos crashes on activation if the deleted row stays selected. Fix: `sleep 2` → `activate` Horos → `sleep 0.5` → `execute:"Select"` on any remaining study.

### Horos XML-RPC Plugin (separate install)

Source: [horosproject/horosplugins — XML-RPC-Plugin](https://github.com/horosproject/horosplugins/blob/master/XML-RPC-Plugin/xmlrpcFilter.m)

If installed, adds 4 extra methods to the HTTP XMLRPC server:

| Method | Parameters | Description |
|--------|-----------|-------------|
| `updateDICOMNode` | `AETitle`, `Port`, `TransferSyntax` | Update a DICOM node config |
| `importFromURL` | `url` | Import DICOM from URL into DB |
| `exportSelectedToPath` | `path` | Export selected files to directory |
| `openSelectedWithTiling` | `rowsTiling`, `columnsTiling` | Open selection in tiled viewer layout |

> Not confirmed installed on this machine (`~/Library/Application Support/Horos/Plugins/` — no XML-RPC plugin found).

---

## Use Cases

### 1. Open study by accession number (confirmed ✅)

```bash
osascript -e 'tell application "Horos"
  invoke XMLRPC method "DisplayStudy" with parameters {{accessionNumber:"11503160373"}}
  OpenViewerForSelected
end tell'
```

Returns: `error:0, elements: name:林訓一, accessionNumber:11503160373, modality:MR, studyName:Rt Ankle Mri, patientID:0002092064, ...`

---

### 2. Import DICOM folder (confirmed ✅ 2026-03-27)

`Horos -l -i <dir>` silently fails. `SelectImageFile` also silently fails. What works: `open POSIX file` (background) + poll/click the dialog.

```bash
STUDY_DIR="/path/to/study folder"
osascript -e 'tell application "Horos" to activate'
sleep 0.3
osascript -e "tell application \"Horos\" to open POSIX file \"$STUDY_DIR\"" &
for i in $(seq 1 20); do
  sleep 0.3
  FOUND=$(osascript -e 'tell application "System Events" to tell process "Horos" to return (exists button "Copy Links" of window 1)' 2>/dev/null)
  if [ "$FOUND" = "true" ]; then
    sleep 0.5
    osascript -e 'tell application "System Events"
  tell process "Horos"
    set frontmost to true
    try
      click button "Copy Links" of window 1
    end try
    try
      click button "Copy Links" of sheet 1 of window 1
    end try
  end tell
end tell'
    break
  fi
done
```

> [!note] Why this dance?
> `open POSIX file` triggers a "Copy Files / Copy Links / Cancel" dialog but blocks until dismissed — running it synchronously prevents the poll loop from starting. Solution: run `open` in background via `&`, poll + click in a separate `osascript` process. Activate Horos first or clicks don't register. Wait ~0.5 s after dialog detected before clicking.

---

### 3. Delete study from DB without closing Horos (confirmed ✅ 2026-03-27)

`FindObject execute:"Delete"` works — DB-only delete, files on disk untouched. Horos stays open.

```bash
ACC_NO="11503160373"
ANCHOR_ACC="<any remaining study accNo>"

# Delete
osascript << EOF
tell application "Horos"
  invoke XMLRPC method "FindObject" with parameters ¬
    {{request:"accessionNumber == '$ACC_NO'", table:"Study", execute:"Delete"}}
end tell
EOF

# Crash fix: re-anchor UI selection
sleep 2
osascript -e 'tell application "Horos" to activate'
sleep 0.5
osascript << EOF
tell application "Horos"
  invoke XMLRPC method "FindObject" with parameters ¬
    {{request:"accessionNumber == '$ANCHOR_ACC'", table:"Study", execute:"Select"}}
end tell
EOF
```

> [!warning] Post-delete crash
> Without the re-anchor step, Horos crashes when you click its icon — the deleted row stays selected in the UI. The fix (sleep 2 → activate → sleep 0.5 → Select another study) was confirmed stable.

---

### 4. Download DICOM from URL into Horos DB

```applescript
tell application "Horos"
  DownloadURL from "http://dicomserver/wado?studyUID=1.2.3..."
end tell
```

Accepts `.dcm`, `.zip`, `.osirixzip`. Optional `Display` parameter controls auto-open.

---

### 5. Get currently open studies (viewer state)

Via HTTP XMLRPC server (needs port enabled in Preferences → Listener):

```bash
curl -s -X POST http://localhost:8080/RPC2 \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?><methodCall>
    <methodName>GetDisplayed2DViewerStudies</methodName>
    <params></params></methodCall>'
```
