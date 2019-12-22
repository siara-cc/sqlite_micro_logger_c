# Sqlite µLogger

Sqlite µLogger is a Fast and Lean database logger that can log data into Sqlite databases even with SRAM as low as 2kb.

# Features

- Low Memory requirement: `page_size` + some stack
- Can write to Sqlite databases with just 2kb RAM with 512 bytes page size
- Can do quick binary search on RowID or Timestamp without any index in logarithmic time
- Recovery possible in case of power failure
- Rolling logs are possible (not implemented yet)
- Can use any media using any IO library/API or even network filesystem
- DMA writes possible (not shown)

# Applications

- Logging sensor data on Microcontrollers with RAM >= 2 kilobytes
- Fast database logging on VMs and Containers having short TTL (turnaround time and memory footprint are important)
- Recording data received from `IoT farms`.

# Performance

This library is found to be faster than the official `sqlite` API functions for logging data, even using Prepared Statements.  However, this was just random testing and no formal test beds or benchmarks are available.  Also, it is to be kept in mind that the speed is because `scope` of this library is limited and it supports only a fraction of what the official library supports.

For reading, the official `sqlite` functions would be faster and less IO intensive as this library does not do any caching.  This library is targetted to run on Microcontrollers having just 2kb RAM.  Even then, it can also be used on Desktops and VMs (such as Docker and Kubernates) where there are constraints on RAM availability.

# Getting started

Clone this repository and compile with `cmake .` and `make` from `Terminal` or `Command Prompt`.

It creates an executable `test_ulog_sqlite` that helps you to check out most the API (In Windows, it would create test_ulog_sqlite.exe). For instance, to create a simple Sqlite database, type:

`./test_ulog_sqlite -c hello.db 512 2 "Hello,World"`

and `hello.db` is created with a table of one row and two columns having the values `Hello` and `World`.  If you open the database in `sqlite3` command line program and do `.dump`, you can see the following output:

```sql
sqlite> .dump
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE t1 (c001,c002);
INSERT INTO t1 VALUES('Hello','World');
COMMIT;
```

Following are the entire command line parameters of `test_ulog_sqlite` executable:

```markdown
Testing Sqlite Micro Logger
---------------------------

Sqlite Micro logger is a library that logs records in Sqlite format 3
using as less memory as possible. This utility is intended for testing it.

Usage
-----

test_ulog_sqlite -c <db_name.db> <page_size> <col_count> <csv_1> ... <csv_n>
    Creates a Sqlite database with the given name and page size
        and given records in CSV format (no comma in data)

test_ulog_sqlite -a <db_name.db> <page_size> <col_count> <csv_1> ... <csv_n>
    Appends to a Sqlite database created using -c above
        with records in CSV format (page_size and col_count have to match)

test_ulog_sqlite -v <db_name.db>
    Attempts to recover <db_name.db> if not finalized

test_ulog_sqlite -r <db_name.db> <rowid>
    Searches <db_name.db> for given row_id and prints result

test_ulog_sqlite -b <db_name.db> <col_idx> <value>
    Searches <db_name.db> and column for given value using
        binary search and prints result. col_idx starts from 0.

test_ulog_sqlite -n
    Runs pre-defined tests and creates databases (verified manually)
```

# API

For creating database through your C/C++ code, a C language context `struct` is to be initialized and supplied to the API functions.  The `struct dblog_write_context` and API for writing databases along with usage description is shown below:

```c++

struct dblog_write_context {
  byte *buf;          // working buffer of size page_size
  byte col_count;     // No. of columns (whether fits into page is not checked)
  byte page_size_exp; // 9=512, 10=1024 and so on upto 16=65536
  byte max_pages_exp; // Maximum data pages (as exponent of 2) after which
                      //   to roll. 0 means no max. Not implemented yet.
  byte page_resv_bytes; // Reserved bytes at end of every page (say checksum)
  // read_fn and write_fn should return no. of bytes read or written
  int32_t (*read_fn)(struct dblog_write_context *ctx, void *buf, uint32_t pos, size_t len);
  int32_t (*write_fn)(struct dblog_write_context *ctx, void *buf, uint32_t pos, size_t len);
  int (*flush_fn)(struct dblog_write_context *ctx); // Success if returns 0
  // following are running values used internally
  uint32_t cur_write_page;
  uint32_t cur_write_rowid;
  byte state;
  int err_no;
};

// Initializes database - writes first page
// and makes it ready for writing data
int dblog_write_init(struct dblog_write_context *wctx);

// Initializes database - writes first page
// and makes it ready for writing data
// Uses the given table name and DDL script
// Table name should match that given in script
int dblog_write_init_with_script(struct dblog_write_context *wctx,
      char *table_name, char *table_script);

// Initalizes database - resets signature on first page
// positions at last page for writing
// If this returns DBLOG_RES_NOT_FINALIZED,
// call dblog_finalize() to first finalize the database
int dblog_init_for_append(struct dblog_write_context *wctx);

// Creates new record with all columns null
// If no more space in page, writes it to disk
// creates new page, and creates a new record
int dblog_append_empty_row(struct dblog_write_context *wctx);

// Creates new record with given column values
// If no more space in page, writes it to disk
// creates new page, and creates a new record
int dblog_append_row_with_values(struct dblog_write_context *wctx,
      uint8_t types[], const void *values[], uint16_t lengths[]);

// Sets value of column in the current record for the given column index
// If no more space in page, writes it to disk
// creates new page, and moves the row to new page
int dblog_set_col_val(struct dblog_write_context *wctx, int col_idx,
                          int type, const void *val, uint16_t len);

// Gets the value of the column for the current record
// Can be used to retrieve the value of the column
// set by dblog_set_col_val
const void *dblog_get_col_val(struct dblog_write_context *wctx, int col_idx, uint32_t *out_col_type);

// Flushes the corrent page to disk
// Page is written only when it becomes full
// If it needs to be written for each record or column,
// this can be used
int dblog_flush(struct dblog_write_context *wctx);

// Flushes data written so far and Updates the last leaf page number
// in the first page to enable Binary Search
int dblog_partial_finalize(struct dblog_write_context *wctx);

// Based on the data written so far, forms Interior B-Tree pages
// according to SQLite format and update the root page number
// in the first page.
int dblog_finalize(struct dblog_write_context *wctx);

// Returns 1 if the database is in unfinalized state
int dblog_not_finalized(struct dblog_write_context *wctx);

// Reads page size from database if not known
int32_t dblog_read_page_size(struct dblog_write_context *wctx);

// Recovers database pointed by given context
// and finalizes it
int dblog_recover(struct dblog_write_context *wctx);
```

Similarly the `struct dblog_read_context` and API for reading databases created using this library, along with usage description is shown below:

```c++
struct dblog_read_context {
  byte *buf;
  // read_fn should return no. of bytes read
  int32_t (*read_fn)(struct dblog_read_context *ctx, void *buf, uint32_t pos, size_t len);
  // following are running values used internally and need not be supplied
  uint32_t last_leaf_page;
  uint32_t root_page;
  uint32_t cur_page;
  uint16_t cur_rec_pos;
  byte page_size_exp;
  byte page_resv_bytes;
};

// Reads a database created using this library,
// checks signature and positions at the first record.
// Cannot be used to read SQLite databases
// not created using this library or modified using other libraries
int dblog_read_init(struct dblog_read_context *rctx);

// Returns number of columns in the current record
int dblog_cur_row_col_count(struct dblog_read_context *rctx);

// Returns value of column at given index.
// Also returns type of column in (out_col_type) according to record format
// See https://www.sqlite.org/fileformat.html#record_format
// For text and blob columns, pass the type to dblog_derive_data_len()
// to get the actual length
const void *dblog_read_col_val(struct dblog_read_context *rctx, int col_idx, uint32_t *out_col_type);

// For text and blob columns, pass the out_col_type
// returned by dblog_read_col_val() to get the actual length
uint32_t dblog_derive_data_len(uint32_t col_type);

// Positions current position at first record
int dblog_read_first_row(struct dblog_read_context *rctx);

// Positions current position at next record
int dblog_read_next_row(struct dblog_read_context *rctx);

// Positions current position at previous record
int dblog_read_prev_row(struct dblog_read_context *rctx);

// Positions current position at last record
// The database should have been finalized
// for this function to work
int dblog_read_last_row(struct dblog_read_context *rctx);

// Performs binary search on the inserted records
// using the given Row ID and positions at the record found
// Does not change position if record not found
int dblog_srch_row_by_id(struct dblog_read_context *rctx, uint32_t rowid);

// Performs binary search on the inserted records
// using the given Value and positions at the record found
// Changes current position to closest match, if record not found
// is_rowid = 1 is used to do Binary Search by RowId
int dblog_bin_srch_row_by_val(struct dblog_read_context *rctx, int col_idx,
      int val_type, void *val, uint16_t len, byte is_rowid);
```

The read API can only be used with databases created using this library.

# Read / Write IO

The `read_fn`, `write_fn` and `flush_fn` are to be supplied with the context to this library.  These are callback functions called by the library to read and write pages.  This mechanism allows any IO library to be used.  This also allows the database to reside in any medium such as `hard disk`, `ssd`, `sd crads` or even `network` folders.  `Opening` and `closing` databases are done outside this library.

This also allows the user to choose between different flavours of IO.  For example, it allows a user choice to use either `open` or `fopen` functions.

# Ensuring integrity

Before writing into the disk (i.e. before calling the callback `write_fn`), simple checksums are calculated and stored in the page.  These checksums are used to discard corrupted pages when finalizing the database.

# Limitations

Following are limitations of this library:

- Only one table per Sqlite database
- Length of table script limited to (`page size` - 100) bytes
- `Select`, `Insert` are not supported.  Instead C API similar to that of Sqlite API is available.
- Index creation and lookup not possible (as of now)

However, the database created can be opened using the `sqlite3` command line utility or any Sqlite GUI utility such as [DB Browser for Sqlite](https://sqlitebrowser.org) and further operations such as index creation and summarization can be carried out from there as though its a regular Sqlite database.  But after doing so, it may not be possible to use it with this library any longer.

# Known issues

- `dblog_set_col_val()` may have defects when writing to the same column with different values consecutively.  Instead, please use `dblog_append_row_with_values()`, which is the fastest way of logging to databases.

# Future plans

- Index creation when finalizing a database
- Allow modification of records
- Rolling logs
- Show how this library can be used in a multi-core, multi-threaded environment
- Add encryption support
- Get it working for esp-idf

# Related Projects

- [Sqlite Micro Logger for Arduino](https://github.com/siara-cc/sqlite_micro_logger_arduino)

# Support

If you find any issues, please create an issue here or contact the author (Arundale Ramanathan) at arun@siara.cc.
