cmake_minimum_required(VERSION 3.21)
project(Deps)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake/chainblocks-cmake-shared)
include(Platform)
include(ExternalProject)

set(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/install/${TARGET_ID})

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

# Expose artifact name to github actions
if(DEFINED ENV{GITHUB_WORKFLOW})
    set(SET_CMD "ARTIFACT_NAME=${TARGET_ID}")
    file(APPEND $ENV{GITHUB_ENV} "${SET_CMD}")
    message("Setting ${SET_CMD} for github actions")
endif()

set(COMMON_CMAKE_ARGS 
    -DBUILD_SHARED_LIBS=OFF
    -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} 
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
)

if(CMAKE_TOOLCHAIN_FILE)
    set(COMMON_CMAKE_ARGS ${COMMON_CMAKE_ARGS} -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE})
endif()

if(IOS)
    set(COMMON_CMAKE_ARGS ${COMMON_CMAKE_ARGS} -DPLATFORM=${PLATFORM})
endif()

set(COMMON_ARGS 
    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
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

if(NOT EMSCRIPTEN)
  ExternalProject_Add(SDL SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/SDL
      ${COMMON_ARGS}
      CMAKE_ARGS 
          -DSDL_RENDER_DISABLED=ON
  )
endif()

if(EMSCRIPTEN OR IOS)
  set(BGFX_BUILD_TOOLS OFF)
else()
  set(BGFX_BUILD_TOOLS ON)
endif()

ExternalProject_Add(bgfx
    GIT_REPOSITORY https://github.com/bkaradzic/bgfx.cmake.git
    GIT_TAG b7b7cd7bdda84433490dca8a12cd931f79debeff
    GIT_SUBMODULES ""
    PATCH_COMMAND git restore . && git apply --whitespace=fix ${CMAKE_CURRENT_SOURCE_DIR}/Patches/bgfx.cmake.patch
    ${COMMON_ARGS}
    CMAKE_ARGS 
        -DBX_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bx
        -DBIMG_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bimg
        -DBGFX_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bgfx
        -DBIMG_DECODE_ENABLE=OFF
        -DBGFX_BUILD_EXAMPLES=OFF
        -DBGFX_BUILD_TOOLS=${BGFX_BUILD_TOOLS}
        -DENABLE_ARC=OFF
)

if(WIN32 OR IOS)
    set(BOOST_CMAKE_ARGS -DCMAKE_ASM_COMPILER=as)
endif()

if(IOS)
    set(BOOST_CMAKE_ARGS ${BOOST_CMAKE_ARGS} -DBOOST_CONTEXT_ARCHITECTURE=${ARCHS})
    if(NOT X86_IOS_SIMULATOR)
        set(BOOST_CMAKE_ARGS ${BOOST_CMAKE_ARGS} -DBOOST_CONTEXT_ABI=aapcs)
    endif()
endif()

set(BOOST_MODULES atomic container lockfree align algorithm)
if(NOT EMSCRIPTEN)
  set(BOOST_MODULES ${BOOST_MODULES} context filesystem process stacktrace asio beast multiprecision)
endif()
list(JOIN BOOST_MODULES "|" BOOST_MODULES_JOINED)

ExternalProject_Add(boost
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/boost
    PATCH_COMMAND cd libs/context && git restore . && git apply --verbose --reject --whitespace=fix ${CMAKE_CURRENT_SOURCE_DIR}/Patches/boost-context.patch
    LIST_SEPARATOR |
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBOOST_INCLUDE_LIBRARIES=${BOOST_MODULES_JOINED}
        -DBOOST_USE_VALGRIND=${VALGRIND}
        ${BOOST_CMAKE_ARGS}
)

ExternalProject_Add(snappy SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/snappy
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DSNAPPY_BUILD_TESTS=OFF
)

ExternalProject_Add(brotli SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/brotli
    PATCH_COMMAND git restore . && git apply --whitespace=fix ${CMAKE_CURRENT_SOURCE_DIR}/Patches/brotli.patch
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBROTLI_DISABLE_TESTS=ON
        -DBROTLI_BUILD_TOOLS=OFF
        -DENABLE_STRICT_TRY_COMPILE=ON
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

ExternalProject_Add(kissfft SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kissfft
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DCMAKE_INSTALL_INCLUDEDIR=include
        -DCMAKE_INSTALL_LIBDIR=lib
        -DKISSFFT_STATIC=ON
        -DKISSFFT_TEST=OFF
        -DKISSFFT_TOOLS=OFF
)

ExternalProject_Add(json SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/json
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DJSON_BuildTests=OFF
)

ExternalProject_Add(kcp SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kcp
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBUILD_TESTING=OFF
)

ExternalProject_Add(replxx SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/replxx 
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DREPLXX_BUILD_EXAMPLES=OFF
        -DREPLXX_BUILD_PACKAGE=OFF
)

ExternalProject_Add(magic_enum SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/magic_enum 
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DMAGIC_ENUM_OPT_BUILD_EXAMPLES=OFF
        -DMAGIC_ENUM_OPT_BUILD_TESTS=OFF
)

ExternalProject_Add(nameof SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/nameof 
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DNAMEOF_OPT_BUILD_EXAMPLES=OFF
        -DNAMEOF_OPT_BUILD_TESTS=OFF
)

ExternalProject_Add(pareto SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/pareto
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBUILD_PYTHON_BINDING=OFF
        -DBUILD_TESTS=OFF
        -DBUILD_EXAMPLES=OFF
        -DBUILD_MATPLOT_TARGETS=OFF
        -DBUILD_BENCHMARKS=OFF
)

ExternalProject_Add(wasm3 SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/wasm3
    PATCH_COMMAND git restore . && git apply --whitespace=fix ${CMAKE_CURRENT_SOURCE_DIR}/Patches/wasm3.patch
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBUILD_NATIVE=OFF
        -DBUILD_WASM3=OFF
)

ExternalProject_Add(xxHash 
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/xxHash/cmake_unofficial
    ${COMMON_ARGS}
    CMAKE_ARGS
        -DBUILD_SHARED_LIBS=OFF
        -DXXHASH_BUILD_XXHSUM=OFF
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
