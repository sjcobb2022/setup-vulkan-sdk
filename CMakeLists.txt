# Vulkan SDK config.json cmake wrapper
# 2022.02.16 humbletim MIT license
# see: https://github.com/humbletim/setup-vulkan-sdk

# cmake variables:
#  -DVULKAN_SDK -- where to install SDK components
#  -DVULKAN_SDK_CONFIG -- config.json that specifies repos/branches
#  -DVULKAN_SDK_COMPONENTS -- which SDK components to install

#  note: a Vulkan SDK config.json can be downloaded using the template:
#   https://vulkan.lunarg.com/sdk/config/VERSION/PLATFORM/config.json
#  (see: https://vulkan.lunarg.com/content/view/latest-sdk-version-api)

cmake_minimum_required(VERSION 3.19)
# note: 3.19+ needed for JSON parsing

include(ExternalProject)
include(cmake/json_utils.cmake)

project(VulkanSDKComponents)

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/modules")


set(AVAILABLE_COMPONENTS "Vulkan-Headers Vulkan-Loader SPIRV-Headers SPIRV-Cross SPIRV-Tools SPIRV-Reflect Glslang shaderc")
# TODO: Vulkan-ExtensionLayer Khronos-Tools LunarG-Tools DXC gfxreconstruct MoltenVK Vulkan-Docs Vulkan-ValidationLayers robin-hood-hashing shaderc

macro(_assert varname)
  if (NOT ${varname})
    message(FATAL_ERROR "assertion failed (${varname}):\n" ${ARGN})
  endif()
endmacro()

function(ls_path path)
  if(WIN32)
    execute_process(COMMAND powershell "-Command" "ls ${path}")
  else()
    execute_process(COMMAND bash "-c" "ls ${path}")
  endif()
endfunction()

_assert(VULKAN_SDK "define -DVULKAN_SDK=<path> to desired VULKAN_SDK installation folder")
_assert(VULKAN_SDK_CONFIG "define -DVULKAN_SDK_CONFIG=<file> to Vulkan SDK config.json")
_assert(VULKAN_SDK_COMPONENTS "define -DVULKAN_SDK_COMPONENTS=\"Vulkan-Headers;Vulkan-Loader;etc.\"\n"
                              "available: ${AVAILABLE_COMPONENTS}")

STRING(REGEX REPLACE "[, ]+" ";" VULKAN_SDK_COMPONENTS "${VULKAN_SDK_COMPONENTS}")

if (DEFINED CMAKE_INSTALL_PREFIX)
  message(STATUS "CMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX} -- replacing with VULKAN_SDK=${VULKAN_SDK}")
  set(CMAKE_INSTALL_PREFIX ${VULKAN_SDK})
endif()

##############################################################################
# manual Vulkan SDK build overrides

# manually list SPIRV-Headers as SPIRV-Tools dependency
set_property(GLOBAL APPEND PROPERTY SPIRV-Tools_deps_extra "SPIRV-Headers\;SPIRV-Headers_SOURCE_DIR")

# manually list robin-hood-hashing as Vulkan-ValidationLayers dependency
set_property(GLOBAL APPEND PROPERTY Vulkan-ValidationLayers_deps_extra "robin-hood-hashing\;ROBIN_HOOD_HASHING_INSTALL_DIR")

set_property(GLOBAL APPEND PROPERTY shaderc_deps_extra "SPIRV-Tools\;spirv-tools_SOURCE_DIR;Glslang\;glslang_SOURCE_DIR")
# set_property(GLOBAL APPEND PROPERTY shaderc_cmake_extra -DCMAKE_DEBUG_POSTFIX:STRING=d)
set_property(GLOBAL APPEND PROPERTY shaderc_cmake_extra -DSHADERC_SKIP_TESTS:BOOL=ON)
set_property(GLOBAL APPEND PROPERTY shaderc_cmake_extra -DSHADERC_ENABLE_SHARED_CRT:BOOL=ON)
set_property(GLOBAL APPEND PROPERTY shaderc_cmake_extra -DSHADERC_SKIP_COPYRIGHT_CHECK:BOOL=ON)
# set_property(GLOBAL APPEND PROPERTY shaderc_cmake_extra -DSHADERC_THIRD_PARTY_ROOT_DIR:FILEPATH=${VULKAN_SDK})

message(STATUS "cmake extra ${shaderc_cmake_extra}")

# set(CMAKE_BINARY_DIR C:/dev/rush.vk/build/_vulkan_build)
# set(CMAKE_BUILD C:/dev/rush.vk/build/_vulkan_build)


# robin-hood-hashing is odd in Vulkan SDK config.json's -- but only the header is actually needed
set_property(GLOBAL APPEND PROPERTY robin-hood-hashing_install_command 
  test -d ${VULKAN_SDK}/include/ || mkdir -v ${VULKAN_SDK}/include/ &&
  cp -av ../robin-hood-hashing/src/include/robin_hood.h ${VULKAN_SDK}/include/
)

# Need to override branch, since config.json always points to the known_good
set_property(GLOBAL APPEND PROPERTY shaderc_commit "1bbf43f210941ba69a2cd05cf3529063f1ff5bb9")

set_property(GLOBAL APPEND PROPERTY robin-hood-hashing_cmake_extra -DRH_STANDALONE_PROJECT:BOOL=OFF)

# if SPIRV-Tools-opt is available as part of installed components enable Glslang to locate it
if(VULKAN_SDK_COMPONENTS MATCHES "Glslang.*SPIRV-Tools|SPIRV-Tools.*Glslang|shaderc")
  message("Glslang + SPIRV-Tools -- patching Glslang dependency on SPIRV-Tools-opt")
  set_property(GLOBAL APPEND PROPERTY Glslang_deps_extra "SPIRV-Tools\;spirv-tools_SOURCE_DIR")
  file(WRITE ${CMAKE_BINARY_DIR}/Glslang_SPIRV_patch.cmake  [====[
    add_library(SPIRV-Tools-opt INTERFACE)
    set_property(TARGET SPIRV-Tools-opt APPEND PROPERTY INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib )
    set_property(TARGET SPIRV-Tools-opt APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/include )
    install(TARGETS SPIRV-Tools-opt EXPORT SPIRV-Tools-opt)
    install(EXPORT SPIRV-Tools-opt DESTINATION lib/cmake)
    set(spirv_tools NOTFOUND)
    set(spirv_tools_opt NOTFOUND)
    find_library(spirv_tools SPIRV-Tools PATHS ${CMAKE_INSTALL_PREFIX}/lib REQUIRED NO_DEFAULT_PATH NO_PACKAGE_ROOT_PATH NO_CMAKE_PATH NO_CMAKE_ENVIRONMENT_PATH NO_SYSTEM_ENVIRONMENT_PATH NO_CMAKE_SYSTEM_PATH NO_CMAKE_FIND_ROOT_PATH)
    find_library(spirv_tools_opt SPIRV-Tools-opt PATHS ${CMAKE_INSTALL_PREFIX}/lib REQUIRED NO_DEFAULT_PATH NO_PACKAGE_ROOT_PATH NO_CMAKE_PATH NO_CMAKE_ENVIRONMENT_PATH NO_SYSTEM_ENVIRONMENT_PATH NO_CMAKE_SYSTEM_PATH NO_CMAKE_FIND_ROOT_PATH)
    set_property(TARGET SPIRV-Tools-opt APPEND PROPERTY INTERFACE_LINK_LIBRARIES ${spirv_tools_opt} ${spirv_tools})
  ]====])
  set_property(GLOBAL APPEND PROPERTY Glslang_patch_command bash -c "echo 'include(${CMAKE_BINARY_DIR}/Glslang_SPIRV_patch.cmake)' | tee External/CMakeLists.txt")
  set_property(GLOBAL APPEND PROPERTY Glslang_cmake_extra -DENABLE_OPT:BOOL=ON)

else()
  # otherwise disable External completely 
  set_property(GLOBAL APPEND PROPERTY Glslang_cmake_extra -DBUILD_EXTERNAL:BOOL=OFF)
endif()

if(VULKAN_SDK_COMPONENTS MATCHES "shaderc")
  file(WRITE ${CMAKE_BINARY_DIR}/shaderc_patch.cmake  [====[

    message(STATUS "======================== SHADER PATCH 1 =====================")

    function(ls_path path)
      if(WIN32)
        execute_process(COMMAND powershell "-Command" "ls ${path}")
      else()
        execute_process(COMMAND bash "-c" "ls ${path}")
      endif()
    endfunction()

    # set(SHADERC_SPIRV_TOOLS_DIR "${CMAKE_BINARY_DIR}/../../../SPIRV-Tools-prefix/src/SPIRV-Tools" CACHE FILEPATH "" FORCE)
    set(spirv-tools_SOURCE_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE FILEPATH "" FORCE)
    # set(SHADERC_SPIRV_HEADERS_DIR "${CMAKE_BINARY_DIR}/../../../SPIRV-Headers-prefix/src/SPIRV-Headers" CACHE FILEPATH "" FORCE)
    set(SPIRV-Headers_SOURCE_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE FILEPATH "" FORCE)
    # set(SHADERC_GLSLANG_DIR "${CMAKE_BINARY_DIR}/../../../Glslang-prefix/src/Glslang" CACHE FILEPATH "" FORCE)
    set(glslang_SOURCE_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE FILEPATH "" FORCE)

    ls_path(${glslang_SOURCE_DIR})

    # find_library(SPIRV-Tools SPIRV-Tools REQUIRED)
    # find_library(SPIRV-Tools-opt SPIRV-Tools-opt REQUIRED)
    # set(SPIRV-Tools SPIRV-Tools PARENT_SCOPE)

    # ls_path(${CMAKE_INSTALL_PREFIX}/lib)
    # ls_path(${CMAKE_INSTALL_PREFIX}/lib/cmake)
    # ls_path(${CMAKE_INSTALL_PREFIX}/include)
    # ls_path(${CMAKE_INSTALL_PREFIX}/include/spirv-tools)

    # ls_path(${CMAKE_INSTALL_PREFIX})

    include(FindPackageHandleStandardArgs)
    include(SelectLibraryConfigurations)

    find_path(glslang_INCLUDE_DIR NAMES     glslang/build_info.h     HINTS "${CMAKE_INSTALL_PREFIX}/include")
    find_library(glslang_LIBRARY            NAMES glslang            HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(GenericCodeGen_LIBRARY     NAMES GenericCodeGen     HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(MachineIndependent_LIBRARY NAMES MachineIndependent HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(OGLCompiler_LIBRARY        NAMES OGLCompiler        HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(OSDependent_LIBRARY        NAMES OSDependent        HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(SPIRV_LIBRARY              NAMES SPIRV              HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(SPIRV-Tools_LIBRARY        NAMES  SPIRV-Tools       HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(SPIRV-Tools-opt_LIBRARY    NAMES SPIRV-Tools-opt    HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(SPIRV-Tools-link_LIBRARY   NAMES SPIRV-Tools-link   HINTS "${CMAKE_INSTALL_PREFIX}/lib")
    find_library(SPIRV-Tools-reduce_LIBRARY NAMES SPIRV-Tools-reduce HINTS "${CMAKE_INSTALL_PREFIX}/lib")

    ls_path(glslang_INCLUDE_DIR)

    set(glslang_INCLUDE_DIRS         ${glslang_INCLUDE_DIR})
    set(glslang_LIBRARIES            ${glslang_LIBRARY})
    set(GenericCodeGen_LIBRARIES     ${GenericCodeGen_LIBRARY})
    set(MachineIndependent_LIBRARIES ${MachineIndependent_LIBRARY})
    set(OGLCompiler_LIBRARIES        ${OGLCompiler_LIBRARY})
    set(OSDependent_LIBRARIES        ${OSDependent_LIBRARY})
    set(SPIRV_LIBRARIES              ${SPIRV_LIBRARY})
    set(SPIRV-Tools_LIBRARIES        ${SPIRV-Tools_LIBRARY})
    set(SPIRV-Tools-opt_LIBRARIES    ${SPIRV-Tools-opt_LIBRARY})
    set(SPIRV-Tools-link_LIBRARIES   ${SPIRV-Tools-link_LIBRARY})
    set(SPIRV-Tools-reduce_LIBRARIES ${SPIRV-Tools-reduce_LIBRARY})

    find_package_handle_standard_args(glslang            DEFAULT_MSG glslang_LIBRARY            glslang_INCLUDE_DIR)
    find_package_handle_standard_args(GenericCodeGen     DEFAULT_MSG GenericCodeGen_LIBRARY     glslang_INCLUDE_DIR)
    find_package_handle_standard_args(MachineIndependent DEFAULT_MSG MachineIndependent_LIBRARY glslang_INCLUDE_DIR)
    find_package_handle_standard_args(OGLCompiler        DEFAULT_MSG OGLCompiler_LIBRARY        glslang_INCLUDE_DIR)
    find_package_handle_standard_args(OSDependent        DEFAULT_MSG OSDependent_LIBRARY        glslang_INCLUDE_DIR)
    find_package_handle_standard_args(SPIRV              DEFAULT_MSG SPIRV_LIBRARY              glslang_INCLUDE_DIR)
    find_package_handle_standard_args(SPIRV-Tools DEFAULT_MSG SPIRV-Tools_LIBRARY glslang_INCLUDE_DIR)
    find_package_handle_standard_args(SPIRV-Tools-opt    DEFAULT_MSG SPIRV-Tools_LIBRARY glslang_INCLUDE_DIR)
    find_package_handle_standard_args(SPIRV-Tools-link   DEFAULT_MSG SPIRV-Tools_LIBRARY glslang_INCLUDE_DIR)
    find_package_handle_standard_args(SPIRV-Tools-reduce DEFAULT_MSG SPIRV-Tools_LIBRARY glslang_INCLUDE_DIR)

    mark_as_advanced(glslang_INCLUDE_DIR glslang_LIBRARY)
    
    if(glslang_FOUND AND NOT TARGET glslang)
      add_library(glslang UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
      set_target_properties(glslang PROPERTIES
        IMPORTED_LOCATION "${glslang_LIBRARY}"
        INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
        INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()
    
    if(GenericCodeGen_FOUND AND NOT TARGET GenericCodeGen)
    add_library(GenericCodeGen UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(GenericCodeGen PROPERTIES
      IMPORTED_LOCATION "${GenericCodeGen_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    if(MachineIndependent_FOUND AND NOT TARGET MachineIndependent)
    add_library(MachineIndependent UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(MachineIndependent PROPERTIES
      IMPORTED_LOCATION "${MachineIndependent_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    if(OGLCompiler_FOUND AND NOT TARGET OGLCompiler)
    add_library(OGLCompiler UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(OGLCompiler PROPERTIES
      IMPORTED_LOCATION "${OGLCompiler_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    if(OSDependent_FOUND AND NOT TARGET OSDependent)
    add_library(OSDependent UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(OSDependent PROPERTIES
      IMPORTED_LOCATION "${OSDependent_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    if(SPIRV_FOUND AND NOT TARGET SPIRV)
    add_library(SPIRV UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(SPIRV PROPERTIES
      IMPORTED_LOCATION "${SPIRV_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    ls_path(${CMAKE_INSTALL_PREFIX}/include/glslang)

    ls_path(${CMAKE_INSTALL_PREFIX}/include/spirv)

    ls_path(${CMAKE_INSTALL_PREFIX}/include/glslang/SPIRV)

    target_include_directories(
      SPIRV 
      INTERFACE ${CMAKE_INSTALL_PREFIX}/include
      INTERFACE ${CMAKE_INSTALL_PREFIX}/include/glslang
      INTERFACE ${CMAKE_INSTALL_PREFIX}/include/glslang/SPIRV)
    
    # include_directories("${glslang_INCLUDE_DIRS}")
    include_directories("${CMAKE_INSTALL_PREFIX}/include;${CMAKE_INSTALL_PREFIX}/include/glslang;${CMAKE_INSTALL_PREFIX}/include/glslang/SPIRV")


    if(SPIRV-Tools_FOUND AND NOT TARGET SPIRV-Tools)
    add_library(SPIRV-Tools UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(SPIRV-Tools PROPERTIES
      IMPORTED_LOCATION "${SPIRV-Tools_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    if(SPIRV-Tools-opt_FOUND AND NOT TARGET SPIRV-Tools-opt)
    add_library(SPIRV-Tools-opt UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(SPIRV-Tools-opt PROPERTIES
      IMPORTED_LOCATION "${SPIRV-Tools-opt_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    if(SPIRV-Tools-link_FOUND AND NOT TARGET SPIRV-Tools-link)
    add_library(SPIRV-Tools-link UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(SPIRV-Tools-link PROPERTIES
      IMPORTED_LOCATION "${SPIRV-Tools-link_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    if(SPIRV-Tools-reduce_FOUND AND NOT TARGET SPIRV-Tools-reduce)
    add_library(SPIRV-Tools-reduce UNKNOWN IMPORTED IMPORTED_GLOBAL TRUE)
    set_target_properties(SPIRV-Tools-reduce PROPERTIES
      IMPORTED_LOCATION "${SPIRV-Tools-reduce_LIBRARY}"
      INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib
      INTERFACE_INCLUDE_DIRECTORIES "${glslang_INCLUDE_DIRS}")
    endif()

    target_link_libraries(SPIRV-Tools-opt    INTERFACE SPIRV-Tools)
    target_link_libraries(SPIRV-Tools-reduce INTERFACE SPIRV-Tools SPIRV-Tools-opt)
    target_link_libraries(SPIRV-Tools-link   INTERFACE SPIRV-Tools-opt)

        
    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads)
    target_link_libraries(OSDependent        INTERFACE Threads::Threads)
    target_link_libraries(MachineIndependent INTERFACE OGLCompiler OSDependent GenericCodeGen)
    target_link_libraries(glslang            INTERFACE OGLCompiler OSDependent MachineIndependent)

    target_link_libraries(SPIRV              INTERFACE MachineIndependent SPIRV-Tools-opt)

    # find_library(SPIRV_TOOLS_LIBRARY_DEBUG NAMES SPIRV-Tools PATH_SUFFIXES DEBUG/lib)
    # find_library(SPIRV_TOOLS_LIBRARY_RELEASE NAMES SPIRV-Tools PATH_SUFFIXES lib)
    # find_path(SPIRV_TOOLS_INCLUDE_DIR NAMES spirv-tools/libspirv.h HINTS ${CMAKE_INSTALL_PREFIX} ${CMAKE_INSTALL_PREFIX}/include/spirv-tools PATH_SUFFIXES include)
    # find_library(SPIRV_TOOLS_OPT_LIBRARY_DEBUG NAMES SPIRV-Tools-opt PATH_SUFFIXES DEBUG/lib)
    # find_library(SPIRV_TOOLS_OPT_LIBRARY_RELEASE NAMES SPIRV-Tools-opt PATH_SUFFIXES lib)
    
    # select_library_configurations(SPIRV_TOOLS)
    # select_library_configurations(SPIRV_TOOLS_OPT)

    # find_package_handle_standard_args(
    #   SPIRV-Tools
    #   DEFAULT_MSG
    #   SPIRV_TOOLS_LIBRARY
    #   SPIRV_TOOLS_OPT_LIBRARY
    #   SPIRV_TOOLS_INCLUDE_DIR
    # )

    # add_library(SPIRV-Tools STATIC IMPORTED)

    # message(STATUS "Found spirv tools include: ${SPIRV_TOOLS_INCLUDE_DIR}")

    # if(SPIRV_TOOLS_LIBRARY_RELEASE)
    #     set_property(TARGET SPIRV-Tools APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)
    #     set_target_properties(SPIRV-Tools PROPERTIES
    #         IMPORTED_LOCATION_RELEASE ${SPIRV_TOOLS_LIBRARY_RELEASE})
    #         message(STATUS "SPIRV-Tools Release: ${SPIRV_TOOLS_LIBRARY_RELEASE}")
    # endif()
    # if(SPIRV_TOOLS_LIBRARY_DEBUG)
    #     set_property(TARGET SPIRV-Tools APPEND PROPERTY IMPORTED_CONFIGURATIONS DEBUG)
    #     set_target_properties(SPIRV-Tools PROPERTIES
    #         IMPORTED_LOCATION_DEBUG ${SPIRV_TOOLS_LIBRARY_DEBUG})
    #         message(STATUS "SPIRV-Tools Debug: ${SPIRV_TOOLS_LIBRARY_DEBUG}")
    # endif()

    # set_target_properties(SPIRV-Tools PROPERTIES INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/include )

    # get_target_property(_SPIRVTOOLS_INTERFACE_INCLUDE_DIRECTORIES SPIRV-Tools INTERFACE_INCLUDE_DIRECTORIES)
    # message(STATUS "${_SPIRVTOOLS_INTERFACE_INCLUDE_DIRECTORIES}")

    # if(NOT TARGET SPIRV-Tools)
    #   message(STATUS " >>>>>>>>>>>>>>>>>> SPIRV-Tools is NOT a target")
    # endif()

    # add_library(SPIRV-Tools-opt STATIC IMPORTED)

    # if(SPIRV_TOOLS_OPT_LIBRARY_RELEASE)
    #   set_property(TARGET SPIRV-Tools-opt APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)
    #   set_target_properties(SPIRV-Tools PROPERTIES
    #     IMPORTED_LOCATION_RELEASE ${SPIRV_TOOLS_OPT_LIBRARY_RELEASE})
    # endif()
    # if(SPIRV_TOOLS_OPT_LIBRARY_DEBUG)
    #     set_property(TARGET SPIRV-Tools-opt APPEND PROPERTY IMPORTED_CONFIGURATIONS DEBUG)
    #     set_target_properties(SPIRV-Tools-opt PROPERTIES
    #         IMPORTED_LOCATION_DEBUG ${SPIRV_TOOLS_OPT_LIBRARY_DEBUG})
    # endif()

    # set_target_properties(SPIRV-Tools-opt PROPERTIES INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/include)
    # set_property(TARGET SPIRV-Tools-opt APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/include )
    # target_link_libraries(SPIRV-Tools-opt INTERFACE SPIRV-Tools)

    # find_library(GLSLANG_LIBRARY_RELEASE NAMES glslang PATH_SUFFIXES lib)
    # find_library(OGLCOMPILER_LIBRARY_RELEASE NAMES OGLCompiler PATH_SUFFIXES lib)
    # find_library(OSDEPENDENT_LIBRARY_RELEASE NAMES OSDependent PATH_SUFFIXES lib)
    # find_library(HLSL_LIBRARY_RELEASE NAMES HLSL PATH_SUFFIXES lib)
    # find_library(SPIRV_LIBRARY_RELEASE NAMES SPIRV PATH_SUFFIXES lib)
    # find_library(GLSLANG_LIBRARY_DEBUG NAMES glslangd PATH_SUFFIXES DEBUG/lib)
    # find_library(OGLCOMPILER_LIBRARY_DEBUG NAMES OGLCompilerd PATH_SUFFIXES DEBUG/lib)
    # find_library(OSDEPENDENT_LIBRARY_DEBUG NAMES OSDependentd PATH_SUFFIXES DEBUG/lib)
    # find_library(SPIRV_LIBRARY_DEBUG NAMES SPIRVd PATH_SUFFIXES DEBUG/lib)
    # find_library(HLSL_LIBRARY_DEBUG NAMES HLSLd PATH_SUFFIXES DEBUG/lib)
    # find_path(GLSLANG_INCLUDE_DIR NAMES glslang/Public/ShaderLang.h PATH_SUFFIXES include)
    # find_path(SPIRV_INCLUDE_DIR NAMES SPIRV/spirv.hpp HINTS ${CMAKE_INSTALL_PREFIX} ${CMAKE_INSTALL_PREFIX}/include/glslang PATH_SUFFIXES include)
    # find_program(GLSLANG_VALIDATOR_EXE NAMES glslangValidator PATH_SUFFIXES bin)
    
    # select_library_configurations(GLSLANG)
    # select_library_configurations(OGLCOMPILER)
    # select_library_configurations(OSDEPENDENT)
    # select_library_configurations(SPIRV)
    # select_library_configurations(HLSL)

    # find_package_handle_standard_args(glslang
    #   DEFAULT_MSG
    #   GLSLANG_LIBRARY
    #   OGLCOMPILER_LIBRARY
    #   OSDEPENDENT_LIBRARY
    #   SPIRV_LIBRARY
    #   HLSL_LIBRARY
    #   GLSLANG_INCLUDE_DIR
    #   SPIRV_INCLUDE_DIR
    # )

    # message(STATUS "glslang:         ${GLSLANG_LIBRARY}")
    # message(STATUS "OGLCompiler:     ${OGLCOMPILER_LIBRARY}")
    # message(STATUS "OSDependant:     ${OSDEPENDENT_LIBRARY}")
    # message(STATUS "SPIRV:           ${SPIRV_LIBRARY}")
    # message(STATUS "HLSL:            ${HLSL_LIBRARY}")
    # message(STATUS "glslang_INCLUDE: ${GLSLANG_INCLUDE_DIR}")
    # message(STATUS "SPIRV_INCLUDE:   ${SPIRV_INCLUDE_DIR}")

    # add_library(OGLCompiler STATIC IMPORTED)
    # if(OGLCOMPILER_LIBRARY_RELEASE)
    #     set_property(TARGET OGLCompiler APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)
    #     set_target_properties(OGLCompiler PROPERTIES
    #         IMPORTED_LOCATION_RELEASE ${OGLCOMPILER_LIBRARY_RELEASE})
    # endif()
    # if(OGLCOMPILER_LIBRARY_DEBUG)
    #     set_property(TARGET OGLCompiler APPEND PROPERTY IMPORTED_CONFIGURATIONS DEBUG)
    #     set_target_properties(OGLCompiler PROPERTIES
    #         IMPORTED_LOCATION_DEBUG ${OGLCOMPILER_LIBRARY_DEBUG})
    # endif()

    # add_library(OSDependent STATIC IMPORTED)
    # if(OSDEPENDENT_LIBRARY_RELEASE)
    #     set_property(TARGET OSDependent APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)
    #     set_target_properties(OSDependent PROPERTIES
    #         IMPORTED_LOCATION_RELEASE ${OSDEPENDENT_LIBRARY_RELEASE})
    # endif()
    # if(OSDEPENDENT_LIBRARY_DEBUG)
    #     set_property(TARGET OSDependent APPEND PROPERTY IMPORTED_CONFIGURATIONS DEBUG)
    #     set_target_properties(OSDependent PROPERTIES
    #         IMPORTED_LOCATION_DEBUG ${OSDEPENDENT_LIBRARY_DEBUG})
    # endif()

    # add_library(HLSL STATIC IMPORTED)
    # if(HLSL_LIBRARY_RELEASE)
    #     set_property(TARGET HLSL APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)
    #     set_target_properties(HLSL PROPERTIES
    #         IMPORTED_LOCATION_RELEASE ${HLSL_LIBRARY_RELEASE})
    # endif()
    # if(HLSL_LIBRARY_DEBUG)
    #     set_property(TARGET HLSL APPEND PROPERTY IMPORTED_CONFIGURATIONS DEBUG)
    #     set_target_properties(HLSL PROPERTIES
    #         IMPORTED_LOCATION_DEBUG ${HLSL_LIBRARY_DEBUG})
    # endif()

    # add_library(glslang STATIC IMPORTED)
    # if(GLSLANG_LIBRARY_RELEASE)
    #     set_property(TARGET glslang APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)
    #     set_target_properties(glslang PROPERTIES
    #         IMPORTED_LOCATION_RELEASE ${GLSLANG_LIBRARY_RELEASE})
    # endif()
    # if(GLSLANG_LIBRARY_DEBUG)
    #     set_property(TARGET glslang APPEND PROPERTY IMPORTED_CONFIGURATIONS DEBUG)
    #     set_target_properties(glslang PROPERTIES
    #         IMPORTED_LOCATION_DEBUG ${GLSLANG_LIBRARY_DEBUG})
    # endif()

    # set_target_properties(glslang PROPERTIES INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include;${CMAKE_INSTALL_PREFIX}/include/glslang")
    # set_property(TARGET glslang APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include/glslang" )
    # target_include_directories(glslang INTERFACE "${CMAKE_INSTALL_PREFIX}/include/glslang")
    # target_link_libraries(glslang INTERFACE OGLCompiler OSDependent HLSL)  

    # add_library(SPIRV UNKNOWN IMPORTED)
    # if(SPIRV_LIBRARY_RELEASE)
    #     set_property(TARGET SPIRV APPEND PROPERTY IMPORTED_CONFIGURATIONS RELEASE)
    #     set_target_properties(SPIRV PROPERTIES
    #         IMPORTED_LOCATION_RELEASE ${SPIRV_LIBRARY_RELEASE}
    #         INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/include/glslang)
    # endif()
    # if(SPIRV_LIBRARY_DEBUG)
    #     set_property(TARGET SPIRV APPEND PROPERTY IMPORTED_CONFIGURATIONS DEBUG)
    #     set_target_properties(SPIRV PROPERTIES
    #         IMPORTED_LOCATION_DEBUG ${SPIRV_LIBRARY_DEBUG})
    # endif()

    # message(STATUS "SPIRV_LIBRARY: ${SPIRV_LIBRARY}")
    # message(STATUS "SPIRV_INCLUDE_DIR: ${SPIRV_INCLUDE_DIR}")
    # message(STATUS "glslang INCLUDE: ${CMAKE_INSTALL_PREFIX}/include/glslang")

    # set_target_properties(SPIRV PROPERTIES INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/include;${CMAKE_INSTALL_PREFIX}/include/glslang")

    # target_include_directories(SPIRV INTERFACE "${CMAKE_INSTALL_PREFIX}/include/glslang")

    # target_link_libraries(SPIRV INTERFACE glslang SPIRV-Tools SPIRV-Tools-opt)

    # find_package_handle_standard_args(glslangValidator
    #     "Cannot find glslangValidator, Shader won't be compiled offline"
    #     GLSLANG_VALIDATOR_EXE
    # )

    # add_executable(glslangValidator IMPORTED)
    # set_target_properties(glslangValidator PROPERTIES IMPORTED_LOCATION ${GLSLANG_VALIDATOR_EXE})

    # mark_as_advanced(
    #   GLSLANG_LIBRARY
    #   OGLCOMPILER_LIBRARY
    #   OSDEPENDENT_LIBRARY
    #   HLSL_LIBRARY
    #   SPIRV_LIBRARY
    # )

    # ls_path(${CMAKE_INSTALL_PREFIX}/include/spirv-tools)
    
    # ls_path(${spirv-tools_SOURCE_DIR})
    # ls_path(${spirv-tools_SOURCE_DIR}/include)   
    # ls_path(${spirv-tools_SOURCE_DIR}/SPIRV-Tools)
    # ls_path(${CMAKE_INSTALL_PREFIX}/libs/cmake)
    
    # if(MSVC)
    #   set(SHADERC_ENABLE_SHARED_CRT ON CACHE BOOL "" FORCE)
    # endif()

    # add_target(${name} PROPERTIES lib [lib...])
    # function(add_target name include)

    #   add_library(${name} INTERFACE)
    #   set_property(TARGET ${name} APPEND PROPERTY INTERFACE_LINK_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/lib )
    #   set_property(TARGET ${name} APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/include )

    #   install(TARGETS ${name} EXPORT ${name})
    #   install(EXPORT ${name} DESTINATION lib/cmake)

    #   message(STATUS " >>>> This should message")
    #   message(STATUS " >>>> ${ARGN}")

    #   foreach(arg ${ARGN})
    #     string(TOLOWER ${arg} lc_name)
    #     message(STATUS "Processing ${name} for ${arg}")
    #     if(EXISTS "${CMAKE_INSTALL_PREFIX}/include/${lc_name}")
    #       message(STATUS "Path Exists ${CMAKE_INSTALL_PREFIX}/include/${lc_name}")
    #       set_property(TARGET ${name} APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/include/${lc_name} )
    #     endif()
        
    #     if(EXISTS "${CMAKE_INSTALL_PREFIX}/include/glslang/${name}")
    #       message(STATUS "Set INTERFACE_INCLUDE_DIRECTORIES for ${name} as ${CMAKE_INSTALL_PREFIX}/include/glslang/${name}")
    #       set_property(TARGET ${name} APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${CMAKE_INSTALL_PREFIX}/include/${name} )
    #     endif()

    #     string(RANDOM LENGTH 10 lib)
    #     find_library(lib ${arg} PATHS ${CMAKE_INSTALL_PREFIX}/lib REQUIRED NO_DEFAULT_PATH NO_PACKAGE_ROOT_PATH NO_CMAKE_PATH NO_CMAKE_ENVIRONMENT_PATH NO_SYSTEM_ENVIRONMENT_PATH NO_CMAKE_SYSTEM_PATH NO_CMAKE_FIND_ROOT_PATH)
    #     set_property(TARGET ${name} APPEND PROPERTY INTERFACE_LINK_LIBRARIES ${lib})
    #   endforeach()

    # endfunction()

    # add_library(SPIRV-Tools-opt INTERFACE)

  ]====])

  set_property(GLOBAL APPEND PROPERTY shaderc_patch_command bash -c "echo \"include(${CMAKE_BINARY_DIR}/shaderc_patch.cmake) $(cat ../shaderc/third_party/CMakeLists.txt)\" > ../shaderc/third_party/CMakeLists.txt")
  execute_process(COMMAND wsl "-e" "awk \'/target_link_libraries(glslangValidator \$\{LIBRARIES\})/ { print; print \"message(STATUS \"Got past the library link\")\"; next }1\' ${CMAKE_BINARY_DIR}/Glslang-prefix/src/Glslang/StandAlone/CMakeLists.txt")
  set_property(GLOBAL APPEND PROPERTY Glslang_cmake_extra -DSHADERC_ENABLE_SHARED_CRT:BOOL=ON)


  # file(WRITE ${CMAKE_BINARY_DIR}/shaderc_patch_2.cmake  [====[

  #   function(ls_path path)
  #     if(WIN32)
  #       execute_process(COMMAND powershell "-Command" "ls ${path}")
  #     else()
  #       execute_process(COMMAND bash "-c" "ls ${path}")
  #     endif()
  #   endfunction()

  #   message(STATUS "======================== SHADER_PATCH 2 =====================")

  # ]====])
  # set_property(GLOBAL APPEND PROPERTY shaderc_patch_command bash -c "echo \"include(${CMAKE_BINARY_DIR}/shaderc_patch_2.cmake) $(cat ../shaderc/CMakeLists.txt)\" > ../shaderc/CMakeLists.txt")
endif()

function(setup_component_dependency key value parentName)
  setup_component(${key} ${parentName})
  set_property(GLOBAL APPEND PROPERTY ${parentName}_cmake_extra -D${value}:FILEPATH=${VULKAN_SDK})
  set_property(GLOBAL APPEND PROPERTY ${parentName}_deps ${key})
endfunction()

function(setup_component name parentName)
    if (TARGET ${name})
      message(STATUS "skipping second ExternalProject_Add ${name}")
      return()
    endif()
    # message(">>>>>>>>>>>>>>>>>>>>>> ${name}_component_defined==${${name}_component_defined}")
    json_get_subprops("${configJson}" "repos" "${name}" PROPERTIES url dependencies)
    _assert(url "could not read URL for name=${name} url=${url}")

    # process config.json dependencies first
    json_foreach("${dependencies}" "" setup_component_dependency ${name})
    # then any override deps_extra's
    get_component_props(${name} deps_extra)
    foreach(kv ${deps_extra})
      message(STATUS "setup component dependency ${deps_extra}")
      setup_component_dependency(${kv} ${name})
    endforeach()

    json_coalesce_subprops("${configJson}" ref "repos" "${name}" PROPERTIES commit tag branch)

    get_component_props(${name} deps cmake_extra patch_command install_command)

    # convert spaces to list (otherwise CMAKE_ARGS seems to force quotes if there are spaces in singular values)
    string(REPLACE " " ";" cmake_extra "${cmake_extra}")
    set(cmake_args
        -Wno-dev
        -DCMAKE_INSTALL_MESSAGE=NEVER
        -DCMAKE_BUILD_TYPE=Release
        -DCMAKE_INSTALL_PREFIX=${VULKAN_SDK}
        # -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
        # -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
        # -DCMAKE_BUILD_WITH_INSTALL_RPATH:BOOL=ON
    )

    list(APPEND cmake_args ${cmake_extra})

    message(STATUS "CMAKE")
    message(STATUS "ExternalProject_Add ${ref} deps=[${deps}] cmake_extra=[${cmake_extra}]")
    message(STATUS ">>>>>>>>>>> REF FOR ${name} IS ${ref}")

    json_get_subprops("${configJson}" "repos" "${name}" PROPERTIES defines)

    message(STATUS " >>> ${defines}")

    ExternalProject_Add("${name}"
      GIT_REPOSITORY "${url}"
      GIT_TAG "${ref}"
      GIT_PROGRESS OFF
      CMAKE_CACHE_ARGS ${cmake_extra}
      CMAKE_ARGS ${cmake_args}
      DEPENDS ${deps}
      PATCH_COMMAND ${patch_command}
      INSTALL_COMMAND ${install_command}
    )

    # ExternalProject_Get_Property("${name}" SOURCE_DIR)
    # message(STATUS "Source dir of ${name} = ${SOURCE_DIR}")

    # message()
    set_property(GLOBAL APPEND PROPERTY VULKAN_COMPONENTS_PROCESSED ${name})
endfunction()

##############################################################################
file(READ ${VULKAN_SDK_CONFIG} configJson)

list(JOIN VULKAN_SDK_COMPONENTS " " REQUESTED_COMPONENTS)
json_get_subprops("${configJson}" PROPERTIES version release release_date)
message("---------------------------------------------------------------------")
message("sdk components:  ${REQUESTED_COMPONENTS}")
message("    build path:  ${CMAKE_BINARY_DIR}")
message("  install path:  ${VULKAN_SDK}")
message(" configuration:  ${VULKAN_SDK_CONFIG}")
message("      .version:  ${version}")
message(" .release_date:  ${release_date}")
message("---------------------------------------------------------------------")

string(REPLACE " " ";" AVAILABLE_COMPONENTS_LIST ${AVAILABLE_COMPONENTS})
foreach(name ${VULKAN_SDK_COMPONENTS})
  list(FIND AVAILABLE_COMPONENTS_LIST ${name} index)
  if (${index} EQUAL -1)
    message(FATAL_ERROR "unknown component: ${name}\navailable: ${AVAILABLE_COMPONENTS}")
  endif()
  setup_component(${name} "${name}")
endforeach()

message("===================================================================")
message("VULKAN_SDK_COMPONENTS: ${VULKAN_SDK_COMPONENTS}")
get_property(VULKAN_COMPONENTS_PROCESSED GLOBAL PROPERTY VULKAN_COMPONENTS_PROCESSED)
message("            PROCESSED: ${VULKAN_COMPONENTS_PROCESSED}")
message("===================================================================")

string(TIMESTAMP TIME_T UTC)
file(WRITE
  ${CMAKE_BINARY_DIR}/sdk.env 
  "VULKAN_SDK=${VULKAN_SDK}
VULKAN_SDK_VERSION=${version}
VULKAN_SDK_BUILD_DATE=${TIME_T}
VULKAN_SDK_RELEASE=${release_date}
VULKAN_SDK_CONFIG=${VULKAN_SDK_CONFIG}
VULKAN_SDK_COMPONENTS=\"${VULKAN_SDK_COMPONENTS}\"
")

install(FILES ${CMAKE_BINARY_DIR}/sdk.env DESTINATION .)
