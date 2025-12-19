# Development

Install prerequisites:

- [Go](https://go.dev/doc/install)
- C/C++ Compiler e.g. Clang on macOS, [TDM-GCC](https://github.com/jmeubank/tdm-gcc/releases/latest) (Windows amd64) or [llvm-mingw](https://github.com/mstorsjo/llvm-mingw) (Windows arm64), GCC/Clang on Linux.

Then build and run Ollama from the root directory of the repository:

```shell
go run . serve
```

> [!NOTE]
> Ollama includes native code compiled with CGO.  From time to time these data structures can change and CGO can get out of sync resulting in unexpected crashes.  You can force a full build of the native code by running `go clean -cache` first. 


## macOS (Apple Silicon)

macOS Apple Silicon supports Metal which is built-in to the Ollama binary. No additional steps are required.

## macOS (Intel)

Install prerequisites:

- [CMake](https://cmake.org/download/) or `brew install cmake`

Then, configure and build the project:

```shell
cmake -B build
cmake --build build
```

Lastly, run Ollama:

```shell
go run . serve
```

## Windows

Install prerequisites:

- [CMake](https://cmake.org/download/)
- [Visual Studio 2022](https://visualstudio.microsoft.com/downloads/) including the Native Desktop Workload
- (Optional) AMD GPU support
    - [ROCm](https://rocm.docs.amd.com/en/latest/)
    - [Ninja](https://github.com/ninja-build/ninja/releases)
- (Optional) NVIDIA GPU support
    - [CUDA SDK](https://developer.nvidia.com/cuda-downloads?target_os=Windows&target_arch=x86_64&target_version=11&target_type=exe_network)
- (Optional) VULKAN GPU support
    - [VULKAN SDK](https://vulkan.lunarg.com/sdk/home) - useful for AMD/Intel GPUs

Then, configure and build the project:

```shell
cmake -B build
cmake --build build --config Release
```

> Building for Vulkan requires VULKAN_SDK environment variable:
> 
> PowerShell
> ```powershell
> $env:VULKAN_SDK="C:\VulkanSDK\<version>"
> ```
> CMD
> ```cmd
> set VULKAN_SDK=C:\VulkanSDK\<version>
> ```

> [!IMPORTANT]
> Building for ROCm requires additional flags:
> ```
> cmake -B build -G Ninja -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++
> cmake --build build --config Release
> ```



Lastly, run Ollama:

```shell
go run . serve
```

## Windows (ARM)

Windows ARM does not support additional acceleration libraries at this time.  Do not use cmake, simply `go run` or `go build`.

## Linux

Install prerequisites:

- [CMake](https://cmake.org/download/) or `sudo apt install cmake` or `sudo dnf install cmake`
- (Optional) AMD GPU support
    - [ROCm](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/install/quick-start.html)
- (Optional) NVIDIA GPU support
    - [CUDA SDK](https://developer.nvidia.com/cuda-downloads)
- (Optional) Moore Threads GPU support
    - [MUSA Toolkit](https://developer.mthreads.com/) - for Moore Threads MTT GPUs
- (Optional) VULKAN GPU support
    - [VULKAN SDK](https://vulkan.lunarg.com/sdk/home) - useful for AMD/Intel GPUs
    - Or install via package manager: `sudo apt install vulkan-sdk` (Ubuntu/Debian) or `sudo dnf install vulkan-sdk` (Fedora/CentOS)
> [!IMPORTANT]
> Ensure prerequisites are in `PATH` before running CMake.


Then, configure and build the project:

```shell
cmake -B build
cmake --build build
```

> [!IMPORTANT]
> Building for MUSA requires enabling the MUSA backend:
> ```
> cmake -B build -DGGML_MUSA=ON -DMUSA_ARCHITECTURES="21;22;31" -DGGML_MUSA_GRAPHS=OFF -DGGML_MUSA_MUDNN_COPY=OFF
> cmake --build build
> ```
> 
> MUSA architecture targets:
> - `21` - MTT S80 (mp_21)
> - `22` - MTT S3000 (mp_22)
> - `31` - Future Moore Threads GPUs
> 
> You can specify multiple architectures separated by semicolons.
>
> **Note:** CUDA Graphs (`GGML_MUSA_GRAPHS`) should be disabled for MUSA due to compatibility issues with stream capture. If you encounter `MUSA_ERROR_ILLEGAL_ADDRESS` or "operation not permitted when stream is capturing" errors, ensure graphs are disabled.

Lastly, run Ollama:

```shell
go run . serve
```

## Docker

```shell
docker build .
```

### ROCm

```shell
docker build --build-arg FLAVOR=rocm .
```

## Running tests

To run tests, use `go test`:

```shell
go test ./...
```

> NOTE: In rare circumstances, you may need to change a package using the new
> "synctest" package in go1.24.
>
> If you do not have the "synctest" package enabled, you will not see build or
> test failures resulting from your change(s), if any, locally, but CI will
> break.
>
> If you see failures in CI, you can either keep pushing changes to see if the
> CI build passes, or you can enable the "synctest" package locally to see the
> failures before pushing.
>
> To enable the "synctest" package for testing, run the following command:
>
> ```shell
> GOEXPERIMENT=synctest go test ./...
> ```
>
> If you wish to enable synctest for all go commands, you can set the
> `GOEXPERIMENT` environment variable in your shell profile or by using:
>
> ```shell
> go env -w GOEXPERIMENT=synctest
> ```
>
> Which will enable the "synctest" package for all go commands without needing
> to set it for all shell sessions.
>
> The synctest package is not required for production builds.

## Library detection

Ollama looks for acceleration libraries in the following paths relative to the `ollama` executable:

* `./lib/ollama` (Windows)
* `../lib/ollama` (Linux)
* `.` (macOS)
* `build/lib/ollama` (for development)

If the libraries are not found, Ollama will not run with any acceleration libraries.

## MUSA-Specific Configuration

### Environment Variables

The following environment variables can be used to configure MUSA backend behavior:

- `MUSA_VISIBLE_DEVICES` - Comma-separated list of MUSA GPU IDs to use. Set to "-1" to disable MUSA and force CPU inference.
- `OLLAMA_MODELS` - Directory where Ollama stores models (applies to all backends)
- `GGML_MUSA` - Set to "1" to enable MUSA backend (build-time)

Example:
```shell
# Use only MUSA GPU 0
export MUSA_VISIBLE_DEVICES=0
go run . serve

# Use MUSA GPUs 0 and 1
export MUSA_VISIBLE_DEVICES=0,1
go run . serve

# Disable MUSA, use CPU only
export MUSA_VISIBLE_DEVICES=-1
go run . serve
```

### Verifying MUSA Support

To verify that MUSA support is active:

1. Check GPU detection at startup:
```shell
go run . serve
```
Look for log messages indicating MUSA devices were detected, including device names, memory capacity, and compute capability.

2. Check device status with Moore Threads tools:
```shell
mthreads-gmi
```

3. Run a model and verify GPU usage:
```shell
# In another terminal
go run . run qwen3:0.6b
```
Monitor GPU memory usage with `mthreads-gmi` to confirm the model is loaded on the GPU.

### Troubleshooting MUSA Issues

#### MUSA Devices Not Detected

**Symptoms:** Ollama falls back to CPU inference, no MUSA devices shown in logs.

**Possible Causes and Solutions:**

1. **MUSA toolkit not installed or not in PATH**
   - Verify MUSA installation: `which mcc`
   - Ensure MUSA libraries are in `LD_LIBRARY_PATH`
   - Add MUSA to PATH: `export PATH=/usr/local/musa/bin:$PATH`

2. **MUSA driver not loaded**
   - Check driver status: `lsmod | grep musa`
   - Load driver if needed: `sudo modprobe musa`
   - Verify with: `mthreads-gmi`

3. **Ollama built without MUSA support**
   - Rebuild with MUSA enabled:
     ```shell
     cmake -B build -DGGML_MUSA=ON -DMUSA_ARCHITECTURES="21;22;31"
     cmake --build build
     ```

4. **MUSA_VISIBLE_DEVICES set incorrectly**
   - Check environment: `echo $MUSA_VISIBLE_DEVICES`
   - Unset if needed: `unset MUSA_VISIBLE_DEVICES`

#### MUSA Out of Memory Errors

**Symptoms:** Model fails to load, "out of memory" errors in logs.

**Solutions:**

1. **Use a smaller model or quantized version**
   - Try Q4_0 or Q5_0 quantization instead of full precision
   - Use a smaller parameter model

2. **Enable partial offloading**
   - Ollama automatically attempts partial offloading
   - Some layers will run on CPU, some on GPU

3. **Close other GPU applications**
   - Check GPU memory usage: `mthreads-gmi`
   - Close other applications using MUSA GPUs

4. **Use multiple GPUs**
   - Set `MUSA_VISIBLE_DEVICES=0,1` to distribute across GPUs

#### MUSA Driver Version Mismatch

**Symptoms:** "driver version mismatch" or "incompatible driver" errors.

**Solutions:**

1. **Update MUSA driver**
   - Check current version: `mthreads-gmi`
   - Install latest driver from Moore Threads website
   - Minimum version: 1.0, Recommended: 1.2+

2. **Update MUSA toolkit**
   - Ensure toolkit version matches driver version
   - Reinstall toolkit if necessary

#### Mixed Backend Issues (MUSA + CUDA/ROCm)

**Symptoms:** Wrong GPU backend selected, device conflicts.

**Solutions:**

1. **Explicitly select MUSA devices**
   - Use `MUSA_VISIBLE_DEVICES` to specify MUSA GPUs
   - Use `CUDA_VISIBLE_DEVICES` or `ROCR_VISIBLE_DEVICES` for other GPUs

2. **Check device enumeration**
   - Ollama logs show which backend is assigned to each device
   - Verify correct backend assignment in startup logs

3. **Disable unwanted backends**
   - Set `CUDA_VISIBLE_DEVICES=-1` to disable CUDA
   - Set `ROCR_VISIBLE_DEVICES=-1` to disable ROCm
   - Set `MUSA_VISIBLE_DEVICES=-1` to disable MUSA

#### Performance Issues

**Symptoms:** Slow inference, lower than expected performance.

**Solutions:**

1. **Verify flash attention is enabled**
   - Check logs for "flash attention" messages
   - MUSA devices should support flash attention by default

2. **Check layer offloading**
   - Logs should show layers offloaded to MUSA
   - If most layers on CPU, increase GPU memory availability

3. **Monitor GPU utilization**
   - Use `mthreads-gmi` to check GPU usage
   - Low utilization may indicate CPU bottleneck

4. **Optimize MUSA build flags**
   - Ensure correct architecture target: `-DMUSA_ARCHITECTURES="21;22"`
   - Enable MUSA graphs: `-DGGML_MUSA_GRAPHS=ON`
   - Enable MUDNN copy: `-DGGML_MUSA_MUDNN_COPY=ON`

#### CUDA Graph / Stream Capture Errors

**Symptoms:** `MUSA_ERROR_ILLEGAL_ADDRESS`, "operation not permitted when stream is capturing", or crashes during model loading.

**Solutions:**

1. **Disable CUDA Graphs**
   - Rebuild with graphs disabled:
     ```shell
     cmake -B build -DGGML_MUSA=ON -DMUSA_ARCHITECTURES="21;22;31" -DGGML_MUSA_GRAPHS=OFF
     cmake --build build
     ```

2. **Disable MUDNN Copy if issues persist**
   - Add `-DGGML_MUSA_MUDNN_COPY=OFF` to cmake flags

3. **Verified working configuration**
   ```shell
   cmake -B build \
     -DGGML_MUSA=ON \
     -DMUSA_ARCHITECTURES="21;22;31" \
     -DGGML_MUSA_GRAPHS=OFF \
     -DGGML_MUSA_MUDNN_COPY=OFF
   cmake --build build
   ```

**Technical Background:** MUSA's CUDA Graph implementation has stricter limitations than NVIDIA CUDA. Certain operations (memory allocation, synchronization) are not permitted during graph capture, which can cause crashes. Disabling graphs avoids these issues with minimal performance impact (typically 5-10%).

#### Build Failures

**Symptoms:** CMake configuration or compilation errors.

**Solutions:**

1. **MUSA compiler not found**
   - Install MUSA toolkit
   - Add to PATH: `export PATH=/usr/local/musa/bin:$PATH`

2. **Missing MUSA headers**
   - Verify toolkit installation
   - Check `/usr/local/musa/include` exists

3. **Linker errors**
   - Ensure MUSA libraries are installed
   - Check `/usr/local/musa/lib` or `/usr/local/musa/lib64`
   - Add to library path: `export LD_LIBRARY_PATH=/usr/local/musa/lib:$LD_LIBRARY_PATH`

For additional help, reach out on [Discord](https://discord.gg/ollama) or file an [issue](https://github.com/ollama/ollama/issues) with:
- MUSA toolkit version (`mcc --version`)
- MUSA driver version (`mthreads-gmi`)
- Ollama version and build configuration
- Complete error logs
