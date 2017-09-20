Introduction
============

This is the benchmark runner for llvm test-suite. It is only available when the
test-suite was built with cmake. It runs benchmarks, checks the results and
collects various metrics such as runtime, compile time or code size.

The runner is implemented as a custom test format for the llvm-lit tool.

`.test` Files
=============

`.test` files specify how to run a benchmark and check the results.

Each line in a `.test` file may specify a `PREPARE:`, `RUN:` or `VERIFY:`
command as a shell command. Each kind can be specified multiple times. A
benchmark run will first execute all `PREPARE:` commands, then all `RUN:`
commands and finally all `VERIFY:` commands. Metrics like runtime or profile
data are collected for the `RUN:` commands.

Commands are specified as shell commands. However only a subset of posix shell
functionality is supported: Assigning environment variables, redirecting input
and outputs, changing the directory and executing commands. (see
`shellcommand.py` for details).

Example:

    RUN: ./mybenchmark --size 500 --verbose > run0.txt
    VERIFY: diff reference_results0.txt run0.txt
    RUN: ./mybenchmark --size 300 --seed 5555 --verbose > run1.txt
    VERIFY: diff reference_results1.txt run1.txt

TODO: Document `METRIC:` lines.

Usage
=====

Running cmake on the test-suite creates a `lit.site.cfg` and `.test` files for
each bechmark. `llvm-lit` can then be used as usual. You may alternatively
install `lit` from the python package index. Examples:

    # Run all benchmark in the current directory an subdirecories one at a time.
    $ llvm-lit -j1 .

    # Run a single benchmark and show executed commands and collected metrics.
    $ llvm-lit -a SingleSource/Benchmarks/Misc/pi.test

    # Run benchmarks with reduced lit output but save collected metrics to json
    # file. This format is used by LNT or viewable by test-suite/utils/compare.py.
    $ llvm-lit . -o result.json -s

Testing Modules
===============

The benchmark runner behaviour is defined and enhanced by testing modules.
Testing modules can modify the command lines to run the benchmark and register
callbacks to collect metrics.

The list of modules is defined in the `lit.site.cfg` file
(`config.test_modules`) which in turn is generated by cmake. The module list is
influenced by a number of cmake flags. For a complete list consult the cmake
code; typical examples are:

- `cmake -DTEST_SUITE_RUN_BENCHMARKS=Off` removes the `run` module, so no
  benchmarks are actually run; this is useful as code size data, compiletime, or
  compilation statistics can be collected anyway.
- `cmake -DTEST_SUITE_RUN_UNDER=qemu` enable the `run_under` module and will
  prefix all benchmark invocations with the specified command (here: `qemu`).
- `cmake -DTEST_SUITE_REMOTE_HOST=xxx` enabled the `remote` module that uses
  ssh to run benchmarks on a remote device (assuming shared file systems).
- `cmake -DTEST_SUITE_PROFILE_GENERATE` compiles benchmark with
  `-fprofile-instr-generate` and enables the `profilegen` module that runs
  `llvm-profdata` after running the benchmarks.

Developing New Modules
----------------------

Testing modules consist of a python module with a `mutatePlan(context, plan)`
function. Modules can:

- Modify the scripts used to execute the benchmark (`plan.preparescript`,
	`plan.runscript`, `plan.verifyscript`). A script is a list of strings with
	shell commands. Modifying the list is best done via the testplan.mutateScript
	function which sets `context.tmpBase` to a unique for the command to be
	modified. The `shellcommand` module helps analyzing and modifying posix shell
	command lines.

- Append a function to the `plan.metric_collectors` list.  The metric collector
  functions are executed after the benchmark scripts ran.  Metrics must be
  returned as a dictionary with int, float or string values (anything supported
  by `lit.Test.toMetricValue()`).

- The `context` object is passed to all testing modules and metric collectors
  and may be used to communicate information between them. `context.executable`
  contains the path to the benchmark executable; `context.config` contains the
  lit configuration coming from a combination of `lit.site.cfg`, `lit.cfg` and
  `lit.local.cfg` files for the benchmark.