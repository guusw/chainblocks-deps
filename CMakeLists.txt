cmake_minimum_required(VERSION 3.21)
project(Deps)

set(BUILD_ID "x86_64-pc-windows-gnu" CACHE STRING "The identifier for the build")
set(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/install/${BUILD_ID})

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
include(Platform)
include(ExternalProject)

set(COMMON_ARGS 
    CMAKE_ARGS 
        -DBUILD_SHARED_LIBS=OFF
        -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} 
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
)

function(add_header_lib NAME INCLUDE_PATH)
    add_library(${NAME} INTERFACE)
    target_include_directories(${NAME} INTERFACE
        $<INSTALL_INTERFACE:include>
    )
    install(DIRECTORY ${INCLUDE_PATH}
        DESTINATION include
        FILES_MATCHING 
            PATTERN "*.h" PATTERN "*.hpp" PATTERN "*.hxx"
            PATTERN ".git" EXCLUDE
            PATTERN ".git*" EXCLUDE
    )
    install(TARGETS ${NAME} EXPORT ${NAME}Targets)
    install(EXPORT ${NAME}Targets
        FILE ${NAME}Config.cmake
        DESTINATION lib/cmake/${NAME}
    )
endfunction()

ExternalProject_Add(json SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/json ${COMMON_ARGS})
ExternalProject_Add(kcp SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kcp ${COMMON_ARGS})
ExternalProject_Add(replxx SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/replxx ${COMMON_ARGS})
ExternalProject_Add(magic_enum SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/magic_enum ${COMMON_ARGS})
ExternalProject_Add(nameof SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/nameof ${COMMON_ARGS})
ExternalProject_Add(pareto SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/pareto
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBUILD_PYTHON_BINDING=OFF
        -DBUILD_TESTS=OFF
        -DBUILD_EXAMPLES=OFF
        -DBUILD_MATPLOT_TARGETS=OFF
        -DBUILD_BENCHMARKS=OFF
)

ExternalProject_Add(SDL SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/SDL
    ${COMMON_ARGS}
    CMAKE_ARGS 
        -DSDL_RENDER_DISABLED=ON
)

ExternalProject_Add(bgfx
    GIT_REPOSITORY https://github.com/bkaradzic/bgfx.cmake.git
    GIT_TAG v1.115.7908-b7b7cd7
    GIT_NO_SUBMODULES
    PATCH_COMMAND git apply --whitespace=fix ${CMAKE_CURRENT_SOURCE_DIR}/Patches/bgfx.cmake.patch
    ${COMMON_ARGS}
    CMAKE_ARGS 
        -DBX_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bx
        -DBIMG_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bimg
        -DBGFX_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bgfx
        -DBIMG_DECODE_ENABLE=OFF
        -DBGFX_BUILD_EXAMPLES=OFF
)

ExternalProject_Add(wasm3 SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/wasm3
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBUILD_NATIVE=OFF
)

ExternalProject_Add(snappy SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/snappy
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DSNAPPY_BUILD_TESTS=OFF
)

ExternalProject_Add(brotli SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/brotli
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBROTLI_DISABLE_TESTS=ON
)

ExternalProject_Add(Catch2 SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/Catch2
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DCATCH_INSTALL_DOCS=OFF
)

ExternalProject_Add(tinygltf SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/tinygltf
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DTINYGLTF_BUILD_LOADER_EXAMPLE=OFF
)

ExternalProject_Add(cpp-taskflow SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/cpp-taskflow
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DTF_BUILD_TESTS=OFF
        -DTF_BUILD_EXAMPLES=OFF
)

ExternalProject_Add(boost
    GIT_REPOSITORY https://github.com/boostorg/boost.git
    GIT_TAG boost-1.77.0
    PATCH_COMMAND cd libs/context && git apply --whitespace=fix ${CMAKE_CURRENT_SOURCE_DIR}/Patches/boost-context.patch
    LIST_SEPARATOR |
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DCMAKE_ASM_COMPILER=as
        -DBOOST_INCLUDE_LIBRARIES=context|filesystem|stacktrace|atomic
        -BOOST_USE_VALGRIND=${VALGRIND}
)

ExternalProject_Add(kissfft SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kissfft
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DCMAKE_INSTALL_INCLUDEDIR=${CMAKE_INSTALL_PREFIX}/include
        -DCMAKE_INSTALL_LIBDIR=${CMAKE_INSTALL_PREFIX}/lib
        -DKISSFFT_STATIC=ON
        -DKISSFFT_TEST=OFF
        -DKISSFFT_TOOLS=OFF
)

ExternalProject_Add(xxHash 
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/xxHash/cmake_unofficial
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBUILD_SHARED_LIBS=OFF
)

if(WIN32)
    set(CONFIGURE_TARGET x86_64-w64-windows-gnu)
    ExternalProject_Add(
        backtrace
        URL ${CMAKE_CURRENT_SOURCE_DIR}/libbacktrace
        CONFIGURE_COMMAND sh configure --prefix=${INSTALL_ROOT} --target=${CONFIGURE_TARGET}
        BUILD_COMMAND make
        INSTALL_COMMAND make install
        BUILD_IN_SOURCE 1
    )
endif()

add_library(stb INTERFACE)
target_include_directories(stb INTERFACE
    $<INSTALL_INTERFACE:include>
)
add_library(stb_vorbis STATIC stb/stb_vorbis.c)
target_include_directories(stb_vorbis PRIVATE stb)
install(DIRECTORY stb
    DESTINATION include
    FILES_MATCHING 
        PATTERN "*.h" PATTERN "*.hpp" PATTERN "*.hxx"
        PATTERN ".git" EXCLUDE
        PATTERN ".git*" EXCLUDE
)
install(TARGETS stb stb_vorbis EXPORT stbTargets)
install(EXPORT stbTargets
    FILE stbConfig.cmake
    DESTINATION lib/cmake/stb
)

install(DIRECTORY bgfx/examples/common/imgui
    DESTINATION include/bgfx/examples
    FILES_MATCHING 
        PATTERN "*.h" PATTERN "*.hpp" PATTERN "*.hxx"
        PATTERN ".git" EXCLUDE
        PATTERN ".git*" EXCLUDE
)

install(DIRECTORY bgfx/3rdparty/iconfontheaders
    DESTINATION include
    FILES_MATCHING 
        PATTERN "*.h" PATTERN "*.hpp" PATTERN "*.hxx"
        PATTERN ".git" EXCLUDE
        PATTERN ".git*" EXCLUDE
)

install(DIRECTORY bgfx/3rdparty/dear-imgui
    DESTINATION include
    FILES_MATCHING 
        PATTERN "*.h" PATTERN "*.hpp" PATTERN "*.hxx"
        PATTERN ".git" EXCLUDE
        PATTERN ".git*" EXCLUDE
)

add_header_lib(linalg linalg)
add_header_lib(miniaudio miniaudio)
add_header_lib(utf8.h utf8.h)
add_header_lib(pdqsort pdqsort)
add_header_lib(spdlog spdlog/include/)
