# Meson build for carquet — a pure C11 Parquet reader/writer.
#
# Standalone: usable via `subproject('carquet')` from the parent project (carquet
# is a git submodule, so there is no meson.wrap). Exposes a `carquet_dep`
# dependency object that consumers link against.
#
# Mirrors the upstream CMakeLists.txt: static library, C11, the compression
# codecs (zstd/zlib/lz4) are linked whenever found, and the SIMD translation
# units are compiled with per-file ISA flags + -fno-lto so runtime CPU-feature
# dispatch stays correct (LTO can inline ISA-specific code into baseline
# callers and SIGILL on older CPUs).
#
# The CLI tool under src/cli/ is intentionally NOT built here — only the
# libcarquet static archive is.

project('carquet', 'c',
    version: '0.7.1',
    default_options: [
        'c_std=c11',
        'default_library=static',
        'warning_level=2',
    ],
)

cc = meson.get_compiler('c')

# --- Warning flags (match upstream CMakeLists, no -Werror) --------------------
# Carquet is third-party code; never promote its warnings to errors so a new
# compiler doesn't break the engine build. warning_level=2 already adds
# -Wall -Wextra; supplement with -Wpedantic and silence unused-parameter noise
# (handlers must match a fixed callback signature).
warning_flags = ['-Wpedantic', '-Wno-unused-parameter']
foreach flag : warning_flags
    if cc.has_argument(flag)
        add_project_arguments(flag, language: 'c')
    endif
endforeach

# --- Optional external compression dependencies -------------------------------
# The compression codec TUs (#include <zstd.h>, <zlib.h>, <lz4.h>) have no
# preprocessor guards — they always compile. required:false mirrors the task;
# if a lib is absent the dependency is skipped, and the missing header surfaces
# as a clear compile error at the relevant TU.
zstd_dep = dependency('libzstd', required: false)
if not zstd_dep.found()
    zstd_dep = dependency('zstd', required: false)
endif
zlib_dep = dependency('zlib', required: false)
lz4_dep = dependency('liblz4', required: false)
if not lz4_dep.found()
    lz4_dep = dependency('lz4', required: false)
endif

compression_deps = []
foreach d : [zstd_dep, zlib_dep, lz4_dep]
    if d.found()
        compression_deps += d
    endif
endforeach

# --- System libraries (Unix) --------------------------------------------------
# libm for log() in bloom_filter.c; pthreads for the reader worker pool.
m_dep = cc.find_library('m', required: false)
thread_dep = dependency('threads', required: false)

carquet_inc = include_directories('include')
# Internal headers (e.g. simd/simd_unaligned.h) are included with paths
# relative to src/, so src/ must be on the private include path for every TU.
# (Mirrors upstream target_include_directories(... PRIVATE src).)
src_inc = include_directories('src')
internal_inc = [carquet_inc, src_inc]

# --- Baseline sources ---------------------------------------------------------
# Every .c under src/ except src/cli/ (the standalone CLI tool, separate target)
# and the per-arch SIMD TUs (built separately below with ISA-specific flags).
carquet_sources = [
    # core
    'src/core/arena.c',
    'src/core/allocator.c',
    'src/core/buffer.c',
    'src/core/bitpack.c',
    'src/core/endian.c',
    'src/core/error.c',
    'src/core/geo_wkb.c',
    # thrift
    'src/thrift/thrift_decode.c',
    'src/thrift/thrift_encode.c',
    'src/thrift/parquet_types.c',
    # encoding
    'src/encoding/plain.c',
    'src/encoding/rle.c',
    'src/encoding/delta.c',
    'src/encoding/delta_length.c',
    'src/encoding/delta_strings.c',
    'src/encoding/dictionary.c',
    'src/encoding/byte_stream_split.c',
    # compression
    'src/compression/lz4.c',
    'src/compression/snappy.c',
    'src/compression/zstd.c',
    'src/compression/gzip.c',
    'src/compression/custom.c',
    # simd dispatch (ISA-specific TUs added per-arch below)
    'src/simd/dispatch.c',
    # reader
    'src/reader/file_reader.c',
    'src/reader/arrow_schema_read.c',
    'src/reader/arrow_c_export.c',
    'src/reader/arrow_c_read.c',
    'src/reader/row_group_reader.c',
    'src/reader/column_reader.c',
    'src/reader/page_reader.c',
    'src/reader/batch_reader.c',
    'src/reader/page_filter.c',
    'src/reader/statistics.c',
    'src/reader/mmap_reader.c',
    'src/reader/worker_pool.c',
    # writer
    'src/writer/file_writer.c',
    'src/writer/row_group_writer.c',
    'src/writer/column_writer.c',
    'src/writer/page_writer.c',
    'src/writer/arrow_schema.c',
    'src/writer/arrow_c_import.c',
    # metadata
    'src/metadata/schema.c',
    'src/metadata/bloom_filter.c',
    'src/metadata/page_index.c',
    # util
    'src/util/crc32.c',
    'src/util/xxhash.c',
]

# --- Architecture & SIMD per-file flags ---------------------------------------
# Runtime CPU-feature detection (simd/detect.c) must stay correct under mixed
# LLVM/macOS toolchains — exclude it from LTO (mirrors upstream CMake).
# Each ISA TU is built as its own object with its -m<isa> + -fno-lto so the
# codegen can't be inlined into baseline callers (would SIGILL before the
# dispatch-table indirection runs).

# carquet uses POSIX.1-2008 (fseeko, ftello) and BSD (madvise) functions that
# are gated behind feature-test macros on glibc (Linux) but unconditionally
# available on macOS/BSD. On glibc, _DEFAULT_SOURCE exposes the BSD subset
# (madvise, MADV_SEQUENTIAL) and _POSIX_C_SOURCE exposes fseeko/ftello.
# On Apple libc, defining _POSIX_C_SOURCE actually HIDES BSD extensions
# (MADV_SEQUENTIAL disappears), so we only add these on Linux.
if host_machine.system() == 'linux'
    carquet_defines = ['-D_POSIX_C_SOURCE=200809L', '-D_DEFAULT_SOURCE']
else
    carquet_defines = []
endif
simd_tu_libs = []   # per-file TU static libs (built with ISA-specific c_args)
simd_tu_srcs = []   # source file each TU lib compiles (for extract_objects)
cpu_family = host_machine.cpu_family()
is_gcc_like = cc.get_id() in ['gcc', 'clang']

# detect.c: -fno-lto on gcc-like to avoid miscompiled feature flags.
detect_extra = is_gcc_like and cc.has_argument('-fno-lto') ? ['-fno-lto'] : []
simd_tu_libs += static_library('carquet_detect_tu',
    'src/simd/detect.c',
    c_args: detect_extra,
    include_directories: internal_inc,
    dependencies: [compression_deps, m_dep, thread_dep],
)
simd_tu_srcs += ['src/simd/detect.c']

if cpu_family == 'x86' or cpu_family == 'x86_64'
    carquet_defines += ['-DCARQUET_ARCH_X86',
                        '-DCARQUET_ENABLE_SSE',
                        '-DCARQUET_ENABLE_AVX',
                        '-DCARQUET_ENABLE_AVX2',
                        '-DCARQUET_ENABLE_AVX512']

    # Map of ISA TU -> per-file compile flags (gcc/clang only).
    simd_tus = is_gcc_like ? {
        'src/simd/x86/sse_ops.c':    ['-msse4.2', '-fno-lto'],
        'src/simd/x86/avx_ops.c':    ['-mavx', '-msse4.2', '-fno-lto'],
        'src/simd/x86/avx2_ops.c':   ['-mavx2', '-mbmi2', '-fno-lto'],
        'src/simd/x86/avx512_ops.c': ['-mavx512f', '-mavx512bw', '-mavx512vl', '-fno-lto'],
    } : {
        'src/simd/x86/sse_ops.c':    [],
        'src/simd/x86/avx_ops.c':    ['/arch:AVX'],
        'src/simd/x86/avx2_ops.c':   ['/arch:AVX2'],
        'src/simd/x86/avx512_ops.c': ['/arch:AVX512'],
    }

    foreach src, flags : simd_tus
        simd_tu_libs += static_library('carquet_' + src.underscorify() + '_tu',
            src,
            c_args: flags,
            include_directories: internal_inc,
            dependencies: [compression_deps, m_dep, thread_dep],
        )
        simd_tu_srcs += [src]
    endforeach
elif cpu_family == 'arm' or cpu_family == 'aarch64'
    carquet_defines += ['-DCARQUET_ARCH_ARM', '-DCARQUET_ENABLE_NEON']
    simd_tu_libs += static_library('carquet_neon_tu',
        'src/simd/arm/neon_ops.c',
        include_directories: internal_inc,
        dependencies: [compression_deps, m_dep, thread_dep],
    )
    simd_tu_srcs += ['src/simd/arm/neon_ops.c']
    # SVE is opt-in upstream (needs -march=armv8-a+sve); left out by default.
endif

# Pull the compiled TU objects out of their per-file archives so the final
# libcarquet.a is a single self-contained archive (matches the CMake build).
simd_objects = []
i = 0
foreach tu_lib : simd_tu_libs
    simd_objects += tu_lib.extract_objects(simd_tu_srcs[i])
    i = i + 1
endforeach

# --- Build the static library -------------------------------------------------
carquet_lib = static_library('carquet',
    carquet_sources,
    objects: simd_objects,         # fold the ISA/detect TUs into the archive
    c_args: carquet_defines,
    include_directories: internal_inc,
    dependencies: [compression_deps, m_dep, thread_dep],
    gnu_symbol_visibility: 'hidden',
)

carquet_dep = declare_dependency(
    link_with: carquet_lib,
    include_directories: carquet_inc,   # public API only — src/ stays private
    dependencies: [compression_deps, thread_dep],
    compile_args: carquet_defines,
)

meson.override_dependency('carquet', carquet_dep)
