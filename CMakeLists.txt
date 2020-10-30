cmake_minimum_required(VERSION 3.9)

project(quickjs C)

add_definitions(-D_CRT_SECURE_NO_WARNINGS)
add_definitions(-DJS_MAN_BOXING)
add_definitions(-DCONFIG_VERSION="2019-09-05")
if(CMAKE_C_COMPILER_ID MATCHES "Clang|GNU")

    add_compile_options(-Wall)
    add_compile_options(-MMD)
    add_compile_options(-MF)
    add_compile_options(-Wno-array-bounds)
    add_compile_options(-Wno-format-truncation)
    add_definitions(-D_GNU_SOURCE)

endif()
if(CMAKE_C_COMPILER_ID MATCHES "MSVC")
    add_compile_options(/wd4244)
    add_compile_options(/wd4018)
    add_compile_options(/wd4146)
    add_compile_options(/wd4267)
endif()

option(CONFIG_BIGNUM "BigNum support" ON)
option(CONFIG_LTO "Enable LTO" ON)
option(CONFIG_TESTS "Run tests" ON)

set(QJS_LIB_OBJS
    cutils.h
    cutils.c
    libregexp.c
    libunicode.c
    quickjs.c
    quickjs-libc.c
    libregexp.h
    libregexp-opcode.h
    libunicode.h
    libunicode-table.h
    list.h
    quickjs.h
    quickjs-atom.h
    quickjs-libc.h
    quickjs-opcode.h
)

if(${CONFIG_BIGNUM})
    add_definitions(-DCONFIG_BIGNUM)
    set(QJS_LIB_OBJS ${QJS_LIB_OBJS} libbf.c)
endif()

add_library(quickjs STATIC ${QJS_LIB_OBJS})
if(CONFIG_LTO)
    set_property(TARGET quickjs PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
endif()

##########################

project(qjsc C)
add_executable(qjsc qjsc.c)
target_link_libraries(qjsc PRIVATE quickjs)
target_link_libraries(qjsc PRIVATE Threads::Threads)

if(CONFIG_LTO)
    set_property(TARGET qjsc PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
endif()

if(UNIX)
target_link_libraries(qjsc PRIVATE dl m)
endif()


##########################

add_custom_command(
    OUTPUT ${CMAKE_SOURCE_DIR}/repl.c
    COMMAND $<TARGET_FILE:qjsc> -c -o repl.c -m repl.js
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
)

add_custom_command(
    OUTPUT ${CMAKE_SOURCE_DIR}/qjscalc.c
    COMMAND $<TARGET_FILE:qjsc> -fbignum -c -o qjscalc.c qjscalc.js
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
)

ADD_CUSTOM_TARGET(generate_files DEPENDS repl.c qjscalc.c)
add_dependencies(generate_files qjsc)
##########################

project(qjs C)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

set(QJS_OBJS
    qjs.c
    repl.c
)

if(${CONFIG_BIGNUM})
    set(QJS_OBJS ${QJS_OBJS} qjscalc.c)
endif()

add_executable(qjs ${QJS_OBJS})
add_dependencies(qjs generate_files)
target_link_libraries(qjs PRIVATE quickjs)
target_link_libraries(qjs PRIVATE Threads::Threads)
if(UNIX)
target_link_libraries(qjs PRIVATE dl m)
endif()

if(CONFIG_LTO)
    set_property(TARGET qjs PROPERTY INTERPROCEDURAL_OPTIMIZATION True)
endif()

##########################
if(CONFIG_TESTS)
    enable_testing()
    if(CONFIG_BIGNUM)
        set(TEST_ARGS --bignum --qjscalc)
    endif()

    file(GLOB TESTS ${CMAKE_SOURCE_DIR}/tests/test_*.js)
    
    foreach(TEST IN ITEMS ${TESTS})
        string(REGEX MATCH "(test_[a-z]+\)\.js$" TEST_NAME ${TEST})
        if(TEST_NAME)
            add_test(NAME ${CMAKE_MATCH_1}
                    COMMAND $<TARGET_FILE:qjs> ${TEST_ARGS} ${TEST}
                    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/tests)
            set_tests_properties(${CMAKE_MATCH_1} PROPERTIES DEPENDS qjs)
        endif()
    endforeach()

    project(bjson)
    add_library(bjson SHARED tests/bjson.c)
    target_link_libraries(bjson quickjs)
    target_compile_definitions(bjson PRIVATE -DJS_SHARED_LIBRARY)
    set_target_properties(bjson PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/tests)
    set_target_properties(bjson PROPERTIES PREFIX "")
endif()