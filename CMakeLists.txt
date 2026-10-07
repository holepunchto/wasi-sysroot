cmake_minimum_required(VERSION 4.0)

find_package(cmake-fetch REQUIRED PATHS node_modules/cmake-fetch)

project(wasi_sysroot C CXX)

include(ExternalProject)

set(WASI_SYSROOT_LIBC_VERSION "wasi-sdk-34" CACHE STRING "The wasi-libc release to build")

# Must match the release of the `llvm-runtime` compiler, as libc++ depends on
# the exact compiler it is built with.
set(WASI_SYSROOT_LLVM_VERSION "23.1.2" CACHE STRING "The LLVM release to build the C++ libraries from")

set(WASI_SYSROOT_TARGETS "wasm32-wasip1" CACHE STRING "The targets to build a sysroot for")

fetch_package(
  "github:WebAssembly/wasi-libc#${WASI_SYSROOT_LIBC_VERSION}"
  SOURCE_DIR libc_source
  SOURCE_SUBDIR fetch-only
)

# wasi-sdk applies the same patches. Two are upstream pull requests,
# llvm/llvm-project#168449 and #186054. The third reverts the part of
# llvm/llvm-project#194317 that defines locale functions on unknown platforms
# such as WASI, where wasi-libc already declares them.
fetch_package(
  "github:llvm/llvm-project#llvmorg-${WASI_SYSROOT_LLVM_VERSION}"
  SOURCE_DIR llvm_source
  SOURCE_SUBDIR fetch-only
  PATCHES
    patches/llvm-libunwind-wasm-declspec.patch
    patches/llvm-libcxxabi-thread-atexit.patch
    patches/llvm-libcxx-wasi-locale.patch
)

set(stage "${CMAKE_CURRENT_BINARY_DIR}/stage")

# The host toolchain passes the resource directory and linker as compiler flags,
# which every sub-build needs as well.
separate_arguments(flags UNIX_COMMAND "${CMAKE_C_FLAGS}")

# The host's archiver may not understand WebAssembly, and the one on macOS
# cannot create the empty archives wasi-libc installs, so use the tools that
# ship with the compiler.
cmake_path(GET CMAKE_C_COMPILER PARENT_PATH tools)

foreach(tool IN ITEMS ar nm ranlib)
  find_program(llvm_${tool} NAMES llvm-${tool} HINTS "${tools}" NO_DEFAULT_PATH REQUIRED)
endforeach()

set(common_args
  "-DCMAKE_BUILD_TYPE=Release"
  "-DCMAKE_MAKE_PROGRAM=${CMAKE_MAKE_PROGRAM}"
  "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
  "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
  "-DCMAKE_ASM_COMPILER=${CMAKE_C_COMPILER}"
  "-DCMAKE_AR=${llvm_ar}"
  "-DCMAKE_NM=${llvm_nm}"
  "-DCMAKE_RANLIB=${llvm_ranlib}"
)

set(eh_flags -fwasm-exceptions -mllvm -wasm-use-legacy-eh=false)

foreach(target IN LISTS WASI_SYSROOT_TARGETS)
  set(sysroot "${stage}/${target}")

  execute_process(
    COMMAND "${CMAKE_C_COMPILER}" ${flags} "--target=${target}" --rtlib=compiler-rt --print-libgcc-file-name
    OUTPUT_VARIABLE builtins
    OUTPUT_STRIP_TRAILING_WHITESPACE
    COMMAND_ERROR_IS_FATAL ANY
  )

  # The driver prints the expected path even when nothing is there.
  if(NOT EXISTS "${builtins}")
    message(FATAL_ERROR
      "'llvm-runtime' has no compiler runtime builtins for '${target}'. The "
      "driver expects them at '${builtins}'.\n"
    )
  endif()

  ExternalProject_Add(
    libc-${target}
    SOURCE_DIR "${libc_source}"
    BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/libc-${target}"
    DOWNLOAD_COMMAND ""
    UPDATE_COMMAND ""
    CMAKE_ARGS
      ${common_args}
      "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}"
      "-DCMAKE_ASM_FLAGS=${CMAKE_C_FLAGS}"
      "-DCMAKE_INSTALL_PREFIX=${sysroot}"
      "-DTARGET_TRIPLE=${target}"
      "-DBUILTINS_LIB=${builtins}"
    USES_TERMINAL_CONFIGURE ON
    USES_TERMINAL_BUILD ON
    USES_TERMINAL_INSTALL ON
  )

  # The C++ libraries are built with and without exception handling, and the
  # driver picks one based on `-fwasm-exceptions`. Both install some of the
  # same files, so they build one after the other.
  set(previous libc-${target})

  foreach(variant IN ITEMS noeh eh)
    if(variant STREQUAL "eh")
      set(exceptions ON)
      set(runtimes "libunwind|libcxx|libcxxabi")
      set(variant_flags ${eh_flags})
    else()
      set(exceptions OFF)
      set(runtimes "libcxx|libcxxabi")
      set(variant_flags)
    endif()

    list(JOIN variant_flags " " variant_flags)

    set(cxx_flags "${CMAKE_CXX_FLAGS} ${variant_flags}")
    set(c_flags "${CMAKE_C_FLAGS} ${variant_flags}")

    set(libdir "/${target}/${variant}")

    ExternalProject_Add(
      libcxx-${target}-${variant}
      DEPENDS ${previous}
      SOURCE_DIR "${llvm_source}/runtimes"
      BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/libcxx-${target}-${variant}"
      DOWNLOAD_COMMAND ""
      UPDATE_COMMAND ""
      LIST_SEPARATOR "|"
      CMAKE_ARGS
        ${common_args}
        "-DCMAKE_C_FLAGS=${c_flags}"
        "-DCMAKE_CXX_FLAGS=${cxx_flags}"
        "-DCMAKE_ASM_FLAGS=${c_flags}"
        "-DCMAKE_INSTALL_PREFIX=${sysroot}"
        "-DCMAKE_INSTALL_INCLUDEDIR=include/${target}/${variant}"
        "-DCMAKE_SYSROOT=${sysroot}"
        -DCMAKE_SYSTEM_NAME=WASI
        -DCMAKE_SYSTEM_PROCESSOR=wasm32
        "-DCMAKE_C_COMPILER_TARGET=${target}"
        "-DCMAKE_CXX_COMPILER_TARGET=${target}"
        "-DCMAKE_ASM_COMPILER_TARGET=${target}"
        # Nothing can link before these libraries exist. LLVM also requires a
        # Unix or Windows platform, and WASI is close enough to Unix.
        -DCMAKE_C_COMPILER_WORKS=ON
        -DCMAKE_CXX_COMPILER_WORKS=ON
        -DLLVM_COMPILER_CHECKED=ON
        -DUNIX=ON
        "-DLLVM_ENABLE_RUNTIMES=${runtimes}"
        -DLIBCXX_ABI_VERSION=2
        -DLIBCXX_CXX_ABI=libcxxabi
        -DLIBCXX_ENABLE_ABI_LINKER_SCRIPT=OFF
        "-DLIBCXX_ENABLE_EXCEPTIONS=${exceptions}"
        -DLIBCXX_ENABLE_FILESYSTEM=ON
        -DLIBCXX_ENABLE_SHARED=OFF
        -DLIBCXX_ENABLE_THREADS=ON
        -DLIBCXX_HAS_EXTERNAL_THREAD_API=OFF
        -DLIBCXX_HAS_MUSL_LIBC=OFF
        -DLIBCXX_HAS_PTHREAD_API=ON
        -DLIBCXX_HAS_WIN32_THREAD_API=OFF
        -DLIBCXX_INCLUDE_BENCHMARKS=OFF
        -DLIBCXX_INCLUDE_TESTS=OFF
        "-DLIBCXX_LIBDIR_SUFFIX=${libdir}"
        "-DLIBCXXABI_ENABLE_EXCEPTIONS=${exceptions}"
        -DLIBCXXABI_ENABLE_SHARED=OFF
        -DLIBCXXABI_ENABLE_THREADS=ON
        -DLIBCXXABI_HAS_EXTERNAL_THREAD_API=OFF
        -DLIBCXXABI_HAS_PTHREAD_API=ON
        -DLIBCXXABI_HAS_WIN32_THREAD_API=OFF
        -DLIBCXXABI_INCLUDE_TESTS=OFF
        "-DLIBCXXABI_LIBDIR_SUFFIX=${libdir}"
        -DLIBCXXABI_SILENT_TERMINATE=ON
        "-DLIBCXXABI_USE_LLVM_UNWINDER=${exceptions}"
        -DLIBUNWIND_ENABLE_SHARED=OFF
        -DLIBUNWIND_ENABLE_THREADS=ON
        -DLIBUNWIND_INCLUDE_TESTS=OFF
        "-DLIBUNWIND_LIBDIR_SUFFIX=${libdir}"
        -DLIBUNWIND_USE_COMPILER_RT=ON
      USES_TERMINAL_CONFIGURE ON
      USES_TERMINAL_BUILD ON
      USES_TERMINAL_INSTALL ON
    )

    set(previous libcxx-${target}-${variant})
  endforeach()

  # The driver only searches the headers of either build once it finds a libc++
  # version in the generic include directory. The prebuild artifacts drop empty
  # directories, so it holds a file.
  file(WRITE "${sysroot}/include/c++/v1/README"
    "The libc++ headers are in 'include/${target}/eh' and 'include/${target}/noeh'.\n"
  )

  add_custom_target(wasi_sysroot_${target} ALL DEPENDS libcxx-${target}-noeh libcxx-${target}-eh)

  install(
    DIRECTORY "${sysroot}/"
    DESTINATION "${target}"
    USE_SOURCE_PERMISSIONS
  )
endforeach()
