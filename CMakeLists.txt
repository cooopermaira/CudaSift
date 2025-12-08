#[[
cmake_minimum_required(VERSION 3.5)

project(cudaSift)
set(cudaSift_VERSION_MAJOR 2)
set(cudaSift_VERSION_MINOR 0)
set(cudaSift_VERSION_PATCH 0)

set(CPACK_PACKAGE_VERSION_MAJOR "${cudaSift_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${cudaSift_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${cudaSift_VERSION_PATCH}")
set(CPACK_GENERATOR "ZIP")
include(CPack)

find_package(OpenCV REQUIRED)
find_package(CUDA)
if (NOT CUDA_FOUND)
  message(STATUS "CUDA not found. Project will not be built.")
endif(NOT CUDA_FOUND)

if (WIN32)
  set(EXTRA_CXX_FLAGS "/DVERBOSE /D_CRT_SECURE_NO_WARNINGS ")
  list(APPEND CUDA_NVCC_FLAGS "-arch=sm_35;--compiler-options;-O2;-DVERBOSE") 
endif()
if (UNIX)
  if (APPLE)
    set(EXTRA_CXX_FLAGS "-DVERBOSE -msse2")
    list(APPEND CUDA_NVCC_FLAGS "-arch=sm_35;--compiler-options;-O2;-DVERBOSE") 
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -msse2 ")
    list(APPEND CUDA_NVCC_FLAGS "-lineinfo;-ccbin;/usr/bin/gcc;--compiler-options;-O2;-D_FORCE_INLINES;-DVERBOSE_NOT")
  endif()
endif()

set(cuda_sources
  cudaImage.cu  
  cudaImage.h  
  cudaSiftH.cu 
  cudaSiftH.h  
  matching.cu  
  cudaSiftD.h  
  cudaSift.h  
  cudautils.h
)  

set(sources
  geomFuncs.cpp  
  mainSift.cpp  
)

include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR} 
)

#SET(CUDA_SEPARABLE_COMPILATION ON)
cuda_add_library(cudasift STATIC ${cuda_sources} ${sources} OPTIONS -arch=compute_86)
#cuda_add_executable(cudasift ${cuda_sources} ${sources} OPTIONS -arch=sm_35)

#cuda_add_executable(l2net l2netD.cu OPTIONS -arch=sm_35)

set_target_properties(cudasift PROPERTIES
  COMPILE_FLAGS "${EXTRA_CXX_FLAGS}"			   
)

target_link_libraries(cudasift ${CUDA_cudadevrt_LIBRARY} ${OpenCV_LIBS})
#  /usr/local/cuda/lib64/libcudadevrt.a ${OpenCV_LIBS} 
#)
 
install(FILES 
  ${cuda_sources} 
  ${sources}
  cudaSiftD.cu
  CMakeLists.txt
  Copyright.txt
  DESTINATION .
)
install(FILES data/left.pgm data/righ.pgm
  DESTINATION data
)
]]
cmake_minimum_required(VERSION 3.10...3.30)

project(cudaSift LANGUAGES CXX CUDA)

# ---- version / packaging (unchanged)
set(cudaSift_VERSION_MAJOR 2)
set(cudaSift_VERSION_MINOR 0)
set(cudaSift_VERSION_PATCH 0)
set(CPACK_PACKAGE_VERSION_MAJOR "${cudaSift_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${cudaSift_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${cudaSift_VERSION_PATCH}")
set(CPACK_GENERATOR "ZIP")
include(CPack)

# ---- deps
find_package(OpenCV REQUIRED)

# Prefer modern CUDA package if available; we’ll fallback to legacy FindCUDA below
if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.18)
  find_package(CUDAToolkit REQUIRED)
endif()

# ---- common sources
set(cuda_sources
        cudaImage.cu
        cudaImage.h
        cudaSiftH.cu
        cudaSiftH.h
        matching.cu
        cudaSiftD.h
        cudaSift.h
        cudautils.h
)

set(sources
        geomFuncs.cpp
        mainSift.cpp
)

# ---- include dirs
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

# ---- portable compiler flags (no -msse2 on ARM)
set(EXTRA_CXX_FLAGS "-DVERBOSE")
if (CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64|i.86")
  # x86/x64 only
  set(EXTRA_CXX_FLAGS "${EXTRA_CXX_FLAGS} -msse2")
endif()

# ======================================================================
# Path A: Modern CMake CUDA (recommended)
# ======================================================================
if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.18)

  # Global CUDA settings (affect this directory)
  set(CMAKE_CUDA_STANDARD 14)
  set(CMAKE_CUDA_STANDARD_REQUIRED ON)

  # Default to Blackwell (GB10) if not provided by parent
  if (NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
    # GB10 Blackwell → compute 12.1 (sm_121)
    set(CMAKE_CUDA_ARCHITECTURES 121)
  endif()

  add_library(cudasift STATIC ${cuda_sources} ${sources})

  target_compile_options(cudasift PRIVATE
          $<$<COMPILE_LANGUAGE:CXX>:-O2 ${EXTRA_CXX_FLAGS}>
          $<$<COMPILE_LANGUAGE:CUDA>:-lineinfo --expt-relaxed-constexpr --expt-extended-lambda>
  )

  target_include_directories(cudasift PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

  target_link_libraries(cudasift
          PRIVATE
          CUDA::cudart
          ${OpenCV_LIBS}
  )

  # ======================================================================
  # Path B: Legacy FindCUDA fallback
  # ======================================================================
else()

  # Older projects used FindCUDA; keep a compatible path
  find_package(CUDA REQUIRED)

  # Clamp legacy arch flags to Blackwell unless parent provided something
  set(CUDA_ARCH_NAME "Manual")
  if (NOT DEFINED CUDA_ARCH_BIN)
    set(CUDA_ARCH_BIN "12.1")   # compute_121 / sm_121
  endif()

  # NVCC flags as a proper list (no hardcoded ccbin)
  list(APPEND CUDA_NVCC_FLAGS
          -lineinfo
          --expt-relaxed-constexpr
          --expt-extended-lambda
          --compiler-options -O2
          -gencode arch=compute_${CUDA_ARCH_BIN},code=sm_${CUDA_ARCH_BIN}
  )

  # Build the library via FindCUDA helper
  cuda_add_library(cudasift STATIC ${cuda_sources} ${sources})

  # Apply host C++ flags (no -msse2 on ARM)
  set_target_properties(cudasift PROPERTIES
          COMPILE_FLAGS "${EXTRA_CXX_FLAGS}"
  )

  target_include_directories(cudasift PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
  target_link_libraries(cudasift ${CUDA_cudadevrt_LIBRARY} ${OpenCV_LIBS})

endif()

# ---- install (unchanged)
install(FILES
        ${cuda_sources}
        ${sources}
        cudaSiftD.cu
        CMakeLists.txt
        Copyright.txt
        DESTINATION .
)

install(FILES data/left.pgm data/righ.pgm
        DESTINATION data
)