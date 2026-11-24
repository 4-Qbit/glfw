cmake_minimum_required(VERSION 3.21...3.28 FATAL_ERROR)

project(GLFW VERSION 3.6.0 LANGUAGES C HOMEPAGE_URL "https://www.glfw.org/")

set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REQUIRED ON)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

string(COMPARE EQUAL "${CMAKE_CURRENT_SOURCE_DIR}" "${CMAKE_SOURCE_DIR}" GLFW_STANDALONE)

option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(GLFW_BUILD_EXAMPLES "Build the GLFW example programs" ${GLFW_STANDALONE})
option(GLFW_BUILD_TESTS "Build the GLFW test programs" ${GLFW_STANDALONE})
option(GLFW_BUILD_DOCS "Build the GLFW documentation" ON)
option(GLFW_INSTALL "Generate installation target" ON)

include(GNUInstallDirs)
include(CMakeDependentOption)

if (DEFINED GLFW_USE_WAYLAND AND UNIX AND NOT APPLE)
    message(FATAL_ERROR
        "GLFW_USE_WAYLAND has been removed; delete the CMake cache and set GLFW_BUILD_WAYLAND and GLFW_BUILD_X11 instead")
endif()

cmake_dependent_option(GLFW_BUILD_WIN32 "Build support for Win32" ON "WIN32" OFF)
cmake_dependent_option(GLFW_BUILD_COCOA "Build support for Cocoa" ON "APPLE" OFF)
cmake_dependent_option(GLFW_BUILD_X11 "Build support for X11" ON "UNIX;NOT APPLE" OFF)
cmake_dependent_option(GLFW_BUILD_WAYLAND "Build support for Wayland" ON "UNIX;NOT APPLE" OFF)

cmake_dependent_option(GLFW_USE_HYBRID_HPG "Force use of high-performance GPU on hybrid systems" OFF
                       "WIN32" OFF)
cmake_dependent_option(USE_MSVC_RUNTIME_LIBRARY_DLL "Use MSVC runtime library DLL" ON
                       "MSVC" OFF)

#What should cursors do,when the pointer is unlocked.
#1:if app not provide new position,put the cursors to the center of its window, and then update focus.
#2:Restore cursors's position to the value(before it locking), and then update focus.
set(GLFW_UNLOCK_CUR_METHOD "1" CACHE STRING
    "Wayland cursor unlock method: 1 = center, 2 = keep previous position")
set_property(CACHE GLFW_UNLOCK_CUR_METHOD PROPERTY STRINGS 1 2)

if (NOT GLFW_UNLOCK_CUR_METHOD STREQUAL "1" AND
    NOT GLFW_UNLOCK_CUR_METHOD STREQUAL "2")
    message(FATAL_ERROR "GLFW_UNLOCK_CUR_METHOD must be 1 or 2")
endif()

#Wp_pointer_warp_v1 support.
option(GLFW_ENABLE_POINTER_WARP "Enable wp_pointer_warp_v1 pointer warping" OFF)

#Xdg_toplevel_icon_v1 support.
option(GLFW_ENABLE_TOPLEVEL_ICON "Enable xdg_toplevel_icon_v1 window icons" OFF)

#Zwp_text_input_v1 support,please noice that, it has been removed since kwin 6.6.90.
option(GLFW_ENABLE_TEXT_INPUT_V1 "Enable zwp_text_input_v1 text input" OFF)

#Zwp_text_input_v3 support.
option(GLFW_ENABLE_TEXT_INPUT_V3 "Enable zwp_text_input_v3 text input" OFF)

#Fake preedit for applications without a preedit layer.
option(GLFW_ENABLE_FAKE_PREEDIT "Inject the IME preedit into the app as typed text" OFF)

#MC style: leave the IME when the cursor is locked for gameplay.
option(GLFW_MC_LEAVE_TEXT_INPUT_WHEN_CURSOR_IS_LOCKED
       "Leave text input when the cursor is locked" OFF)

if (GLFW_ENABLE_FAKE_PREEDIT AND NOT GLFW_BUILD_WAYLAND)
    message(FATAL_ERROR "GLFW_ENABLE_FAKE_PREEDIT requires GLFW_BUILD_WAYLAND")
endif()

if (GLFW_MC_LEAVE_TEXT_INPUT_WHEN_CURSOR_IS_LOCKED AND NOT GLFW_BUILD_WAYLAND)
    message(FATAL_ERROR "GLFW_MC_LEAVE_TEXT_INPUT_WHEN_CURSOR_IS_LOCKED requires GLFW_BUILD_WAYLAND")
endif()

if (GLFW_MC_LEAVE_TEXT_INPUT_WHEN_CURSOR_IS_LOCKED AND NOT (GLFW_ENABLE_TEXT_INPUT_V1 OR GLFW_ENABLE_TEXT_INPUT_V3))
    message(FATAL_ERROR "GLFW_MC_LEAVE_TEXT_INPUT_WHEN_CURSOR_IS_LOCKED requires GLFW_ENABLE_TEXT_INPUT_V1 or GLFW_ENABLE_TEXT_INPUT_V3")
endif()

set(GLFW_LIBRARY_TYPE "${GLFW_LIBRARY_TYPE}" CACHE STRING
    "Library type override for GLFW (SHARED, STATIC, OBJECT, or empty to follow BUILD_SHARED_LIBS)")

if (GLFW_LIBRARY_TYPE)
    string(COMPARE EQUAL "${GLFW_LIBRARY_TYPE}" "SHARED" GLFW_BUILD_SHARED_LIBRARY)
else()
    set(GLFW_BUILD_SHARED_LIBRARY ${BUILD_SHARED_LIBS})
endif()

list(APPEND CMAKE_MODULE_PATH "${GLFW_SOURCE_DIR}/CMake/modules")

find_package(Threads REQUIRED)

#--------------------------------------------------------------------
# Report backend selection
#--------------------------------------------------------------------
if (GLFW_BUILD_WIN32)
    message(STATUS "Including Win32 support")
endif()
if (GLFW_BUILD_COCOA)
    message(STATUS "Including Cocoa support")
endif()
if (GLFW_BUILD_WAYLAND)
    message(STATUS "Including Wayland support")
endif()
if (GLFW_BUILD_X11)
    message(STATUS "Including X11 support")
endif()

#--------------------------------------------------------------------
# Apply Microsoft C runtime library option
# This is here because it also applies to tests and examples
#--------------------------------------------------------------------
if (MSVC AND NOT USE_MSVC_RUNTIME_LIBRARY_DLL)
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

#--------------------------------------------------------------------
# Create generated files
#--------------------------------------------------------------------
include(CMakePackageConfigHelpers)

set(GLFW_CONFIG_PATH "${CMAKE_INSTALL_LIBDIR}/cmake/glfw3")

configure_package_config_file(CMake/glfw3Config.cmake.in
                              src/glfw3Config.cmake
                              INSTALL_DESTINATION "${GLFW_CONFIG_PATH}"
                              NO_CHECK_REQUIRED_COMPONENTS_MACRO)

write_basic_package_version_file(src/glfw3ConfigVersion.cmake
                                 VERSION ${GLFW_VERSION}
                                 COMPATIBILITY SameMajorVersion)

#--------------------------------------------------------------------
# Add subdirectories
#--------------------------------------------------------------------
add_subdirectory(src)

if (GLFW_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

if (GLFW_BUILD_TESTS)
    add_subdirectory(tests)
endif()

if (GLFW_BUILD_DOCS)
    add_subdirectory(docs)
endif()

#--------------------------------------------------------------------
# Install files other than the library
# The library is installed by src/CMakeLists.txt
#--------------------------------------------------------------------
if (GLFW_INSTALL)
    install(DIRECTORY include/GLFW DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
            FILES_MATCHING PATTERN glfw3.h PATTERN glfw3native.h)

    install(FILES "${GLFW_BINARY_DIR}/src/glfw3Config.cmake"
                  "${GLFW_BINARY_DIR}/src/glfw3ConfigVersion.cmake"
            DESTINATION "${GLFW_CONFIG_PATH}")

    install(EXPORT glfwTargets FILE glfw3Targets.cmake
            EXPORT_LINK_INTERFACE_LIBRARIES
            DESTINATION "${GLFW_CONFIG_PATH}")
    install(FILES "${GLFW_BINARY_DIR}/src/glfw3.pc"
            DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")

    # Only generate this target if no higher-level project already has
    if (NOT TARGET uninstall)
        configure_file(CMake/cmake_uninstall.cmake.in
                       cmake_uninstall.cmake IMMEDIATE @ONLY)

        add_custom_target(uninstall
                          "${CMAKE_COMMAND}" -P
                          "${GLFW_BINARY_DIR}/cmake_uninstall.cmake")
        set_target_properties(uninstall PROPERTIES FOLDER "GLFW3")
    endif()
endif()

