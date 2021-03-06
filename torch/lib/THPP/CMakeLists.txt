CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
SET(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

################################################################################
# Helper functions
################################################################################

FUNCTION(EXCLUDE_DIR list_name dir_name)
  # A helper that excludes all files that contain dir_name in their file path
  SET(local_list ${${list_name}})
  FOREACH(source ${local_list})
    IF(${source} MATCHES ${dir_name})
      MESSAGE(STATUS "Excluding " ${source} " from the build")
      LIST(REMOVE_ITEM local_list ${source})
    ENDIF()
  ENDFOREACH()
  SET(${list_name} ${local_list} PARENT_SCOPE)
ENDFUNCTION()

################################################################################

if(${CMAKE_VERSION} VERSION_LESS "2.8.12")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
else(${CMAKE_VERSION} VERSION_LESS "2.8.12")
  if(${CMAKE_VERSION} VERSION_LESS "3.1")
    add_compile_options(-std=c++11) # CMake 2.8.12 to 3.1
  endif(${CMAKE_VERSION} VERSION_LESS "3.1")
endif(${CMAKE_VERSION} VERSION_LESS "2.8.12")

IF(NOT Torch_FOUND)
  FIND_PACKAGE(Torch REQUIRED)
ENDIF()

FIND_PACKAGE(CUDA 5.5)
IF(CUDA_FOUND)
  INCLUDE_DIRECTORIES(${CUDA_INCLUDE_DIRS})
  INCLUDE_DIRECTORIES("${CUDA_SDK_ROOT_DIR}/common/inc")
ENDIF()

# Can be compiled standalone
IF(NOT THPP_INSTALL_BIN_DIR OR NOT THPP_INSTALL_LIB_DIR OR NOT THPP_INSTALL_INCLUDE_DIR)
  SET(THPP_INSTALL_BIN_DIR "bin" CACHE PATH "THPP install binary subdirectory")
  SET(THPP_INSTALL_LIB_DIR "lib" CACHE PATH "THPP install library subdirectory")
  SET(THPP_INSTALL_INCLUDE_DIR "include" CACHE PATH "THPP install include subdirectory")
ENDIF()

FILE(GLOB_RECURSE base_h RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "*.h" "*.hpp")
FILE(GLOB_RECURSE base_cpp "*.cpp")

SET(all_cpp ${base_cpp})
SET(all_h ${base_h})

EXCLUDE_DIR(all_cpp ".*/generic/.*\\.cpp$")

IF(NOT CUDA_FOUND)
  EXCLUDE_DIR(all_h ".*THC.*.hpp$")
  EXCLUDE_DIR(all_cpp ".*THC.*.cpp$")
  EXCLUDE_DIR(all_cpp "TraitsCuda.cpp")
ENDIF()


INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})
ADD_LIBRARY(THPP SHARED ${all_cpp})
SET_TARGET_PROPERTIES(THPP PROPERTIES VERSION 1 SOVERSION 1)

if(NOT ${CMAKE_VERSION} VERSION_LESS "3.1")
    SET_PROPERTY(TARGET THPP PROPERTY CXX_STANDARD 11)
endif(NOT ${CMAKE_VERSION} VERSION_LESS "3.1")

TARGET_LINK_LIBRARIES(THPP LINK_PRIVATE ${TH_LIBRARIES})

INSTALL(TARGETS THPP
  RUNTIME DESTINATION "${THPP_INSTALL_BIN_DIR}"
  LIBRARY DESTINATION "${THPP_INSTALL_LIB_DIR}"
  ARCHIVE DESTINATION "${THPP_INSTALL_LIB_DIR}")

FOREACH(HEADER ${all_h})
  STRING(REGEX MATCH "(.*)[/\\]" DIR ${HEADER})
  INSTALL(FILES ${HEADER} DESTINATION ${THPP_INSTALL_INCLUDE_DIR}/THPP/${DIR})
ENDFOREACH()
