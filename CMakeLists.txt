cmake_minimum_required(VERSION 3.24)

if (NOT (UNIX OR WIN32))
  message(FATAL_ERROR "unsupported target platform")
endif()

#Build options
option(BUILD_SHARED_LIBS "Build shared library?" ON)
option(USE_OWN_FFTW "Use own FFTW installation?" OFF)
option(BUILD_BENCHMARK "Build Benchmark?" OFF)
option(BUILD_MATLAB "Build Matlab mex?" OFF)

if(NOT DEFINED CMAKE_BUILD_TYPE OR NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING
      "Build type: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()
message("Build type is ${CMAKE_BUILD_TYPE}")

set(CMAKE_CURRENT_BINARY_DIR ${CMAKE_BINARY_DIR})

#Installation options

  set(FCWT_MATLAB_DIR "../samples/matlab" CACHE
   STRING "Name of the build directory for the MATLAB MEX-files")
  set(FCWT_LIB_INSTALL_DIR "lib" CACHE
    STRING "Name of the installation directory for the libraries relative to ${CMAKE_INSTALL_PREFIX}")
  set(FCWT_INCLUDE_INSTALL_DIR "include" CACHE
    STRING "Name of the installation directory for the includes relative to ${CMAKE_INSTALL_PREFIX}")

if(UNIX)
  set(FCWT_CMAKE_INSTALL_DIR "share/fcwt/cmake" CACHE
    STRING "Name of the installation directory for the CMake stuff relative to ${CMAKE_INSTALL_PREFIX}")

elseif(WIN32)
  set(FCWT_CMAKE_INSTALL_DIR "cmake" CACHE
    STRING "Name of share directory relative to ${CMAKE_INSTALL_PREFIX}")
endif()

# set the project name
project(fcwt VERSION 2.0 DESCRIPTION "Highly optimized implementation of the Continuous Wavelet Transform")

# specify the C++ standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED True)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}")

if(UNIX)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -mavx -O2")
elseif(WIN32)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++20 /arch:AVX /O2 /W1")
endif()

#set cmake linker flags
link_libraries("-lm")

link_directories("${PROJECT_SOURCE_DIR}/deps/libs")

set(libraryheaders 
    "./src/fcwt.h"
)
set(implementationheaders 
    "./samples/cpp/main.h" 
) 
set(benchmarkheaders
    "./src/bench/benchmark.h"
    "./src/fcwt.h"
    "./src/bench/rwave-bench.h" 
    "./src/bench/wavelib-bench.h"
)

set(librarysources 
    "./src/fcwt.cpp"
)
set(implementationsources 
    "./samples/cpp/main.cpp" 
)
set(benchmarksources
    "./src/bench/benchmark.cpp"
    "./src/fcwt.cpp"
    "./src/bench/rwave-bench.cpp"
    "./src/bench/wavelib-bench.cpp"
)

set(headers ${libraryheaders} ${implementationheaders})
set(sources ${librarysources} ${implementationsources})

# targets 
set(fcwt_library_target "fcwt")

#library
if(BUILD_SHARED_LIBS)
    message("Building of shared library is enabled.")
    add_definitions(-DFCWT_LIBRARY_DLL_BUILDING)
else()
    message("Building of static library is enabled.")
endif()

add_library(${fcwt_library_target} ${libraryheaders} ${librarysources})

set_target_properties(${fcwt_library_target}
    PROPERTIES
    LINKER_LANGUAGE "CXX"
    VERSION ${PROJECT_VERSION}
)

# add the executable
if(BUILD_BENCHMARK)
    add_executable(fCWT_benchmark ${benchmarkheaders} ${benchmarksources}) 
endif()

add_executable(fcwt_example ${headers} ${sources})
include_directories("${PROJECT_SOURCE_DIR}/deps/include")
include_directories("${PROJECT_SOURCE_DIR}/src")
include_directories("${PROJECT_SOURCE_DIR}/src/bench")
include_directories("/usr/local/include")
include_directories("${PROJECT_SOURCE_DIR}/src/mex")
link_directories("${PROJECT_SOURCE_DIR}/deps/libs")
link_directories("/usr/local/lib")

if(APPLE)
    set(OpenMP_INCLUDES "/usr/local/opt/libomp/include")
    set(OpenMP_LIBRARIES "/usr/local/opt/libomp/lib")
    set(OpenMP_C_FLAGS "-Xpreprocessor -fopenmp")
    set(OpenMP_CXX_FLAGS "-Xpreprocessor -fopenmp")
    set(OpenMP_C_LIB_NAMES "libomp" "libgomp" "libiomp5")
    set(OpenMP_CXX_LIB_NAMES "libomp" "libgomp" "libiomp5")
    set(OpenMP_libomp_LIBRARY ${OpenMP_C_LIB_NAMES})
    set(OpenMP_libgomp_LIBRARY ${OpenMP_C_LIB_NAMES})
    set(OpenMP_libiomp5_LIBRARY ${OpenMP_C_LIB_NAMES})
    set(OpenMP_omp_LIBRARY ${OpenMP_C_LIB_NAMES})
    set(OpenMP_gomp_LIBRARY ${OpenMP_C_LIB_NAMES})
    set(OpenMP_iomp5_LIBRARY ${OpenMP_C_LIB_NAMES})
endif()

find_package(OpenMP 5.0)

if (OPENMP_FOUND)
    if(APPLE)
        target_include_directories(${fcwt_library_target} PUBLIC ${OpenMP_INCLUDES})
        target_link_directories(${fcwt_library_target} PUBLIC ${OpenMP_LIBRARIES})
        target_link_libraries(${fcwt_library_target} PUBLIC "-lomp")

        target_include_directories(fcwt_example PUBLIC ${OpenMP_INCLUDES})
        target_link_directories(fcwt_example PUBLIC ${OpenMP_LIBRARIES})
        target_link_libraries(fcwt_example PUBLIC "-lomp")

        if(BUILD_BENCHMARK)
            target_include_directories(fcwt_benchmark PUBLIC ${OpenMP_INCLUDES})
            target_link_directories(fcwt_benchmark PUBLIC ${OpenMP_LIBRARIES})
            target_link_libraries(fcwt_benchmark PUBLIC "-lomp")
        endif()
    elseif(WIN32)
        target_link_libraries(${fcwt_library_target} PUBLIC "libomp.lib")
        target_link_libraries(fcwt_example PUBLIC "libomp.lib")
        if(BUILD_BENCHMARK)
            target_link_libraries(fcwt_benchmark PUBLIC "libomp.lib")
        endif()
    endif()
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
else()
    add_compile_definitions (SINGLE_THREAD)
endif()

if(NOT USE_OWN_FFTW)
    #check if we are on a arm64 mac
    if(APPLE)
        message("Building for OSX")
        
        find_library (
            FFTW
            NAMES fftw3fmac # what to look for
            HINTS "${PROJECT_SOURCE_DIR}/deps/libs" # where to look
            NO_DEFAULT_PATH # do not search system default paths
            REQUIRED
        )
        find_library (
            FFTW_OMP
            NAMES fftw3f_ompmac # what to look for
            HINTS "${PROJECT_SOURCE_DIR}/deps/libs" # where to look
            NO_DEFAULT_PATH # do not search system default paths
            REQUIRED
        )

    #check if we are on a linux machine
    elseif(UNIX)
        message("Building for Linux")
        
        find_library (
            FFTW
            NAMES fftw3fl # what to look for
            HINTS "${PROJECT_SOURCE_DIR}/depse/libs" # where to look
            NO_DEFAULT_PATH # do not search system default paths
            REQUIRED
        )
        find_library (
            FFTW_OMP
            NAMES fftw3f_ompl # what to look for
            HINTS "${PROJECT_SOURCE_DIR}/deps/libs" # where to look
            NO_DEFAULT_PATH # do not search system default paths
            REQUIRED
        )

    #check if we are on a windows machine
    elseif(WIN32)
        message("Building for Windows")
        find_library (
            FFTW
            NAMES fftw3f # what to look for
            HINTS "${PROJECT_SOURCE_DIR}/deps/libs" # where to look
            NO_DEFAULT_PATH # do not search system default paths
            REQUIRED
        )
    endif()
else()
    message("Building using own FFTW installation.")
    find_library (
        FFTW
        NAMES fftw3f libfftw3f # what to look for
        REQUIRED
    )
    if(UNIX)
        find_library (
            FFTW_OMP
            NAMES fftw3f_omp libfftw3f_omp # what to look for
            REQUIRED
        )
    endif()
endif()

if(BUILD_BENCHMARK)
    find_library (
        WAVELIB
        NAMES wavelib libwavelib # what to look for
        REQUIRED
    )
endif()

# check if we found the library
message(${PROJECT_SOURCE_DIR})
message("STATUS OF LIBRARY: FFTW: [${FFTW}]")
if(UNIX)
	message("STATUS OF LIBRARY: FFTW_OMP: [${FFTW_OMP}]")
endif()
if(BUILD_BENCHMARK)
    message("STATUS OF LIBRARY: WAVELIB: [${WAVELIB}]")
endif()


if(FFTW)
    target_link_libraries(fcwt_example PRIVATE ${FFTW})
    if(BUILD_BENCHMARK)
        target_link_libraries(fcwt_benchmark PRIVATE ${FFTW})
    endif()
    target_link_libraries(${fcwt_library_target} PRIVATE ${FFTW})
else()
    message(SEND_ERROR "Did not find fftw3f (float version of FFTW). If you want to use your own FFTW installation, please set USE_OWN_FFTW to ON.")
endif()

if(UNIX)
	if(FFTW_OMP)
		target_link_libraries(fcwt_example PRIVATE ${FFTW_OMP})
        if(BUILD_BENCHMARK)
		    target_link_libraries(fcwt_benchmark PRIVATE ${FFTW_OMP})
        endif()
		target_link_libraries(${fcwt_library_target} PRIVATE ${FFTW_OMP})
	else()
        message(SEND_ERROR "Did not find fftw3f_omp (float and multithreaded omp version of FFTW). If you want to use your own FFTW installation, please set USE_OWN_FFTW to ON.")
	endif()
endif()

if(BUILD_BENCHMARK)
    if (NOT WAVELIB)
        message(SEND_ERROR "Did not find wavelib. Please note that you need to install wavelib yourself. See README.md for more information.")
    else()
        target_link_libraries(fcwt_benchmark PRIVATE ${WAVELIB})
    endif()
endif()

if(WIN32)
	file(COPY "${PROJECT_SOURCE_DIR}/deps/libs/fftw3f.dll" DESTINATION ${PROJECT_BINARY_DIR}/Debug)
	file(COPY "${PROJECT_SOURCE_DIR}/deps/libs/fftw3f.lib" DESTINATION ${PROJECT_BINARY_DIR}/Debug)
	file(COPY "${PROJECT_SOURCE_DIR}/deps/libs/fftw3f.dll" DESTINATION ${PROJECT_BINARY_DIR}/Release)
	file(COPY "${PROJECT_SOURCE_DIR}/deps/libs/fftw3f.lib" DESTINATION ${PROJECT_BINARY_DIR}/Release)
    if (OPENMP_FOUND)
        file(COPY "${PROJECT_SOURCE_DIR}/deps/libs/libomp.dll" DESTINATION ${PROJECT_BINARY_DIR}/Debug)
	    file(COPY "${PROJECT_SOURCE_DIR}/deps/libs/libomp.lib" DESTINATION ${PROJECT_BINARY_DIR}/Debug)
        file(COPY "${PROJECT_SOURCE_DIR}/deps/libs/libomp.dll" DESTINATION ${PROJECT_BINARY_DIR}/Release)
	    file(COPY "${PROJECT_SOURCE_DIR}/deps/libs/libomp.lib" DESTINATION ${PROJECT_BINARY_DIR}/Release)
    endif()
    if(BUILD_BENCHMARK)
	    file(COPY ${WAVELIB} DESTINATION ${PROJECT_BINARY_DIR}/Debug)
	    file(COPY ${WAVELIB} DESTINATION ${PROJECT_BINARY_DIR}/Release)
    endif()
endif()


if(BUILD_MATLAB)
    set(MATLAB_FIND_DEBUG)
    
    #Add MATLAB 2021a as possible export
    set(MATLAB_ADDITIONAL_VERSIONS "R2020b=9.9")
    
    find_package(Matlab)
    
    matlab_add_mex(
        NAME fCWTmex
        SRC src/mex/fcwtmex.cpp src/fcwt.h src/fcwt.cpp
        OUTPUT_NAME ${FCWT_MATLAB_DIR}/fCWT
        LINK_TO ${FFTW} ${FFTW_OMP}
        R2018a
    )

    #set compile options
    set_target_properties(fCWTmex PROPERTIES COMPILE_FLAGS "-g -DSINGLE_THREAD")

    matlab_add_mex(
        NAME fCWTmexplan
        SRC src/mex/fcwtplan.cpp src/fcwt.h src/fcwt.cpp
        OUTPUT_NAME ${FCWT_MATLAB_DIR}/fCWT_create_plan
        LINK_TO ${FFTW} ${FFTW_OMP}
        R2018a
    )

    set_target_properties(fCWTmexplan PROPERTIES COMPILE_FLAGS "-g -DSINGLE_THREAD")
endif()

# Installing

install(FILES ${libraryheaders}
    DESTINATION ${FCWT_INCLUDE_INSTALL_DIR})

install(TARGETS ${fcwt_library_target}
    EXPORT ${fcwt_library_target}_export
    ARCHIVE  DESTINATION ${FCWT_LIB_INSTALL_DIR}
    LIBRARY  DESTINATION ${FCWT_LIB_INSTALL_DIR}
    RUNTIME  DESTINATION ${FCWT_LIB_INSTALL_DIR}
    INCLUDES DESTINATION ${FCWT_INCLUDE_INSTALL_DIR})

install(EXPORT ${fcwt_library_target}_export
  DESTINATION ${FCWT_CMAKE_INSTALL_DIR}
  FILE ${fcwt_library_target}-config.cmake)