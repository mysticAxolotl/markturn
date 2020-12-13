# Build for markturn

cmake_minimum_required(VERSION 3.18.4)

project(markturn VERSION "1.0.0")

option(LINK_STATIC "Build marktrun with static library" OFF)

# Finds all necessary libraries or sets if find is not possible
find_package(CURL REQUIRED)
find_package(LibXml2 REQUIRED)

# Finds the include path to yaml and json cmake does not get the 
# full path though, so it's necessary to add the rest of the path
find_path(YAMLCPP_DIR NAMES "yaml-cpp/yaml.h" REQUIRED)
set(YAMLCPP_DIR "${YAMLCPP_DIR}/yaml-cpp/")

# @NOTE Why isn't "determines", "Determines"
# determines whether program is building on fedora or ubuntu 
execute_process (
    COMMAND bash -c "head -n 1 /etc/os-release"
    OUTPUT_VARIABLE UNIX_TYPE
)
# @NOTE No need here for the enclosing ${...}
if(${UNIX_TYPE} STREQUAL "NAME=Fedora\n")
    find_path(JSONCPP_DIR NAMES "json/json.h" REQUIRED)
else()
    find_path(JSONCPP_DIR NAMES "jsoncpp/json/json.h" REQUIRED)
endif()

# Adds the rest of the needed json path
set(JSONCPP_DIR "${JSONCPP_DIR}/jsoncpp/")

# Displays where each include library is installed
message("-- curl library: " ${CURL_INCLUDE_DIR})
message("-- libxml2 library: " ${LIBXML2_INCLUDE_DIR})
message("-- yaml library: " ${YAMLCPP_DIR})
message("-- json library: " ${JSONCPP_DIR})
set(LIBRARY_INCLUDE_DIRS ${LIBXML2_INCLUDE_DIR} ${YAMLCPP_DIR} ${JSONCPP_DIR})

# Gets all the cpp files for libmarkturn
file(GLOB LIBRARY_SOURCES "src/*Format.cpp" "src/libmarkturn.cpp")

# Build markturn
set(MARKTURN_SOURCE "src/markturn.cpp")
add_executable(markturn ${MARKTURN_SOURCE})
if(LINK_STATIC)
    message("-- Static library")
    target_link_libraries(markturn PUBLIC markturn_static)
else()
    message("-- Shared library")
    target_link_libraries(markturn PUBLIC markturn_shared)
endif()
install(TARGETS markturn RUNTIME)

# Create object library of all formats
add_library(libmarkturnobj OBJECT ${LIBRARY_SOURCES})
target_include_directories(libmarkturnobj PUBLIC ${LIBRARY_INCLUDE_DIRS})

# Static markturn Library
add_library(markturn_static STATIC $<TARGET_OBJECTS:libmarkturnobj>)
target_link_libraries(markturn_static PUBLIC ${LIBXML2_LIBRARY})
set_target_properties(markturn_static PROPERTIES OUTPUT_NAME markturn)
install(TARGETS markturn_static LIBRARY)

# Shared markturn library
add_library(markturn_shared SHARED $<TARGET_OBJECTS:libmarkturnobj>)
target_link_libraries(markturn_shared PUBLIC ${LIBXML2_LIBRARY})
set_target_properties(markturn_shared PROPERTIES OUTPUT_NAME markturn)
install(TARGETS markturn_shared LIBRARY)

# Creates test programs for markturn
enable_testing()
file(GLOB TEST_SOURCES "test/*.cpp")
foreach( TEST_FILE ${TEST_SOURCES})
    get_filename_component(TEST_NAME ${TEST_FILE} NAME_WLE)

    # Creates an executable test program from test sources
    add_executable(${TEST_NAME} ${TEST_FILE})
    target_include_directories(${TEST_NAME} PUBLIC ${LIBXML2_INCLUDE_DIR})
    target_include_directories(${TEST_NAME} PUBLIC src)
    target_link_libraries(${TEST_NAME} PUBLIC ${LIBXML2_LIBRARY})
    add_test(NAME ${TEST_NAME} COMMAND ${TEST_NAME})

    message("-- Added test ${TEST_NAME} from program ${TEST_FILE}")
endforeach()

# Lets cmake generate simple markturn packages
include(CPack)
