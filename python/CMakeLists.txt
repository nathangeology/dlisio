project(dlisio-python)

if(NOT BUILD_PYTHON)
    return()
endif()

find_package(PythonInterp REQUIRED)

if(NOT PYTHON_EXECUTABLE)
    message(WARNING "Could not find python - skipping python bindings")
    message(WARNING "Select specific python distribution with "
                    "-DPYTHON_EXECUTABLE=bin/python"
    )
    return()
endif()

set(python ${PYTHON_EXECUTABLE})
set(setup.py ${CMAKE_CURRENT_SOURCE_DIR}/setup.py)

if (NOT WIN32)
    # setuptools on microsoft compilers doesn't support the --library-dir or
    # --build-dir flag and crashes, so only pass it on non-microsoft platforms
    set(build_ext build_ext --library-dirs ${CMAKE_CURRENT_SOURCE_DIR}
                            --rpath ${CMAKE_CURRENT_SOURCE_DIR}
    )
endif()

add_custom_target(
    dlisio-python ALL
    COMMENT "Building python library with setup.py"
    SOURCES ${setup.py}
    DEPENDS ${setup.py}
    VERBATIM
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}

    # setuptools on windows breaks spectacularly when the library isn't
    # available in the same directory, and build_ext --library-dirs is not
    # support on msvc is not supported, so we must copy out the libsegyio core
    # object and put it here
    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:dlisio>
                                     $<TARGET_FILE_NAME:dlisio>

    COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_LINKER_FILE:dlisio>
                                     $<TARGET_LINKER_FILE_NAME:dlisio>

    COMMAND ${python} ${setup.py} ${build_ext} build
)

add_dependencies(dlisio-python dlisio)

install(CODE "
if (DEFINED ENV{DESTDIR})
    get_filename_component(abs-destdir \"\$ENV{DESTDIR}\" ABSOLUTE)
    set(root_destdir --root=\${abs-destdir})
endif()

execute_process(COMMAND
    ${python} ${setup.py}
        install --prefix=${CMAKE_INSTALL_PREFIX}
                --single-version-externally-managed
                --record installed-files
                \${root_destdir}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)")



if(NOT BUILD_TESTING)
    return()
endif()

# run tests with setup.py test
# this is very slow compared to invoking pytest directly, but setuptools will
# copy the built extension into the tree as it sees fit
add_test(NAME python.unit
         COMMAND ${python} ${setup.py} test
         WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
)
