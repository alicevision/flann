#include_directories(${CMAKE_SOURCE_DIR}/include algorithms ext util nn .)

add_definitions(-D_FLANN_VERSION=${FLANN_VERSION})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/flann/config.h.in ${CMAKE_CURRENT_SOURCE_DIR}/flann/config.h)

file(GLOB_RECURSE C_SOURCES flann.cpp)
file(GLOB_RECURSE CPP_SOURCES flann_cpp.cpp)
file(GLOB_RECURSE CU_SOURCES *.cu)

#Set flags if we build CUDA versions
if (BUILD_CUDA_LIB)
    SET(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};-DFLANN_USE_CUDA")
    if(CMAKE_COMPILER_IS_GNUCC)
		set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};-Xcompiler;-fPIC;" )
        if (NVCC_COMPILER_BINDIR)
            set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};--compiler-bindir=${NVCC_COMPILER_BINDIR}")
        endif()
    else()
	    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS};" )
    endif()
endif()

if(NOT BUILD_SHARED_LIBS)
  add_library(flann_cpp_s STATIC ${CPP_SOURCES})
  target_link_libraries(flann_cpp_s PUBLIC lz4::lz4)
  set_property(TARGET flann_cpp_s PROPERTY OUTPUT_NAME flann_cpp)
  if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_CLANG)
      set_target_properties(flann_cpp_s PROPERTIES COMPILE_FLAGS -fPIC)
  endif()

  if (BUILD_CUDA_LIB)
      cuda_add_library(flann_cuda_s STATIC ${CU_SOURCES})
      target_link_libraries(flann_cuda_s lz4::lz4)
      set_property(TARGET flann_cuda_s PROPERTY OUTPUT_NAME flann_cuda)
      set_property(TARGET flann_cpp_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC FLANN_USE_CUDA)
  endif()
endif()

if(BUILD_SHARED_LIBS)
  if(CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_COMPILER_IS_GNUCC)
      add_library(flann_cpp SHARED ${CPP_SOURCES})
      set_target_properties(flann_cpp PROPERTIES LINKER_LANGUAGE CXX)
      target_link_libraries(flann_cpp PUBLIC lz4::lz4)
      if (BUILD_CUDA_LIB)
        cuda_add_library(flann_cuda SHARED ${CU_SOURCES})
        set_target_properties(flann_cuda PROPERTIES LINKER_LANGUAGE CXX)
        set_property(TARGET flann_cpp PROPERTY COMPILE_DEFINITIONS FLANN_USE_CUDA)
      endif()
  else()
      add_library(flann_cpp SHARED ${CPP_SOURCES})
      target_link_libraries(flann_cpp PUBLIC lz4::lz4)
      # export lz4 headers, so that MSVC to creates flann_cpp.lib
      set_target_properties(flann_cpp PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS YES)
      if (BUILD_CUDA_LIB)
        cuda_add_library(flann_cuda SHARED ${CU_SOURCES})
        target_link_libraries(flann_cuda lz4::lz4)
        set_property(TARGET flann_cpp PROPERTY COMPILE_DEFINITIONS FLANN_USE_CUDA)
      endif()
  endif()
    
  set_target_properties(flann_cpp PROPERTIES
     VERSION ${FLANN_VERSION}
     SOVERSION ${FLANN_SOVERSION}
     DEFINE_SYMBOL FLANN_EXPORTS
  )

  if (BUILD_CUDA_LIB)
      set_target_properties(flann_cuda PROPERTIES
         VERSION ${FLANN_VERSION}
         SOVERSION ${FLANN_SOVERSION}
         DEFINE_SYMBOL FLANN_EXPORTS
      )
  endif()
endif()

if (USE_MPI AND HDF5_IS_PARALLEL)
  if(BUILD_SHARED_LIBS)
      add_executable(flann_mpi_server flann/mpi/flann_mpi_server.cpp)
      target_link_libraries(flann_mpi_server flann_cpp ${HDF5_LIBRARIES} ${MPI_LIBRARIES} ${Boost_LIBRARIES})

      add_executable(flann_mpi_client flann/mpi/flann_mpi_client.cpp)
      target_link_libraries(flann_mpi_client flann_cpp ${HDF5_LIBRARIES} ${MPI_LIBRARIES} ${Boost_LIBRARIES})

      install (TARGETS flann_mpi_client flann_mpi_server
          DESTINATION bin)
  endif()
endif()

if (BUILD_C_BINDINGS)
  if(NOT BUILD_SHARED_LIBS)
    add_library(flann_s STATIC ${C_SOURCES})
    target_link_libraries(flann_s PUBLIC lz4::lz4)
    set_property(TARGET flann_s PROPERTY OUTPUT_NAME flann)
    if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_CLANG)
        set_target_properties(flann_s PROPERTIES COMPILE_FLAGS -fPIC)
    endif()
    set_property(TARGET flann_s PROPERTY COMPILE_DEFINITIONS FLANN_STATIC)
  endif()
  
  if(BUILD_SHARED_LIBS)
    if(CMAKE_SYSTEM_NAME STREQUAL "Linux" AND CMAKE_COMPILER_IS_GNUCC)
        add_library(flann SHARED ${C_SOURCES})
        target_link_libraries(flann PUBLIC lz4::lz4)
        set_target_properties(flann PROPERTIES LINKER_LANGUAGE CXX)
    else()
        add_library(flann SHARED ${C_SOURCES})
        target_link_libraries(flann PUBLIC lz4::lz4)

        if(MINGW AND OPENMP_FOUND)
          target_link_libraries(flann gomp)
        endif()
    endif()

    set_target_properties(flann PROPERTIES
       VERSION ${FLANN_VERSION}
       SOVERSION ${FLANN_SOVERSION}
       DEFINE_SYMBOL FLANN_EXPORTS
    )
  endif()
endif()

if(WIN32)
  if (BUILD_C_BINDINGS AND BUILD_MATLAB_BINDINGS)
      install (
          TARGETS flann
          RUNTIME DESTINATION share/flann/matlab
      )
  endif()
endif(WIN32)

if(NOT BUILD_SHARED_LIBS)
  list(APPEND FLANN_TARGETS_CPP "flann_cpp_s")
  list(APPEND FLANN_TARGETS_C "flann_s")
  list(APPEND FLANN_TARGETS_CUDA "flann_cuda_s")
endif()
if(BUILD_SHARED_LIBS)
  list(APPEND FLANN_TARGETS_CPP "flann_cpp")
  list(APPEND FLANN_TARGETS_C "flann")
  list(APPEND FLANN_TARGETS_CUDA "flann_cuda")
endif()

install (
    TARGETS ${FLANN_TARGETS_CPP}
    EXPORT ${targets_export_name}
    INCLUDES DESTINATION include
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION ${FLANN_LIB_INSTALL_DIR}
    ARCHIVE DESTINATION ${FLANN_LIB_INSTALL_DIR}
)

if (BUILD_CUDA_LIB)
    install (
        TARGETS ${FLANN_TARGETS_CUDA}
        EXPORT ${targets_export_name}
        INCLUDES DESTINATION include
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION ${FLANN_LIB_INSTALL_DIR}
        ARCHIVE DESTINATION ${FLANN_LIB_INSTALL_DIR}
    )
endif()

if (BUILD_C_BINDINGS)
    install (
        TARGETS ${FLANN_TARGETS_C}
        EXPORT ${targets_export_name}
        INCLUDES DESTINATION include
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION ${FLANN_LIB_INSTALL_DIR}
        ARCHIVE DESTINATION ${FLANN_LIB_INSTALL_DIR}
    )
endif()

install (
    DIRECTORY flann
    DESTINATION include
    FILES_MATCHING PATTERN "*.h" PATTERN "*.hpp"
)
