cmake_minimum_required(VERSION 3.0.0)
project(cppe-core VERSION 0.3.1 LANGUAGES CXX)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)
include(ExternalProject)

if(ENABLE_OPENMP)
    find_package(OpenMP REQUIRED)
    if (OPENMP_FOUND)
        set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
        set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
    endif()
endif()

list(APPEND CPPE_SOURCES
  metadata.cc
  utils/string_utils.cc
  core/potfile_reader.cc
  core/pot_manipulation.cc
  core/cppe_state.cc
  core/multipole_expansion.cc
  core/electric_fields.cc
  core/math.cc
  core/bmatrix.cc
  # tensors
  core/tensors/tensors_autogen.cc
  core/tensors/tensors_recursive.cc
  # fmm
  core/fmm/calculate.cc
  core/fmm/tree.cc
  core/fmm/utils.cc
  core/fmm/field_m0.cc
  core/fmm/field_m1.cc
  core/fmm/field_m2.cc
)

add_library(cppe
  ${CPPE_SOURCES}
)


target_include_directories(cppe PUBLIC ${EIGEN3_INCLUDE_DIRS})

if(${ENABLE_PYTHON_INTERFACE})
    message(STATUS "Building Python interface for cppe.")

    if(MSVC)
      set(PYBIND11_CPP_STANDARD "/std:c++${CMAKE_CXX_STANDARD}")
    else()
      set(PYBIND11_CPP_STANDARD "-std=c++${CMAKE_CXX_STANDARD}")
    endif()
    find_package(pybind11 CONFIG REQUIRED)
    message(STATUS "${Cyan}Using pybind11${ColourReset}: ${pybind11_INCLUDE_DIR} (version ${pybind11_VERSION} for Py${PYTHON_VERSION_STRING} and ${PYBIND11_CPP_STANDARD})")
    message(STATUS "${Cyan}Using Python ${PYTHON_VERSION_MAJOR}.${PYTHON_VERSION_MINOR}${ColourReset}: ${PYTHON_EXECUTABLE}")

    pybind11_add_module(pycppe SHARED NO_EXTRAS
        ${CPPE_SOURCES}
        python_iface/export_molecule.cc
        python_iface/export_potential.cc
        python_iface/export_fields.cc
        python_iface/export_utils.cc
        python_iface/export_state.cc
        python_iface/export_math.cc
        python_iface/export_tensors.cc
        python_iface/export_fmm.cc
        python_iface/export_cppe.cc
    )
    target_include_directories(pycppe PUBLIC ${EIGEN3_INCLUDE_DIRS})
    set_target_properties(pycppe PROPERTIES PREFIX "${PYTHON_MODULE_PREFIX}"
                                            OUTPUT_NAME cppe
                                            EXPORT_NAME cppe
                                            SUFFIX "${PYTHON_MODULE_EXTENSION}")
endif()

set(PN "cppe")
#### Installation ####
if (INSTALL_DEVEL_HEADERS)
    install(DIRECTORY "core"
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PN}
            FILES_MATCHING PATTERN "*.hh")
    install(DIRECTORY "utils"
            DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PN}
            FILES_MATCHING PATTERN "*.hh")
endif()
install(FILES libcppe.hh
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PN})

install(TARGETS cppe
        EXPORT "${PN}Targets"
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})

if(${ENABLE_PYTHON_INTERFACE})
    install(TARGETS pycppe
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}${PYMOD_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_LIBDIR}${PYMOD_INSTALL_LIBDIR})
    
    # install the Python tests
    install(DIRECTORY "../tests"
        DESTINATION ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}${PYMOD_INSTALL_LIBDIR}/cppe
        FILES_MATCHING PATTERN "*.py" PATTERN "*.pot" PATTERN "*.hdf5")
endif()

target_compile_definitions(cppe INTERFACE USING_${PN})
target_include_directories(cppe INTERFACE $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>)

set(CMAKECONFIG_INSTALL_DIR "share/cmake/${PN}")
configure_package_config_file(cmake/${PN}Config.cmake.in
                         "${CMAKE_CURRENT_BINARY_DIR}/${PN}Config.cmake"
                         INSTALL_DESTINATION ${CMAKECONFIG_INSTALL_DIR})
write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/${PN}ConfigVersion.cmake
                            VERSION ${${PN}_VERSION}
                            COMPATIBILITY AnyNewerVersion)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PN}Config.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/${PN}ConfigVersion.cmake
        DESTINATION ${CMAKECONFIG_INSTALL_DIR})
install(EXPORT "${PN}Targets"
        NAMESPACE "${PN}::"
        DESTINATION ${CMAKECONFIG_INSTALL_DIR})
