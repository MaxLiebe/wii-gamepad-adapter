cmake_minimum_required(VERSION 3.16)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

# Both dependencies are git submodules:
#   git submodule update --init --recursive
set(WIIMOTE_ROOT "${CMAKE_CURRENT_LIST_DIR}/external/wiimote-anything")
set(BLUEPAD32_ROOT "${CMAKE_CURRENT_LIST_DIR}/external/bluepad32")

if(NOT EXISTS "${WIIMOTE_ROOT}/CMakeLists.txt" OR NOT EXISTS "${BLUEPAD32_ROOT}/external/btstack/src")
    message(FATAL_ERROR "Submodules missing. Run: git submodule update --init --recursive")
endif()

set(ENV{BTSTACK_ROOT} "${BLUEPAD32_ROOT}/external/btstack")

# WiimoteAnything is a component named after its directory (wiimote-anything);
# Bluepad32 and its BTstack port live under bluepad32/src/components.
set(EXTRA_COMPONENT_DIRS
    "${WIIMOTE_ROOT}"
    "${BLUEPAD32_ROOT}/src/components"
)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)
include($ENV{BTSTACK_ROOT}/port/esp32/components/btstack/cmake/BTstack_project.cmake)

project(wii_gamepad_adapter)
