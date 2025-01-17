cmake_minimum_required (VERSION 3.12)
# find_package(Python3) need version >= 3.12

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# compile for wasm 
# emcmake cmake .. -DBUILD_WASM=ON 
# compile for python
# cmake .. -DBUILD_PYTHON=ON -DPYTHON_LIBRARY="C:\\python3\\libs\\Python36.lib" -DPYTHON_INCLUDE_DIR="C:\\python3\\include"

project (asyncflow)
set(PLATFORM x64)

option (BUILD_PYTHON "build python version" OFF)
option (BUILD_WASM "build WASM version" OFF)
option (BUILD_LUAJIT "build luajit version" OFF)
option (BUILD_TEST   "build test project" OFF)

if(${BUILD_WASM})
    add_definitions(-DBUILD_WASM)
endif(${BUILD_WASM})

if(${BUILD_TEST})
    if(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
        # add gcov compile flags
        message(STATUS "build with gcov flags")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -fPIC -fprofile-arcs -ftest-coverage -fsanitize=address") 
        set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} -g -pthread -fPIC -fprofile-arcs -ftest-coverage -fsanitize=address")
    endif()
endif(${BUILD_TEST})

if(FLOWCHART_DEBUG)
    # asio
    add_library(asio INTERFACE)
    target_include_directories(asio INTERFACE "thirdparty/asio-1.12.1/include")
    target_compile_definitions(asio INTERFACE ASIO_STANDALONE)

    # websocketpp
    add_library(websocketpp INTERFACE)
    target_include_directories(websocketpp INTERFACE "thirdparty")
    target_compile_definitions (websocketpp INTERFACE _WEBSOCKETPP_CPP11_STRICT_)
    if(MSVC)
    target_compile_options(websocketpp INTERFACE /wd4996 /wd4995 /wd4355)
    endif(MSVC)

    # build debug
    add_definitions(-DFLOWCHART_DEBUG -D_SINGLE_THREAD_MODE)
endif(FLOWCHART_DEBUG)

# msgpack
add_library(msgpack INTERFACE)
target_include_directories(msgpack INTERFACE "thirdparty/msgpack-c-cpp-3.0.1/include")

# rapidjson
add_library(rapidjson INTERFACE)
target_include_directories(rapidjson INTERFACE "thirdparty")

# python
if(${BUILD_PYTHON})
	find_package (Python3 COMPONENTS Development)	
	if(NOT ${Python3_FOUND})
		message(FATAL_ERROR "cannot find python")
	endif(NOT ${Python3_FOUND})

	add_definitions(-DUSING_PYTHON)
endif(${BUILD_PYTHON})

if (CMAKE_SYSTEM_NAME MATCHES "Linux")
	add_compile_options(-fPIC)
else()
	add_definitions(-D_WIN32_WINNT=0x0601)
endif (CMAKE_SYSTEM_NAME MATCHES "Linux")



# source files
aux_source_directory("src/core" CORE_SRCS)
aux_source_directory("src/export" EXPORT_SRCS)
aux_source_directory("src/debug" DEBUG_SRCS)
aux_source_directory("src/util" UTIL_SRCS)
aux_source_directory("src/js" JS_SRCS)

# set source folder in visual studio project
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
#source_group(include FILES ${HEADERS})
source_group(core FILES ${CORE_SRCS})
source_group(export FILES ${EXPORT_SRCS})
source_group(debug FILES ${DEBUG_SRCS})
source_group(util FILES ${UTIL_SRCS})
source_group(js FILES ${JS_SRCS})

# header files
file(GLOB HEADERS "include/*.h")
source_group(include ${HEADERS})

file(GLOB CORE_HEADERS "include/core/*.h")
source_group(include\\core FILES ${CORE_HEADERS})
list(APPEND HEADERS ${CORE_HEADERS})

file(GLOB UTIL_HEADERS "include/util/*.h")
source_group(include\\util FILES ${UTIL_HEADERS})
list(APPEND HEADERS ${UTIL_HEADERS})

# asyncflow source files
add_library (asyncflow STATIC ${CORE_SRCS} ${DEBUG_SRCS} ${EXPORT_SRCS} ${UTIL_SRCS}  ${HEADERS})
target_include_directories(asyncflow PUBLIC "include")
target_compile_features(asyncflow PUBLIC cxx_std_11)
set_target_properties(asyncflow PROPERTIES CXX_EXTENSIONS OFF)
if(FLOWCHART_DEBUG)
target_link_libraries(asyncflow asio websocketpp msgpack rapidjson)
else(FLOWCHART_DEBUG)
target_link_libraries(asyncflow msgpack rapidjson)
endif(FLOWCHART_DEBUG)

if(${BUILD_WASM})
    aux_source_directory("src/js" JS_SRCS)
    add_executable (asyncflow-js ${JS_SRCS})
    target_include_directories(asyncflow-js PUBLIC "include")
    target_compile_features(asyncflow-js PUBLIC cxx_std_11)
    set_target_properties(asyncflow-js PROPERTIES CXX_EXTENSIONS OFF)
    if(FLOWCHART_DEBUG)
        target_link_libraries(asyncflow-js asyncflow msgpack rapidjson)
    else(FLOWCHART_DEBUG)
        target_link_libraries(asyncflow-js asyncflow msgpack rapidjson)
    endif(FLOWCHART_DEBUG)

    message(STATUS "Setting compilation target to WASM")
    #set(CMAKE_EXECUTABLE_SUFFIX ".wasm.js")
    set_target_properties(asyncflow-js PROPERTIES LINK_FLAGS "-s WASM=1 -s EXTRA_EXPORTED_RUNTIME_METHODS='[ccall, cwrap]' -s --js-library ${CMAKE_SOURCE_DIR}/src/js/pkg.js")
endif(${BUILD_WASM})

if(${BUILD_PYTHON})	
# todo add different libs to different config, should test if PYTHON_DEBUG_LIBRARIES exist
# target_link_libraries(asyncflow debug ${PYTHON_DEBUG_LIBRARIES} optimized ${PYTHON_LIBRARIES})
    aux_source_directory("src/py" PY_SRCS)
    add_library (asyncflow-py SHARED ${PY_SRCS})
    target_compile_features(asyncflow-py PUBLIC cxx_std_11)
    set_target_properties(asyncflow-py PROPERTIES CXX_EXTENSIONS OFF)
    target_link_libraries(asyncflow-py asyncflow Python3::Python msgpack rapidjson)
    if(MSVC)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MD")
    	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MDd")   # cmake > 3.15 => MSVC_RUNTIME_LIBRARY
    endif (MSVC)  
endif(${BUILD_PYTHON})

if(${BUILD_LUAJIT})    
    set(LUAJIT_INCLUDE_PATH CACHE PATH "luajit header files folder")
    set(LUAJIT_LIB CACHE FILEPATH "fullpath of luajit lib")    

    # add luajit
    add_library(luajit STATIC IMPORTED)
    set_target_properties(luajit PROPERTIES IMPORTED_LOCATION "${LUAJIT_LIB}"
    INTERFACE_INCLUDE_DIRECTORIES "${LUAJIT_INCLUDE_PATH}")    

    aux_source_directory("src/lua" LUA_SRCS)
    add_library (asyncflow-lua SHARED ${LUA_SRCS})
    target_compile_features(asyncflow-lua PUBLIC cxx_std_11)
    set_target_properties(asyncflow-lua PROPERTIES CXX_EXTENSIONS OFF)
    target_link_libraries(asyncflow-lua asyncflow luajit msgpack rapidjson)

if(${BUILD_TEST})
    aux_source_directory("src/test" LUA_TEST_SRCS)    
    add_executable(lua_test ${LUA_TEST_SRCS} ${LUA_SRCS})
    target_compile_features(lua_test PUBLIC cxx_std_11)
    target_include_directories(lua_test PUBLIC "src/lua")
    set_target_properties(lua_test PROPERTIES CXX_EXTENSIONS OFF)
    target_link_options(lua_test PRIVATE "-Wl,-rpath=${LUAJIT_INCLUDE_PATH}")
    target_link_options(lua_test PRIVATE "-fsanitize=address")
    target_link_libraries(lua_test asyncflow luajit msgpack rapidjson ${CMAKE_DL_LIBS})
endif(${BUILD_TEST})


endif(${BUILD_LUAJIT})

add_custom_target("distclean"
        COMMAND make clean
        COMMAND rm CMakeCache.txt Makefile cmake_install.cmake
        COMMAND rm -rf CMakeFiles)
