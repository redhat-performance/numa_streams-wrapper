# NUMA STREAMS (Memory Bandwidth) Benchmark Wrapper

## Description

This wrapper facilitates the automated execution of the NUMA STREAMS memory bandwidth benchmark. NUMA STREAMS measures sustainable memory bandwidth across NUMA topologies by running the STREAM benchmark (Copy, Scale, Add, Triad operations) with varying thread counts and array sizes, providing insight into memory subsystem performance across sockets and NUMA nodes.

The wrapper provides:
- Automated download, build, and execution of the AMD STREAM Dynamic benchmark.
- Automatic cache-topology-aware array sizing.
- Thread count scaling from 1 to total CPUs (powers of 2).
- Multiple array sizes per thread count (512, 1000, 2000, 4000 MiB).
- Support for x86_64 and aarch64 architectures (including ARM Neoverse SLC detection).
- Configurable compiler optimization levels (O2, O3).
- Configurable cache sizing parameters.
- Result collection and processing with per-socket CSV output.
- Integration with test_tools framework.
- Optional Performance Co-Pilot (PCP) integration.

## Command-Line Options

```text
NUMA STREAMS Options:
  --cache_cap_size <value>: Cap the maximum cache size to this value.
      Default: no cap (0).
  --cache_multiply <value>: Multiply cache sizes by this factor at each step.
      Must be greater than 1. Default: 2.
  --cache_start_factor <value>: Start the cache size at base_cache * this factor.
      Must be greater than 0. Default: 1.
  --mem_offset <value>: Memory offset value passed to the test runner.
      Default: 0.
  --nsizes <value>: Maximum number of cache sizes to test. Default: 3.
  --numa_iterations <value>: Number of NUMA test iterations. Default: 10.
  --opt2 <value>: If value is not 0, run with compiler optimization level O2.
      Default: 1 (enabled). Currently not executed by default (see Notes).
  --opt3 <value>: If value is not 0, run with compiler optimization level O3.
      Default: 1 (enabled).
  --results_dir <string>: Directory to place results into.
      Default: results_numa_streams_<tuned>_<timestamp>.
  --size_list <x,y,...>: Comma-separated list of array sizes in bytes.
      Overrides automatic cache-based sizing. Default: 0 (auto).
  --threads_multiple <value>: Multiply thread count by this factor at each step.
      Must be greater than 1. Default: 2.

General test_tools options:
  --debug: Enable bash -x debug output for wrapper troubleshooting.
  --home_parent <value>: Parent home directory. If not set, defaults to current working directory.
  --host_config <value>: Host configuration name, defaults to current hostname.
  --iterations <value>: Number of times to run the test, defaults to 1.
  --json_skip: Skip JSON conversion of test CSV results.
  --no_pkg_install: Do not install any packages (system or pip). Useful for pre-provisioned systems.
  --no_system_packages: Do not install system packages via the package manager. Pip packages are still installed.
  --no_pip_packages: Do not install Python pip packages. System packages are still installed.
  --run_label <value>: Label to associate with the run. No default.
  --run_user: User that is actually running the test on the test system. Defaults to current user.
  --sys_type: Type of system working with (aws, azure, hostname). Defaults to hostname.
  --sysname: Name of the system running, used in determining config files. Defaults to hostname.
  --test_tools_release <tag>: Version tag of test_tools-wrappers to check out and use.
  --tuned_setting: Used in naming the results directory. For RHEL, defaults to current active tuned profile.
      For non-RHEL systems, defaults to 'none'. If set to a profile name, activates that tuned profile.
  --use_pcp: Enable Performance Co-Pilot monitoring during test execution.
  --verify_skip: Skip result verification against the Pydantic schema.
  --tools_git <value>: Git repo to retrieve the required tools from.
      Default: https://github.com/redhat-performance/test_tools-wrappers
  --usage: Display this usage message.
```

## What the Script Does

The wrapper consists of two scripts: `numa_streams_run` (entry point) and `run_numa_stream` (core test runner). Together they perform the following workflow:

1. **Environment Setup**:
   - Clones the test_tools-wrappers repository if not present (default: ~/test_tools).
   - Sources error codes and general setup utilities.
   - Gathers hardware information via `gather_data`.
   - Uses `invoke_test` to manage output logging to `/tmp/numa_streams.out`.

2. **Package Installation**:
   - Installs required dependencies via package_tool using `numa_streams.json`.
   - Dependencies are defined for different OS variants (RHEL, Ubuntu, SLES, Amazon Linux).

3. **Cache Topology Detection**:
   - Detects the number of NUMA nodes from `lscpu`.
   - Reads the top-level cache size from `/sys/devices/system/cpu/cpu0/cache/index*/size`.
   - Counts the number of top-level caches from `shared_cpu_list` entries.
   - For ARM Neoverse systems without exposed L3 cache: uses a 32 MiB per-node System-Level Cache (SLC) estimate.
   - Calculates base cache size in longs (divides by 8 for 64-bit elements).

4. **Benchmark Installation**:
   - Clones and builds `jemalloc` from source (https://github.com/jemalloc/jemalloc.git) for optimized memory allocation.
   - Downloads the AMD STREAM Dynamic benchmark from Phoronix Test Suite (http://www.phoronix-test-suite.com/benchmark-files/amd-stream-dynamic-1.tar.xz).
   - Patches the benchmark source to remove AMD-specific compiler flags (`-fnt-store=aggressive`, `-mavx2`).
   - Disables CPU governor calls for cloud system compatibility.
   - Builds the benchmark using the system GCC compiler.

5. **Test Execution**:
   - Runs the benchmark across a thread count sweep: 1, 2, 4, 8, ... up to total CPUs (powers of 2, plus a final run at total CPU count).
   - For each thread count, tests four array sizes: 512, 1000, 2000, 4000 MiB.
   - Each test runs with `--ntimes 100` (100 passes per measurement).
   - Optionally collects PCP data per array-size test.

6. **Data Collection**:
   - Captures raw output for each thread-count and array-size combination.
   - Records the four STREAM metrics: Copy, Scale, Add, Triad (in MB/s).
   - Extracts the highest MB/s value for each metric across all runs into `highest.csv`.
   - Extracts the minimum average time for each metric into `min.csv`.
   - Captures system configuration metadata (kernel version, NUMA nodes, CPU count, threads per core, cores per socket, sockets, model name, total memory).

7. **Result Processing**:
   - Sorts results by buffer size and socket count.
   - Generates `results_numa_streams.csv` with per-socket breakdowns of Copy, Scale, Add, and Triad rates.
   - Records test status ("Ran" or "Failed") based on whether data was successfully captured.

8. **Output**:
   - Creates timestamped results directory in `/tmp/results_numa_streams_<tuned>_<timestamp>`.
   - Saves raw output files, processed CSV, and system metadata.
   - Creates a tar archive of results.
   - Optionally saves PCP performance data.
   - Archives results to configured storage location via `save_results`.

## Dependencies

**Location of underlying workload**: Downloaded from http://www.phoronix-test-suite.com/benchmark-files/amd-stream-dynamic-1.tar.xz. Additionally, jemalloc is cloned from https://github.com/jemalloc/jemalloc.git.

**General packages required**:

- **RHEL**: git, gcc, bc, perf, zip, unzip, numactl, wget, gcc-c++, autoconf, hwloc, hwloc-gui, libomp
- **Ubuntu**: git, bc, zip, unzip, numactl, libnuma-dev, g++, autoconf, hwloc, libomp5-20
- **SLES**: git, gcc, make, bc, perf, unzip, zip, libnuma1, numactl
- **Amazon Linux**: git, gcc, bc, zip, unzip, numactl, wget, gcc-c++, autoconf, hwloc, hwloc-gui, libomp

To run:
```bash
git clone https://github.com/redhat-performance/numa_streams-wrapper
cd numa_streams-wrapper/numa_streams
./numa_streams_run
```

The script will automatically detect the system cache topology, build the benchmark, and execute the full test sweep.

## The NUMA STREAMS Benchmark

STREAM is a synthetic benchmark that measures sustainable memory bandwidth by performing four simple vector operations on large arrays. The NUMA STREAMS variant extends this by testing across NUMA topologies with varying thread counts and array sizes to characterize memory bandwidth scaling.

### STREAM Operations

1. **Copy** (`a[i] = b[i]`): Measures pure memory copy bandwidth. 2 memory accesses per element (1 read, 1 write).

2. **Scale** (`a[i] = q * b[i]`): Measures bandwidth with a scalar multiply. 2 memory accesses per element plus computation.

3. **Add** (`a[i] = b[i] + c[i]`): Measures bandwidth with vector addition. 3 memory accesses per element (2 reads, 1 write).

4. **Triad** (`a[i] = b[i] + q * c[i]`): Measures bandwidth with scalar multiply and vector addition. 3 memory accesses per element plus computation.

### Performance Metric

Results are reported in **MB/s** (megabytes per second) for each operation. Higher values indicate better memory bandwidth. The Triad rate is the most commonly cited metric as it exercises the full memory subsystem with both reads and writes.

### Array Sizes

The wrapper tests four fixed array sizes: 512, 1000, 2000, and 4000 MiB. Larger arrays ensure the working set exceeds cache capacity, forcing data access from main memory. This reveals true memory bandwidth rather than cache bandwidth.

### Thread Scaling

Thread counts follow a geometric progression (powers of 2): 1, 2, 4, 8, 16, ... up to the total CPU count. This reveals how memory bandwidth scales with thread concurrency and identifies the saturation point where adding more threads no longer increases bandwidth.

## Output Files

The results directory contains:

- **results_numa_streams.csv**: CSV file with per-socket Copy, Scale, Add, and Triad rates, plus system configuration metadata.
- **highest.csv**: The highest MB/s value achieved for each STREAM operation across all runs.
- **min.csv**: The minimum average time for each STREAM operation across all runs.
- **\<size\>meg_threads=\<N\>_passes=100**: Raw output files for each array-size and thread-count combination.
- **numa_streams.out**: Full script execution log.
- **PCP data** (if --use_pcp option used): Performance Co-Pilot monitoring data.

## Examples

### Basic run with defaults
```bash
./numa_streams_run
```
This runs with:
- Automatic cache topology detection
- O3 optimization
- Thread counts: 1, 2, 4, ... up to total CPUs
- Array sizes: 512, 1000, 2000, 4000 MiB
- 100 passes per measurement

### Run with custom thread multiplier
```bash
./numa_streams_run --threads_multiple 4
```
Scales thread counts by 4x at each step (1, 4, 16, 64, ...) instead of the default 2x, reducing the total number of test points.

### Run with custom number of cache sizes
```bash
./numa_streams_run --nsizes 6
```
Tests up to 6 different cache sizes instead of the default 3.

### Run with specific array sizes
```bash
./numa_streams_run --size_list 256,512,1024,2048,4096
```
Overrides automatic sizing with an explicit list of array sizes in bytes.

### Run with cache size cap
```bash
./numa_streams_run --cache_cap_size 8192
```
Caps the maximum cache size to 8192 KiB, limiting the largest array tested.

### Run with custom cache start factor
```bash
./numa_streams_run --cache_start_factor 2 --cache_multiply 4
```
Starts at 2x the base cache size and multiplies by 4x at each step.

### Run multiple iterations
```bash
./numa_streams_run --iterations 3
```
Runs the full test suite 3 times.

### Run with PCP monitoring
```bash
./numa_streams_run --use_pcp
```
Collects Performance Co-Pilot data during the run.

### Run with custom results directory
```bash
./numa_streams_run --results_dir /tmp/my_streams_results
```
Saves results to a specific directory instead of the auto-generated timestamped path.

### Combination example
```bash
./numa_streams_run --threads_multiple 2 --nsizes 4 --cache_multiply 2 \
    --iterations 3 --use_pcp
```
Tests 4 cache sizes with 2x thread scaling, runs 3 iterations, and collects PCP data.

## How Cache Sizing Works

The script automatically detects the system's cache topology to determine appropriate test parameters:

### Base Cache Size Detection
1. Reads all cache level sizes from `/sys/devices/system/cpu/cpu0/cache/index*/size`.
2. Selects the largest cache (typically L3).
3. For ARM Neoverse systems without exposed L3: estimates 32 MiB SLC per NUMA node.
4. Converts to longs (divides by 8) since NUMA STREAMS operates on `long` elements.

### Top-Level Cache Count
1. Reads `shared_cpu_list` for each CPU's L3 cache (index3).
2. Counts unique entries to determine the number of L3 cache domains.
3. Falls back to NUMA node count if L3 cache info is unavailable.

### Thread Count Progression
1. Starts at 1 thread.
2. Multiplies by `--threads_multiple` (default: 2) at each step.
3. Continues until reaching total CPU count.
4. Always includes a final run at exactly the total CPU count.
5. This produces a geometric series: 1, 2, 4, 8, ..., N_cpus.

### Array Sizes
The test runner uses four fixed array sizes for each thread count: 512, 1000, 2000, and 4000 MiB. These sizes are chosen to span from fitting within aggregate cache to significantly exceeding it, revealing the transition from cache-speed to memory-speed bandwidth.

## Return Codes

The script uses the following exit codes:
- **0**: Success
- **1**: General failure (git clone failure, package installation failure, invalid parameter values)
- **E_USAGE**: Invalid usage/arguments (from test_tools error_codes)

Parameter validation:
- `--cache_multiply` must be greater than 1 (exits with error if not).
- `--cache_start_factor` must be greater than 0 (exits with error if not).
- `--threads_multiple` must be greater than 1 (prints a warning and defaults to 2 if set to 1).

## Notes

### Architecture Support
- **x86_64**: Full support for AMD and Intel CPUs. Cache topology is read from sysfs.
- **aarch64**: Supported with special handling for ARM Neoverse systems where the System-Level Cache (SLC) is not exposed through standard sysfs interfaces. A 32 MiB per-node SLC estimate is used.

### Benchmark Source
The wrapper uses the AMD STREAM Dynamic benchmark from the Phoronix Test Suite, not the standard McCalpin STREAM. Key differences:
- Uses `jemalloc` for memory allocation.
- Includes a Python-based runner (`run_stream_dynamic.py`) with thread and array size parameters.
- AMD-specific compiler flags (`-fnt-store=aggressive`, `-mavx2`) are automatically patched out for portability.
- CPU governor calls are disabled for cloud compatibility.

### Compiler Optimization
- The default optimization level is O3 (`-O3`).
- While `--opt2` is accepted as a parameter, the current code only executes the O3 run path (`numa_streams_run` function). The O2 option is parsed but not used in the default execution flow.

### Memory Requirements
- The largest default array size is 4000 MiB per test. With multiple threads, total memory usage can be significant.
- Ensure the system has sufficient free memory to avoid swapping, which would invalidate bandwidth measurements.

### Performance Tips
- Run on an idle system for accurate bandwidth measurements.
- Disable CPU frequency scaling (use performance governor) for reproducible results.
- Consider the active tuned profile on RHEL systems.
- The Triad result is the most representative metric for overall memory bandwidth.
- Compare results across socket counts to evaluate NUMA scaling efficiency.
- Use `--threads_multiple 2` (default) for fine-grained thread scaling analysis.

### Missing Features
- **No results schema validation**: Unlike other wrappers, this wrapper does not include a `results_schema.py` file or call `verify_results`. Results are not validated against a Pydantic schema.
- **No JSON output**: The wrapper does not call `csv_to_json`. Results are only available in CSV format.
- **No PCP OpenMetrics reset file**: The `openmetrics_numa_streams_reset.txt` file is not present, so PCP metric definitions are not standardized.

### Troubleshooting
- If the benchmark fails to build, verify that `gcc`, `g++`, `autoconf`, and `wget` are installed.
- If jemalloc fails to build, verify that `autoconf` is available.
- If the download from Phoronix fails, check network connectivity and the availability of the URL.
- If thread counts seem wrong, verify NUMA topology with `lscpu` and check `/sys/devices/system/cpu/cpu*/cache/` entries.
- If ARM systems report incorrect cache sizes, the 32 MiB SLC estimate may not match your hardware — use `--size_list` to override.
- The full script execution log is saved to `numa_streams.out` in the results directory.
