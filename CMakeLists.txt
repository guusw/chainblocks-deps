cmake_minimum_required(VERSION 3.21)
project(Deps)

option(BUILD_ID "The identifier for the build" "x86_64-pc-windows-gnu")
set(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/install/${BUILD_ID})

set(CONFIGURE_TARGET x86_64-w64-windows-gnu)
set(BUILD_TYPE Release)

include(ExternalProject)

function(add_project NAME)
    math(EXPR REMAINING_ARGC "${ARGC}-1")
    if(REMAINING_ARGC) 
        list(SUBLIST ARGV 1 ${REMAINING_ARGC} REMAINING_ARGS)
        message(STATUS "REMAINING_ARGS: " ${REMAINING_ARGS})
    else()
        set(REMAINING_ARGS "")
    endif()
    message(STATUS "add_project(${NAME} ${REMAINING_ARGS})")

    ExternalProject_Add(${NAME} 
        CMAKE_ARGS 
            -DBUILD_SHARED_LIBS=OFF
            -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} 
            -DCMAKE_BUILD_TYPE=${BUILD_TYPE}
        ${REMAINING_ARGS}
    )
endfunction()

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

add_project(json SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/json)
add_project(kcp SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kcp)
add_project(replxx SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/replxx)
add_project(magic_enum SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/magic_enum)
add_project(nameof SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/nameof)
add_project(pareto SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/pareto
    CMAKE_ARGS
        -DBUILD_PYTHON_BINDING=OFF
        -DBUILD_TESTS=OFF
        -DBUILD_EXAMPLES=OFF
        -DBUILD_MATPLOT_TARGETS=OFF
        -DBUILD_BENCHMARKS=OFF
)

add_project(SDL SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/SDL
    CMAKE_ARGS 
        -DSDL_RENDER_DISABLED=ON
)

add_project(bgfx SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/bgfx.cmake
    CMAKE_ARGS 
        -DBX_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bx
        -DBIMG_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bimg
        -DBGFX_DIR=${CMAKE_CURRENT_SOURCE_DIR}/bgfx
)

add_project(wasm3 SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/wasm3
    CMAKE_ARGS
        -DBUILD_NATIVE=OFF
)

add_project(snappy SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/snappy
    CMAKE_ARGS
        -DSNAPPY_BUILD_TESTS=OFF
)

add_project(brotli SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/brotli
    CMAKE_ARGS
        -DBROTLI_DISABLE_TESTS=ON
)

add_project(Catch2 SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/Catch2
    CMAKE_ARGS
        -DCATCH_INSTALL_DOCS=OFF
)

add_project(tinygltf SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/tinygltf
    CMAKE_ARGS
        -DTINYGLTF_BUILD_LOADER_EXAMPLE=OFF
)

add_project(cpp-taskflow SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/cpp-taskflow
    CMAKE_ARGS
        -DTF_BUILD_TESTS=OFF
        -DTF_BUILD_EXAMPLES=OFF
)

add_project(boost SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/boost
    LIST_SEPARATOR |
    CMAKE_ARGS
        -DCMAKE_ASM_COMPILER=as
        -DBOOST_INCLUDE_LIBRARIES=context|filesystem|stacktrace|atomic
)

add_project(kissfft SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/kissfft
    CMAKE_ARGS
        -DCMAKE_INSTALL_INCLUDEDIR=${CMAKE_INSTALL_PREFIX}/include
        -DCMAKE_INSTALL_LIBDIR=${CMAKE_INSTALL_PREFIX}/lib
        -DKISSFFT_STATIC=ON
        -DKISSFFT_TEST=OFF
        -DKISSFFT_TOOLS=OFF
)

add_project(xxHash 
    SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/xxHash/cmake_unofficial
    CMAKE_ARGS
        -DBUILD_SHARED_LIBS=OFF
)

ExternalProject_Add(
  backtrace
  URL ${CMAKE_CURRENT_SOURCE_DIR}/libbacktrace
  CONFIGURE_COMMAND sh configure --prefix=${INSTALL_ROOT} --target=${CONFIGURE_TARGET}
  BUILD_COMMAND make
  INSTALL_COMMAND make install
  BUILD_IN_SOURCE 1
)

add_header_lib(linalg linalg)
add_header_lib(miniaudio miniaudio)
add_header_lib(stb stb)
add_header_lib(utf8.h utf8.h)
add_header_lib(pdqsort pdqsort)
add_header_lib(spdlog spdlog/include/)

# add_project(ImGuizmo URL ${CMAKE_CURRENT_SOURCE_DIR}/ImGuizmo)
# add_project(imgui_club URL ${CMAKE_CURRENT_SOURCE_DIR}/imgui_club)
# add_project(implot URL ${CMAKE_CURRENT_SOURCE_DIR}/implot)
