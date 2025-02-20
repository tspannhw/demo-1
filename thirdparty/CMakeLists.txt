# Copyright (C) 2019-2020 Zilliz. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance
# with the License. You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software distributed under the License
# is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express
# or implied. See the License for the specific language governing permissions and limitations under the License

add_compile_options( -O3 -fPIC -Wno-error -fopenmp )

if ( NOT KNOWHERE_VERBOSE_THIRDPARTY_BUILD )
    set( EP_LOG_OPTIONS LOG_CONFIGURE 1 LOG_BUILD 1 LOG_INSTALL 1 LOG_DOWNLOAD 1 )
else ()
    set( EP_LOG_OPTIONS )
endif ()

set( MAKE_BUILD_ARGS "-j6" )

include( FetchContent )
set( FETCHCONTENT_BASE_DIR  ${KNOWHERE_BINARY_DIR}/3rdparty_download )
set( FETCHCONTENT_QUIET     OFF )

if( CUSTOM_THIRDPARTY_DOWNLOAD_PATH )
    set( THIRDPARTY_DOWNLOAD_PATH ${CUSTOM_THIRDPARTY_DOWNLOAD_PATH} )
else()
    set( THIRDPARTY_DOWNLOAD_PATH ${CMAKE_BINARY_DIR}/3rdparty_download/download )
endif()
message( STATUS "Thirdparty downloaded file path: ${THIRDPARTY_DOWNLOAD_PATH}" )
# ----------------------------------------------------------------------
# assume built-in pthreads on MacOS
if ( APPLE )
    set( CMAKE_THREAD_LIBS_INIT "-lpthread" )
    set( CMAKE_HAVE_THREADS_LIBRARY 1 )
    set( CMAKE_USE_WIN32_THREADS_INIT 0 )
    set( CMAKE_USE_PTHREADS_INIT 1 )
    set( THREADS_PREFER_PTHREAD_FLAG ON )
endif()

# Find pthreads
set( THREADS_PREFER_PTHREAD_FLAG ON )
find_package( Threads REQUIRED )


# ****************************** Thirdparty googletest ***************************************
if ( KNOWHERE_BUILD_TESTS )
    if( NOT TARGET gtest::gtest_main )
        add_subdirectory( gtest )
    endif()

    if( NOT TARGET benchmark::benchmark_main )
        add_subdirectory( google_benchmark )
    endif()
endif()

# ****************************** Thirdparty NGT ***************************************
if ( LINUX )
    add_subdirectory( NGT )
endif()

# ****************************** Thirdparty Faiss ***************************************
set( FAISS_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/faiss )

macro(build_faiss)
    message( STATUS "Building FAISS-${FAISS_VERSION} from source" )
    set( FAISS_PREFIX "${KNOWHERE_BINARY_DIR}/thirdparty/faiss_ep-prefix" )
    set( FAISS_INCLUDE_DIR "${FAISS_PREFIX}/include" )
    set( FAISS_STATIC_LIB
            "${FAISS_PREFIX}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}faiss${CMAKE_STATIC_LIBRARY_SUFFIX}" )

    if ( CCACHE_FOUND )
        set( FAISS_C_COMPILER "${CCACHE_FOUND} ${CMAKE_C_COMPILER}" )
        if ( KNOWHERE_GPU_VERSION )
            set( FAISS_CXX_COMPILER "${CMAKE_CXX_COMPILER}" )
            set( FAISS_CUDA_COMPILER "${CCACHE_FOUND} ${CMAKE_CUDA_COMPILER}" )
        else ()
            set( FAISS_CXX_COMPILER "${CCACHE_FOUND} ${CMAKE_CXX_COMPILER}" )
        endif()
    else ()
        set( FAISS_C_COMPILER "${CMAKE_C_COMPILER}" )
        set( FAISS_CXX_COMPILER "${CMAKE_CXX_COMPILER}" )
    endif ()

    set( FAISS_CONFIGURE_ARGS
            "--prefix=${FAISS_PREFIX}"
            "CC=${FAISS_C_COMPILER}"
            "CXX=${FAISS_CXX_COMPILER}"
            "NVCC=${FAISS_CUDA_COMPILER}"
            "CFLAGS=${EP_C_FLAGS}"
            "CXXFLAGS=${EP_CXX_FLAGS} -msse4.2 -O3 -fpermissive -I ${KNOWHERE_SOURCE_DIR}/knowhere"
            --without-python )

    message( STATUS "${FAISS_CONFIGURE_ARGS}" )

    message( STATUS "Build Faiss with OpenBlas/LAPACK" )
    set( FAISS_CONFIGURE_ARGS ${FAISS_CONFIGURE_ARGS}
         "LDFLAGS=-L${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}"
         )

    if ( KNOWHERE_GPU_VERSION )
        if ( NOT MILVUS_CUDA_ARCH OR MILVUS_CUDA_ARCH STREQUAL "DEFAULT" )
            set( FAISS_CONFIGURE_ARGS ${FAISS_CONFIGURE_ARGS}
                 "--with-cuda=${CUDA_TOOLKIT_ROOT_DIR}"
                 "--with-cuda-arch=-gencode=arch=compute_60,code=sm_60 -gencode=arch=compute_61,code=sm_61 -gencode=arch=compute_70,code=sm_70 -gencode=arch=compute_75,code=sm_75"
                 )
        else ()
            STRING( REPLACE ";" " " MILVUS_CUDA_ARCH "${MILVUS_CUDA_ARCH}" )
            set( FAISS_CONFIGURE_ARGS ${FAISS_CONFIGURE_ARGS}
                 "--with-cuda=${CUDA_TOOLKIT_ROOT_DIR}"
                 "--with-cuda-arch=${MILVUS_CUDA_ARCH}"
                 )
        endif ()
    else ()
        set( FAISS_CONFIGURE_ARGS ${FAISS_CONFIGURE_ARGS}
                 "CPPFLAGS=-DUSE_CPU"
                 --without-cuda )
    endif ()

    message( STATUS "Building FAISS with configure args -${FAISS_CONFIGURE_ARGS}" )

    #Notice: there will not set LDFLAGS value
    if ( DEFINED ENV{FAISS_SOURCE_URL} )
        set( FAISS_SOURCE_URL "$ENV{FAISS_SOURCE_URL}" )
        externalproject_add( faiss_ep
                URL
                ${FAISS_SOURCE_URL}
                ${EP_LOG_OPTIONS}
                CONFIGURE_COMMAND
                "./configure"
                ${FAISS_CONFIGURE_ARGS}
                BUILD_COMMAND
                ${MAKE} ${MAKE_BUILD_ARGS} all
                BUILD_IN_SOURCE
                1
                INSTALL_COMMAND
                ${MAKE} install
                BUILD_BYPRODUCTS
                ${FAISS_STATIC_LIB} )
    else ()
        externalproject_add( faiss_ep
                DOWNLOAD_COMMAND
                ""
                SOURCE_DIR
                ${FAISS_SOURCE_DIR}
                ${EP_LOG_OPTIONS}
                CONFIGURE_COMMAND
                "./configure"
                ${FAISS_CONFIGURE_ARGS}
                BUILD_COMMAND
                ${MAKE} ${MAKE_BUILD_ARGS} all
                BUILD_IN_SOURCE
                1
                INSTALL_COMMAND
                ${MAKE} install
                BUILD_BYPRODUCTS
                ${FAISS_STATIC_LIB} )
    endif ()

    ExternalProject_Add_StepDependencies( faiss_ep configure knowhere_utils )

    if ( NOT BLAS_FOUND )
        message( STATUS "OpenBLAS BOUNDED" )
        ExternalProject_Add_StepDependencies( faiss_ep configure openblas_ep knowhere_utils )
    endif()

    file( MAKE_DIRECTORY "${FAISS_INCLUDE_DIR}" )
    add_library( faiss STATIC IMPORTED )

    set_target_properties( faiss
            PROPERTIES
                IMPORTED_GLOBAL                 TRUE
                IMPORTED_LOCATION               "${FAISS_STATIC_LIB}"
                INTERFACE_INCLUDE_DIRECTORIES   "${FAISS_INCLUDE_DIR}")

    target_link_libraries( faiss INTERFACE knowhere_utils openblas )
    add_dependencies( faiss faiss_ep )

endmacro()

if ( LINUX )
    if ( KNOWHERE_WITH_FAISS AND NOT TARGET faiss_ep )
        if ( FAISS_SOURCE STREQUAL "AUTO" )
            # find_package(FAISS MODULE)
            if ( NOT FAISS_FOUND )
                build_faiss()
            endif ()
        elseif ( FAISS_SOURCE STREQUAL "BUNDLED")
            build_faiss()
        elseif ( FAISS_SOURCE STREQUAL "SYSTEM" )
            find_package( FAISS REQUIRED )
        endif ()

        get_target_property( FAISS_INCLUDE_DIR faiss INTERFACE_INCLUDE_DIRECTORIES )
        include_directories( SYSTEM "${FAISS_INCLUDE_DIR}" )
        link_directories( SYSTEM ${FAISS_PREFIX}/lib/ )
    endif ()

    install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/annoy/"
            DESTINATION "include/annoy"
            FILES_MATCHING
            PATTERN "*.h"
            )

    install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/faiss/"
            DESTINATION "include/faiss"
            FILES_MATCHING
            PATTERN "*.h"
            )

    install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/hnswlib/"
            DESTINATION "include/hnswlib"
            FILES_MATCHING
            PATTERN "*.h"
            )

    install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/NGT/lib/NGT/"
            DESTINATION "include/NGT"
            FILES_MATCHING
            PATTERN "*.h"
            )
endif ()

install(DIRECTORY "${KNOWHERE_SOURCE_DIR}/thirdparty/easyloggingpp/"
        DESTINATION "include/easyloggingpp"
        FILES_MATCHING 
        PATTERN "*.h" 
)

