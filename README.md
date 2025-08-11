# NAME

Benchmark::MCE - Perl multi-core benchmarking framework

# SYNOPSIS

    use Benchmark::MCE;

    # Run 2 benchmarks (custom functions) and time them on a single core:
    my %stat_single = suite_run({
      threads => 1,
      bench   => {
        Bench1 => sub { ...code1... },
        Bench2 => '...code2...'       # String is also fine
      }
    );

    # Run each across multiple cores.
    # Use the extended (arrayref) definition to check for correctness of output.
    my %stat_multi = suite_run({
      threads => system_identity(1),  # Workers count equal to system logical cores
      bench   => {
        Bench1 => [\&code1, $expected_output1],
        Bench2 => [\&code2, $expected_output2],
      }
    );
    
    # Calculate the multi/single core scalability
    my %scal = calc_scalability(\%stat_single, \%stat_multi);

# DESCRIPTION

A benchmarking framework originally designed for the [Benchmark::DKbench](https://metacpan.org/pod/Benchmark%3A%3ADKbench) multi-core
CPU benchmarking suite. Released as a stand-alone to be used for custom benchmarks
of any type, as well as other kinds of stress-testing, throughput testing etc. 

You define custom functions (usually with randomized workloads) that can be run on
any number of parallel workers, using the low-overhead Many-Core Engine ([MCE](https://metacpan.org/pod/MCE)).

# FUNCTIONS

## `system_identity`

    my $cores = system_identity($quiet?);

Prints out software/hardware configuration and returns the number of logical cores
detected using [System::CPU](https://metacpan.org/pod/System%3A%3ACPU).

Any argument will suppress printout and will only return the number of cores.

## `suite_run`

    my %stats = suite_run(\%options);

Runs the benchmark suite given the `%options` and prints results. Returns a hash
with run stats that looks like this:

    %stats = (
      $bench_name_1 => {times => [ ... ], scores => [ ... ]},
       ...
      _total => {times => [ ... ], scores => [ ... ]},
      _opt   => {iter => $iterations, threads => $no_threads, ...}
    );

Note that the times reported will be average times per thread (or per function
call if you prefer), however the scores reported (if a reference time is supplied)
are sums across all threads. So you expect for ideal scaling 1 thread vs 2 threads
to return the same times, double the scores.

### Options:

- `bench` (HashRef) **required**:
A hashref with keys being your unique custom benchmark names and values being
arrays:

    `name => [ $coderef, $expected?, $ref_time?, $quick_arg?, $normal_arg? ]`

    where:

    - `$coderef` **required**:
    Reference to your benchmark function. See ["BENCHMARK FUNCTIONS"](#benchmark-functions) for more details.
    - `$expected`:
    Expected output of the benchmark function on successful run (for PASS/FAIL - PASS
    will be always assumed is parameter is undefined).
    - `$ref_time`:
    Reference time in seconds for score of 1000.
    - `$quick_arg`:
    Argument to pass to the benchmark function in `quick` mode (for workload scaling).
    - `$normal_arg`:
    Argument to pass to the benchmark function in normal mode (for workload scaling).

- `threads` (Int; default 1):
Parallel benchmark threads. They are [MCE](https://metacpan.org/pod/MCE) workers, so not 'threads' in the technical
sense. Each of the benchmarks defined will launch on each of the threads, hence the
total workload is multiplied by the number of `threads`. Times will be averaged
across threads, while scores will be summed.
- `iter` (Int; default 1):
Number of suite iterations (with min/max/avg at the end when > 1).
- `include` (Regex):
Only run benchmarks whose names match regex.
- `exclude` (Regex):
Skip benchmarks whose names match regex.
- `time` (Bool):
Report time (sec) instead of score. Set to true by `quick` or if at least one
benchmark has no reference time declared. Otherwise score output is the default.
- `quick` (Bool; default 0):
Use each benchmark's quick argument and imply `time=1`.
- `scale` (Int; default 1):
Scale the bench workload (number of calls of the benchmark functions) by x times.
Forced to 1 with `quick` or `no_mce`.
- `stdev` (Bool; default 0):
Show relative standard deviation (for `iter` > 1).
- `sleep` (Int; default 0):
Number of seconds to sleep after each benchmark run.
- `duration` (Int, seconds):
Minimum duration in seconds for suite run (overrides `iter`).
- `srand` (Int; default 1):
Define a fixed seed to keep runs reproducible when your benchmark functions use
`rand`. The seed will be passed to `srand` before each call to a benchmark
function. Set to 0 to skip rand seeding.
- `no_pass` (Bool; default 0):
Do check for Pass/Fail even if reference output is defined.
- `no_mce` (Bool; default 0):
Do not run under [MCE::Loop](https://metacpan.org/pod/MCE%3A%3ALoop) (sets `threads=1`, `scale=1`).

## `calc_scalability`

    my %scal = calc_scalability(\%stat_single, \%stat_multi, $keep_outliers?);

Given the `%stat_single` results of a single-threaded `suite_run` and `%stat_multi`
results of a multi-threaded run, will calculate, print and return the multi-thread
scalability (including averages, ranges etc for multiple iterations).

Unless `$keep_outliers` is true, the overall scalability is an average after droping
Benchmarks that are non-scaling outliers (over 2\*stdev less than the mean).

The result hash return looks like this:

    %scal = (
      bench_name => $bench_avg_scalability,
       ...
      _total => $total_avg_scalability
    );

## `suite_calc`

    my ($stats, $stats_multi, $scal) = suite_calc(\%suite_run_options, , $keep_outliers?);

Convenience function that combines 3 calls, [suite\_run](https://metacpan.org/pod/suite_run) with `threads=>1`,
[suite\_run](https://metacpan.org/pod/suite_run) with `threads=>system_identity(1)` and [calc\_scalability](https://metacpan.org/pod/calc_scalability) with
the results of those two, returning hashrefs with the results of all three calls.

For single-core systems (or when `system_identity(1)` does not return > 1)
only `$stats` will be returned.

# BENCHMARK FUNCTIONS

The benchmark functions will be called with two parameters that you can choose to
take advantage of.
The first one is what you define as either the `$quick_arg` or `$normal_arg`,
with the intention being to have a way to run a `quick` mode that lets you test with
smaller workloads. The second argument will be an integer that's the chunk number
from [MCE::Loop](https://metacpan.org/pod/MCE%3A%3ALoop) - it will be 1 for the call on the first thread, 2 from the second
thread etc, so your function may track which worker/chunk is running.

The function may return a string, usually a checksum, that will be checked against
the (optional) `$expected` parameter to show a Pass/Fail (useful for verifying
correctness, stress testing, etc.).

Example:

    use Benchmark::MCE;
    use Math::Trig qw/:great_circle :pi/;

    sub great_circle {
      my $size  = shift || 1;  # Optionally have an argument that scales the workload
      my $chunk = shift;       # Optionally use the chunk number
      my $dist = 0;
      $dist +=
        great_circle_distance(rand(pi), rand(2 * pi), rand(pi), rand(2 * pi))
          for 1 .. $size;
      return $dist; # Returning a value is optional for the Pass/Fail functionality
    }

    my %stats = suite_run({
        bench => { 'Math::Trig' =>  # A unique name for the benchmark
          [
          \&great_circle,      # Reference to bench function
          '3144042.81433949',  # Reference output - determines Pass/Fail (optional)
          5.5,                 # Seconds to complete in normal mode for score = 1000 (optional)
          1000000,             # Argument to pass for quick mode (optional)
          5000000              # Argument to pass for normal mode (optional)
          ]},
      }
    );

# STDOUT / QUIET MODE

Normally function calls will print results to `STDOUT` as well as return them.
You can suppress STDOUT by setting:

    $Benchmark::MCE::QUIET = 1;

# NOTES

The framework uses a monotonic timer for non-Windows systems with at least v1.9764
of `Time::HiRes` (`$Benchmark::MCE::MONO_CLOCK` will be true).

# AUTHOR

Dimitrios Kechagias, `<dkechag at cpan.org>`

# BUGS

Please report any bugs or feature requests on [GitHub](https://github.com/SpareRoom/Benchmark-MCE).

# GIT

[https://github.com/SpareRoom/Benchmark-MCE](https://github.com/SpareRoom/Benchmark-MCE)

# LICENSE AND COPYRIGHT

Copyright (c) 2021-2025 Dimitrios Kechagias.
Copyright (c) 2025 SpareRoom.

This is free software; you can redistribute it and/or modify it under
the same terms as the Perl 5 programming language system itself.
