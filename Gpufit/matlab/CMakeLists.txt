
# MATLAB Gpufit binding

find_package( Matlab COMPONENTS MX_LIBRARY )

if( NOT Matlab_FOUND )
  message( STATUS "Matlab and/or MX_Library NOT found - skipping Gpufit Matlab binding!" )
  return()
endif()

# set C++ ABI version on Linux to work with Matlab (see documentation)

if( CMAKE_CXX_COMPILER_ID STREQUAL GNU )
  if( Matlab VERSION_LESS 9.1 )
    message( STATUS "Matlab ${Matlab} is not supported - skipping Gpufit Matlab binding!" )
    return()
  endif()
  set( abi_version 8 ) # g++ 4.9 (most likely)
  string( APPEND CMAKE_CXX_FLAGS " -fabi-version=${abi_version}" )
endif()

# MATLAB MEX FILE for gpufit

set( Headers
  )

set( Sources
  mex/GpufitMex.cpp
  )

add_library( GpufitMex SHARED
  ${Headers}
  ${Sources}
  )

set_target_properties( GpufitMex
  PROPERTIES
		PREFIX ""
		SUFFIX .${Matlab_MEX_EXTENSION}
		RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}"
)

target_include_directories( GpufitMex PRIVATE ${Matlab_INCLUDE_DIRS} ${PROJECT_SOURCE_DIR} )

target_link_libraries( GpufitMex Gpufit ${Matlab_LIBRARIES} )

if( WIN32 )
  SET(CMAKE_SHARED_LINKER_FLAGS "/export:mexFunction")
endif()

add_matlab_launcher( GpufitMex "${CMAKE_CURRENT_SOURCE_DIR}" )

# MATLAB MEX FILE for cuda_available

set( Headers
  )

set( Sources
  mex/GpufitCudaAvailableMex.cpp
  )

add_library( GpufitCudaAvailableMex SHARED
  ${Headers}
  ${Sources}
  )

set_target_properties( GpufitCudaAvailableMex
  PROPERTIES
		PREFIX ""
		SUFFIX .${Matlab_MEX_EXTENSION}
		RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}"
)

target_include_directories( GpufitCudaAvailableMex PRIVATE ${Matlab_INCLUDE_DIRS} ${PROJECT_SOURCE_DIR} )

target_link_libraries( GpufitCudaAvailableMex Gpufit ${Matlab_LIBRARIES} )

if( WIN32 )
  SET(CMAKE_SHARED_LINKER_FLAGS "/export:mexFunction")
endif()

add_matlab_launcher( GpufitCudaAvailableMex "${CMAKE_CURRENT_SOURCE_DIR}" )

# MATLAB Gpufit PACKAGE

set( build_directory "${CMAKE_BINARY_DIR}/${CMAKE_CFG_INTDIR}/matlab" )
set( package_files
  "${CMAKE_CURRENT_SOURCE_DIR}/EstimatorID.m"
  "${CMAKE_CURRENT_SOURCE_DIR}/gpufit.m"
  "${CMAKE_CURRENT_SOURCE_DIR}/gpufit_cuda_available.m"
  "${CMAKE_CURRENT_SOURCE_DIR}/gpufit_version.m"
  "${CMAKE_CURRENT_SOURCE_DIR}/ModelID.m"
  "${CMAKE_CURRENT_SOURCE_DIR}/README.txt"
)

if(NOT USE_CUBLAS)
	set( binary_gpufit $<TARGET_FILE:Gpufit> )
else()
	set( binary_gpufit $<TARGET_FILE:Gpufit> ${CUBLAS_DLL})
endif()

set( binary_mex $<TARGET_FILE:GpufitMex> )
set( binary_mex2 $<TARGET_FILE:GpufitCudaAvailableMex> )

add_custom_target( MATLAB_GPUFIT_PACKAGE
  COMMAND ${CMAKE_COMMAND} -E
    remove_directory ${build_directory}
  COMMAND ${CMAKE_COMMAND} -E
    make_directory ${build_directory}
  COMMAND ${CMAKE_COMMAND} -E
    copy_if_different ${package_files} ${build_directory}
  COMMAND ${CMAKE_COMMAND} -E
    copy_if_different ${binary_gpufit} ${build_directory}
  COMMAND ${CMAKE_COMMAND} -E
    copy_if_different ${binary_mex} ${build_directory}	
  COMMAND ${CMAKE_COMMAND} -E
    copy_if_different ${binary_mex2} ${build_directory}	
  COMMENT "Creating Gpufit Matlab package"
)
set_property( TARGET MATLAB_GPUFIT_PACKAGE PROPERTY FOLDER CMakePredefinedTargets )
add_dependencies( MATLAB_GPUFIT_PACKAGE Gpufit GpufitMex GpufitCudaAvailableMex)

# add launcher
