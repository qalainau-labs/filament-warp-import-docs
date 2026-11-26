# Changelog

All notable changes to `qalainau/filament-warp-import` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.5.1] - 2026-11-27

### Fixed

- Scheduled imports read large CSV, TSV and .xlsx files row by row instead of holding every row in memory. A 950,000-row, 54 MB CSV used to need more than 500 MB on top of the file and failed on servers with `memory_limit` at 512M or lower; it now needs about the size of the file. The rows, headers and errors are the same as before.
- The plugin's stylesheet no longer hides the Filament sidebar on desktop screens until the page's JavaScript has loaded. Its `[x-cloak]` rule now only applies inside the import wizard.

## [1.5.0] - 2026-11-25

### Removed

- `AnthropicSuggester`, `aiSuggestions(model:)` and direct calls enabled by `services.anthropic.key`.

### Changed

- AI suggestions require a `warp.ai` gateway or an explicit `using` class. A provider key alone no longer shows AI controls; saved model settings are ignored.

### Added

- `GatewaySuggester` for columns, values and rewrites, `aiFeature()` metering names, and gateway `unavailable()` messages on their own line, in small muted text, below the disabled suggestion buttons.
- PCRE execution limits during preview and import; preview excludes expensive rules and explains which were skipped.
- Gateway evaluation cases for all three Import AI features.

### Fixed

- Templates with identical update times are ordered by descending ID for consistent selection.

## [1.4.2] - 2026-11-25

### Fixed

- **Result counts when some rows had no changes**: rows whose values were already the same as the record are not written, but the result had no count for them, so the counts did not add up (for example "All 4" with every other count at 0). The result now has an "Unchanged" tab (shown when there are such rows), and the "Not imported" table no longer lists them, so it matches its count.
- **Template notice after choosing "None"**: after a template was applied, choosing "None" in the template menu still said "Applied the template “…”." The notice now goes away.
- **Values hidden by "Choose…"**: in a narrow column, a value that is not one of the choices was cut short by the "Choose…" button next to it (for example "Deutschla… Auswählen…"). When the value does not fit, the button now shows only ▾ (its name stays in the tooltip and for screen readers).
- **Two names for the same rows in the review**: the badge of a row with no changes said "Same" while its filter said "Unchanged" (and "Change" next to "Changed"). The badges now use the filters' names in every language.

## [1.4.1] - 2026-11-24

### Fixed

- **Result counts of an import with child sheets**: after importing orders and their items, the "All", "Created", "Updated" and "Failed" counts showed the total of every sheet (13 = 4 orders + 9 items) while the table below showed only the selected tab's rows. The counts now belong to the selected tab, like the table.

- **Problems with two or more columns**: when a row had errors in two columns with choices (for example Country and Rank), the Problems column shortened both messages and their "Choose…" buttons until none could be read ("Cou… Choose… Ra… Choose…"). "Choose…" is now in the cell with the value, and the Problems column reads as one sentence, shortened at the end (hover for all of it).

- **"Sheet 1" for a CSV with child sheets**: in the mapping step, the parent's tab is named after what is imported (for example "Orders"), as in the review, and the child's sheet menu no longer lists the parent's own sheet, which could not be chosen.

- **Blank review table while the rows load**: after "Check", the table showed only its headers for about a second. It now says "Loading…" until the first rows arrive.

- **Message for required fields without a column**: pressing Check with a required field left on "Do not import" showed only the field's name in red ("Code") and kept it after a column was chosen or AI matched it. It now says "Choose a column for: Code." in the panel's language, and updates or disappears as the columns are chosen.

### Changed

- When an import has child sheets, the parent's review tab is named after what is imported (for example "Orders") instead of the file's sheet name ("Sheet 1" for a CSV), and the import button counts each part in words: "Import 4 orders · 9 order items".

- The value mapping window puts its two AI tools in separate, headed boxes ("Let AI choose for values that can't be imported" and "Ask AI to change the values in words"). While an AI change is waiting to be applied or discarded, the window's Apply button is disabled and says why.

- In the mapping step, the file column menus are wide enough for the header names and "Do not import", opening "Combine or split" no longer changes the widths of the table's columns, and "First values" shows the combined or split values.

- In the review, the Problems column is narrower (16rem) so more of the file's columns fit, and the column menu buttons are larger, with a chevron icon and a background.

- A value that is not one of a column's choices is described in the panel's language ("Country “UK” is not one of the choices.") instead of the app's validation message, and values are quoted the way each language quotes them.

- The value mapping window explains each AI tool in one line, and the example in "Ask AI to change the values in words" follows the column (choices, numbers, dates or text) instead of always being about phone numbers. After an AI change is applied or removed, the review says so ("AI changed 4 values in “Phone”."). For columns without choices, the field for each value is a plain field ("Keep as it is") instead of a search field, with a line that explains it.

- Model names inside sentences are written as in the middle of a sentence ("Same sheet as orders", "Sheet for order items"), and the import heading uses the sentence form in languages other than English and German ("Importar clientes" instead of "Importar Clientes").

- Wording: Japanese uses 「取り込んだデータ」 everywhere and a shorter required-for-new badge (「新規は必須」), Korean avoids the 이(가) form and uses one kind of quotes, Spanish uses «…» quotes, and French keeps « and » on the same line as the quoted words and titles the window "Importer des clients".

## [1.4.0] - 2026-11-23

### Added

- Read OpenDocument (.ods) spreadsheets and Excel 97–2003 (.xls) workbooks, with optional PhpSpreadsheet support for .xls (5.9 or later; earlier versions can hang on damaged .xls files and are treated as not installed), including scheduled imports.

- Download CSV and Excel sample files from the file step, using importer example headers and values, with child sheets in Excel and a `sampleFile(false)` option.

- Ask AI to change column values in words, review before/after examples, and apply or remove validated replacement rules. Templates and scheduled imports keep the rules. Only with `->aiSuggestions()` (`rewrites: false` hides it); the values of that column are sent to the AI provider. Custom suggesters implement the new `Ai\SuggestsRewrites` interface.

- Combine columns or split a column into selected parts in the mapping step, with previews, value mapping, child sheets, templates and scheduled imports.

- Import parents and children from one sheet of detail rows, with automatic selection, a `sameSheet: true` default, and original file row numbers.

- Choose the header row, skip title rows above the table, or import every row without headers. Templates and scheduled imports keep the choice.

### Changed

- New migration `update_warp_import_add_tenant_indexes` adds an index on `(tenant_type, tenant_id, importer)` to `warp_import_templates` and `warp_import_schedules`, so saved mappings and scheduled imports are looked up within one tenant. Indexes only: no data changes, and an app that already added the same index keeps its own. Run `php artisan migrate`.

- Upload .ods and .xls files to the server for reading, then immediately delete the temporary files. CSV, TSV and .xlsx files continue to be read in the browser.

- Detect the header row automatically by default, including child sheets and pasted cells. Previously the first row was always used as the header.

## [1.3.1] - 2026-11-20

### Fixed

- **Console error `warpImport is not defined`**: on pages where Livewire re-renders the import wizard inside another Alpine component, the browser console could show `warpImport is not defined` before the component's script had arrived. Empty components are now registered up front, and the real ones replace them as soon as they load. No visible change.

## [1.3.0] - 2026-11-15

### Added

- `canAccessUsing()` and `canWriteUsing()` on `WarpImportPlugin` decide, from your app's roles, who can open the history, the scheduled imports and your `ImportPage`s, and who can only look at them. An `ImportPage` can also override `canWrite()`. See [Restricting pages per role](README.md#restricting-pages-per-role).

## [1.2.3] - 2026-11-14

### Fixed

- Searching the values of a related column while mapping values no longer fails on PostgreSQL when the related attribute is a number column (`operator does not exist: integer ~~* text`).

## [1.2.2] - 2026-11-12

### Changed

- Now requires Filament 5.10 or later (was 5.0). Parts of the plugin did not work on older 5.x versions, which were never tested. The tests now also run on the lowest supported versions before each release.

### Fixed

- Importing from a URL: a malformed address such as `http:///file` is now refused with the usual "not a valid URL" message instead of a server error.

## [1.2.1] - 2026-11-10

### Fixed

- The "Records" column of the import history, the approval request notification and the names of the child imports now use the plural label of the panel resource for the model (`getTitleCasePluralModelLabel()`, so in the app's language, such as 「商品」), instead of the English class name (`Products`). Without a resource for the model, the class name is used as before.

## [1.2.0] - 2026-11-10

### Changed

- Values that are exactly the name (label) or the value of an option, apart from upper and lower case (`Gold`, `United States`), are now mapped on their own while checking, and saved in the value mapping, so they are no longer errors. Values that only look alike (spaces, full-width letters) are still suggested in the value mapping window.
- The review counts rows that point at the same record as another row under a new "Duplicates" card (blue, like their label) instead of "Errors" (the card shows only when there are such rows). `RowQueries::counts()` has a new `duplicate` count. On the Errors or Duplicates card, **Skip all N rows with errors or duplicates** skips both kinds at once, as "Skip all rows with errors" did before. Scheduled imports still treat these rows like errors (stop, or skip them).
- Scheduled imports made from an import now use the panel's time zone (the same as the history and `config('app.timezone')`), not the browser's. The list of scheduled imports shows the time zone after "Next:", writes the time in 24 hours like "Every day at 06:00", and shows a "Not run yet" badge instead of "—".
- In the value mapping window, the field for related records looks like a search field (a magnifying glass, "Search", no ▼), with a line that says how to search and what an empty field does. The message of a duplicate row is no longer red.
- The history column "Records" is now "Imported into" (it shows the kind of records, not a number).
- French: "Importer des :label" and the heading "Importation : :label".

### Fixed

- On phones, "Run now" in the list of scheduled imports no longer ends up outside the table: the list is stacked on phones, and "Run now" is an outlined button.

## [1.1.3] - 2026-11-07

### Changed

- The protection for downloading from links (images, scheduled imports) is now shared with the other Warp plugins. It works as before; IPv4 addresses written inside IPv6 (`::ffff:10.0.0.1`) are checked by their IPv4 part in every case.

## [1.1.2] - 2026-11-07

### Fixed

- Records that are not in the file: a row your importer skips by returning `null` from `resolveRecord()` after
  looking up its record (`$product = Product::firstOrNew([...]); if (...) return null;`) now keeps that record in the
  file. Before, the record counted as not in the file and was changed, or deleted with `missingRecords(delete: true)`.
  A later row of the same record is no longer marked as its duplicate.

### Changed

- README, the Boost guideline and the skill: which importer hooks run while checking and which only when a row is
  written (refuse rows in `beforeValidate()` … `afterFill()`, call APIs and send mail in `afterSave()` /
  `afterUpdate()`), refusing a row with `RowImportFailedException` instead of returning `null`, and that
  `warpImportQuery()` narrows only the records not in the file, not `resolveRecord()`.

## [1.1.1] - 2026-11-06

### Changed

- README: a "Testing" section with two helpers and a full feature test of an import (review, writing, a refused row,
  records not in the file). It explains that `updateColumns()`, `keepBlankCells()` and `missingRecords()` only take
  effect through what the test sends to `warpImportStart` (`updateColumns`, `keepBlanks`, `sync`), as the browser does;
  a test that leaves them out overwrites every column.

## [1.1.0] - 2026-11-06

### Added

- Instructions for AI coding assistants (Claude Code, Cursor, Codex) through [Laravel Boost](https://laravel.com/docs/boost):
  a guideline that Boost adds to `CLAUDE.md` and `AGENTS.md`, and the `warp-import-development` skill for building
  imports and testing them. Select `qalainau/filament-warp-import` in `php artisan boost:install`, or add it to
  `"packages"` in `boost.json`. See [AI coding assistants (Laravel Boost)](README.md#ai-coding-assistants-laravel-boost).
- `php artisan warp:assign-tenant "App\Models\Team" 1` (with `--dry-run`): for apps that add tenancy later, gives the imports, templates and schedules saved without a tenant to one tenant, so the tenant panel shows them. Every row without a tenant moves, in one transaction. The tenant is written as the class name, the way Warp Import stores it, also in apps with a morph map.
- README: **Adding tenancy to an app that already has imports** and **Testing with tenancy** (how to open a tenant in a test, and what another team's IDs do: most are simply not shown, some answer 404).

## [1.0.0] - 2026-11-02

### Added

- CSV files saved by Excel in other languages: the encoding is told apart from the file (Windows-1252, Shift_JIS, CP949, GBK or Big5) and shown in the columns step, where users can choose another one; scheduled imports read the file in the encoding of the first import. Also the currencies `NT$`, `HK$`, `￦`, `RMB` and `元`, and Taiwan's Minguo dates (`115/1/6`, `民國115年1月6日`). Checked with an Excel CSV of each of the ten languages (`tests/resources/locales`).
- European and other files: numbers with decimal commas (`1.234,56 €`, `12,99`, `1 234,5`, `1'234.50`), dates like `31.12.2026`, `31/12/2026` and `12/31/2026`, month names in German, French, Spanish, Italian and Portuguese (`6. Januar 2026`, `1er janvier 2026`, `6 de enero de 2026`), Korean dates (`2026년 1월 6일`, `2026. 1. 6.`) and yes / no in those languages (`ja` / `nein`, `oui` / `non`, `sí` / `no`, `sim` / `não`, `예` / `아니요`, `是` / `否`). New clean-up options **Decimal comma**, **Day first** and **Month first** decide `1,234`, `1.234` and `05/06/2026`; the wizard picks them from the values in the column.
- `WarpImportAction`, a drop-in for Filament's `ImportAction` that takes the same `Importer` class and opens the import wizard in a full-screen modal.
- `ImportPage`, the same wizard as a resource page.
- Reading CSV, TSV and Excel (.xlsx) files in the browser, UTF-8 or the encodings Excel saves CSV in (listed under the drop zone), and pasting cells from Excel or Google Sheets.
- Column mapping guessed from names, labels and the importer's `guess()`, with the first values of each column and per-column clean-up (trim, full-width to half-width, numbers like `１，２００円` and `△300`, Japanese era dates like `令和8年10月20日`, はい／いいえ and ○／×, postal codes, phone numbers).
- Review before writing: every row runs through the importer without saving (inside a rolled-back transaction) and is shown as new, changed (with the old and new value of each field), unchanged or error. Rows that point at the same record are flagged. Numbers, dates and yes/no values that Filament would silently turn into 0 or true are reported as errors.
- Map values: the different values of a column with their row counts, unknown ones first, matched to the column's options (enums, `in` rules), related records or yes/no, with suggestions for values that differ only in case or width (`kitchen` → `Kitchen`, `販売中` → `on_sale`). Choice values are shown by their labels.
- Find and replace in a column, optionally only for the whole cell or only in rows with errors. Only the affected rows are checked again.
- Import without a queue worker: the browser writes 200 rows at a time with a progress bar, pause and resume. Each row runs in its own transaction (a savepoint), so a failing row is rolled back on its own; rows that hit a busy database are tried again. `->queue(threshold)` hands large imports to Laravel's queue and sends a database notification with the importer's `getCompletedNotificationBody()`.
- Results: created, updated and failed rows, and the failed rows as a CSV with the original headers and a column with the reasons, ready to fix and import again. Rows that were not written are marked **Not imported** with the reason (a duplicate of row 2, an error, left out, a parent row that was not imported) instead of the review's labels.
- Images and files: `->files('image', disk:, directory:, maxKilobytes:, types:, privateNetwork:)`. Cells hold a link or a file name from a ZIP (a small **Add a ZIP of images…** button on the first screen; when the column step finds file names and no ZIP, it says how many and offers the same button, and says nothing for links); the review checks them and shows the link or name, the importer gets the stored path (content-addressed, so the same picture is Unchanged), files are downloaded or extracted when the row is written (size, type and HTML pages checked; internal network addresses refused, redirects included) and removed on undo.
- AI suggestions: `->aiSuggestions(model:, samples:, using:)`. **Suggest with AI** in the column step maps the fields that are still unmapped to the unused headers (with their first values), and in the value mapping maps the values a column does not accept to the ones it does, with Claude (`config('services.anthropic.key')`) or your own `SuggestsMappings` class. Suggestions are marked **AI**, never overwrite the user's choices and go through the review like everything else; answers naming unknown fields, headers or values are dropped. 20 questions a minute per user.
- The wizard opens as a large modal over the list (a margin around it, up to 1920 px wide; full screen on phones). `->modalWidth(Width::Screen)` makes it fill the screen.
- Scheduled imports: `->schedules(disks: [...], urls:, privateNetwork:)`. **Run on a schedule…** on the result saves the import's columns and settings with a link (Google Sheets share links included) or a file on a listed disk (its files are offered), every hour, day or week in the user's time zone. Each run matches columns by header name, checks the rows like the review, stops and notifies on rows with errors (or skips them), writes on the queue, and asks for approval when the import needs it. A **Scheduled imports** page lists the last and next runs (the last result in the reader's language), with **Run now**, **Edit** and **Delete**. Runs through `warp-import:run-schedules`, added to Laravel's scheduler.
- Approval: `->requireApproval(approvers: ..., when: ...)`. The import button becomes **Request approval** (with a note); the import waits read only, approvers get a database notification and see it in their history, and **Approve and import** writes it from the approver's browser or the queue, while **Send back…** returns it with a reason. Fixed rows after the import need approval too. Events `ImportApprovalRequested`, `ImportApproved`, `ImportRejected`.
- Records that are not in the file: `->missingRecords(delete: true)` or `->missingRecords(update: [...], label: 'mark them as not for sale')` lets users treat a file as the complete list (opt-in per import), narrowed by `warpImportQuery()` on the importer. The checkbox has a collapsed plain-language explanation with a worked example. The review lists them in a **Not in imported data** tab where any can be kept, the import button shows the count and asks before writing, they are handled after the file rows with the model policy's `delete` / `update` checked per record, and undo restores them (soft deletes restored, hard deletes inserted again).
- Updating existing records: **Leave the current value when a cell is empty** and a per-column **Update existing** in the column step (`->keepBlankCells()`, `->updateColumns([...])` for the defaults). Kept cells are faded in the review, are not validated and never count as changes; templates keep both settings.
- **Choose…** on review errors of choice, relationship and yes / no columns, next to the value from the file (`Prefecture “愛知”: …`): opens the value mapping of the column at that row's value, with the values the column accepts, instead of leaving users with "The selected … is invalid.".
- Dates with English month names (`Jan 6, 2026`, `6 January 2026`, `06-Jan-2026`) are read as dates; the English clean-up labels and messages lead with English examples (`$1,200`, `Y / N`).
- English, Japanese, German, French, Spanish, Italian, Brazilian Portuguese, Korean and Chinese (simplified and traditional) translations.
- Fix rows after the import: the result counts the rows that were not imported (skipped, errors, failed), and **Fix the rows that were not imported** opens them in the review table with all its tools. Failed rows are checked again; already imported rows cannot change; the fixed rows are written into the same import, so the counts add up and one undo takes back everything. `ImportCompleted` is dispatched again after each such write.
- Undo: created records are deleted and updated ones get their old values back, for 30 days by default (`->undoable(days)`, `->undoable(false)`). Records that someone changed after the import are kept and listed. Values are compared through the model's casts and restored without mutators.
- Recent imports in the wizard and an import history page (`WarpImportPlugin`) to open, resume or undo an import. `ImportCompleted` and `ImportUndone` events. Dates are shown in the panel's timezone (`FilamentTimezone`). Old imports are pruned with `model:prune`. **Download** on the history gives back what was imported as a CSV with the file's headers (a ZIP of CSVs for several sheets), with the cells fixed in the review.
- Templates: save the column mapping, clean-up and value mapping under a name, privately or shared (`->templates(shared: false)` to keep them private, `->templates(false)` to turn them off). The mapping is kept by header name, so it fits files with the columns in another order; a file with the same headers gets its template automatically.
- Parent and child sheets: `->child(OrderItemImporter::class, relationship: 'order', parentColumn: 'order_no', sheet: 'Items')` imports another sheet of the same workbook as children of the rows in the main sheet. Child rows are linked to the parent row with the same key; the child importer needs no column for its parent, Warp associates it. Children of a parent row that will not be imported are held back and released when the parent is fixed. Parents are written first; undo removes children first.
- With parent and child sheets, the import button counts rows per sheet ("Import Orders 2 · Order Items 2").
- Opening a recent import that was removed after the list was shown (pruning) says so and refreshes the list.
- Files that are not text or xlsx (images, PDFs) are refused instead of read as garbage; Excel 97–2003 (.xls) files get a message to save them as .xlsx or CSV; UTF-16 text saved by Excel as "Unicode Text" is read. Read errors are shown in the page's language.
- Rows with only spaces are skipped like empty rows.
- Numbers in scientific notation (`1e3`) are written out in full; values Filament would misread as numbers (`1-2`, `0x1A`) and dates that do not exist (`2026/02/30`, which PHP rolls over to March 2) are errors.
- A row with a value that cannot be read still shows the errors of its other columns, so everything can be fixed at once.
- Esc closes the column menu, the value and replace panels and the clean-up list wherever the focus is, without closing the import modal; focus moves into the menu and panels and returns afterwards; menus work with the arrow keys and panels keep Tab inside.
- Touch screens: double-tap a cell to change it, also in Safari on iPhone.
- Phones and short windows: the wizard scrolls and the column step is shown as one card per field.
- Light and dark mode, the same in Chrome, Safari and Firefox.
- Many-to-many columns (`->multiple(',')->relationship()`): values are mapped one by one and the related records attached.
- The importer's options form (`getOptionsFormComponents()`) in the column step; its values are added to the importer's options, as with `ImportAction`.
- `->maxRows()` to limit the rows of an import.
- Rows whose save a model event stopped (`saving` returning `false`) are reported as failed.
- Keyboard navigation in the review table (arrow keys, Page Up/Down, Home/End, Enter or F2 to change a cell, Space to skip), with `aria-activedescendant`.
- Closing the wizard stops sending rows; a paused import is resumed from the history.
- Imports and recent imports stay inside the current tenant.
- Multi-tenancy: queued imports restore the panel as well as the tenant, so the resources' tenant scopes apply in the worker — rows update only the current tenant's records, and new records are given the tenant (also through a pivot table). A queued import whose tenant has been deleted, or whose user has left the tenant (checked again every 500 rows), stops writing, stays paused and notifies the user. Relationship columns that point at a model without a resource in the panel are kept to the current tenant through the panel's ownership relationship (suggestions, the check and the import); a model without that relationship stops the wizard with a clear error. A [Multi-tenancy](README.md#multi-tenancy) section in the README.
- Fix a cell in the review table and the row is checked again; skip rows one by one or all rows with errors.

### Security

- Links (images, scheduled imports) cannot reach the internal network: every IPv4 and IPv6 address of the host is checked, the download connects to the address that was checked (a DNS answer that changes in between cannot reach inside), redirects are checked one by one, and ranges PHP lets through (`100.64.0.0/10`, `198.18.0.0/15`, NAT64 and 6to4 addresses) are refused.
- Downloaded images and files that run scripts in a browser (HTML, SVG, XML, PHP) are refused, even when their extension is allowed.
- Downloaded CSV files put an apostrophe before cells Excel would run as formulas; importing the file again takes it off.
- The import history continues the import it opened and does not start a new one, where the import action's permissions do not apply.
- An approver approves only the request they saw; approvers outside the tenant are not asked; notifications escape what people typed.
- Scheduled imports read only table files on a disk (also after an edit), stop when their creator can no longer use the panel, and stop before a sync that would remove more than a quarter of the matching records.
