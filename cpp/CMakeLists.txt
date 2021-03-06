#=============================================================================
# Copyright (c) 2018-2020, NVIDIA CORPORATION.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#=============================================================================

cmake_minimum_required(VERSION 3.18 FATAL_ERROR)

# If `CMAKE_CUDA_ARCHITECTURES` is not defined, build for all supported architectures. If
# `CMAKE_CUDA_ARCHITECTURES` is set to an empty string (""), build for only the current
# architecture. If `CMAKE_CUDA_ARCHITECTURES` is specified by the user, use user setting.

# This needs to be run before enabling the CUDA language due to the default initialization behavior
# of `CMAKE_CUDA_ARCHITECTURES`, https://gitlab.kitware.com/cmake/cmake/-/issues/21302
if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES OR CMAKE_CUDA_ARCHITECTURES STREQUAL "ALL")
  set(CUDF_BUILD_FOR_ALL_ARCHS TRUE)
elseif(CMAKE_CUDA_ARCHITECTURES STREQUAL "")
  set(CUDF_BUILD_FOR_DETECTED_ARCHS TRUE)
endif()

project(CUDF VERSION 0.19.0 LANGUAGES C CXX)

# Needed because GoogleBenchmark changes the state of FindThreads.cmake,
# causing subsequent runs to have different values for the `Threads::Threads` target.
# Setting this flag ensures `Threads::Threads` is the same value in first run and subsequent runs.
set(THREADS_PREFER_PTHREAD_FLAG ON)

###################################################################################################
# - build options ---------------------------------------------------------------------------------

option(USE_NVTX "Build with NVTX support" ON)
option(BUILD_TESTS "Configure CMake to build tests" ON)
option(BUILD_BENCHMARKS "Configure CMake to build (google) benchmarks" OFF)
option(BUILD_SHARED_LIBS "Build cuDF shared libraries" ON)
option(JITIFY_USE_CACHE "Use a file cache for JIT compiled kernels" ON)
option(CUDF_USE_ARROW_STATIC "Build and statically link Arrow libraries" OFF)
option(PER_THREAD_DEFAULT_STREAM "Build with per-thread default stream" OFF)
option(DISABLE_DEPRECATION_WARNING "Disable warnings generated from deprecated declarations." OFF)
# Option to enable line info in CUDA device compilation to allow introspection when profiling / memchecking
option(CUDA_ENABLE_LINEINFO "Enable the -lineinfo option for nvcc (useful for cuda-memcheck / profiler" OFF)
# cudart can be statically linked or dynamically linked. The python ecosystem wants dynamic linking
option(CUDA_STATIC_RUNTIME "Statically link the CUDA runtime" OFF)

message(VERBOSE "CUDF: Build with NVTX support: ${USE_NVTX}")
message(VERBOSE "CUDF: Configure CMake to build tests: ${BUILD_TESTS}")
message(VERBOSE "CUDF: Configure CMake to build (google) benchmarks: ${BUILD_BENCHMARKS}")
message(VERBOSE "CUDF: Build cuDF shared libraries: ${BUILD_SHARED_LIBS}")
message(VERBOSE "CUDF: Use a file cache for JIT compiled kernels: ${JITIFY_USE_CACHE}")
message(VERBOSE "CUDF: Build and statically link Arrow libraries: ${CUDF_USE_ARROW_STATIC}")
message(VERBOSE "CUDF: Build with per-thread default stream: ${PER_THREAD_DEFAULT_STREAM}")
message(VERBOSE "CUDF: Disable warnings generated from deprecated declarations: ${DISABLE_DEPRECATION_WARNING}")
message(VERBOSE "CUDF: Enable the -lineinfo option for nvcc (useful for cuda-memcheck / profiler: ${CUDA_ENABLE_LINEINFO}")
message(VERBOSE "CUDF: Statically link the CUDA runtime: ${CUDA_STATIC_RUNTIME}")

# Set a default build type if none was specified
set(DEFAULT_BUILD_TYPE "Release")
set(CUDF_BUILD_TESTS ${BUILD_TESTS})
set(CUDF_BUILD_BENCHMARKS ${BUILD_BENCHMARKS})

set(CUDF_CXX_FLAGS "")
set(CUDF_CUDA_FLAGS "")
set(CUDF_CXX_DEFINITIONS "")
set(CUDF_CUDA_DEFINITIONS "")

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(VERBOSE "CUDF: Setting build type to '${DEFAULT_BUILD_TYPE}' since none specified.")
    set(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}" CACHE STRING  "Choose the type of build." FORCE)
    # Set the possible values of build type for cmake-gui
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)

# Set RMM logging level
set(RMM_LOGGING_LEVEL "INFO" CACHE STRING "Choose the logging level.")
set_property(CACHE RMM_LOGGING_LEVEL PROPERTY STRINGS "TRACE" "DEBUG" "INFO" "WARN" "ERROR" "CRITICAL" "OFF")
message(VERBOSE "CUDF: RMM_LOGGING_LEVEL = '${RMM_LOGGING_LEVEL}'.")

if (NOT CUDF_GENERATED_INCLUDE_DIR)
    set(CUDF_GENERATED_INCLUDE_DIR ${CUDF_BINARY_DIR})
endif()

###################################################################################################
# - conda environment -----------------------------------------------------------------------------

if("$ENV{CONDA_BUILD}" STREQUAL "1")
    set(CMAKE_PREFIX_PATH "$ENV{BUILD_PREFIX};$ENV{PREFIX};${CMAKE_PREFIX_PATH}")
    set(CONDA_INCLUDE_DIRS "$ENV{BUILD_PREFIX}/include" "$ENV{PREFIX}/include")
    set(CONDA_LINK_DIRS "$ENV{BUILD_PREFIX}/lib" "$ENV{PREFIX}/lib")
    message(VERBOSE "CUDF: Conda build detected, CMAKE_PREFIX_PATH set to: ${CMAKE_PREFIX_PATH}")
elseif(DEFINED ENV{CONDA_PREFIX})
    set(CMAKE_PREFIX_PATH "$ENV{CONDA_PREFIX};${CMAKE_PREFIX_PATH}")
    set(CONDA_INCLUDE_DIRS "$ENV{CONDA_PREFIX}/include")
    set(CONDA_LINK_DIRS "$ENV{CONDA_PREFIX}/lib")
    message(VERBOSE "CUDF: Conda environment detected, CMAKE_PREFIX_PATH set to: ${CMAKE_PREFIX_PATH}")
endif()

###################################################################################################
# - compiler options ------------------------------------------------------------------------------

# * find CUDAToolkit package
# * determine GPU architectures
# * enable the CMake CUDA language
# * set other CUDA compilation flags
include(cmake/Modules/ConfigureCUDA.cmake)

###################################################################################################
# - dependencies ----------------------------------------------------------------------------------

# find zlib
find_package(ZLIB REQUIRED)
# add third party dependencies using CPM
include(cmake/thirdparty/CUDF_GetCPM.cmake)
# find boost
include(cmake/thirdparty/CUDF_FindBoost.cmake)
# find jitify
include(cmake/thirdparty/CUDF_GetJitify.cmake)
# find thrust/cub
include(cmake/thirdparty/CUDF_GetThrust.cmake)
# find rmm
include(cmake/thirdparty/CUDF_GetRMM.cmake)
# find arrow
include(cmake/thirdparty/CUDF_GetArrow.cmake)
# find dlpack
include(cmake/thirdparty/CUDF_GetDLPack.cmake)
# find libcu++
include(cmake/thirdparty/CUDF_GetLibcudacxx.cmake)
# Stringify libcudf and libcudacxx headers used in JIT operations
include(cmake/Modules/StringifyJITHeaders.cmake)

###################################################################################################
# - library targets -------------------------------------------------------------------------------

add_library(cudf
    src/aggregation/aggregation.cpp
    src/aggregation/aggregation.cu
    src/aggregation/result_cache.cpp
    src/ast/linearizer.cpp
    src/ast/transform.cu
    src/binaryop/binaryop.cpp
    src/binaryop/compiled/binary_ops.cu
    src/binaryop/jit/code/kernel.cpp
    src/binaryop/jit/code/operation.cpp
    src/binaryop/jit/code/traits.cpp
    src/bitmask/null_mask.cu
    src/column/column.cu
    src/column/column_device_view.cu
    src/column/column_factories.cpp
    src/column/column_view.cpp
    src/comms/ipc/ipc.cpp
    src/copying/concatenate.cu
    src/copying/contiguous_split.cu
    src/copying/copy.cpp
    src/copying/copy.cu
    src/copying/copy_range.cu
    src/copying/gather.cu
    src/copying/get_element.cu
    src/copying/pack.cpp
    src/copying/sample.cu
    src/copying/scatter.cu
    src/copying/shift.cu
    src/copying/slice.cpp
    src/copying/split.cpp
    src/datetime/datetime_ops.cu
    src/dictionary/add_keys.cu
    src/dictionary/decode.cu
    src/dictionary/detail/concatenate.cu
    src/dictionary/detail/merge.cu
    src/dictionary/dictionary_column_view.cpp
    src/dictionary/dictionary_factories.cu
    src/dictionary/encode.cu
    src/dictionary/remove_keys.cu
    src/dictionary/replace.cu
    src/dictionary/search.cu
    src/dictionary/set_keys.cu
    src/filling/fill.cu
    src/filling/repeat.cu
    src/filling/sequence.cu
    src/groupby/groupby.cu
    src/groupby/hash/groupby.cu
    src/groupby/sort/group_argmax.cu
    src/groupby/sort/group_argmin.cu
    src/groupby/sort/groupby.cu
    src/groupby/sort/group_collect.cu
    src/groupby/sort/group_count.cu
    src/groupby/sort/group_max.cu
    src/groupby/sort/group_min.cu
    src/groupby/sort/group_nth_element.cu
    src/groupby/sort/group_nunique.cu
    src/groupby/sort/group_quantiles.cu
    src/groupby/sort/group_std.cu
    src/groupby/sort/group_sum.cu
    src/groupby/sort/sort_helper.cu
    src/hash/hashing.cu
    src/interop/dlpack.cpp
    src/interop/from_arrow.cpp
    src/interop/to_arrow.cpp
    src/io/avro/avro.cpp
    src/io/avro/avro_gpu.cu
    src/io/avro/reader_impl.cu
    src/io/comp/brotli_dict.cpp
    src/io/comp/cpu_unbz2.cpp
    src/io/comp/debrotli.cu
    src/io/comp/gpuinflate.cu
    src/io/comp/snap.cu
    src/io/comp/uncomp.cpp
    src/io/comp/unsnap.cu
    src/io/csv/csv_gpu.cu
    src/io/csv/durations.cu
    src/io/csv/reader_impl.cu
    src/io/csv/writer_impl.cu
    src/io/functions.cpp
    src/io/json/json_gpu.cu
    src/io/json/reader_impl.cu
    src/io/orc/dict_enc.cu
    src/io/orc/orc.cpp
    src/io/orc/reader_impl.cu
    src/io/orc/stats_enc.cu
    src/io/orc/stripe_data.cu
    src/io/orc/stripe_enc.cu
    src/io/orc/stripe_init.cu
    src/io/orc/timezone.cpp
    src/io/orc/writer_impl.cu
    src/io/parquet/compact_protocol_writer.cpp
    src/io/parquet/page_data.cu
    src/io/parquet/page_dict.cu
    src/io/parquet/page_enc.cu
    src/io/parquet/page_hdr.cu
    src/io/parquet/parquet.cpp
    src/io/parquet/reader_impl.cu
    src/io/parquet/writer_impl.cu
    src/io/statistics/column_stats.cu
    src/io/utilities/data_sink.cpp
    src/io/utilities/datasource.cpp
    src/io/utilities/parsing_utils.cu
    src/io/utilities/type_conversion.cpp
    src/jit/cache.cpp
    src/jit/launcher.cpp
    src/jit/parser.cpp
    src/jit/type.cpp
    src/join/cross_join.cu
    src/join/hash_join.cu
    src/join/join.cu
    src/join/semi_join.cu
    src/lists/contains.cu
    src/lists/copying/concatenate.cu
    src/lists/copying/copying.cu
    src/lists/copying/gather.cu
    src/lists/copying/segmented_gather.cu
    src/lists/count_elements.cu
    src/lists/extract.cu
    src/lists/drop_list_duplicates.cu
    src/lists/lists_column_factories.cu
    src/lists/lists_column_view.cu
    src/lists/segmented_sort.cu
    src/merge/merge.cu
    src/partitioning/partitioning.cu
    src/partitioning/round_robin.cu
    src/quantiles/quantile.cu
    src/quantiles/quantiles.cu
    src/reductions/all.cu
    src/reductions/any.cu
    src/reductions/max.cu
    src/reductions/mean.cu
    src/reductions/min.cu
    src/reductions/minmax.cu
    src/reductions/nth_element.cu
    src/reductions/product.cu
    src/reductions/reductions.cpp
    src/reductions/scan.cu
    src/reductions/std.cu
    src/reductions/sum.cu
    src/reductions/sum_of_squares.cu
    src/reductions/var.cu
    src/replace/clamp.cu
    src/replace/nans.cu
    src/replace/nulls.cu
    src/replace/replace.cu
    src/reshape/byte_cast.cu
    src/reshape/explode.cu
    src/reshape/interleave_columns.cu
    src/reshape/tile.cu
    src/rolling/grouped_rolling.cu
    src/rolling/jit/code/kernel.cpp
    src/rolling/jit/code/operation.cpp
    src/rolling/rolling.cu
    src/round/round.cu
    src/scalar/scalar.cpp
    src/scalar/scalar_factories.cpp
    src/search/search.cu
    src/sort/is_sorted.cu
    src/sort/rank.cu
    src/sort/segmented_sort.cu
    src/sort/sort_column.cu
    src/sort/sort.cu
    src/sort/stable_sort_column.cu
    src/sort/stable_sort.cu
    src/stream_compaction/apply_boolean_mask.cu
    src/stream_compaction/distinct_count.cu
    src/stream_compaction/drop_duplicates.cu
    src/stream_compaction/drop_nans.cu
    src/stream_compaction/drop_nulls.cu
    src/strings/attributes.cu
    src/strings/capitalize.cu
    src/strings/case.cu
    src/strings/char_types/char_cases.cu
    src/strings/char_types/char_types.cu
    src/strings/combine.cu
    src/strings/contains.cu
    src/strings/convert/convert_booleans.cu
    src/strings/convert/convert_datetime.cu
    src/strings/convert/convert_durations.cu
    src/strings/convert/convert_fixed_point.cu
    src/strings/convert/convert_floats.cu
    src/strings/convert/convert_hex.cu
    src/strings/convert/convert_integers.cu
    src/strings/convert/convert_ipv4.cu
    src/strings/convert/convert_urls.cu
    src/strings/copying/concatenate.cu
    src/strings/copying/copying.cu
    src/strings/extract.cu
    src/strings/filling/fill.cu
    src/strings/filter_chars.cu
    src/strings/findall.cu
    src/strings/find.cu
    src/strings/find_multiple.cu
    src/strings/padding.cu
    src/strings/regex/regcomp.cpp
    src/strings/regex/regexec.cu
    src/strings/replace/backref_re.cu
    src/strings/replace/backref_re_large.cu
    src/strings/replace/backref_re_medium.cu
    src/strings/replace/multi_re.cu
    src/strings/replace/replace.cu
    src/strings/replace/replace_re.cu
    src/strings/split/partition.cu
    src/strings/split/split.cu
    src/strings/split/split_record.cu
    src/strings/strings_column_factories.cu
    src/strings/strings_column_view.cu
    src/strings/strings_scalar_factories.cpp
    src/strings/strip.cu
    src/strings/substring.cu
    src/strings/translate.cu
    src/strings/utilities.cu
    src/strings/wrap.cu
    src/structs/copying/concatenate.cu
    src/structs/structs_column_factories.cu
    src/structs/structs_column_view.cpp
    src/structs/utilities.cu
    src/table/table.cpp
    src/table/table_device_view.cu
    src/table/table_view.cpp
    src/text/detokenize.cu
    src/text/edit_distance.cu
    src/text/generate_ngrams.cu
    src/text/ngrams_tokenize.cu
    src/text/normalize.cu
    src/text/replace.cu
    src/text/stemmer.cu
    src/text/subword/data_normalizer.cu
    src/text/subword/load_hash_file.cu
    src/text/subword/subword_tokenize.cu
    src/text/subword/wordpiece_tokenizer.cu
    src/text/tokenize.cu
    src/transform/bools_to_mask.cu
    src/transform/encode.cu
    src/transform/jit/code/kernel.cpp
    src/transform/mask_to_bools.cu
    src/transform/nans_to_nulls.cu
    src/transform/transform.cpp
    src/transpose/transpose.cu
    src/unary/cast_ops.cu
    src/unary/math_ops.cu
    src/unary/nan_ops.cu
    src/unary/null_ops.cu
    src/utilities/default_stream.cpp
)

set_target_properties(cudf
    PROPERTIES BUILD_RPATH                         "\$ORIGIN"
               # set target compile options
               CXX_STANDARD                        14
               CXX_STANDARD_REQUIRED               ON
               CUDA_STANDARD                       14
               CUDA_STANDARD_REQUIRED              ON
               POSITION_INDEPENDENT_CODE           ON
               INTERFACE_POSITION_INDEPENDENT_CODE ON
)

target_compile_options(cudf
            PRIVATE "$<$<COMPILE_LANGUAGE:CXX>:${CUDF_CXX_FLAGS}>"
                    "$<$<COMPILE_LANGUAGE:CUDA>:${CUDF_CUDA_FLAGS}>"
)

target_compile_definitions(cudf
            PUBLIC "$<$<COMPILE_LANGUAGE:CXX>:${CUDF_CXX_DEFINITIONS}>"
                   "$<$<COMPILE_LANGUAGE:CUDA>:${CUDF_CUDA_DEFINITIONS}>"
)

# Disable Jitify log printing. See https://github.com/NVIDIA/jitify/issues/79
target_compile_definitions(cudf PRIVATE "JITIFY_PRINT_LOG=0")

# Specify include paths for the current target and dependents
target_include_directories(cudf
           PUBLIC      "$<BUILD_INTERFACE:${DLPACK_INCLUDE_DIR}>"
                       "$<BUILD_INTERFACE:${JITIFY_INCLUDE_DIR}>"
                       "$<BUILD_INTERFACE:${LIBCUDACXX_INCLUDE_DIR}>"
                       "$<BUILD_INTERFACE:${CUDF_SOURCE_DIR}/include>"
                       "$<BUILD_INTERFACE:${CUDF_GENERATED_INCLUDE_DIR}/include>"
           PRIVATE     "$<BUILD_INTERFACE:${CUDF_SOURCE_DIR}/src>"
           INTERFACE   "$<INSTALL_INTERFACE:include>")

# Add Conda library paths if specified
if(CONDA_LINK_DIRS)
    target_link_directories(cudf PUBLIC "$<BUILD_INTERFACE:${CONDA_LINK_DIRS}>")
endif()

# Add Conda include paths if specified
if(CONDA_INCLUDE_DIRS)
    target_include_directories(cudf PUBLIC "$<BUILD_INTERFACE:${CONDA_INCLUDE_DIRS}>")
endif()

# Instruct jitify to use the kernel JIT cache
if(JITIFY_USE_CACHE)
    target_compile_definitions(cudf PUBLIC JITIFY_USE_CACHE "CUDF_VERSION=${PROJECT_VERSION}")
endif()

# Per-thread default stream
if(PER_THREAD_DEFAULT_STREAM)
    target_compile_definitions(cudf PUBLIC CUDA_API_PER_THREAD_DEFAULT_STREAM)
endif()

# Disable NVTX if necessary
if(NOT USE_NVTX)
    target_compile_definitions(cudf PUBLIC NVTX_DISABLE)
endif()

# Define spdlog level
target_compile_definitions(cudf PUBLIC "SPDLOG_ACTIVE_LEVEL=SPDLOG_LEVEL_${RMM_LOGGING_LEVEL}")

# Compile stringified JIT sources first
add_dependencies(cudf stringify_run)

# Specify the target module library dependencies
target_link_libraries(cudf
           PUBLIC ZLIB::ZLIB
                  Boost::filesystem
                  ${ARROW_LIBRARIES}
                  cudf::Thrust
                  rmm::rmm)

if(CUDA_STATIC_RUNTIME)
    target_link_libraries(cudf PUBLIC CUDA::nvrtc CUDA::cudart_static CUDA::cuda_driver)
else()
    target_link_libraries(cudf PUBLIC CUDA::nvrtc CUDA::cudart CUDA::cuda_driver)
endif()

file(WRITE "${CUDF_BINARY_DIR}/fatbin.ld"
[=[
SECTIONS
{
  .nvFatBinSegment : { *(.nvFatBinSegment) }
  .nv_fatbin : { *(.nv_fatbin) }
}
]=])
target_link_options(cudf PRIVATE "${CUDF_BINARY_DIR}/fatbin.ld")

add_library(cudf::cudf ALIAS cudf)

###################################################################################################
# - tests and benchmarks --------------------------------------------------------------------------
###################################################################################################

if (CUDF_BUILD_TESTS OR CUDF_BUILD_BENCHMARKS)
    # Find or install GoogleTest
    CPMFindPackage(NAME GTest
        VERSION         1.10.0
        GIT_REPOSITORY  https://github.com/google/googletest.git
        GIT_TAG         release-1.10.0
        GIT_SHALLOW     TRUE
        OPTIONS         "INSTALL_GTEST OFF"
        # googletest >= 1.10.0 provides a cmake config file -- use it if it exists
        FIND_PACKAGE_ARGUMENTS "CONFIG")
    # Add GTest aliases if they don't already exist.
    # Assumes if GTest::gtest doesn't exist, the others don't either.
    # TODO: Is this always a valid assumption?
    if(NOT TARGET GTest::gtest)
        add_library(GTest::gtest ALIAS gtest)
        add_library(GTest::gmock ALIAS gmock)
        add_library(GTest::gtest_main ALIAS gtest_main)
        add_library(GTest::gmock_main ALIAS gmock_main)
    endif()
    if(GTest_ADDED)
        install(TARGETS gmock
                        gtest
                        gmock_main
                        gtest_main
            DESTINATION lib
            EXPORT cudf-targets)
    endif()
endif()

###################################################################################################
# - add tests -------------------------------------------------------------------------------------

if(CUDF_BUILD_TESTS)
    # include CTest module -- automatically calls enable_testing()
    include(CTest)
    add_subdirectory(tests)
endif()

###################################################################################################
# - add benchmarks --------------------------------------------------------------------------------

if(CUDF_BUILD_BENCHMARKS)
    # Find or install GoogleBench
    CPMFindPackage(NAME benchmark
        VERSION         1.5.2
        GIT_REPOSITORY  https://github.com/google/benchmark.git
        GIT_TAG         v1.5.2
        GIT_SHALLOW     TRUE
        OPTIONS         "BENCHMARK_ENABLE_TESTING OFF"
                        "BENCHMARK_ENABLE_INSTALL OFF")
    if(benchmark_ADDED)
        install(TARGETS benchmark
                        benchmark_main
            DESTINATION lib
            EXPORT cudf-targets)
    endif()
    add_subdirectory(benchmarks)
endif()

###################################################################################################
# - install targets -------------------------------------------------------------------------------

include(GNUInstallDirs)

set(INSTALL_CONFIGDIR ${CMAKE_INSTALL_LIBDIR}/cmake/cudf)
set(CMAKE_INSTALL_DEFAULT_COMPONENT_NAME cudf)

# install target for cudf_base and the proxy libcudf.so
install(TARGETS cudf
        DESTINATION lib
        EXPORT cudf-targets)

install(DIRECTORY
            ${CUDF_SOURCE_DIR}/include/cudf
            ${CUDF_SOURCE_DIR}/include/cudf_test
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(DIRECTORY
            ${CUDF_GENERATED_INCLUDE_DIR}/include/libcxx
            ${CUDF_GENERATED_INCLUDE_DIR}/include/libcudacxx
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/libcudf)

include(CMakePackageConfigHelpers)

configure_package_config_file(cmake/cudf-config.cmake.in "${CUDF_BINARY_DIR}/cmake/cudf-config.cmake"
                              INSTALL_DESTINATION "${INSTALL_CONFIGDIR}")

write_basic_package_version_file("${CUDF_BINARY_DIR}/cmake/cudf-config-version.cmake"
                                 COMPATIBILITY SameMinorVersion)

install(FILES   "${CUDF_BINARY_DIR}/cmake/cudf-config.cmake"
                "${CUDF_BINARY_DIR}/cmake/cudf-config-version.cmake"
                "${CUDF_SOURCE_DIR}/cmake/install/FindThrust.cmake"
    DESTINATION "${INSTALL_CONFIGDIR}")

install(EXPORT  cudf-targets
    FILE        cudf-targets.cmake
    NAMESPACE   cudf::
    DESTINATION "${INSTALL_CONFIGDIR}")

################################################################################################
# - build export -------------------------------------------------------------------------------
configure_package_config_file(cmake/cudf-build-config.cmake.in ${CUDF_BINARY_DIR}/cudf-config.cmake
                              INSTALL_DESTINATION ${CUDF_BINARY_DIR})

write_basic_package_version_file(${CUDF_BINARY_DIR}/cudf-config-version.cmake
                                 COMPATIBILITY SameMinorVersion)

if(TARGET arrow_shared)
    get_target_property(arrow_is_imported arrow_shared IMPORTED)
    if(NOT arrow_is_imported)
        export(TARGETS arrow_shared arrow_cuda_shared
            FILE ${CUDF_BINARY_DIR}/cudf-arrow-targets.cmake
            NAMESPACE   cudf::)
    endif()
elseif(TARGET arrow_static)
    get_target_property(arrow_is_imported arrow_static IMPORTED)
    if(NOT arrow_is_imported)
        export(TARGETS arrow_static arrow_cuda_static
            FILE ${CUDF_BINARY_DIR}/cudf-arrow-targets.cmake
            NAMESPACE   cudf::)
    endif()
endif()

export(EXPORT cudf-targets
    FILE ${CUDF_BINARY_DIR}/cudf-targets.cmake
    NAMESPACE   cudf::)


###################################################################################################
# - make documentation ----------------------------------------------------------------------------

# doc targets for cuDF
add_custom_command(OUTPUT CUDF_DOXYGEN
                   WORKING_DIRECTORY ${CUDF_SOURCE_DIR}/doxygen
                   COMMAND doxygen Doxyfile
                   VERBATIM)

add_custom_target(docs_cudf DEPENDS CUDF_DOXYGEN)
