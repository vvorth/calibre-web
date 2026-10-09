# Kobo Sync in Calibre-Web: how it works

A condensed reference for Calibre-Web's Kobo sync: what it does, how it's built, and which protocol details matter if you want to re-implement it elsewhere. For the plan to bring this to inpx-web, see
[`docs/kobo-sync-integration.md` in vvorth/inpx-web](https://github.com/vvorth/inpx-web/blob/claude/sharp-ritchie-3qlntr/docs/kobo-sync-integration.md).

## 1. The idea in one paragraph

A Kobo e-reader reads the server address it syncs with from `api_endpoint` in
`.kobo/Kobo/Kobo eReader.conf` (section `[OneStoreServices]`). Calibre-Web pretends to be the
Kobo Store: you set `api_endpoint=https://<cw-host>/kobo/<auth_token>`. The device then calls
Calibre-Web for `initialization`, `library/sync`, metadata, covers, reading state and collections,
and downloads book files from the URLs Calibre-Web hands it. Calibre-Web can optionally pass any
request it doesn't handle (store, wishlist, purchased books) on to the real `storeapi.kobo.com`
and merge in the store's sync results, so purchased books keep working.

## 2. Code map

| File | Role |
|---|---|
| `cps/kobo.py` | Flask blueprint `/kobo/<auth_token>/…`: every Kobo API endpoint, the sync algorithm, metadata/entitlement builders, reading-state mapping, collections (tags), store proxying, and the fallback `NATIVE_KOBO_RESOURCES` list |
| `cps/kobo_auth.py` | Generates and deletes the per-user token, sets up the URL-token auth decorator `requires_kobo_auth`, and includes research notes on Kobo's real auth |
| `cps/services/SyncToken.py` | Sync cursor that round-trips through the `x-kobo-synctoken` header as base64 JSON |
| `cps/kobo_sync_status.py` | Helpers for `kobo_synced_books` (what's on the device) and `archived_book` (what to remove) |
| `cps/ub.py` | App DB models: `RemoteAuthToken`, `KoboSyncedBooks`, `ArchivedBook`, `KoboReadingState`, `KoboBookmark`, `KoboStatistics`, `ReadBook`, `Shelf.kobo_sync`, `ShelfArchive`, `User.kobo_only_shelves_sync` |
| `cps/main.py` | Registers the blueprints only when `config_kobo_sync` is on; applies a `3/minute` limiter that each authenticated handler then clears |
| `cps/shelf.py`, `cps/web.py`, `cps/admin.py`, `cps/editbooks.py` | Hooks: per-shelf "sync to Kobo" flag, per-user "only sync selected shelves", force full sync, and invalidating the synced flag on edit, archive or format delete |
| `cps/helper.py` | `get_download_link`/`do_download_file`: on-the-fly KEPUB metadata embedding and `.kepub.epub` filename for Kobo |
| `cps/epub.py` | `get_epub_layout()` detects fixed-layout (`pre-paginated`) EPUBs and reports them as `EPUB3FL` |
| `themes/standard/templates/generate_kobo_auth_url.html` | Shows the `api_endpoint=` line to paste into the config |

Config flags: `config_kobo_sync` (needs a restart), `config_kobo_proxy`, `config_external_port`
(used when not behind a reverse proxy), `config_kepubifypath`.

## 3. Authentication

* Kobo's real auth uses a `UserKey`/`DeviceId` and Bearer tokens. Calibre-Web **ignores all of it**.
* Each user gets a random 32-hex token (`RemoteAuthToken`, `token_type=1`, never expires), generated from the profile page.
* `url_value_preprocessor` pops `<auth_token>` from the URL into `flask.g`. `@requires_kobo_auth` looks up the user, calls `login_user()` and clears rate limits; an unknown token gets a 401.
* `/v1/auth/device` and `/v1/auth/refresh` return a dummy random `AccessToken`/`RefreshToken` (or the store's response when proxying), which is just enough to keep the device happy.
* Download URLs embed the same token (`/kobo/<token>/download/<id>/<fmt>`), so the device needs no cookies.
* Because the token is in the URL, it ends up in access logs. HTTPS is effectively required.

## 4. Endpoints

| Device call | Calibre-Web behaviour |
|---|---|
| `GET /v1/initialization` | Returns `Resources`: the store's (when proxying) or the built-in list, with `library_sync`, `image_host`, `image_url_template`, `image_url_quality_template` rewritten to point at Calibre-Web. Sets header `x-kobo-apitoken: e30=` |
| `GET /v1/library/sync` | **The core:** see §5 |
| `GET /v1/library/<uuid>/metadata` | `[get_metadata(book)]`; unknown UUID → proxied to the store |
| `GET/PUT /v1/library/<uuid>/state` | Read or write the reading state (§7) |
| `DELETE /v1/library/<uuid>` | User deleted the book on the device → `ArchivedBook.is_archived=True`, row removed from `kobo_synced_books` |
| `POST /v1/library/tags` | Device created a collection → creates or extends a `Shelf`, returns its uuid |
| `PUT/DELETE /v1/library/tags/<id>` | Rename or delete a shelf |
| `POST /v1/library/tags/<id>/items` and `/items/delete` | Add or remove books on a shelf (matched by `RevisionId` = book uuid) |
| `GET /<uuid>/<w>/<h>[/<q>]/<grey>/image.jpg` | Cover thumbnail sized by height (≤500 small, ≤1000 medium, otherwise large); unknown → 307 to `cdn.kobo.com` when proxying |
| `GET /download/<book_id>/<fmt>` | Book file (KEPUB is served as `*.kepub.epub`) |
| `/v1/user/*`, `/v1/products/*`, `/v1/analytics/*`, `/v1/deals`, `/v1/affiliate`, `/v1/assets`, … | Not implemented: proxied (GET → `307` redirect; other methods are proxied server-side because the device turns redirected POSTs into GETs) or answered with `{}`. `/v1/user/loyalty/benefits` and `/v1/analytics/gettests` have minimal local stubs |

## 5. The sync algorithm (`HandleSyncRequest`)

State lives in two places:

1. **The SyncToken** (on the device, echoed back in the `x-kobo-synctoken` header), holding
   `books_last_created`, `books_last_modified`, `reading_state_last_modified`,
   `tags_last_modified`, `archive_last_modified` (epoch floats) plus `raw_kobo_store_token`.
   A token containing `.` is a real Kobo store token and is stored untouched for proxying.
2. **Server tables:** `kobo_synced_books` (user × book already sent) and `archived_book`.

Steps:

```
require download role
token = SyncToken.from_headers()
if user has 0 rows in kobo_synced_books:  # first sync or "force full sync"
    reset all token timestamps to datetime.min
reconnect calibre DB  # pick up books added outside CW

candidates = books with a KEPUB/EPUB file
             AND id NOT IN kobo_synced_books(user)
             AND visible to user (common_filters, archived allowed)
  if user.kobo_only_shelves_sync:
             AND on one of the user's shelves with kobo_sync=1
             AND BookShelf.date_added > token.books_last_modified
  order by last_modified, id

for book in candidates.limit(100):
    convert EPUB→KEPUB with kepubify if missing (synchronously!)
    entitlement = {BookEntitlement, BookMetadata}
      + ReadingState if its last_modified > token.reading_state_last_modified
    NewEntitlement if max(book.timestamp, shelf.date_added) > token.books_last_created
    else ChangedEntitlement
    advance books_last_modified / books_last_created
    insert into kobo_synced_books

cont_sync = candidates.count() > 0   # still more → "x-kobo-sync: continue"

changed reading states (last_modified > token, not already sent above, limit 100)
    → ChangedReadingState
sync_shelves() → NewTag / ChangedTag / DeletedTag   (§8)

books_last_created only advances when the batch is final
response = JSON array; headers: x-kobo-synctoken (new), x-kobo-sync: continue (if more)
if proxying and final batch: also call the store's sync, append its results, merge its token
```

How changes propagate:

* **New books:** they aren't in `kobo_synced_books` yet, so they're picked up.
* **Edited books:** `editbooks.py` calls `remove_synced_book(id, all=True)` when metadata changes and `last_modified` is bumped. The book becomes a candidate again and goes out as `ChangedEntitlement`. Deleting an EPUB/KEPUB format does the same.
* **Removed or archived books:** the "archive" flag in the web UI, or a delete on the device, sets `ArchivedBook.is_archived`. The book is re-sent with `BookEntitlement.IsRemoved=true`, which removes it from the device.
* **Force full sync** (profile/admin button) deletes the user's `kobo_synced_books` rows, which resets the token logic.
* **Switching on "only sync selected shelves"** runs `update_on_sync_shelfs()`. It archives already-synced books that aren't on a sync shelf and queues `DeletedTag` for unsynced shelves.

## 6. Payload shapes (what the device actually needs)

`BookEntitlement`:
`{Accessibility:"Full", ActivePeriod:{From:now}, Created, CrossRevisionId:uuid, Id:uuid,
IsRemoved:bool, IsHiddenFromArchive:false, IsLocked:false, LastModified, OriginCategory:"Imported",
RevisionId:uuid, Status:"Active"}`

`BookMetadata`:
`{Categories:[<zero-uuid>], CoverImageId:uuid, CrossRevisionId, EntitlementId, RevisionId, WorkId (all = uuid),
CurrentDisplayPrice:{CurrencyCode:"USD",TotalAmount:0}, CurrentLoveDisplayPrice:{TotalAmount:0},
Description, DownloadUrls:[{Format:"KEPUB"|"EPUB3"|"EPUB"|"EPUB3FL", Size:<bytes>, Url, Platform:"Generic"}],
ExternalIds:[], Genre:<zero-uuid>, IsEligibleForKoboLove:false, IsInternetArchive:false, IsPreOrder:false,
IsSocialEnabled:true, Language:<ISO-639-1>, PhoneticPronunciations:{}, PublicationDate,
Publisher:{Imprint:"",Name}, Title, Contributors:[names], ContributorRoles:[{Name}],
Series:{Name, Number, NumberFloat, Id:uuid3(DNS, series_name)}}`

Rules that matter:

* **One stable UUID per book** is used for every id field. Covers are looked up by the same UUID (`CoverImageId`).
* If a KEPUB exists, **only** KEPUB URLs are offered. Otherwise an EPUB is offered as `EPUB3` and `EPUB`. Fixed-layout EPUBs are reported as `EPUB3FL`.
* Timestamps use the format `%Y-%m-%dT%H:%M:%SZ` (UTC).
* The response is built with `json.dumps` instead of `jsonify` because the device is sensitive to how Unicode is escaped (`charset=utf-8`).

## 7. Reading state

The Kobo tracks four timestamped objects. Calibre-Web maps them to tables:

| Kobo object | Table | Fields |
|---|---|---|
| `ReadingState` | `kobo_reading_state` | `last_modified`, `priority_timestamp` |
| `StatusInfo` | `book_read_link` (`ReadBook`, shared with the web UI's read flag) | `Status` ReadyToRead/Reading/Finished ↔ 0/2/1, `TimesStartedReading`, `LastTimeStartedReading` |
| `Statistics` | `kobo_statistics` | `SpentReadingMinutes`, `RemainingTimeMinutes` |
| `CurrentBookmark` | `kobo_bookmark` | `ProgressPercent`, `ContentSourceProgressPercent`, `Location{Value,Type,Source}` |

* `PUT …/state` copies the fields in and answers `{RequestResult:"Success", UpdateResults:[{EntitlementId, CurrentBookmarkResult, StatisticsResult, StatusInfoResult, LastModified, PriorityTimestamp}]}`.
* A SQLAlchemy `before_flush` listener bumps `KoboReadingState.last_modified` whenever a child row changes. That's how a "mark as read" in the web UI reaches the device on the next sync as `ChangedReadingState`.
* Whole-number progress values are sent as integers (`_clean_progress`), because the firmware expects that.

## 8. Collections (Shelves ↔ Kobo "Tags")

* Each shelf is sent as `{Tag:{Id:shelf.uuid, Name, Created, LastModified, Type:"UserTag", Items:[{RevisionId:book_uuid, Type:"ProductRevisionTagItem"}]}}`, as `NewTag` (created after the cursor) or `ChangedTag`.
* A shelf is selected when `shelf.last_modified` or any `BookShelf.date_added` is newer than `tags_last_modified`.
* Deleted shelves are recorded in `shelf_archive` and sent once as `DeletedTag`, then the row is purged.
* Collection edits made on the device go back through the `/v1/library/tags*` endpoints (§4).

## 9. KEPUB handling

* If `kepubify` is configured, `EPUB→KEPUB` conversion runs **synchronously inside the sync request** for each candidate. Opening the Kobo-setup page converts **every** EPUB in the library.
* On download, when `config_embed_metadata` is on, current metadata is written into the KEPUB's OPF (`do_kepubify_metadata_replace`) and the file is streamed from a temp copy that's cleaned up afterwards.

## 10. Known quirks and limits (useful when designing a re-implementation)

1. **Removing a book from a sync shelf doesn't remove it from the device.** Only the collection membership changes. Removal happens only through the archive flag, a delete on the device, or toggling the "only shelves" mode.
2. Change detection depends on clearing `kobo_synced_books` rows from edit hooks. Anything that changes `metadata.db` outside Calibre-Web (desktop Calibre) is **not** noticed as an edit; only brand-new books are.
3. `archive_last_modified` is computed and stored in the token but never used to filter anything.
4. In `sync_shelves`, the filter `not ub.Shelf.kobo_sync` is a Python `not` on a column object, not a SQL `NOT`, so that query most likely never returns rows. Unsynced shelves are removed through `update_on_sync_shelfs` instead.
5. `SyncToken.from_headers` validates the outer object against `data_schema_v1` instead of `data_json`, so validation of the data part is weaker than intended.
6. Per-user, not per-device: two devices on one token share `kobo_synced_books` and will fight over it.
7. KEPUB conversion during sync can make the request slow enough to time out on large first syncs. Batches are capped at 100 items, with `x-kobo-sync: continue`.

## 11. Minimal recipe to re-implement Kobo sync anywhere

1. Route `/kobo/<token>/…` to your service and authenticate by token.
2. Serve `v1/initialization` with resources that point `library_sync` and the image templates back at you.
3. Keep a per-device set of "books already sent" and a stable UUID per book.
4. On `v1/library/sync`, send new or changed books as `NewEntitlement`/`ChangedEntitlement` and removed books with `IsRemoved:true`. Page with `x-kobo-sync: continue` and round-trip an opaque `x-kobo-synctoken`.
5. Serve covers by UUID and files at the `DownloadUrls` you advertised, with an accurate `Size`. Prefer KEPUB.
6. Accept `PUT v1/library/<uuid>/state` and return state changes as `ChangedReadingState`.
7. Optionally, send collections as `NewTag`/`ChangedTag`/`DeletedTag` and handle the `/v1/library/tags*` endpoints.
8. Stub or proxy everything else so the firmware doesn't complain.
