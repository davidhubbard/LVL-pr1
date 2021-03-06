cmake_minimum_required (VERSION 2.8.11)
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    add_definitions(-DVK_USE_PLATFORM_WIN32_KHR -DVK_USE_PLATFORM_WIN32_KHX -DWIN32_LEAN_AND_MEAN)
    set(DisplayServer Win32)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Android")
    add_definitions(-DVK_USE_PLATFORM_ANDROID_KHR -DVK_USE_PLATFORM_ANDROID_KHX)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    if (BUILD_WSI_XCB_SUPPORT)
        add_definitions(-DVK_USE_PLATFORM_XCB_KHR -DVK_USE_PLATFORM_XCB_KHX)
    endif()

    if (BUILD_WSI_XLIB_SUPPORT)
       add_definitions(-DVK_USE_PLATFORM_XLIB_KHR -DVK_USE_PLATFORM_XLIB_KHX -DVK_USE_PLATFORM_XLIB_XRANDR_EXT)
    endif()

    if (BUILD_WSI_WAYLAND_SUPPORT)
       add_definitions(-DVK_USE_PLATFORM_WAYLAND_KHR -DVK_USE_PLATFORM_WAYLAND_KHX)
    endif()

    if (BUILD_WSI_MIR_SUPPORT)
        add_definitions(-DVK_USE_PLATFORM_MIR_KHR -DVK_USE_PLATFORM_MIR_KHX)
        include_directories(${MIR_INCLUDE_DIR})
    endif()
else()
    message(FATAL_ERROR "Unsupported Platform!")
endif()

set(ICD_JSON_FILES VkICD_mock_icd)

if (WIN32)
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        if (CMAKE_GENERATOR MATCHES "^Visual Studio.*")
            foreach (config_file ${ICD_JSON_FILES})
                FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/windows/${config_file}.json src_json)
                FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>/${config_file}.json dst_json)
                add_custom_target(${config_file}-json ALL
                    COMMAND copy ${src_json} ${dst_json}
                    VERBATIM
                    )
                set_target_properties(${config_file}-json PROPERTIES FOLDER ${LVL_TARGET_FOLDER})
            endforeach(config_file)
        else()
            foreach (config_file ${ICD_JSON_FILES})
                FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/windows/${config_file}.json src_json)
                FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/${config_file}.json dst_json)
                add_custom_target(${config_file}-json ALL
                    COMMAND copy ${src_json} ${dst_json}
                    VERBATIM
                    )
            endforeach(config_file)
        endif()
    endif()
else()
    # extra setup for out-of-tree builds
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        foreach (config_file ${ICD_JSON_FILES})
            add_custom_target(${config_file}-json ALL
                COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/linux/${config_file}.json
                VERBATIM
                )
        endforeach(config_file)
    endif()
endif()
# For ICD with a direct dependency on a project with the same name, use it.
if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
    foreach (config_file ${ICD_JSON_FILES})
        add_dependencies(${config_file}-json ${config_file})
    endforeach(config_file)
endif()

add_custom_target(generate_icd_files DEPENDS
    mock_icd.h
    mock_icd.cpp
    )
set_target_properties(generate_icd_files PROPERTIES FOLDER ${LVL_TARGET_FOLDER})

if (WIN32)
    macro(add_vk_icd target)
    FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/VkICD_${target}.def DEF_FILE)
    add_custom_target(copy-${target}-def-file ALL
        COMMAND ${CMAKE_COMMAND} -E copy_if_different ${DEF_FILE} VkICD_${target}.def
        VERBATIM
    )
    set_target_properties(copy-${target}-def-file PROPERTIES FOLDER ${LVL_TARGET_FOLDER})
    add_library(VkICD_${target} SHARED ${ARGN} VkICD_${target}.def)
    add_dependencies(VkICD_${target} generate_icd_files)
    #target_link_Libraries(VkICD_${target} VkICD_utils)
    #add_dependencies(VkICD_${target} generate_helper_files VkICD_utils)
    endmacro()
else()
    macro(add_vk_icd target)
    add_library(VkICD_${target} SHARED ${ARGN})
    #target_link_Libraries(VkICD_${target} VkICD_utils)
    add_dependencies(VkICD_${target} generate_icd_files)
    set_target_properties(VkICD_${target} PROPERTIES LINK_FLAGS "-Wl,-export-dynamic,-Bsymbolic,--exclude-libs,ALL")
    if(INSTALL_ICD_FILES)
        install(TARGETS VkICD_${target} DESTINATION ${CMAKE_INSTALL_LIBDIR})
    endif()
    endmacro()
endif()

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/../loader
    ${CMAKE_CURRENT_SOURCE_DIR}/../include/vulkan
    ${CMAKE_CURRENT_BINARY_DIR}
    ${PROJECT_BINARY_DIR}
    ${CMAKE_BINARY_DIR}
)

if (WIN32)
    set (CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -D_CRT_SECURE_NO_WARNINGS")
    set (CMAKE_C_FLAGS_RELEASE   "${CMAKE_C_FLAGS_RELEASE} -D_CRT_SECURE_NO_WARNINGS")
    set (CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} -D_CRT_SECURE_NO_WARNINGS")
    set (CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} -D_CRT_SECURE_NO_WARNINGS")
    set (CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_CXX_FLAGS_DEBUG} -D_CRT_SECURE_NO_WARNINGS /bigobj")
    set (CMAKE_C_FLAGS_DEBUG     "${CMAKE_C_FLAGS_DEBUG} -D_CRT_SECURE_NO_WARNINGS /bigobj")
    # Turn off transitional "changed behavior" warning message for Visual Studio versions prior to 2015.
    # The changed behavior is that constructor initializers are now fixed to clear the struct members.
    add_compile_options("$<$<AND:$<CXX_COMPILER_ID:MSVC>,$<VERSION_LESS:$<CXX_COMPILER_VERSION>,19>>:/wd4351>")
else()
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wpointer-arith -Wno-unused-function -Wno-sign-compare")
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wpointer-arith -Wno-unused-function -Wno-sign-compare")
endif()

run_vk_xml_generate(mock_icd_generator.py mock_icd.h)
run_vk_xml_generate(mock_icd_generator.py mock_icd.cpp)

add_vk_icd(mock_icd mock_icd.cpp mock_icd.h)

