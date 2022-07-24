#######################################################################
# SWIG Embedding
#######################################################################
#
# SWIG generation is driven by the following SWIG macros:
#
#   - swig_add_module
#   - swig_add_library
#   - swig_link_libraries
#
# those macros are driven by CMAKE_SWIG_FLAGS, and SWIG_MODULE_XXXXXXX_EXTRA_DEPS
# where XXXXXXX is the name of the target object in swig_add_library(XXXXXXX ...)


find_package(Python3 REQUIRED COMPONENTS Interpreter Development NumPy)
if (BUILD_TESTING)
    find_package(Pytest)
endif()

# XXXX: this include_directories should be removed and transitive properties
#       of OpenMEEG::OpenMEEG and OpenMEEG::OpenMEEGMaths should be used instead

include_directories(${Python3_INCLUDE_DIRS}
                    ${Python3_NumPy_INCLUDE_DIRS}
                    ${CMAKE_CURRENT_SOURCE_DIR}
                    ${PROJECT_SOURCE_DIR}/OpenMEEG/include/
                    ${PROJECT_SOURCE_DIR}/OpenMEEGMaths/include/
)

cmake_policy(SET CMP0078 NEW)
set(SWIG_TARGET_NAME "openmeeg")
cmake_policy(SET CMP0086 NEW) # Use NEW policy as we do not use SWIG_MODULE_NAME

find_package(SWIG REQUIRED)

# load SWIG macros. The path to UseSWIG.cmake is provided by SWIG_USE_FILE which is set in FindSWIG

if (SWIG_USE_FILE STREQUAL "")
    message(FATAL_ERROR "unexpected: the variable SWIG_USE_FILE is the empty string. Did you run FindSWIG.cmake?")
else()
    include(${SWIG_USE_FILE})
endif()

list(APPEND CMAKE_SWIG_FLAGS -v -O)
set(SWIG_MODULE_openmeeg_EXTRA_DEPS openmeeg/numpy.i ${SWIG_MODULE_openmeeg_EXTRA_DEPS})
set_source_files_properties(openmeeg/openmeeg.i PROPERTIES CPLUSPLUS ON GENERATED_COMPILE_DEFINITIONS SWIG_PYTHON_SILENT_MEMLEAK)
# This puts openmeeg.py in openmeeg/
swig_add_library(${SWIG_TARGET_NAME} LANGUAGE python OUTPUT_DIR ${CMAKE_CURRENT_BINARY_DIR}/openmeeg SOURCES openmeeg/openmeeg.i)
# This puts _openmeeg.so in openmeeg/
set_target_properties(${SWIG_TARGET_NAME} PROPERTIES LIBRARY_OUTPUT_DIRECTORY openmeeg/)
if ( MSVC )
    set_target_properties(${SWIG_TARGET_NAME} PROPERTIES LIBRARY_OUTPUT_DIRECTORY_DEBUG openmeeg/)
    set_target_properties(${SWIG_TARGET_NAME} PROPERTIES LIBRARY_OUTPUT_DIRECTORY_RELEASE openmeeg/)
endif()

if (PYTHON_FORCE_EXT_SUFFIX)
    # For PyPy, see
    # https://github.com/conda-forge/openmeeg-feedstock/pull/20#issuecomment-1161729565
    if(NOT Python3_EXT_SUFFIX)
        if(UNIX)
            set(Python3_EXT_SUFFIX ".${Python3_SOABI}.so")
        else()
            set(Python3_EXT_SUFFIX ".${Python3_SOABI}.pyd")
        endif()
    endif()

    set_target_properties(openmeeg PROPERTIES SUFFIX "${Python3_EXT_SUFFIX}")
    message(STATUS "Python forced suffix ${Python3_EXT_SUFFIX}")
else()
    message(STATUS "Python default suffix ${Python3_EXT_SUFFIX}")
endif()

if (APPLE)
    swig_link_libraries(openmeeg OpenMEEG::OpenMEEG)
else()
    swig_link_libraries(openmeeg ${Python3_LIBRARIES} OpenMEEG::OpenMEEG)
endif()

set(PYTHON_SITE_ARCH ${Python3_SITEARCH})
if (PYTHON_INSTALL_RELATIVE)
    string(REGEX MATCH "[Ll]ib.*packages" PYTHON_SITE_ARCH ${PYTHON_SITE_ARCH})
endif()
file(TO_CMAKE_PATH "${PYTHON_SITE_ARCH}" PYTHON_SITE_ARCH)
string(REGEX REPLACE "\n$" "" PYTHON_SITE_ARCH "${PYTHON_SITE_ARCH}")
message(STATUS "Python installation directory ${PYTHON_SITE_ARCH}")

# Copy our .py files over, dealing with Windows wildcard issues
add_custom_command(
    TARGET ${SWIG_TARGET_NAME} PRE_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/wrapping/python/openmeeg ${CMAKE_CURRENT_BINARY_DIR}/openmeeg
    COMMENT "Python: Copying ${CMAKE_SOURCE_DIR}/wrapping/python/openmeeg to ${CMAKE_CURRENT_BINARY_DIR}/openmeeg"
    VERBATIM
)
add_custom_command(
    TARGET ${SWIG_TARGET_NAME} PRE_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy
    ${CMAKE_SOURCE_DIR}/wrapping/python/MANIFEST.in
    ${CMAKE_SOURCE_DIR}/wrapping/python/setup.py
    ${CMAKE_SOURCE_DIR}/wrapping/python/README.rst
    ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Python: Copying MANIFEST.in, setup.py, and README.rst to ${CMAKE_CURRENT_BINARY_DIR}"
    VERBATIM
)

# Installation just copies the directory
install(DIRECTORY  ${OpenMEEG_BINARY_DIR}/wrapping/python/openmeeg
        DESTINATION ${PYTHON_SITE_ARCH}
        DIRECTORY_PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_READ GROUP_EXECUTE WORLD_READ WORLD_EXECUTE)

if (APPLE)
    set_target_properties(${SWIG_TARGET_NAME} PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
endif()
if (PYTHON_COPY_RUNTIME_DLLS)
    add_custom_command(TARGET ${SWIG_TARGET_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_RUNTIME_DLLS:${SWIG_TARGET_NAME}> ${CMAKE_BINARY_DIR}
        COMMAND_EXPAND_LISTS
        COMMENT "Python: Copying runtime DLLs for ${SWIG_TARGET_NAME}"
        VERBATIM
    )
    add_custom_command(TARGET ${SWIG_TARGET_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_RUNTIME_DLLS:OpenMEEG> ${CMAKE_BINARY_DIR}
        COMMAND_EXPAND_LISTS
        COMMENT "Python: Copying runtime DLLs for OpenMEEG"
        VERBATIM
    )
    add_custom_command(TARGET ${SWIG_TARGET_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_RUNTIME_DLLS:OpenMEEGMaths> ${CMAKE_BINARY_DIR}
        COMMAND_EXPAND_LISTS
        COMMENT "Python: Copying runtime DLLs for OpenMEEGMaths"
        VERBATIM
    )
endif()

# Test Python wrapping

if (BUILD_TESTING)
    set(PYTHON_TEST_FILES test_vector.py test_matrix.py test_sensors.py test_geom_import.py test_python.py test_python2.py test_mesh.py test_make_geometry.py)
    foreach (test ${PYTHON_TEST_FILES})
        add_test(openmeeg_python_${test} ${Python3_EXECUTABLE} -m pytest ${OpenMEEG_BINARY_DIR}/wrapping/python/openmeeg/tests/${test})
        if (USE_MKL AND BUILD_SHARED_LIBS AND NOT MKL_USE_sdl)
            # due to this bug in the MKL
            message(WARNING "Due to a bug in the MKL when used from python: see https://software.intel.com/en-us/forums/intel-distribution-for-python/topic/628976 "
                "workaround: set environment variable:\nexport LD_PRELOAD=\"${MKL_ROOT_DIR}/lib/intel64/libmkl_core.so:"
                "${MKL_ROOT_DIR}/lib/intel64/libmkl_sequential.so\"")
            set_tests_properties(openmeeg_python_${test} PROPERTIES ENVIRONMENT LD_PRELOAD=${MKL_ROOT_DIR}/lib/intel64/libmkl_core.so:${MKL_ROOT_DIR}/lib/intel64/libmkl_sequential.so)
        endif()
        set_tests_properties(openmeeg_python_${test} PROPERTIES ENVIRONMENT "PYTHONPATH=${OpenMEEG_BINARY_DIR}/wrapping/python OPENMEEG_DATA_PATH=${OpenMEEG_SOURCE_DIR}/data")
    endforeach()

    if (BUILD_REFERENCE_DOC)
        add_test(openmeeg_python_doc ${Python3_EXECUTABLE} ${OpenMEEG_BINARY_DIR}/wrapping/python/openmeeg/tests/test_doc.py)
        set_tests_properties(openmeeg_python_doc PROPERTIES ENVIRONMENT PYTHONPATH=${OpenMEEG_BINARY_DIR}/wrapping/python)
    endif()

endif()