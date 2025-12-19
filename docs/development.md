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
    - [MUSA Toolkit](https://developer.mthreads.com/) - for Moore Threads MTT S80/S3000 GPUs
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
> ```shell
> cmake -B build -DGGML_MUSA=ON -DMUSA_ARCHITECTURES="21;22;31" \
>       -DGGML_MUSA_GRAPHS=ON -DGGML_MUSA_MUDNN_COPY=ON
> cmake --build build
> ```
> 
> MUSA architecture targets:
> - `21` - MTT S80
> - `22` - MTT S3000
> - `31` - Future Moore Threads GPUs

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

## MUSA (Moore Threads)

### Build Options

| Option | Description | Default |
|--------|-------------|---------|
| `GGML_MUSA` | Enable MUSA backend | OFF |
| `MUSA_ARCHITECTURES` | Target GPU architectures (21=S80, 22=S3000, 31=future) | - |
| `GGML_MUSA_GRAPHS` | Enable CUDA Graph optimization | OFF |
| `GGML_MUSA_MUDNN_COPY` | Enable MUDNN memory copy optimization | OFF |

### Build Commands

Basic build:
```shell
cmake -B build -DGGML_MUSA=ON -DMUSA_ARCHITECTURES="21;22;31"
cmake --build build
```

Optimized build (recommended):
```shell
cmake -B build -DGGML_MUSA=ON -DMUSA_ARCHITECTURES="21;22;31" \
      -DGGML_MUSA_GRAPHS=ON -DGGML_MUSA_MUDNN_COPY=ON
cmake --build build
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `MUSA_VISIBLE_DEVICES` | Comma-separated list of GPU IDs to use. Set to `-1` to disable MUSA. |

### Verifying MUSA Support

1. Check GPU detection at startup - look for MUSA devices in logs
2. Use `mthreads-gmi` to monitor GPU status and memory usage
3. Run a model and verify GPU memory is being used

### Troubleshooting

**MUSA devices not detected:**
- Verify MUSA toolkit is installed and in PATH
- Check driver is loaded: `mthreads-gmi`
- Ensure Ollama was built with `-DGGML_MUSA=ON`

**Out of memory errors:**
- Use a smaller or quantized model
- Close other GPU applications
- Use multiple GPUs with `MUSA_VISIBLE_DEVICES=0,1`

**Build failures:**
- Ensure MUSA toolkit is installed
- Add to PATH: `export PATH=/usr/local/musa/bin:$PATH`
- Add to library path: `export LD_LIBRARY_PATH=/usr/local/musa/lib:$LD_LIBRARY_PATH`
