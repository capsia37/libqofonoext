cmake_minimum_required(VERSION 3.12.0 FATAL_ERROR)
project(qofonoextdeclarative LANGUAGES CXX)

include(GNUInstallDirs)

set(PLUGIN_SOURCES
    qofonoextdeclarativeplugin.cpp
    qofonoextmodemlistmodel.cpp
    qofonoextsimlistmodel.cpp
)

add_library(qofonoextdeclarative SHARED ${PLUGIN_SOURCES})

target_include_directories(qofonoextdeclarative PRIVATE
    ${CMAKE_SOURCE_DIR}/src ${QOFONO_QT_INCLUDE_DIRS}
)

target_link_libraries(qofonoextdeclarative PRIVATE
    Qt5::Qml
    Qt5::DBus
    qofonoext
    ${QOFONO_QT_LIBRARIES}
)

install(TARGETS qofonoextdeclarative
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}/qt5/qml/org/nemomobile/ofono
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/org/nemomobile/ofono/qmldir
    ${CMAKE_CURRENT_BINARY_DIR}/org/nemomobile/ofono/plugins.qmltypes
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/qt5/qml/org/nemomobile/ofono
)

# Helper to replicate Qt6 qt_add_qml_module behavior for Qt5 builds.
# Generates a qmldir and plugins.qmltypes file for the given target.
function(qt5_add_qml_module TARGET)
    set(oneValueArgs URI VERSION)
    cmake_parse_arguments(QMLMOD "" "${oneValueArgs}" "" ${ARGN})

    if(NOT QMLMOD_URI)
        message(FATAL_ERROR "qt5_add_qml_module: URI is required")
    endif()

    get_target_property(_outname ${TARGET} OUTPUT_NAME)
    if(NOT _outname)
        set(_outname ${TARGET})
    endif()

    find_program(_QMLPLUGINDUMP_EXECUTABLE NAMES qmlplugindump-qt5 qmlplugindump)
    if(NOT _QMLPLUGINDUMP_EXECUTABLE)
    message(FATAL_ERROR "qmlplugindump not found")
    endif()
    
    set(_plugin_filename "${CMAKE_SHARED_LIBRARY_PREFIX}${_outname}${CMAKE_SHARED_LIBRARY_SUFFIX}")
    string(REPLACE "." "/" QMLMOD_URI_PATH "${QMLMOD_URI}")
    set(_qmldir_dir "${CMAKE_CURRENT_BINARY_DIR}/${QMLMOD_URI_PATH}")
    set(_qmldir_path "${_qmldir_dir}/qmldir")
    set(_qmltypes_out "${_qmldir_dir}/plugins.qmltypes")

    file(MAKE_DIRECTORY "${_qmldir_dir}")
    file(WRITE ${_qmldir_path} "module ${QMLMOD_URI}\n")
    file(APPEND ${_qmldir_path} "plugin ${_outname}\n")
    file(APPEND ${_qmldir_path} "typeinfo plugins.qmltypes\n")

    add_custom_command(TARGET ${TARGET}
        POST_BUILD
        BYPRODUCTS ${_qmltypes_out}
        COMMAND ${CMAKE_COMMAND} -E copy
            $<TARGET_FILE:${TARGET}>
            "${_qmldir_dir}/${_plugin_filename}"
        COMMAND ${CMAKE_COMMAND} -E env
            QML2_IMPORT_PATH=${CMAKE_CURRENT_BINARY_DIR}
            QT_QML_IMPORT_PATH=${CMAKE_CURRENT_BINARY_DIR}
            QT_PLUGIN_PATH=${_qmldir_dir}
            ${_QMLPLUGINDUMP_EXECUTABLE}
                -nonrelocatable
                -output ${_qmltypes_out}
                ${QMLMOD_URI} ${QMLMOD_VERSION}
        COMMENT "Generating plugins.qmltypes for ${QMLMOD_URI}"
    )
endfunction()

qt5_add_qml_module(
    qofonoextdeclarative
    URI org.nemomobile.ofono
    VERSION 1.0
)