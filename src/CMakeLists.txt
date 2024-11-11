set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

find_package(Cython REQUIRED)

set(cython_module re2)

set(re2_include_dir "${PROJECT_SOURCE_DIR}/src")
set(cython_output "${CMAKE_CURRENT_SOURCE_DIR}/${cython_module}.cpp")
set(cython_src ${cython_module}.pyx)
# Track cython sources
file(GLOB cy_srcs *.pyx *.pxi *.h)

# .pyx -> .cpp
add_custom_command(OUTPUT ${cython_output}
                   COMMAND ${Cython_EXECUTABLE}
                           -a -3
                           --fast-fail
                           --cplus -I ${re2_include_dir}
                           --output-file ${cython_output} ${cython_src}
                   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                   DEPENDS ${cy_srcs}
                   COMMENT "Cythonizing extension ${cython_src}")

pybind11_add_module(${cython_module} MODULE ${cython_output})

target_compile_definitions(
    ${cython_module} PRIVATE VERSION_INFO=${SCM_VERSION_INFO}
)

# here we get to jump through some hoops to find libre2 on the manylinux
# docker CI images, etc
if(NOT MSVC)
  find_package(re2 CONFIG NAMES re2)
endif()

if(re2_FOUND)
  message(STATUS "System re2 found")
  target_link_libraries(${cython_module} PRIVATE re2::re2)
else()
  message(STATUS "Trying PkgConfig")
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(RE2 IMPORTED_TARGET re2)

  if(RE2_FOUND)
    include_directories(${RE2_INCLUDE_DIRS})
    target_link_libraries(${cython_module} PRIVATE PkgConfig::RE2)
  else()
    # last resort for manylinux: just try it
    message(STATUS "Blindly groping instead")
    link_directories("/usr/lib64" "/usr/lib")
    target_link_libraries(${cython_module} PRIVATE "libre2.so")
  endif()
endif()

if(APPLE)
  # macos/appleclang needs this
  target_link_libraries(${cython_module} PUBLIC pybind11::module)
  target_link_libraries(${cython_module} PUBLIC pybind11::python_link_helper)
endif()

if(MSVC)
  target_compile_options(${cython_module} PUBLIC /utf-8)
  target_link_libraries(${cython_module} PUBLIC ${Python_LIBRARIES})
  target_link_libraries(${cython_module} PUBLIC pybind11::windows_extras)
endif()
