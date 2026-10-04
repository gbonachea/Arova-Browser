cmake_minimum_required(VERSION 3.21)
project(Arova VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

# --- Qt 6.11 (por defecto busca la instalacion oficial en /opt/qt) ------------
# Si no se pasa -DCMAKE_PREFIX_PATH, se anade el prefijo de Qt 6.11 detectado
# para que un simple `cmake -B build` use Qt 6.11.2 y no el Qt del sistema.
set(AROVA_DEFAULT_QT_PREFIXES
    "/opt/qt/6.11.2/gcc_64"
    "/opt/qt/6.11/gcc_64"
    "/opt/qt/6.11.2"
    "/opt/qt/6.11"
)

if(NOT CMAKE_PREFIX_PATH)
    foreach(_qt_prefix IN LISTS AROVA_DEFAULT_QT_PREFIXES)
        if(EXISTS "${_qt_prefix}/lib/cmake/Qt6/Qt6Config.cmake")
            list(APPEND CMAKE_PREFIX_PATH "${_qt_prefix}")
            message(STATUS "Qt: usando prefijo por defecto ${_qt_prefix}")
            break()
        endif()
    endforeach()
endif()

# Qt 6.11 puede venir tanto en /opt/qt/<ver>/gcc_64 (instalador) como instalado
# de forma estandar en el sistema. Anadimos el prefijo detectado a la lista de
# busqueda de Qt para que WebEngineWidgets se resuelva siempre en la misma
# instalacion que Core/Gui.
list(APPEND QT_ADDITIONAL_PACKAGES_PREFIX_PATH ${CMAKE_PREFIX_PATH})

find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets Network WebEngineWidgets)

message(STATUS "Qt version: ${Qt6_VERSION} (${Qt6_DIR})")

# --- Herramientas de Qt (moc/uic/rcc) y LD_LIBRARY_PATH -----------------------
# moc, uic y rcc llevan RUNPATH $ORIGIN/../lib, pero LD_LIBRARY_PATH tiene
# prioridad sobre el RUNPATH. Si el entorno apunta a otro Qt (por ejemplo el
# Qt del sistema), las herramientas cargan esas librerias y fallan con errores
# de simbolos versionados, p. ej. "version `Qt_6.11' not found".
get_filename_component(_qt_prefix "${Qt6_DIR}/../../.." ABSOLUTE)
get_filename_component(_qt_libdir "${_qt_prefix}/lib" ABSOLUTE)
get_filename_component(_qt_libexec "${_qt_prefix}/libexec" ABSOLUTE)

if(NOT EXISTS "${_qt_libdir}/libQt6Core.so.6")
    set(_qt_libdir "")
endif()

# Se prueba cada herramienta tal cual esta; solo se envuelven las que fallan.
# (moc va enlazado estaticamente con Qt y no suele suffer el problema, uic y rcc
#  si dependen de libQt6Core de forma dinamica.)
if(_qt_libdir)
    set(AROVA_QT_TOOL_DIR "${CMAKE_BINARY_DIR}/arova-qt-tools")
    set(AROVA_QT_TOOL_WRAPPED OFF)
    foreach(_tool moc uic rcc)
        if(EXISTS "${_qt_libexec}/${_tool}")
            execute_process(
                COMMAND "${_qt_libexec}/${_tool}" -v
                RESULT_VARIABLE _tool_rc
                OUTPUT_VARIABLE _tool_out
                ERROR_VARIABLE _tool_err
            )
            string(FIND "${_tool_err}${_tool_out}" "not found" _tool_broken)
            if(_tool_rc GREATER 0 OR NOT _tool_broken EQUAL -1)
                message(STATUS
                    "Qt: ${_tool} no puede cargar las librerias de Qt (LD_LIBRARY_PATH), se envuelve")
                file(MAKE_DIRECTORY "${AROVA_QT_TOOL_DIR}")
                set(_wrapper "${AROVA_QT_TOOL_DIR}/${_tool}")
                file(WRITE "${_wrapper}"
                    "#!/bin/sh\n"
                    "# Generado por CMake: fija la libreria de Qt usada en la compilacion.\n"
                    "LD_LIBRARY_PATH=\"${_qt_libdir}\${LD_LIBRARY_PATH:+:\$LD_LIBRARY_PATH}\"\n"
                    "export LD_LIBRARY_PATH\n"
                    "exec \"${_qt_libexec}/${_tool}\" \"$@\"\n"
                )
                file(CHMOD "${_wrapper}"
                    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE
                                GROUP_READ GROUP_EXECUTE
                                WORLD_READ WORLD_EXECUTE
                )
                set(AROVA_QT_TOOL_WRAPPED ON)
                if(_tool STREQUAL "moc")
                    set(CMAKE_AUTOMOC_EXECUTABLE "${_wrapper}" CACHE STRING "" FORCE)
                elseif(_tool STREQUAL "uic")
                    set(CMAKE_AUTOUIC_EXECUTABLE "${_wrapper}" CACHE STRING "" FORCE)
                else()
                    set(CMAKE_AUTORCC_EXECUTABLE "${_wrapper}" CACHE STRING "" FORCE)
                endif()
            endif()
        endif()
    endforeach()

    # La propia configuracion tambien puede lanzar herramientas de Qt.
    if(AROVA_QT_TOOL_WRAPPED)
        if(DEFINED ENV{LD_LIBRARY_PATH} AND NOT "$ENV{LD_LIBRARY_PATH}" STREQUAL "")
            set(ENV{LD_LIBRARY_PATH} "${_qt_libdir}:$ENV{LD_LIBRARY_PATH}")
        else()
            set(ENV{LD_LIBRARY_PATH} "${_qt_libdir}")
        endif()
    endif()
endif()

add_definitions(-DPKGDATADIR="${CMAKE_INSTALL_PREFIX}/share/arova")

add_subdirectory(src)
