cmake_minimum_required(VERSION 3.12)
# We need to specify all languages that will be used in subprojects as well
# This is to avoid retriggering function that sets CMAKE_SYSTEM specific configuration
# It appears to happen each time a new compiler needs to be found.
# We need to avoid that since we need to patch some of the settings
project(FTorch_CESM_WRAPPER LANGUAGES Fortran C CXX)
enable_language(Fortran)
# Force use of Fortran compiler for linking
set(CMAKE_Fortran_LINK_EXECUTABLE
    "${CMAKE_Fortran_COMPILER} <FLAGS> <CMAKE_Fortran_LINK_FLAGS> <OBJECTS> -o <TARGET> <LINK_LIBRARIES>"
)

if("${USE_FTORCH}" STREQUAL "ON")
  # Workaround:
  # When building on Derecho, CESM sets a "Catamount" as CMAKE_SYSTEM_NAME
  # since it was an OS without dynamic linking capability, it makes CMAKE raise
  # error if a build script request a SHARED library to be build.
  #
  # Some of the Torch dependencies (utf8_range pulled  by protobuf, pulled by Caffe2)
  # always request a dynamic library, this makes CMake fail on configuration.
  #
  # However, we are not running an Catamount so we have access to dynamic linking.
  # We need to override the property below to tell CMake it is allowed to create
  # shared libraries
  set_property(GLOBAL PROPERTY TARGET_SUPPORTS_SHARED_LIBS TRUE)

  message("Building FTorch library and wrapper ${USE_FTORCH} ${CMAKE_BINARY_DIR}")
  add_subdirectory(src)
  add_library(ftorch_wrapper SHARED  FTorch_cesm_interface.F90)
  target_link_libraries(ftorch_wrapper PRIVATE ftorch)
  set_target_properties(ftorch PROPERTIES
    Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}/external/ftorch
  )
  # The cpp file in this library will cause cmake to prefer the cxx compiler for the link step, we override and use fc
  set_target_properties(ftorch PROPERTIES
    LINKER_LANGUAGE Fortran  
  )
  # Expose that module directory to anything that links against ftorch
  target_include_directories(ftorch
    PUBLIC
    $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/external/ftorch>
  )
else()
  message("Building FTorch wrapper only ${USE_FTORCH}")
  add_library(ftorch_wrapper SHARED  FTorch_cesm_interface.F90)
endif()


