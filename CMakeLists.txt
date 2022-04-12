# To use this cmake to compile and install a shared library, define BUILD_INSTALLABLE_LIB at configure time.
# If BUILD_INSTALLABLE_LIB is not defined, this cmake file is for compile tests and is not usable for your cmake app.
cmake_minimum_required(VERSION 3.18)
option(BUILD_INSTALLABLE_LIB "Build installable library" OFF)
if(BUILD_INSTALLABLE_LIB)
    set(TINY_EXR_TARGET tinyEXR)
    project(${TINY_EXR_TARGET} CXX)
    if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
        set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/installed"
            CACHE PATH "Install path prefix, prepended onto install directories." FORCE
            )
        set(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT False)
    endif()
    # Require strict C++11 mode(e.g. `-std=c++11`)
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_EXTENSIONS OFF)

    # use zlib or minizip if zlib is not found
    find_package(ZLIB QUIET)
    if(ZLIB_FOUND)
        message(STATUS "tinyEXR uses zlib")
        set(tinyEXR_USES_ZLIB True)
    else()
        message(STATUS "tinyEXR uses miniz")
        set(tinyEXR_USES_ZLIB False)
        enable_language(C)
        add_library(miniz STATIC deps/miniz/miniz.c)
        target_include_directories(miniz PRIVATE deps/miniz)
    endif()

    set(TINYEXR_SOURCES ${PROJECT_SOURCE_DIR}/tinyexr.cc)
    add_library(${TINY_EXR_TARGET} SHARED  ${TINYEXR_SOURCES} tinyexr.h)
    add_library(${TINY_EXR_TARGET}::${TINY_EXR_TARGET} ALIAS ${TINY_EXR_TARGET})
    target_compile_features(${TINY_EXR_TARGET} PUBLIC cxx_std_11)

    if(tinyEXR_USES_ZLIB)
        target_compile_definitions(${TINY_EXR_TARGET} PUBLIC TINYEXR_USE_MINIZ=0)
        target_include_directories(
            ${TINY_EXR_TARGET} PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
            $<INSTALL_INTERFACE:include>
        )
        target_link_libraries(${TINY_EXR_TARGET} PUBLIC ZLIB::ZLIB)
    else()
        target_compile_definitions(${TINY_EXR_TARGET} PUBLIC TINYEXR_USE_MINIZ=1)
        target_include_directories(
            ${TINY_EXR_TARGET}
            PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/deps/miniz>
            $<INSTALL_INTERFACE:include>
        )
        target_link_libraries(${TINY_EXR_TARGET} PRIVATE miniz)
    endif()

    # install header files, generate and install cmake config files for find_package()
    #
    include(GNUInstallDirs)
    include(CMakePackageConfigHelpers)
    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${TINY_EXR_TARGET}Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${TINY_EXR_TARGET}Config.cmake @ONLY
    )
    install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/tinyexr.h
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/tinyEXR
        )
    if(NOT tinyEXR_USES_ZLIB)
        install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/deps/miniz/miniz.h
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/tinyEXR
            )
    endif()
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${TINY_EXR_TARGET}Config.cmake
        DESTINATION ${CMAKE_INSTALL_DATADIR}/${TINY_EXR_TARGET}/cmake
        )
    export(TARGETS ${TINY_EXR_TARGET}
        FILE ${CMAKE_CURRENT_BINARY_DIR}/${TINY_EXR_TARGET}Targets.cmake
        )
    install(EXPORT ${TINY_EXR_TARGET}Targets NAMESPACE ${TINY_EXR_TARGET}::
        DESTINATION ${CMAKE_INSTALL_DATADIR}/${TINY_EXR_TARGET}/cmake
        )
    install(
        TARGETS ${TINY_EXR_TARGET}
        EXPORT ${TINY_EXR_TARGET}Targets
        INCLUDES
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/tinyEXR
    )

else()
    set(BUILD_TARGET "tinyexr")
    set(SAMPLE_TARGET "test_tinyexr")

    project(${BUILD_TARGET} CXX)

    # options
    option(TINYEXR_BUILD_SAMPLE "Build a sample" ON)
    option(TINYEXR_USE_MINIZ "Use miniz" ON)

    # cmake modules
    list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
    list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/sanitizers)
    find_package(Sanitizers) # Address sanitizer (-DSANITIZE_ADDRESS=ON)

    # Require strict C++11 mode(e.g. `-std=c++11`)
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_EXTENSIONS OFF)

    set_property(GLOBAL PROPERTY USE_FOLDERS ON)

    set(TINYEXR_SOURCES
        ${PROJECT_SOURCE_DIR}/tinyexr.cc
        )

    if(TINYEXR_USE_MINIZ)
      enable_language(C)
      add_library(miniz STATIC deps/miniz/miniz.c)
      target_include_directories(miniz PUBLIC deps/miniz)
      set_target_properties(miniz PROPERTIES FOLDER "deps")
      list(APPEND TINYEXR_EXT_LIBRARIES miniz)
    endif()

    add_library(${BUILD_TARGET} ${TINYEXR_SOURCES} ${TINYEXR_DEP_SOURCES})
    add_sanitizers(${BUILD_TARGET})

    target_include_directories(${BUILD_TARGET} PRIVATE ${PROJECT_SOURCE_DIR})
    target_link_libraries(${BUILD_TARGET} ${TINYEXR_EXT_LIBRARIES} ${CMAKE_DL_LIBS})

    # Increase warning level for clang.
    IF (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
      set_source_files_properties(${TINYEXR_SOURCES} PROPERTIES COMPILE_FLAGS "-Weverything -Werror -Wno-padded -Wno-c++98-compat-pedantic -Wno-documentation -Wno-unused-member-function")
    ENDIF ()

    if (TINYEXR_BUILD_SAMPLE)

      set(TINYEXR_SAMPLE_SOURCES
          ${PROJECT_SOURCE_DIR}/test_tinyexr.cc
          )

      add_executable(${SAMPLE_TARGET} ${TINYEXR_SAMPLE_SOURCES})
      target_link_libraries(${SAMPLE_TARGET} ${TINYEXR_EXT_LIBRARIES})
      add_sanitizers(${SAMPLE_TARGET})
      if (WIN32)
        target_compile_definitions(${SAMPLE_TARGET} PUBLIC UNICODE)
        target_compile_definitions(${SAMPLE_TARGET} PUBLIC _UNICODE)

        # Set ${SAMPLE_TARGET} as a startup project for VS IDE
        set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT ${SAMPLE_TARGET})

        # For easier debugging in VS IDE(cmake 3.8.0 or later required) Set working
        # directory to ${BUILD_TARGET} git repo root.
        if(CMAKE_VERSION VERSION_GREATER 3.8.0)
          set_target_properties(${SAMPLE_TARGET}
                  PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY
                  "${CMAKE_CURRENT_SOURCE_DIR}")
        endif()
      endif(WIN32)

    endif (TINYEXR_BUILD_SAMPLE)
endif()
