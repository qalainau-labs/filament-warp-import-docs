# Warp Import

![An import of 100,000 products under review: new, changed and error rows, with the old and new value of every changed field](https://raw.githubusercontent.com/qalainau-labs/filament-warp-import-docs/main/art/import.png)

**Import CSV and Excel files into Filament with the `Importer` classes you already have — and see every row, every change and every error before anything is written. Fix mistakes in place, import 100,000 rows without a queue worker, and undo an import with one click.**

<a href="https://demo.warpui.com/import"><img src="https://raw.githubusercontent.com/qalainau-labs/filament-warp-import-docs/main/art/live-demo.png" alt="Try the live demo: demo.warpui.com/import" width="592"></a>

Drop a spreadsheet of products, customers or orders with their items, review what would be created and changed, fix the errors and import — then undo it — at **[demo.warpui.com/import](https://demo.warpui.com/import)**. No sign-up needed; the data is reset every hour.

Filament's own `ImportAction` takes a CSV file, hands it to a queue and tells you afterwards how many rows failed, in a CSV of failed rows. Warp Import keeps your `Importer` and changes everything around it. The file is read in the browser. Every row runs through your importer *without saving*, so users see which rows will be new, which will change and how, and which have errors, before anything happens. They fix cells in a table, map "東京都" or "Sold out" to the values your app expects, and import. If something went wrong anyway, they undo the import: created records are deleted and changed ones get their old values back.

```php
use App\Filament\Imports\ProductImporter;
use Qalainau\FilamentWarpImport\Actions\WarpImportAction;

protected function getHeaderActions(): array
{
    return [
        // was: ImportAction::make()->importer(ProductImporter::class)
        WarpImportAction::make()->importer(ProductImporter::class),
    ];
}
```

## Contents

- [Why](#why)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [How rows are checked](#how-rows-are-checked)
- [Images and files](#images-and-files)
- [Approval](#approval)
- [Scheduled imports](#scheduled-imports)
- [Records that are not in the file](#records-that-are-not-in-the-file)
- [Editing in a spreadsheet](#editing-in-a-spreadsheet)
- [Updating existing records](#updating-existing-records)
- [Clean-up and value mapping](#clean-up-and-value-mapping)
- [AI suggestions](#ai-suggestions)
- [Importing and undoing](#importing-and-undoing)
- [Parent and child sheets](#parent-and-child-sheets)
- [Templates](#templates)
- [Import history](#import-history)
- [Multi-tenancy](#multi-tenancy)
- [Security](#security)
- [Limitations](#limitations)

## Why

Measured on the server (checking, writing and undoing; the browser adds the time to send the rows), importing products through a plain Filament importer with a relationship, a number, a boolean and a date column — 2,000 rows update existing products, the rest are new:

| | SQLite | MySQL 8.4 | PostgreSQL 18 |
| --- | --- | --- | --- |
| Check 10,000 rows | 3.6 s | 6.2 s | 6.9 s |
| Import 10,000 rows | 6.1 s | 8.4 s | 9.7 s |
| Undo 10,000 rows | 1.8 s | 2.5 s | 2.5 s |
| Check 100,000 rows | 43 s | 57 s | 65 s |
| Import 100,000 rows | 63 s | 95 s | 103 s |
| Undo 100,000 rows | 18 s | 33 s | 28 s |

On a laptop (Apple M4 Max, PHP 8.4, one PHP process). The work is done in steps of 500 to 1,000 rows, so memory stays flat however large the file is.

There is no queue worker to set up: the browser sends the work in small steps and shows the progress. Users can pause and resume, and an import interrupted by a closed tab continues from the history.

## Features

![Values in the file mapped to the options of the column](https://raw.githubusercontent.com/qalainau-labs/filament-warp-import-docs/main/art/values.png)

- **Your Filament importers, unchanged**: columns, `rules()`, `relationship()`, `castStateUsing()`, `fillRecordUsing()`, `resolveRecord()`, the hooks (`beforeSave()` and so on), `getOptionsFormComponents()` and `getCompletedNotificationBody()` all work as with `ImportAction`.
- **CSV, TSV, Excel and pasted cells**: `.csv`, `.tsv` and `.xlsx`, UTF-8 or Shift_JIS (what Japanese Excel saves), every sheet of a workbook, Excel dates as dates. Or paste a block of cells from Excel or Google Sheets. Files are read in the browser and never uploaded.
- **Column mapping** guessed from your column names, labels and `guess()`, ignoring case, width and separators, with the first values of each column next to it.
- **Review before writing**: every row is new, changed (with the old and new value of each field, related records by name), unchanged or an error. Rows that point at the same record are flagged. Filter by outcome and scroll through 100,000 rows.
- **Fix in place**: double-click a cell (double-tap on a phone or tablet, or press Enter) to change the value from the file and the row is checked again. Skip single rows or every row with an error. Find and replace in a column.
- **Clean-up** per column: trim, numbers like `$1,200` and `(300)`, dates like `Oct 20, 2026`, `2026/10/20` and Excel serials, yes/no like `Y` / `N` — and the Japanese forms too: full-width digits, `１，２００円`, `△300`, `令和8年10月20日`, はい／いいえ and ○／×, postal codes and phone numbers. Numbers, dates and yes/no values that Filament would silently turn into `0` or `true` become errors instead.
- **AI suggestions** (optional): **Suggest with AI** maps headers like "Client ID" or "Mail" and values like "England" or "VIP" with Claude — you check them before anything is imported.
- **Value mapping**: the different values of a column with their row counts, unknown values first, mapped to the column's options (enums, `in` rules), related records or yes/no, with suggestions. An invalid value in the review has a **Choose…** button that opens it.
- **Undo** for 30 days: created records are deleted, changed ones get their old values back. Records someone changed after the import are kept and listed.
- **Parent and child sheets**: orders and their items, in two sheets of one workbook, in one import.
- **Templates**: save the mapping, clean-up and value mapping; files with the same headers get them automatically next time.
- **Images and files**: photos and documents from links or a ZIP, put on a Laravel disk when the row is written, removed again on undo.
- **Approval**: one person prepares the import, another reviews the same changes and approves or sends it back.
- **Scheduled imports**: import a supplier's price list from a link, a Google Sheets spreadsheet or storage every hour, day or week, with the same columns and clean-up; rows with errors stop the run and notify you.
- **Sync with a full list**: optionally delete or deactivate the records a file leaves out — listed in the review, confirmed before writing, undone with the rest.
- **Edit in a spreadsheet**: download the current records with the importer's headers, change them in Excel and import the file back — only the changed rows are written.
- **Results**: created, updated and failed rows. Rows that were not imported (skipped, errors, failed) can be fixed right there and imported into the same import, or downloaded as a CSV with the reasons.
- **History** of imports to reopen, resume or undo them, and `->queue()` for imports you would rather run in the background.
- Light and dark mode, keyboard navigation, translations (English, Japanese, German, French, Spanish, Italian, Brazilian Portuguese, Korean and Chinese, simplified and traditional), and Filament's [multi-tenancy](#multi-tenancy).

## Requirements

- PHP 8.3+
- Laravel 12 or 13
- Filament 5
- SQLite, MySQL 8, MariaDB 10.11+ or PostgreSQL

## Installation

> **Warp Import is a commercial plugin.** A license is required. After purchase you receive credentials for the private Composer repository.

Add the repository and authenticate:

```bash
composer config repositories.qalainau composer https://qalainau.privato.pub/composer
composer config --auth http-basic.qalainau.privato.pub "your-email" "your-license-key"
```

This writes your license key to `auth.json` in the project. Add `auth.json` to `.gitignore`, and on CI or servers pass the credentials in the `COMPOSER_AUTH` environment variable instead.

Install the package, run the migration and publish Filament's assets:

```bash
composer require qalainau/filament-warp-import
php artisan migrate
php artisan filament:assets
```

The migration creates three tables: `warp_import_runs` (one row per import), `warp_import_rows` (the rows of an import while it is reviewed, and what is needed to undo it) and `warp_import_templates`. Your tables are not changed. To edit the migration, publish it with `php artisan vendor:publish --tag=filament-warp-import-migrations`. Publish it before you run `php artisan migrate` for the first time: Warp Import then runs your copy instead of its own. If the tables already exist, change them with a migration of your own instead.

Unlike Filament's `ImportAction`, Warp Import does not need the `imports`, `failed_import_rows` and `job_batches` tables, or a queue worker.

> Run `php artisan filament:assets` again after every update of the package.

## Usage

### Replace the import action

Wherever you use Filament's `ImportAction`, use `WarpImportAction` with the same importer:

```php
use App\Filament\Imports\ProductImporter;
use Qalainau\FilamentWarpImport\Actions\WarpImportAction;

class ListProducts extends ListRecords
{
    protected static string $resource = ProductResource::class;

    protected function getHeaderActions(): array
    {
        return [
            WarpImportAction::make()
                ->importer(ProductImporter::class)
                ->options(['source' => 'admin']),  // the importer's $this->options, as with ImportAction
        ];
    }
}
```

The action opens the wizard in a large modal over the list: **File → Columns → Review → Import**. It is a normal Filament action, so `->visible()`, `->authorize()`, `->label()`, `->icon()` and the rest work as usual. Its label is "Import products" by default.

The modal leaves a margin around it on a computer, so the list stays visible behind it, and grows up to 1920 px wide for files with many columns; on a phone it fills the screen. To fill the whole window everywhere, use `->modalWidth(Width::Screen)` (`Filament\Support\Enums\Width`).

The importer is an ordinary Filament importer. If you do not have one yet, `php artisan make:filament-importer Product --generate` creates it. A short example:

```php
use Filament\Actions\Imports\ImportColumn;
use Filament\Actions\Imports\Importer;
use Filament\Actions\Imports\Models\Import;
use Illuminate\Validation\Rule;

class ProductImporter extends Importer
{
    protected static ?string $model = Product::class;

    public static function getColumns(): array
    {
        return [
            ImportColumn::make('sku')->label('SKU')->requiredMapping()->rules(['required', 'max:32']),
            ImportColumn::make('name')->requiredMappingForNewRecordsOnly()->rules(['required', 'max:255']),
            ImportColumn::make('category')->relationship(resolveUsing: 'name')->guess(['カテゴリ']),
            ImportColumn::make('price')->numeric()->rules(['required', 'integer', 'min:0']),
            ImportColumn::make('is_active')->boolean(),
            ImportColumn::make('released_on')->rules(['nullable', 'date']),
            ImportColumn::make('status')->rules(['required', Rule::enum(ProductStatus::class)]),
        ];
    }

    public function resolveRecord(): ?Product
    {
        return Product::firstOrNew(['sku' => $this->data['sku']]);
    }

    public static function getCompletedNotificationBody(Import $import): string
    {
        return "Imported {$import->successful_rows} products.";
    }
}
```

`resolveRecord()` decides whether a row creates or changes a record. Return `firstOrNew()` to update existing records by a key, `new Product` to always create, or `null` to skip the row (it shows as "Skipped").

### A page instead of a modal

To give the wizard a whole page, add an `ImportPage` to the resource:

```php
use App\Filament\Imports\ProductImporter;
use Qalainau\FilamentWarpImport\Import;
use Qalainau\FilamentWarpImport\Pages\ImportPage;

class ImportProducts extends ImportPage
{
    protected static string $resource = ProductResource::class;

    public function import(Import $import): Import
    {
        return $import->importer(ProductImporter::class);
    }
}
```

and register it in the resource's `getPages()`, for example `'import' => ImportProducts::route('/import')`. The page uses the resource's access rules (`canAccess()`). `$import` has the same methods as the action: `importer()`, `options()`, `undoable()`, `queue()`, `templates()` and `child()`.

### All options

| Method | Default | What it does |
| --- | --- | --- |
| `->importer(ProductImporter::class)` | required | The Filament importer. |
| `->options([...])` | `[]` | Values for the importer's `$this->options`. The values of the importer's options form (`getOptionsFormComponents()`) are added to them, as with `ImportAction`. |
| `->undoable(30)` | 30 days | How long an import can be undone. `->undoable(false)` turns undo off: the old values are not kept. |
| `->queue(50_000)` | off | Imports with at least this many rows to write run on Laravel's queue instead of in the browser. `->queue()` always uses the queue. |
| `->maxRows(20_000)` | none | The most rows an import may have (all sheets together). Larger files are refused when they are read. |
| `->templates(true, shared: true)` | on, shareable | Saving and applying templates. `shared: false` keeps every template private; `->templates(false)` turns them off. Both accept closures, for example to check a policy. |
| `->child(OrderItemImporter::class, relationship: 'order', parentColumn: 'order_no', sheet: 'Items', label: null)` | — | Imports another sheet of the workbook as children. See [Parent and child sheets](#parent-and-child-sheets). |
| `->files('image', disk: 'public', directory: 'products')` | — | An image or file column: a link or a file name from a ZIP, downloaded or extracted when the row is written. See [Images and files](#images-and-files). |
| `->requireApproval(fn () => User::role('manager')->get(), when: fn (User $user) => ! $user->isManager())` | off | Someone else approves an import before it is written. See [Approval](#approval). |
| `->schedules(disks: ['supplier'])` | off | Lets users run an import again on a schedule, from a link or a file on a disk. See [Scheduled imports](#scheduled-imports). |
| `->aiSuggestions()` | off | **Suggest with AI** in the column step and the value mapping, with Claude. See [AI suggestions](#ai-suggestions). |
| `->missingRecords(delete: true)` / `->missingRecords(update: ['is_active' => false])` | off | Lets users treat a file as the complete list: records that are not in it are deleted or changed. See [Records that are not in the file](#records-that-are-not-in-the-file). |
| `->currentData(false)` | on | Hides **Download current data** on the first screen. See [Editing in a spreadsheet](#editing-in-a-spreadsheet). |
| `->keepBlankCells()` | off | For existing records, an empty cell leaves the current value instead of clearing it. Users can change it in the column step. See [Updating existing records](#updating-existing-records). |
| `->updateColumns(['price', 'stock'])` | all columns | The columns that may change on existing records; the others are only used for new records. Users can change it in the column step. |

## How rows are checked

When the user presses **Check**, the rows are sent to the server in chunks of up to 1,000 rows (and under Livewire's 1 MB request limit). Each row goes through your importer exactly as Filament would import it — `remapData()`, `castData()`, `resolveRecord()`, the mapping checks, `beforeValidate()`, `validateData()`, `afterValidate()`, `beforeFill()`, `fillRecord()`, `afterFill()` — and stops before `saveRecord()`. Warp then reads which attributes of the record would change.

| Outcome | Meaning |
| --- | --- |
| **New** | `resolveRecord()` returned a record that does not exist yet. |
| **Changed** | An existing record whose attributes would change. The table shows the old value struck through and the new one; related records are shown by the attribute you resolve them by (`name`), not by their key. |
| **Unchanged** | An existing record that would stay the same. It is not written. |
| **Error** | A validation rule failed, a related record was not found, a number, date or yes/no value could not be read, or the importer threw `RowImportFailedException`. The message is shown next to the row and the cell is marked. |
| **Duplicate** | An earlier row in the file points at the same record (the same existing record, or the same attributes for a new one, ignoring case). Only the first row is imported. Fixing either row updates both. |
| **Skipped** | The user skipped the row, or `resolveRecord()` returned `null`. |

All of this happens inside a database transaction that is rolled back, so nothing your importer writes while checking — for example with `firstOrCreate()` in `resolveRecord()` — stays in the database. Hooks that act outside the database (sending mail, calling an API) do run, so keep those in `afterSave()`, which is not called while checking.

Numbers, dates and yes/no values are checked before your rules run. Filament reads `abc` in a numeric column as `0` and `いいえ` in a boolean column as `true`; Warp reports them as errors instead. The type of a column comes from the column itself (`numeric()`, `boolean()`, `enum()`, `relationship()`), the model's casts and the rules.

## Images and files

Product photos, logos, PDFs: a column of your importer can take a link or a file name, and Warp puts the file on a disk and gives your importer its path.

```php
// The importer: an ordinary column that stores a path
ImportColumn::make('image')->rules(['nullable', 'string', 'max:255']),

// The action
WarpImportAction::make()
    ->importer(ProductImporter::class)
    ->files('image', disk: 'public', directory: 'products'),
```

In the file, write either a link (`https://cdn.example.com/mug.png`) or a file name (`mug.png`). For file names, users add the files in one ZIP on the first screen of the wizard (**Images and files → Add a ZIP…**); names are matched without case and without the folders inside the ZIP.

- **Review**: the link's form and extension, and whether a name is in the ZIP, are checked; the table shows the link or name. Your importer sees the path the file will get (`products/3f2a9c….png`). The path comes from the link, or from the name and contents in the ZIP, so importing the same picture again is **Unchanged** and nothing is downloaded twice.
- **Import**: the file is downloaded or extracted and put on the disk before the row is written. A file that is too large, of the wrong type, not a file (an HTML error page) or that cannot be downloaded makes that row fail with the reason; other rows go on.
- **Undo** removes the files the import put on the disk.
- **Security**: only `http` and `https` links. Links to internal network addresses — `127.0.0.1`, `10.x`, `192.168.x`, `169.254.x` and other private or reserved ranges — are refused, including where a redirect leads, so an import cannot make your server call machines inside your network. Pass `privateNetwork: true` to allow them.

| Argument | Default | |
| --- | --- | --- |
| `disk` | `'public'` | The Laravel disk to put the files on. |
| `directory` | `''` | The directory on the disk. |
| `maxKilobytes` | `5120` | The largest file. |
| `types` | `['jpg', 'jpeg', 'png', 'gif', 'webp']` | The extensions accepted (from the link's path or the name in the ZIP). |
| `privateNetwork` | `false` | Allow links to internal network addresses. |

The ZIP is uploaded with Livewire's temporary uploads, so its size is limited by `livewire.temporary_file_upload.rules` (12 MB by default) and your server's upload limits; raise them for large photo sets. The ZIP is kept with the import and deleted with it. Files go on the parent sheet's columns, not on [child sheets](#parent-and-child-sheets). The PHP `zip` extension is needed for ZIPs.

## Approval

For data where a wrong import is expensive — prices, stock, accounts — let one person prepare the import and another approve it:

```php
WarpImportAction::make()
    ->importer(ProductImporter::class)
    ->requireApproval(
        approvers: fn () => User::query()->where('role', 'manager')->get(),
        when: fn (User $user): bool => $user->role !== 'manager',   // managers import directly
    ),
```

- The person importing goes through the wizard as usual. Instead of **Import**, the button says **Request approval for 42 rows**; they can add a note. The import waits — nothing is written — and the review becomes read only. **Withdraw request** takes it back.
- The approvers get a Filament database notification with a **Review** button that opens the import in the [history](#import-history) (the panel needs `->databaseNotifications()` and the user model Laravel's `Notifiable`). Imports waiting for them are also listed in their history.
- An approver sees the same review — every new row, every change with its old and new value, the records [not in the file](#records-that-are-not-in-the-file) — and presses **Approve and import**, which writes the rows from their browser (or the queue), or **Send back…** with a reason. Approvers cannot change rows.
- The person who asked is notified either way. A sent-back import shows the reason; they fix it and request approval again. Rows [fixed after the import](#importing-and-undoing) need approval too.
- The person importing never approves their own import, even if they are in the list. Both can open the result and undo it.

Events: `ImportApprovalRequested`, `ImportApproved` and `ImportRejected`, each with the `ImportRun` as `$run`, for mail or Slack. The approvers are fixed when the request is sent.

## Scheduled imports

A supplier updates a price list every night, a stock count lands on an SFTP server every hour, the sales team keeps a Google Sheets spreadsheet: import the file once by hand, then let Warp import it again on a schedule.

```php
WarpImportAction::make()
    ->importer(ProductImporter::class)
    ->schedules(disks: ['supplier']),   // links, and files on the "supplier" disk
```

1. Import the file as usual: map the columns, set up the clean-up and value mapping, import.
2. On the result, press **Run on a schedule…**. Give it a name, say where the file is — **At a link** or **In storage** (a disk you listed; the files on it are offered) — how often (every hour, day or week, at a time in the user's time zone) and what to do with rows that have errors.
3. **Scheduled imports** in the navigation lists the user's schedules with the last run and the next one. **Run now** runs one at once; the menu has **Last import**, **Edit** (name, file, time, pause) and **Delete**.

Each run:

- **Fetches the file**: CSV, TSV, text or `.xlsx` (the first sheet), UTF-8, UTF-16 or Shift_JIS, up to 100 MB. A Google Sheets share link (`https://docs.google.com/spreadsheets/d/…/edit#gid=0`) is turned into its CSV export; the spreadsheet must be shared with anyone who has the link.
- **Maps the columns by their header names**, so a file with the columns in another order, or with extra columns, still works. If a mapped header is gone, the run fails and names it.
- **Checks every row** exactly like the review, with the clean-up, value mapping, [update settings](#updating-existing-records), importer options and [records not in the file](#records-that-are-not-in-the-file) of the first import, as the user who created the schedule, in the same panel and tenant.
- **Rows with errors**: by default the run stops before writing anything and notifies the user; **Last import** (or the notification's **Review**) opens it in the review to fix the rows and import by hand. Choose **Skip those rows and import the rest** to write the good rows anyway; the list says how many were skipped.
- **Writes the rows** on the queue, like `->queue()`: the user gets the usual notification and the run is in the [history](#import-history), where it can be undone. If the import needs [approval](#approval), the run asks for it instead.
- A file with nothing new or changed is recorded as **Nothing new**, without a notification.

The list shows the outcome of the last run: **Imported** (with the counts), **Importing**, **Waiting for approval**, **Stopped**, **Nothing new** or **Failed** (the file could not be fetched or read, a column is missing, or the user can no longer import there). **Stopped** and **Failed** also send a database notification, so the panel needs `->databaseNotifications()`.

**What you need on the server**: Warp adds `warp-import:run-schedules` to Laravel's scheduler (every minute), so the usual cron entry `* * * * * php artisan schedule:run` must be there, and a queue worker for the runs. Nothing runs while the schedule is paused.

| Argument | Default | |
| --- | --- | --- |
| `$condition` | `true` | A bool or closure: who may create schedules (`fn () => auth()->user()->isAdmin()`). |
| `disks` | `[]` | The Laravel disks users may pick files from, for example an `sftp` or `s3` disk your supplier uploads to. The `.csv`, `.tsv`, `.txt` and `.xlsx` files on them are offered in the form. Empty: links only. |
| `urls` | `true` | Allow links. `false` allows files on the `disks` only. |
| `privateNetwork` | `false` | Allow links to internal network addresses. |

- **Security**: links are checked like [image links](#images-and-files) — `http` and `https` only, and internal network addresses (`127.0.0.1`, `10.x`, `192.168.x`, `169.254.x` and other private or reserved ranges) are refused when the schedule is saved and again on each run, including where a redirect leads. Only the disks you list can be read, and the form lists their files to everyone who may create schedules, so list a disk that holds import files only.
- Each user sees and changes only their own schedules (in the current tenant). A schedule runs with its creator's permissions; if they can no longer import there, the run fails.
- [Parent and child sheets](#parent-and-child-sheets) cannot be scheduled yet.
- `->maxRows()` applies to each run.

## Records that are not in the file

Some files are the complete list: this week's price list, the products a supplier still sells, the staff who work here. Turn on `missingRecords()` and users can let such a file decide what happens to the records it leaves out:

```php
WarpImportAction::make()
    ->importer(ProductImporter::class)
    ->missingRecords(update: ['is_active' => false]),   // or ->missingRecords(delete: true)
```

The column step then has a checkbox — **Also change the products that are not in the file (Active: no)** or **Also delete the products that are not in the file** — which is off for every import until the user ticks it. With it ticked:

- The review has a **Not in file** tab listing every record the file leaves out, with its current values. Untick a record to keep it. A record counts as in the file when any row points at it, including rows with errors and rows the user skipped, so a typo in a row does not delete its record.
- The import button says what will happen (**Import 3 rows · delete 47**), and pressing it asks once more, with the number, before anything is written.
- The file rows are written first, then the records that are not in the file are changed or deleted, each in its own savepoint. If the model has a policy, `update` or `delete` must allow each record; a record the database refuses to delete (a foreign key) is listed as failed with the reason.
- The result counts them (**Marked** or **Deleted**) and **Undo** puts them back: changed records get their old values, soft-deleted records are restored, and other deleted records are inserted again with their original key and values. Records the database deleted along with them (cascades) are not restored, so prefer `update:` or soft deletes for records with children.

The records considered are the same as for [Download current data](#editing-in-a-spreadsheet): all records of the importer's model in the current tenant, narrowed by `warpImportQuery()` on the importer if it has one. This only applies to the parent sheet, not to [child sheets](#parent-and-child-sheets).

## Editing in a spreadsheet

The first screen of the wizard has **Download current data (CSV)**. It downloads the records with one column per importer column, headed by the column labels — so when the file comes back, every column maps itself. Change prices in Excel or Google Sheets, import the file, and the review shows exactly the rows you changed as **Changed**; everything else is **Unchanged** and is not written. Undo works as for any import.

What goes into the file:

| Column | Written as |
| --- | --- |
| Related records (`->relationship()`) | the attribute you resolve them by (`Kitchen`, not `3`); many-to-many columns joined with their separator (`New,Sale`) |
| Choices (enums, `in` rules) | the value (`on_sale`), which the import accepts as it is |
| Yes / no | `yes` / `no` |
| Dates | `2026-01-05`, with the time when there is one (`2026-01-05 09:30:00`) |

Columns that do not fill a model attribute (only `fillRecordUsing()`) and the model's `$hidden` attributes, such as passwords, are left out. The file is built in the browser in steps of 500 records, with a progress count.

The records are all records of the importer's model, in the current [tenant](#multi-tenancy). To narrow them, add a static method to the importer:

```php
use Illuminate\Database\Eloquent\Builder;

class ProductImporter extends Importer
{
    public static function warpImportQuery(Builder $query): Builder
    {
        return $query->where('is_archived', false);
    }
}
```

The download is offered to users who may list the records: if the model has a policy, its `viewAny` must allow it. `->currentData(false)` turns it off. Child sheets are not included in the download.

## Updating existing records

A file that only carries prices should change prices, not wipe out names. Two settings in the column step decide what happens to records that already exist; new records always get every mapped column.

- **Leave the current value when a cell is empty.** With this ticked, an empty cell does nothing to an existing record: the column is not validated and not filled for that row, so a `required` rule does not fail and the old value stays. Without it, Warp does what Filament does: the empty value is validated and written (unless the column uses Filament's `ignoreBlankState()`).
- **Update existing** (a tick per column). Unticked columns are not validated or filled for existing records, in any row. A price list with the product name in it, for example, keeps every name as it is but still creates new products with their names.

In the review, the cells that an existing record keeps are shown faded and hatched ("Not updated: the existing record keeps its value"), and they never count as changes. Numbers, dates and yes/no values in these cells are not checked either, so a broken value in a column you do not update does not stop the row. Undo only puts back what was written.

Set the defaults on the action — users can still change them, and templates keep them:

```php
WarpImportAction::make()
    ->importer(ProductImporter::class)
    ->keepBlankCells()
    ->updateColumns(['price', 'stock']),
```

Both settings apply to the parent sheet and to [child sheets](#parent-and-child-sheets) alike; "Update existing" is chosen per sheet.

## Clean-up and value mapping

![The column mapping with the first values and the clean-up of each column](https://raw.githubusercontent.com/qalainau-labs/filament-warp-import-docs/main/art/mapping.png)

Each column has a clean-up applied to the value from the file before your importer sees it. The defaults depend on the type of the column; users change them in the mapping step.

| Clean-up | Example | Default for |
| --- | --- | --- |
| Trim spaces (also full-width spaces) | `␣ABC␣` → `ABC` | every column |
| Full-width letters and digits to half-width | `ＡＢＣ１２３` → `ABC123` | e-mail columns |
| Half-width katakana to full-width | `ｶﾀｶﾅ` → `カタカナ` | — |
| Numbers | `$1,200`, `€1,200`, `¥1,200` and `１，２００円` → `1200`, `(300)` and `△300` → `-300` | numeric columns |
| Dates | `Oct 20, 2026`, `20 October 2026`, `20-Oct-2026`, `2026/10/20 9:05`, `20261020`, `46315` (Excel), `令和8年10月20日`, `R8.10.20`, `2026年10月20日` → `2026-10-20` | date columns |
| Yes / no | yes／no, Y／N, true／false, on／off, 1／0, はい／いいえ, ○／×, 有効／無効 → `1` / `0` | boolean columns |
| Postal code | `〒1000001`, `100 0001` → `100-0001` | columns named `postal`, `zip` |
| Phone number | `０３ー１２３４ー５６７８` → `03-1234-5678` | columns named `phone`, `tel`, `fax` |
| Upper case, lower case | | e-mail columns: lower case |

A value that cannot be cleaned up stays as it is, so the row shows an error with the original value.

**Value mapping** (the ▾ menu on a column in the review) lists the different values of a column with the number of rows, unknown values first, and lets users choose what each becomes:

- **Choice columns** — an enum (from `enum()`, the model's cast or a `Rule::enum()` rule) or an `in` rule — offer the options by label. A value that matches an option's label or value apart from case and width is suggested (`販売中` → `on_sale`).
- **Related records** offer the records of the relationship, searched as you type, by the attribute you resolve them by. A value that matches one apart from case and width is suggested (`kitchen` → `Kitchen`).
- **Yes / no** columns offer yes and no.
- Columns with several values in a cell (`->multiple(',')`) are mapped value by value.

In the review, an error on a choice column, a related record or a yes / no column ("The selected prefecture is invalid.") shows the value from the file and a **Choose…** button. It opens the value mapping of that column at the value of the row, so users see the values the column accepts instead of guessing.

Only the rows with a changed value are checked again. **Find and replace** works the same way, on the value from the file, for part of the cell or the whole cell, in every row or only in rows with errors.

## AI suggestions

Files from other systems rarely use your names: "Client ID" for the customer code, "Mail" for the email, "Membership level" for the rank, and values like "England", "Deutschland" or "VIP". Turn on `aiSuggestions()` and users can ask Claude to map them:

```php
WarpImportAction::make()
    ->importer(CustomerImporter::class)
    ->aiSuggestions(),
```

```php
// config/services.php
'anthropic' => [
    'key' => env('ANTHROPIC_API_KEY'),
],
```

- **Columns**: **Suggest with AI** above the column table sends the fields that are not mapped yet (name, label, type, whether required, the first choices) and the headers that are not used yet with up to three values from each. The answers fill only fields that are still **Do not import**, and are marked **AI** until the user changes them.
- **Values**: in the value mapping of a column (**Map values…** or **Choose…** next to an error), **Suggest with AI** sends the values the column does not accept as written (up to 150) and the values it accepts (up to 300; for a relationship, the first 300 records by name). The answers fill only values without a mapping, are marked **AI**, and are written only when the user presses **Apply**; the rows are then checked again as usual.
- Nothing is imported on the AI's word: every suggestion goes through the same review as everything else. Answers that name a field, header or value that does not exist, or a header twice, are dropped.
- If the key is missing or Claude does not answer, the user sees a short message and carries on by hand. Each user can ask 20 times a minute.

| Argument | Default | |
| --- | --- | --- |
| `$condition` | `true` | A bool or closure: who sees the button. |
| `model` | `'claude-haiku-4-5'` | The Claude model. Haiku is fast and costs a fraction of a cent per question; `claude-sonnet-5` thinks harder. |
| `samples` | `true` | Send the first values of each column with the headers. `false` sends the headers only (value suggestions always send the values they are about). |
| `using` | `null` | A class implementing `Qalainau\FilamentWarpImport\Ai\SuggestsMappings`, to use another service or your own rules instead of Claude. |

**Privacy**: headers and sample values, and in the value mapping the values of one column, are sent to Anthropic (or to your `using` class) when the user presses the button — never on their own. Turn samples off, or leave the feature off, for data that must not leave your servers.

```php
use Qalainau\FilamentWarpImport\Ai\SuggestsMappings;

class OpenAiSuggester implements SuggestsMappings
{
    public function columns(array $fields, array $headers): array
    {
        // $fields: [['name' => 'code', 'label' => 'Code', 'type' => 'text', 'required' => true], ...]
        // $headers: [['index' => 0, 'header' => 'Client ID', 'samples' => ['C0003', 'C9101']], ...]
        return ['code' => 0];   // field name => header index
    }

    public function values(array $field, array $values, array $options): array
    {
        // $values: ['England', 'VIP'], $options: [['value' => 'gb', 'label' => 'United Kingdom'], ...]
        return ['England' => 'gb'];   // file value => accepted value
    }
}

->aiSuggestions(using: OpenAiSuggester::class)
```

Throw a `RuntimeException` with a message for the user when the service fails.

## Importing and undoing

![The result of an import with the option to undo it](https://raw.githubusercontent.com/qalainau-labs/filament-warp-import-docs/main/art/undo.png)

**Import** writes the new and changed rows with your importer — the same steps as Filament, including `saveRecord()`, `saveRelationships()` and the save hooks. The browser writes 200 rows at a time and shows the progress. **Pause** stops after the current step; rows written so far stay written, and **Resume** continues. Closing the modal or the tab pauses the import too; open it again from the recent imports in the wizard or from the [history](#import-history).

Each step runs in one transaction and each row in a savepoint: a row that fails — a database error, an exception in a hook, `RowImportFailedException`, or a model event that returns `false` from `saving` — is rolled back on its own and listed as failed, and the other rows are written. Rows that run into a busy database (`database is locked`, a deadlock) are tried again. A row is checked once more when it is written, so a record that changed since the review is validated against its current state.

The result lists created, updated and failed rows. **Download failed rows** gives a CSV with the original headers and a column with the reasons, ready to fix and import again.

**Fixing rows after the import.** The result also counts the rows that were **Not imported**: rows you skipped, rows with errors and rows that failed while writing (rows that were unchanged need nothing and are not counted). **Fix the rows that were not imported** opens them in the review table with the same tools as before the import — change cells, map values, find and replace, include skipped rows — and **Import N rows** writes them into the same import:

- Failed rows are checked again first, so a row that failed because something was missing (a category someone has added since, a record that was locked) now shows whether it would pass.
- Rows that were already imported are not shown and cannot be changed (edit those records in your resource as usual). Value mapping and find and replace only touch the rows that were not imported.
- A fixed row that points at a record this import already wrote is flagged as the same record and not written twice.
- The counts on the result and in the history add up, and **Undo** takes back everything the import wrote, the first rows and the later ones together. The undo period starts again from the last write.
- The column mapping cannot be changed at this point (the rows already written used it). **Back to the result** leaves without writing; nothing is lost, and you can come back to it from the result or the history.

With `->queue()`, the browser shows the progress while a job writes the rows. The job acts as the user who started the import, in the same panel and tenant (see [Multi-tenancy](#multi-tenancy)), re-queues itself every 40 seconds so that it never hits a worker timeout, and sends a Filament database notification with your importer's `getCompletedNotificationBody()` when it is done (the user model needs Laravel's `Notifiable` trait, and the panel `->databaseNotifications()`). Users can close the window.

**Undo** is offered on the result and in the history until the undo period ends (30 days by default). It deletes the records the import created and writes back the old values of the records it changed. A record that someone changed after the import is not touched; it is listed as **Kept**. Values are compared through the model's casts (a `date` column read back as `2026-01-05` equals the stored `2026-01-05 00:00:00`) and written back without mutators, so a hashed password is not hashed again. Children are removed before their parents.

Undo restores the record's own attributes. Records attached with `belongsToMany` (`saveRelationships()` calls `attach()`) and other side effects of your hooks are not undone.

Events: `Qalainau\FilamentWarpImport\Events\ImportCompleted` and `ImportUndone`, both with the `ImportRun` as `$run`. `ImportCompleted` is dispatched again each time fixed rows are imported into an import that was already completed; `$run->created_rows` and the other counts are the totals so far.

## Parent and child sheets

![Orders and their items in two sheets of one workbook](https://raw.githubusercontent.com/qalainau-labs/filament-warp-import-docs/main/art/orders.png)

An order and its items, an invoice and its lines, a product and its variants: put them in two sheets of one workbook and import both at once.

```php
WarpImportAction::make()
    ->importer(OrderImporter::class)
    ->child(
        OrderItemImporter::class,
        relationship: 'order',      // OrderItem::order(), a belongsTo
        parentColumn: 'order_no',   // the column of OrderImporter that the items refer to
        sheet: 'Items',             // the sheet the wizard picks first
    );
```

The child importer is an ordinary importer for the child model. It needs **no column for its parent**: in the wizard, users pick the sheet with the items and the column in it that holds the order number (guessed from the parent column's name, label and `guess()`). Each item is linked to the order row with the same order number, ignoring case and spaces, and Warp associates the order with `$item->order()->associate($order)` before your `fillRecord()` runs.

- In the review, each sheet has a tab with its own counts. An item whose order row will not be imported — it has an error, is skipped, is a duplicate, or there is no order with that number — shows as an error with the reason, and comes back as soon as the order row is fixed.
- Orders are written before items. An item whose order failed to save fails too.
- Undo removes the items before the orders.
- `->child()` can be called more than once for several child sheets. Templates cover the main sheet.

## Templates

Users who import the same kind of file every week save the mapping as a template (**Save as template…** in the review). A template keeps the column mapping by header name — so it fits a file with the columns in another order — the clean-up, and the value mapping.

When a file is opened whose headers are the same as a template's (in any order, ignoring case, width and spaces), the template is applied automatically and the wizard says so. Others can be chosen from the list above the mapping. Templates are private or shared with everyone who can import with the same importer (in the same [tenant](#multi-tenancy)); only their owner can delete them.

## Import history

Register the plugin in the panel to add an **Imports** page that lists the user's imports:

```php
use Qalainau\FilamentWarpImport\WarpImportPlugin;

$panel->plugin(
    WarpImportPlugin::make()
        ->navigationGroup('Settings')
        ->navigationSort(90)
        ->navigationLabel('Imports')    // default: "Imports"
        ->navigationIcon('heroicon-o-arrow-up-tray'),
);
```

Each import shows when it started, the file, the kind of records, its status, the created, updated and failed rows and until when it can be undone. **Open** shows its result, resumes it if it was paused, or undoes it. Users see only their own imports, and in a panel with [tenancy](#multi-tenancy) only those of the current tenant. Without the plugin, the wizard still lists the five most recent imports of its importer.

Imports left in the review for more than a day, and imports whose undo period has ended, are removed with Laravel's pruning (a paused import is kept until it is finished). The model is in a package, so name it in the schedule:

```php
use Illuminate\Support\Facades\Schedule;
use Qalainau\FilamentWarpImport\Models\ImportRun;

Schedule::command('model:prune', ['--model' => [ImportRun::class]])->daily();
```

Pruning removes the rows kept for review and undo; the imported records are not touched.

## Multi-tenancy

In a panel with Filament's [multi-tenancy](https://filamentphp.com/docs/5.x/users/tenancy) (`->tenant(Team::class)`), each team imports into its own records and sees only its own imports and templates. Your importer stays the same: like Filament's `ImportAction`, Warp Import lets the panel's resources do the scoping, so there is no tenant code to write in `resolveRecord()`.

### Setting up the panel

A complete example: teams, users in several teams, and products that belong to a team.

```php
// app/Providers/Filament/AppPanelProvider.php
use App\Models\Team;
use Qalainau\FilamentWarpImport\WarpImportPlugin;

public function panel(Panel $panel): Panel
{
    return $panel
        ->id('app')
        ->path('app')
        ->login()
        ->tenant(Team::class, slugAttribute: 'slug')  // ownershipRelationship: 'team' is the default
        ->discoverResources(in: app_path('Filament/Resources'), for: 'App\Filament\Resources')
        ->databaseNotifications()                      // only needed for ->queue()
        ->plugin(WarpImportPlugin::make());            // the import history, per team
}
```

```php
// app/Models/User.php
use Filament\Models\Contracts\FilamentUser;
use Filament\Models\Contracts\HasTenants;

class User extends Authenticatable implements FilamentUser, HasTenants
{
    public function teams(): BelongsToMany
    {
        return $this->belongsToMany(Team::class);
    }

    public function getTenants(Panel $panel): Collection
    {
        return $this->teams;
    }

    public function canAccessTenant(Model $tenant): bool
    {
        return $this->teams()->whereKey($tenant)->exists();
    }

    public function canAccessPanel(Panel $panel): bool
    {
        return true;
    }
}
```

```php
// app/Models/Product.php
class Product extends Model
{
    public function team(): BelongsTo
    {
        return $this->belongsTo(Team::class);
    }

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);   // Category has a team() relationship too
    }
}
```

```php
// app/Filament/Resources/Products/Pages/ListProducts.php — the importer has no tenant code
WarpImportAction::make()->importer(ProductImporter::class)

// app/Filament/Imports/ProductImporter.php
public function resolveRecord(): ?Product
{
    return Product::firstOrNew(['sku' => $this->data['sku']]);  // finds only the current team's product
}
```

`ProductResource` and `CategoryResource` are ordinary resources in the panel. That is what makes the queries of `Product` and `Category` scoped to the current team.

### What is kept per tenant

- **Records that are updated.** `resolveRecord()` finds records through the resource's tenant scope, so a row with the same SKU as another team's product creates a new product for the current team instead of changing theirs.
- **New records** belong to the current team. Filament sets the `team` relationship when the record is created; when records belong to teams through a pivot table (`belongsToMany`), the team is attached after the record is created.
- **Related records.** `->relationship()` columns find the category by name within the current team, and the value mapping lists and suggests only the current team's categories.
- **Imports.** Recent imports, the history page, opening an import by its ID, resuming and undoing only work in the team that started the import. Opening another team's import returns 404.
- **Templates** are saved for the current team, private or shared within it. Another team cannot see, apply or delete them, and can use the same name.
- **Undo** finds the imported records through the same scope.
- **Queued imports.** With `->queue()`, the job restores the panel, the team and the user before it writes rows, so the scoping above applies in the worker as well. If the team has been deleted, or the user who started the import is no longer in it (`canAccessTenant()` returns false, checked before every 500 rows), the job stops writing: the import is paused as if the user had pressed Pause (the rows written so far stay), and the user gets a notification that says why.

In a panel without tenancy, nothing changes: imports and templates are stored without a tenant, and a panel without tenancy does not show the imports of a tenant panel.

### Columns and migrations

Warp Import's own tables already have `tenant_type` and `tenant_id` columns; there is nothing to add. These columns, like `user_id`, are strings, so teams and users with UUID or ULID keys work on every database. Your models need what Filament's tenancy needs anyway: a `team_id` column and a `team()` relationship (or a pivot table, see below). Make unique indexes per team, for example `$table->unique(['team_id', 'sku'])`, so two teams can have the same SKU.

### Records shared by all teams, and other relationship names

Warp Import follows each resource's tenancy settings:

```php
class CountryResource extends Resource
{
    // Countries are the same for every team: not scoped, and new countries are not given a team
    protected static bool $isScopedToTenant = false;
}

class InvoiceResource extends Resource
{
    // Invoices belong to a team through the company() relationship instead of team()
    protected static ?string $tenantOwnershipRelationshipName = 'company';
}

class DocumentResource extends Resource
{
    // Documents belong to teams through a pivot table: a belongsToMany teams() relationship
    protected static ?string $tenantOwnershipRelationshipName = 'teams';
}
```

To change the relationship name for the whole panel, use `->tenant(Team::class, ownershipRelationship: 'company')`. Imports and templates themselves are always kept per team.

### Things to keep in mind

- **The scope comes from a resource.** Filament scopes a model only when the panel has a resource for it. If you import into a model without a resource, scope it yourself in `resolveRecord()` (`Product::where('team_id', Filament::getTenant()->getKey())->firstOrNew([...])`). A resource that is not shown in the navigation (`$shouldRegisterNavigation = false`) is enough.
- **Related records without a resource.** When a `->relationship()` column points at a model that has no resource in the panel (a `Supplier` that belongs to a team, say), Warp Import keeps it to the current team itself, through the panel's ownership relationship (`team()` by default): the value suggestions, the check before writing and the import only find the team's suppliers. If that model has no relationship to the team, the wizard stops with an error that says so, instead of looking through every team's records. For records shared by all teams (countries, units), resolve the column yourself with `->relationship(resolveUsing: fn (string $state) => Country::firstWhere('name', $state))`; Warp then leaves it to your code.
- **`unique` and `exists` rules ignore the scope.** Laravel's validation rules query the table directly. In an importer's `->rules()`, add the team: `Rule::unique('products', 'sku')->where('team_id', Filament::getTenant()->getKey())`.
- **Your own queries** in importer hooks (`beforeSave()`, `afterCreate()`…) are scoped only when they use a model with a resource in the panel.
- **The notification** of a queued import goes only to the person who started it, so other users and other members of the team never see it. If that person is in several teams, it shows in their notification bell whichever team they have open: while they work in Osaka, they can see "Import finished" for an import they started in Tokyo. This is how Filament's own database notifications work. To say which team the import was for, add the team's name in your importer's `getCompletedNotificationBody()`. The job restores the import's team before it sends the notification, so `Filament::getTenant()` returns it:

  ```php
  use Filament\Actions\Imports\Models\Import;
  use Filament\Facades\Filament;

  public static function getCompletedNotificationBody(Import $import): string
  {
      $team = Filament::getTenant(); // the team the import was started in

      return ($team ? "{$team->name}: " : '').number_format($import->successful_rows).' rows imported.';
  }
  ```
- **Queued imports remember the panel by its ID.** If you rename the panel's `->id()` while imports are queued, the job uses the first panel whose tenant model matches the import's tenant.
- A record moved to another team after the import is not touched by undo.

## Security

- **Who can import** is decided by where you put the wizard: the action's `->visible()` / `->authorize()`, or the page's `canAccess()`. The importer runs as the signed-in user. As with Filament's `ImportAction`, `resolveRecord()` runs without policy checks; add your own checks there if users may only import into some records.
- **The importer cannot be changed from the browser.** The wizard keeps the importer class and options in locked Livewire properties, and every request checks that the import belongs to the signed-in user, to the same importer and to the current tenant ([Multi-tenancy](#multi-tenancy)).
- **Files are read in the browser** and only the cell values are sent. Files larger than 100 MB, cells longer than 32,767 characters and more than 5,000,000 cells are refused; each request carries at most 2,000 rows.
- **Templates** are visible to their owner and, if shared, to users of the same importer in the same tenant. Only the owner can delete a template.
- The rows of an import are kept until the undo period ends (with `->undoable(false)`, only the result is kept). They contain the values from the file and the old values of changed records; prune them as shown above.

## Translations

The wizard is translated into English, Japanese, German, French, Spanish, Italian, Brazilian Portuguese (`pt_BR`), Korean, and Chinese, simplified (`zh_CN`) and traditional (`zh_TW`). It follows the panel's locale (`app()->getLocale()`). Publish the translations to change or add languages:

```bash
php artisan vendor:publish --tag=filament-warp-import-translations
```

Messages from validation rules come from your application's translations.

## Keyboard

In the review table:

| Key | Action |
| --- | --- |
| Arrow keys | Move between cells |
| Page Up / Page Down | Move a page up or down |
| Home / End | First or last column; with Ctrl or ⌘, first or last row |
| Enter or F2 | Change the cell's value from the file |
| Enter / Esc (while changing) | Save / cancel |
| Space | Import or skip the row |

## Limitations

- Formulas in Excel files are imported as their last calculated values, and only the first 16,384 columns are read. `.xls` (Excel 97–2003) is not supported; save the file as `.xlsx` or CSV.
- Undo does not detach records attached with `belongsToMany` and does not undo side effects of your hooks.
- `fillRecordUsing()` closures that change attributes other than the column's own are shown under "Problems" as extra changes, by attribute name.
- Hooks that act outside the database run while rows are checked (see [How rows are checked](#how-rows-are-checked)).

## Troubleshooting

**Nothing happens when a file is dropped.** Run `php artisan filament:assets` after installing or updating, and check that the page shows no JavaScript errors.

**Every row is "Changed".** Something in `resolveRecord()` or `fillRecord()` sets a value that differs from what is stored on every row, for example a timestamp or a value with a different format. The changed fields are shown in the review.

**"database is locked" with SQLite.** Warp retries rows that hit a locked database, but long imports into SQLite next to other writers can still fail. Set a busy timeout for the connection (`'busy_timeout' => 5000` in `config/database.php`).

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## Support

Ask a question or report a bug at https://warpui.com/support?plugin=import. Do not post your license key in your question.

## License

Warp Import is commercial software. See [LICENSE.md](LICENSE.md) for the license terms.
