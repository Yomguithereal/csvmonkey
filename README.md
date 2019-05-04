# csvmonkey

This is a header-only vectorized, lazy-decoding, zero-copy CSV file parser. The
C++ version can tokenize ~1.8 GiB/sec of raw input in a single thread. For a
basic summing task, the Python version is ~7x faster than `csv.reader` and ~18x
faster than `csv.DictReader`, while maintaining a similarly usable interface.

Requires a CPU supporting Intel SSE4.2 and a C++11 compiler that bundles
`smmintrin.h`. For non-SSE4.2 machines, a reasonable fallback implementation is
also provided. It still requires a ton of work. For now it's mostly toy code.

As of writing, csvmonkey comfortably leads <a
href="https://bitbucket.org/ewanhiggs/csv-game">Ewan Higg's csv-game</a>
microbenchmark of 24 CSV parsers with a 30% margin.


## How It Works

* **Vectorized**: scanning for byte values that influence parser state is done
  using Intel SSE 4.2's PCMPISTRI instruction. This instruction supports many
  operating modes, including one that will locate the first occurence of up to
  four distinct values within a 16 byte vector. This allows searching 16 input
  bytes to locate the end of line, escape, quote, or field separators in a
  single CPU instruction.

* **Zero Copy**: the user is responsible for providing the memory buffers the
  parser will search. The output of the parser is an array of column offsets
  within a row, each containing a flag to indicate whether any escape
  character was detected.

* **Lazy Decoding**: input data is not copied or unescaped until it is
  requested. Since a flag is stored to indicate the presence of escapes, a fast
  path is possible that avoids any bytewise decode loop in the usual case where
  no escape is present.

* **Header Only**: the parser has no third-party dependencies, just some
  templates defined in ``csvmonkey.hpp``.


## Python Usage

1. `pip install -e . # or similar`
1. `import csvmonkey`
1. `csvmonkey.from_path()` for a memory-mapped file, `csvmonkey.from_file()`
   for any file-like object with a `read()` method, or `csvmonkey.from_iter()`
   for any iterable object that yields lines or file chunks, e.g.
   `from_iter(file("ram.csv"))`.

By default a file header is expected and read away during construction. If your
CSV lacks a header, specify `header=False`.

By default a magical `Row` object is yielded during iteration. This object is
only a window into the currently parsed data, and will become invalid upon the
next iteration. Row data can be accessed either by index or by key (if
`header=True`) using:

```
for row in reader:
    row[20]  # by index
    row["UnBlendedCost"]  # by header value
```

Element access causes the relevant chunk of the row to be copied to the heap
and returned as a Python string.

If the CSV lacks a header, but dict-like behaviour is desired, pass a header
field explicitly as `header=("a", "b", "c", "d")`.

Rows may be converted to dicts via `row.asdict()` or tuples using
`row.astuple()`. If you want rows to be produced directly as dict or tuple,
pass `yields="tuple"` or `yields="dict"` keyword arguments.


### Unicode Support

Unicode is supported for character sets where delimiters and line endings are
represented by one byte. Columns default to being returned as ``bytes``. To
enable Unicode support, pass an ``encoding`` parameter, and optionally an
``errors`` parameter.

* ``None``, "bytes": Return bytes (default)
* "utf-8": Decode as UTF-8
* "ascii": Decode as ASCII
* "latin1": Decode as LATIN1
* "locale": Decode according to the active C locale
* "...": Decode according some codec "..." known to Python

Where possible, prefer exact spelling and case matching one of the above
encodings, to ensure an associated fast path is used.


## Python Benchmark

ram.csv is 614MiB with 1,540,093 records of 22 columns and approximately 418
bytes per record. An anonymized version is checked into LFS as
``testdata/anon-ram.csv.zstd``.


| Mode                     | Xeon E5530 (Sum) | Xeon E5530 (noop) | Core i5-2435M (Sum) | Core i5-2335M (noop) |
|--------------------------|------------------|-------------------|---------------------|----------------------|
| csvmonkey Lazy Decode    | 0.9s             | 0.444s            | 1.29s               | -                    |
| csvmonkey yields="tuple" | 1.87s            | 1.4s              | 2.17s               | -                    |
| csvmonkey yields="dict"  | 4.57s            | 4.26s             | 5.04s               | -                    |
| csv.reader               | 5.88s            | 5.31s             | 11.1s               | -                    |
| csv.DictReader           | 16.3s            | 15.2s             | 25.0s               | -                    |

### Command lines

```
# csvmonkey Lazy Decode (Sum)
$ python -m timeit -n 1 -r 1 -s 'import csvmonkey' 'sum(float(row["UnBlendedCost"]) for row in csvmonkey.from_path("ram.csv"))'

# csvmonkey Lazy Decode (noop)
$ python -m timeit -n 1 -r 1 -s 'import csvmonkey' 'all(csvmonkey.from_path("ram.csv"))'

# csv.DictReader (Sum)
$ python -m timeit -n 1 -r 1 -s 'import csv' 'sum(float(row["UnBlendedCost"]) for row in csv.DictReader(file("ram.csv")))'

# csv.DictReader (noop)
$ python -m timeit -n 1 -r 1 -s 'import csv' 'all(csv.DictReader(file("ram.csv")))'

# csv.reader (Sum)
$ python -m timeit -n 1 -r 1 -s 'import csv' 'r = csv.reader(file("ram.csv")); next(r); sum(float(row[20]) for row in r)'

# csv.reader (noop)
$ python -m timeit -n 1 -r 1 -s 'import csv' 'all(csv.reader(file("ram.csv")))'

# csvmonkey yields="tuple" (Sum)
$ python -m timeit -n 1 -r 1 -s 'import csvmonkey' 'sum(float(row[20]) for row in csvmonkey.from_path("ram.csv", yields="tuple"))'

# csvmonkey yields="tuple" (noop)
$ python -m timeit -n 1 -r 1 -s 'import csvmonkey' 'all(csvmonkey.from_path("ram.csv", yields="tuple"))'

# csvmonkey yields="dict" (Sum)
$ python -m timeit -n 1 -r 1 -s 'import csvmonkey' 'sum(float(row["UnBlendedCost"]) for row in csvmonkey.from_path("ram.csv", yields="dict"))'

# csvmonkey yields="dict" (noop)
$ python -m timeit -n 1 -r 1 -s 'import csvmonkey' 'all(csvmonkey.from_path("ram.csv", yields="dict"))'
```


## C++ Usage

1. Copy `csvmonkey.hpp` to your project and include it.
1. `CFLAGS=-msse4.2 -O3`
1. See `Makefile` for an example of producing a profile-guided build (worth an
   extra few %).
1. Instantiate `MappedFileCursor` (zero copy) or `FdStreamCursor` (buffered), attach it to a `CsvReader`.
1. Invoke `read_row()` and use `row().by_value()` to pick out `CsvCell` pointers for your desired columns.
1. Pump `read_row()` in a loop and use cell's `ptr()`, `size()`, `as_str()`, `equals()` and `as_double()` methods while `read_row()` returns true.


# TODO

* Finish Python 3 support
* Ensure Python ReaderObject is always 16-byte aligned
* Fix handling of last row when it:
    * lacks newline, or
    * is truncated after final quote, or
    * is truncated within a quote, or
    * is truncated within an escape
* Restartable: fix quadratic behaviour when `StreamCursor` yields lines and CSV
  rows span lines
* ~~Python `from_file()` that uses `read()` in preference to `__iter__()`.~~
* ~~Fix CRLF / LFCR handling.~~
* ~~`StreamCursor` error / exception propagation.~~
* ~~Remove hard 256 column limit & fix crash if it's exceeded.~~
* ~~Ensure non-SSE fallback return codes match SSE when not found.~~
* ~~Map single zero page after file pages in MappedFileCursor~~
* ~~Add trailing 16 NUL bytes to BufferedStreamCursor~~
* ~~Remove hard-coded page size~~
* ~~(Single byte separator) Unicode support.~~
* (Multi byte separator) Unicode support.
