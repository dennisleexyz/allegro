# Allegro

This is a set of shell scripts for importing data from financial institutions into [ledger][]/[hledger][] [plain text accounting][] systems.

[ledger]: https://ledger-cli.org/
[hledger]: https://hledger.org/
[plain text accounting]: https://plaintextaccounting.org/

## Etymology

Both "allegro" and *ledger* are fuzzy-matched by "legr". The name came to mind presumably due to my musical background, and I thought its meaning would be a good fit for the goal I had set out to achieve. It grants alphabet privilege by sorting toward the top of alphabetized lists.

## Dependencies

- [ledger](https://ledger-cli.org/)
- [ledger-autosync](https://github.com/egh/ledger-autosync/) for OFX formatted data

The scripts for the most part aim to target a POSIX environment. Aside from the following exceptions, any incompatibilites are to be considered bugs resulting from development in a GNU/Linux environment.

- `venmo csv` uses `awk --csv` from One True Awk, Gawk (default on OpenBSD, GNU/Linux).

## Description

The tooling is designed with a particular filesystem layout in mind. Directories group transaction logs and other data by financial institution. The following example will be referenced throughout this section.

```
.
├── bofa
│   ├── ckg
│   │   ├── acct
│   │   ├── stmt.qfx
│   │   └── url
│   └── crc
│       ├── 2025-02-08--2025-03-03.qfx
│       ├── acct
│       └── url
├── c1
│   ├── 2025-01-26--2025-02-22.ofx
│   ├── acct
│   └── url
├── citi
│   ├── acct
│   ├── fid
│   ├── Since Feb 21, 2025.OFX
│   └── url
├── fidelity
│   ├── 2024q4.csv
│   ├── csv
│   ├── date-format
│   └── url
├── pp
│   ├── 2025-02.csv
│   └── url
└── venmo
    ├── 2025-02.csv
    ├── csv
    ├── date-format
    └── url
```

The `convert` utility wraps
- `ledger convert` for institutions providing CSV data (Fidelity, PayPal, Venmo)
	- calling an executable in the institution's directory named `csv` taking all `.csv` files in that directory (non-recursively) as command-line arguments to pre-process them.
	- reading a plain text file in the institution's directory named `date-format` containing an *strftime*(3) format string representing the format of the date field provided in the institution's CSV file data.
	- setting the account to `assets:$a` where `$a` is the name of the insitution's directory.
- `ledger-autosync` for institutions providing OFX data (Bank of America, Capital One, Citi)
	- optionally accepting, as the first command-line argument, the path to a file containing rules for matching `payee, account` pairs delimited by a tab character ('\t') and otherwise setting the account to `Expenses:Unknown` (hardcoding this value and using capital letters since `ledger convert` seems to do likewise).
	- reading a plain text file in the institution's directory named `acct` containing an account name.
	- setting the OFX FID to the contents of, if it exists, a plain text file in the institution's directory named `fid`. This is required by `ledger-autosync` for OFX files not providing FID (like Citi's).

By keeping opening balances in a separate file, you can easily delete and reinitialize your journal without losing data. This may be useful for regenerating it with different payee/account matching rules. If `start.j` contains the starting balances of your accounts in ledger format,

```
# first invocation
cat start.j <(./convert rules.tsv) > ledger.dat

# subsequent invocations
# preview changes
./convert rules.tsv | less -F
./convert rules.tsv | ledger -f - [options] [command] [arguments]
# commit changes
./convert rules.tsv >> ledger.dat
```

The `dl` utility recursively searches the directories provided as command line arguments for files named "url" and opens them in `$BROWSER` if set, or *xdg-open*(1) otherwise. For example, `dl pp` opens `https://www.paypal.com/reports/dlog`, and `dl bofa` is equivalent to `dl bofa/ckg bofa/crc`.

The `ofxmv` and `venmv` utilities accept OFX and Venmo CSV files respectively as command-line arguments and rename them to ISO 8601 date format based on extracting either the file's `DTSTART` and `DTEND` fields (OFX) or date information provided in the existing filename (Venmo CSV). The format is an interval `YYYY-MM-DD--YYYY-MM-DD.csv` for OFX and `YYYY-MM[--[YYYY-]MM].csv` for Venmo CSV. No action is performed on directories or files which do not contain the information in the expected format. Existing files are not clobbered (overwritten).

The `payees` utility accepts data in ledger journal format as `stdin` and outputs a sorted, deduplicated list of payees, with the sequence " #" and everything after it trimmed. This is designed for constructing your payee/account matching rules. I found that this stripped cheque numbers and location information (for businesses with multiple locations) without losing anything of value. Test it for yourself with your own data: `diff --color -u0 <(ledger payees) <(ledger print | ./payees)`.

The `venmo/rm` utility accepts files as command-line arguments and deletes them if they are exactly 31 lines long (the length of a CSV file downloaded from Venmo containing zero transaction records).

## Bugs

- `ledger convert`, `ledger-autosync`, *xdg-open*(1) don't play nice with streaming inputs into their `stdin` or reading multiple command-line operands.
	- For `convert` `*.csv`, we stream input into a temporary file and then make `ledger convert` read from that instead.
	- For `convert` `*.[oq]fx`/`dl`, we iterate (loop) over operands and make individual calls to `ledger-autosync`/*xdg-open*(1) to handle them separately.
