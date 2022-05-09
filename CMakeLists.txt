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

    # OFF CACHE BOOL "" FORCE

    # message(STATUS "Make directory shaderc_third_party")
    # make_directory("../../../shaderc_third_party")
    message(STATUS "List Path : _vulkan_build")
    ls_path("../../../")

    message(STATUS "List Path : shaderc-build")

    ls_path("./")

    # message(STATUS "Create symlinks from libs")
    # execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "../../../SPIRV-Tools-prefix/src/SPIRV-Tools" "../shaderc/third_party/spirv-tools")
    # execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "../../../SPIRV-Tools-prefix/src/SPIRV-Tools" "third_party/spirv-tools")
    # execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "../../../SPIRV-Headers-prefix/src/SPIRV-Headers" "../shaderc/third_party/spirv-headers")
    # execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "../../../Glslang-prefix/src/Glslang" "../shaderc/third_party/glslang")

    # message(STATUS "List Path : thirdparty")

    # ls_path("third_party")

    # message(STATUS "List Path : shaderc-build")

    # ls_path(".")

    # message(STATUS "Setting shaderc files")

    # set(SHADERC_SPIRV_TOOLS_DIR "../../../SPIRV-Tools-prefix/src/SPIRV-Tools" CACHE FILEPATH "" FORCE)
    # set(spirv-tools_SOURCE_DIR "../../../SPIRV-Tools-prefix/src/SPIRV-Tools" CACHE FILEPATH "" FORCE)
    # set(SHADERC_SPIRV_HEADERS_DIR "../../../SPIRV-Headers-prefix/src/SPIRV-Headers" CACHE FILEPATH "" FORCE)
    # set(SPIRV-Headers_SOURCE_DIR "../../../SPIRV-Headers-prefix/src/SPIRV-Headers" CACHE FILEPATH "" FORCE)
    # set(SHADERC_GLSLANG_DIR "../../../Glslang-prefix/src/Glslang" CACHE FILEPATH "" FORCE)
    # set(glslang_SOURCE_DIR "../../../Glslang-prefix/src/Glslang" CACHE FILEPATH "" FORCE)

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

    # set(GLSLANGLIBS
    # glslang
    # HLSL
    # OSDependent
    # OGLCompiler
    # SPIRV)

    # find_file(GlslangToSpv NAMES "GlslangToSpv.h" HINTS ${CMAKE_INSTALL_PREFIX}/include ${CMAKE_INSTALL_PREFIX}/include/glslang ${CMAKE_INSTALL_PREFIX}/include/glslang/SPIRV ${CMAKE_INSTALL_PREFIX}/include/SPIRV/1.0 ${CMAKE_INSTALL_PREFIX}/include/SPIRV/1.1 ${CMAKE_INSTALL_PREFIX}/include/SPIRV/1.2 ${CMAKE_INSTALL_PREFIX}/include/SPIRV/unified1)

    # ls_path(".")
    # ls_path(${CMAKE_BINARY_DIR})
    # ls_path(${PROJECT_SOURCE_DIR})
    # ls_path("D:/a/rush.vk/rush.vk/_vulkan_build")
    # ls_path("D:/a/rush.vk/rush.vk/_vulkan_build/Glslang-prefix/src/Glslang")
    # ls_path("D:/a/rush.vk/rush.vk/_vulkan_build/Glslang-prefix/src/Glslang/glslang")
    # ls_path("D:/a/rush.vk/rush.vk/_vulkan_build/Glslang-prefix/src/Glslang/glslang/Include")
    
    # add_library(SPIRV-Tools-opt INTERFACE)

  ]====])
  # set_property(GLOBAL APPEND PROPERTY shaderc_patch_command bash -c "echo 'include(${CMAKE_BINARY_DIR}/shaderc_patch.cmake)' | tee third_party/CMakeLists.txt")

  file(WRITE ${CMAKE_BINARY_DIR}/shaderc_patch_2.cmake  [====[

    function(ls_path path)
      if(WIN32)
        execute_process(COMMAND powershell "-Command" "ls ${path}")
      else()
        execute_process(COMMAND bash "-c" "ls ${path}")
      endif()
    endfunction()

    message(STATUS "======================== SHADER_PATCH 2 =====================")
    message(STATUS "listing current directory")
    ls_path("./")

    message(STATUS "cat ./CMakeLists.txt")
    execute_process(COMMAND powershell "-Command" "cat ./CMakeLists.txt")
    message(STATUS "cat third_party/CMakeLists.txt")
    execute_process(COMMAND powershell "-Command" "cat third_party/CMakeLists.txt")

    # message(STATUS ">>>>>>>>> CREATE SYMLINKS FOR LIBS")
    # execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "../../../SPIRV-Tools-prefix/src/SPIRV-Tools" "../shaderc/third_party/spirv-tools")
    # execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "../../../SPIRV-Tools-prefix/src/SPIRV-Tools" "third_party/spirv-tools")
    # execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "../../../SPIRV-Headers-prefix/src/SPIRV-Headers" "../shaderc/third_party/spirv-headers")
    # execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink "../../../Glslang-prefix/src/Glslang" "../shaderc/third_party/glslang")

  ]====])
  set_property(GLOBAL APPEND PROPERTY shaderc_patch_command bash -c "echo 'include(${CMAKE_BINARY_DIR}/shaderc_patch_2.cmake)' | tee ./CMakeLists.txt && echo 'include(${CMAKE_BINARY_DIR}/shaderc_patch.cmake)' | tee third_party/CMakeLists.txt")
  
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
