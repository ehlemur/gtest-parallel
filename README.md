# gtest-parallel

_This is not an official Google product._

`gtest-parallel` is a script that executes [Google
Test](https://github.com/google/googletest) binaries in parallel, providing good
speedup for single-threaded tests (on multi-core machines) and tests that do not
run at 100% CPU (on single- or multi-core machines).

The script works by listing the tests of each binary, and then executing them on
workers in separate processes. This works fine so long as the tests are self
contained and do not share resources (reading data is fine, writing to the same
log file is probably not).

## Basic Usage

_For a full list of options, see `--help`._

    $ ./gtest-parallel path/to/binary...

This shards all enabled tests across a number of workers, defaulting to the
number of cores in the system. If your system uses Python 2, but you have no
python2 binary, run `python gtest-parallel` instead of `./gtest-parallel`.

To run only a select set of tests, run:

    $ ./gtest-parallel path/to/binary... --gtest_filter=Foo.*:Bar.*

This filter takes the same parameters as Google Test, so -Foo.\* can be used for
test exclusion as well. This is especially useful for slow tests (that you're
not working on), or tests that may not be able to run in parallel.

## Flakiness

Flaky tests (tests that do not deterministically pass or fail) often cause a lot
of developer headache. A test that fails only 1% of the time can be very hard to
detect as flaky, and even harder to convince yourself of having fixed.

`gtest-parallel` supports repeating individual tests (`--repeat=`), which can be
very useful for flakiness testing. Some tests are also more flaky under high
loads (especially tests that use realtime clocks), so raising the number of
`--workers=` well above the number of available core can often cause contention
and be fruitful for detecting flaky tests as well.

    $ ./gtest-parallel out/{binary1,binary2,binary3} --repeat=1000 --workers=128

The above command repeats all tests inside `binary1`, `binary2` and `binary3`
located in `out/`. The tests are run `1000` times each on `128` workers (this is
more than I have cores on my machine anyways). This can often be done and then
left overnight if you've no initial guess to which tests are flaky and which
ones aren't. When you've figured out which tests are flaky (and want to fix
them), repeat the above command with `--gtest_filter=` to only retry the flaky
tests that you are fixing.

Note that repeated tests do run concurrently with themselves for efficiency, and
as such they have problem writing to hard-coded files, even if they are only
used by that single test. `tmpfile()` and similar library functions are often
your friends here.

## Running Tests Within Test Cases Sequentially

Sometimes tests within a single test case use globally-shared resources
(hard-coded file paths, sockets, etc.) and cannot be run in parallel. Running
such tests in parallel will either fail or be flaky (if they happen to not
overlap during execution, they pass). So long as these resources are only shared
within the same test case `gtest-parallel` can still provide some parallelism.

For such binaries where test cases are independent, `gtest-parallel` provides
`--serialize_test_cases` that runs tests within the same test case sequentially.
While generally not providing as much speedup as fully parallel test execution,
this permits such binaries to partially benefit from parallel execution.

## Multiple Invocations

When running multiple `gtest-parallel` instances, separate `--output_dir`
locations should be used. `gtest-parallel` cleans out this directory on startup,
to avoid mixing old stale logs with new ones. This will discard any logs from
ongoing invocations, or otherwise fail (throw) because the files are in use
(depending on platform). This is not enforced by a lock file as cleaning these
up would require unnecessary user intervention.
