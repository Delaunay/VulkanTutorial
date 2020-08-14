CMAKE_MINIMUM_REQUIRED(VERSION 3.13)

PROJECT(VulkanTutorial C CXX)

SET(CMAKE_RELEASE_POSTFIX "")
SET(CMAKE_DEBUG_POSTFIX "-debug")

SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# Compiler-flag
IF(${CMAKE_BUILD_TYPE} MATCHES "Debug")
    MESSAGE(STATUS "Building Debug Version")
    ADD_DEFINITIONS(-O0)
    ADD_LINK_OPTIONS(-rdynamic)         # For stack trace with symbols
ELSE()
    MESSAGE(STATUS "Building Release Version")
    ADD_DEFINITIONS(-O2)
    ADD_DEFINITIONS(-DNDEBUG)
ENDIF()

SET(CMAKE_CXX_STANDARD 20)
SET(CXX_STANDARD_REQUIRED ON)

FILE(GLOB SHADERS_SRC code/*.frag code/*.vert)
FILE(GLOB DEMO_SRC    code/*.cpp)

# apt-get install libglfw3-dev
FIND_PACKAGE(glfw3 REQUIRED)

# apt-get install libglm-dev
FIND_PACKAGE(glm REQUIRED)

FIND_PACKAGE(Vulkan REQUIRED)

INCLUDE_DIRECTORIES(dependencies/stb)
INCLUDE_DIRECTORIES(dependencies/tinyobjloader)

# Find Shaders and compile them
# move it out into a cmake module
# ===============================
FILE(GLOB_RECURSE SHADERS_SRC *.frag *.vert)
SET(SHADER_COMPILER /opt/vulkan/1.2.148.0/x86_64/bin/glslc)


ADD_CUSTOM_TARGET(shaders_all)

FUNCTION(add_shader SHADER)
  # Get filename only
  get_filename_component(SHADER_FILE_NAME ${SHADER} NAME)

  # Get relative path to shader
  FILE(RELATIVE_PATH SHADER_FILE_PATH ${CMAKE_CURRENT_SOURCE_DIR} ${SHADER})

  # Get relative directories
  get_filename_component(SHADER_PATH ${SHADER_FILE_PATH} DIRECTORY)

  # make the shader folder structure mirror the build folder
  # so the path is consistent between source and build
  # later when we will zip all shaders together for fast loading
  FILE(MAKE_DIRECTORY "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${SHADER_PATH}")
  SET(SHADER_OUT "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${SHADER_PATH}/${SHADER_FILE_NAME}.spv")

  ADD_CUSTOM_TARGET(
    shader_${SHADER_FILE_NAME}
    COMMAND ${SHADER_COMPILER} ${SHADER} -g -o ${SHADER_OUT}
    DEPENDS ${SHADER}
    BYPRODUCTS ${SHADER_OUT}
    COMMENT "Building ${SHADER}.spv"
  )
  ADD_DEPENDENCIES(shaders_all shader_${SHADER_FILE_NAME})
ENDFUNCTION()



ADD_CUSTOM_TARGET(shaders ALL SOURCES ${SHADERS_SRC})
FOREACH(SHADER ${SHADERS_SRC})
  add_shader(${SHADER})
ENDFOREACH()

ADD_CUSTOM_TARGET(asset_dir_all)

FUNCTION(asset_dir dirname)
  ADD_CUSTOM_TARGET(link_${dirname} ALL COMMAND
    ${CMAKE_COMMAND} -E create_symlink
    "${CMAKE_SOURCE_DIR}/${dirname}"
    "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/${dirname}")

  ADD_DEPENDENCIES(asset_dir_all link_${dirname})
ENDFUNCTION()

asset_dir("resources")

FUNCTION(add_demo_exec source)
    # Get filename only
    get_filename_component(FILE_NAME ${source} NAME)
    ADD_EXECUTABLE(${FILE_NAME} ${source})
    TARGET_LINK_LIBRARIES(${FILE_NAME} glfw Vulkan::Vulkan)

    ADD_DEPENDENCIES(${FILE_NAME} shaders_all asset_dir_all)
ENDFUNCTION()

FOREACH(source ${DEMO_SRC})
  add_demo_exec(${source})
ENDFOREACH()



