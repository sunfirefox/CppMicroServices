project(CppMicroServicesExamples)

cmake_minimum_required(VERSION 2.8)

find_package(CppMicroServices NO_MODULE REQUIRED)

include_directories(${CppMicroServices_INCLUDE_DIRS})

#-----------------------------------------------------------------------------
# Set C/CXX flags
#-----------------------------------------------------------------------------

if(${CMAKE_PROJECT_NAME} STREQUAL ${PROJECT_NAME})
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CppMicroServices_CXX_FLAGS}")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} ${CppMicroServices_CXX_FLAGS_RELEASE}")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${CppMicroServices_CXX_FLAGS_DEBUG}")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CppMicroServices_C_FLAGS}")
  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${CppMicroServices_C_FLAGS_RELEASE}")
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${CppMicroServices_C_FLAGS_DEBUG}")
endif()

#-----------------------------------------------------------------------------
# Init output directories
#-----------------------------------------------------------------------------

set(CppMicroServicesExamples_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")
set(CppMicroServicesExamples_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")
set(CppMicroServicesExamples_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")

foreach(_type ARCHIVE LIBRARY RUNTIME)
  if(NOT CMAKE_${_type}_OUTPUT_DIRECTORY)
    set(CMAKE_${_type}_OUTPUT_DIRECTORY ${CppMicroServicesExamples_${_type}_OUTPUT_DIRECTORY})
  endif()
endforeach()


function(CreateExample _name)
  add_library(Example-${_name} SHARED ${ARGN})
  if(${_name}_DEPENDS)
    foreach(_dep ${${_name}_DEPENDS})
      include_directories(${CppMicroServicesExamples_SOURCE_DIR}/${_dep})
      target_link_libraries(Example-${_name} Example-${_dep})
    endforeach()
  endif()
  target_link_libraries(Example-${_name} ${CppMicroServices_LIBRARIES})
  set_target_properties(Example-${_name} PROPERTIES
    LABELS Examples
    OUTPUT_NAME ${_name}
  )
endfunction()

add_subdirectory(eventlistener)
add_subdirectory(dictionaryservice)
add_subdirectory(frenchdictionary)
add_subdirectory(dictionaryclient)
add_subdirectory(dictionaryclient2)
add_subdirectory(dictionaryclient3)
add_subdirectory(spellcheckservice)
add_subdirectory(spellcheckclient)
add_subdirectory(driver)

#-----------------------------------------------------------------------------
# Test if examples compile against an install tree and if the
# Makefile example compiles
#-----------------------------------------------------------------------------

if(US_BUILD_TESTING)
  enable_testing()

  set(_example_tests )

  set(_install_dir "${CppMicroServices_BINARY_DIR}/install_test/${CMAKE_INSTALL_PREFIX}")

  add_test(NAME usInstallCleanTest
           COMMAND ${CMAKE_COMMAND} -E remove_directory "${_install_dir}")

  add_test(NAME usInstallTest
           WORKING_DIRECTORY ${CppMicroServices_BINARY_DIR}
           COMMAND ${CMAKE_COMMAND} --build ${CppMicroServices_BINARY_DIR} --target install)
  set_tests_properties(usInstallTest PROPERTIES
                       ENVIRONMENT "DESTDIR=${CppMicroServices_BINARY_DIR}/install_test"
                       DEPENDS usInstallCleanTest)

  set(_examples_binary_dir "${CppMicroServices_BINARY_DIR}/examples_build")

  add_test(NAME usExamplesCleanTest
           COMMAND ${CMAKE_COMMAND} -E remove_directory "${_examples_binary_dir}")

  add_test(NAME usExamplesCreateDirTest
           COMMAND ${CMAKE_COMMAND} -E make_directory "${_examples_binary_dir}")
  set_tests_properties(usExamplesCreateDirTest PROPERTIES
                       DEPENDS usExamplesCleanTest)

  add_test(NAME usExamplesConfigureTest
           WORKING_DIRECTORY ${_examples_binary_dir}
           COMMAND ${CMAKE_COMMAND}
                   -G ${CMAKE_GENERATOR}
                   -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
                   "-DCppMicroServices_DIR:PATH=${_install_dir}/${CONFIG_CMAKE_DIR}"
                   "${CMAKE_CURRENT_LIST_DIR}")
  set_tests_properties(usExamplesConfigureTest PROPERTIES
                       DEPENDS "usInstallTest;usExamplesCreateDirTest")

  add_test(NAME usExamplesBuildTest
           WORKING_DIRECTORY ${_examples_binary_dir}
           COMMAND ${CMAKE_COMMAND} --build .)
  set_tests_properties(usExamplesBuildTest PROPERTIES
                       DEPENDS usExamplesConfigureTest)

  list(APPEND _example_tests usInstallCleanTest usInstallTest usExamplesCleanTest
       usExamplesCreateDirTest usExamplesConfigureTest usExamplesBuildTest)

  # The makefile is Linux specific, so only try to build the Makefile example
  # if we are on a proper system
  if(UNIX AND NOT APPLE)
    find_program(MAKE_COMMAND NAMES make gmake)
    find_program(CXX_COMMAND NAMES g++)
    mark_as_advanced(MAKE_COMMAND CXX_COMMAND)
    if(MAKE_COMMAND AND CXX_COMMAND)
      add_test(NAME usMakefileExampleCleanTest
               WORKING_DIRECTORY ${CppMicroServices_SOURCE_DIR}/examples/makefile
               COMMAND ${MAKE_COMMAND} clean)
      add_test(NAME usMakefileExampleTest
               WORKING_DIRECTORY ${CppMicroServices_SOURCE_DIR}/examples/makefile
               COMMAND ${MAKE_COMMAND})
      set_tests_properties(usMakefileExampleTest PROPERTIES
                           DEPENDS "usMakefileExampleCleanTest;usInstallTest"
                           ENVIRONMENT "CppMicroServices_ROOT=${CppMicroServices_BINARY_DIR}/install_test${CMAKE_INSTALL_PREFIX};CppMicroServices_CXX_FLAGS=${CppMicroServices_CXX_FLAGS}")
      list(APPEND _example_tests usMakefileExampleCleanTest usMakefileExampleTest)
    endif()
  endif()

  if(US_TEST_LABELS)
    set_tests_properties(${_example_tests} PROPERTIES LABELS "${US_TEST_LABELS}")
  endif()

endif()
