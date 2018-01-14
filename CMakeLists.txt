# MIT License
#
# Copyright (c) 2018 Joerg Hallmann
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#
# https://github.com/joerghall/pypack
#
cmake_minimum_required(VERSION 3.5)

if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/_submodules/cbt/cmake/setup.cmake)
    include(${CMAKE_CURRENT_SOURCE_DIR}/_submodules/cbt/cmake/setup.cmake)
else ()
    message(FATAL_ERROR "Unable to find buildtools, please check if the submodule is initialized")
endif ()

#discover_source_version()

set(CMAKE_CXX_STANDARD 14)

if(APPLE)
    set(CMAKE_OSX_DEPLOYMENT_TARGET 10.10)
endif()

project(pybuild)

enable_testing()

file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CPACK_OUTPUT_FILE_PREFIX ${CMAKE_BINARY_DIR}/packages)

# Standard locations
#
set(SOURCE_UPLOAD_DIR ${CMAKE_BINARY_DIR}/upload)
set(SOURCE_DOWNLOAD_DIR ${CMAKE_BINARY_DIR}/download)
set(SOURCE_INSTALL_DIR ${CMAKE_BINARY_DIR}/install)
set(SOURCE_SRC_DIR ${CMAKE_BINARY_DIR}/src)
set(TARGET_INSTALL_DIR ${CMAKE_BINARY_DIR}/build)

# Python specifics
#
set(PYTHON_VERSION 2.7.14)
set(PYTHON_CHECKSUM cee2e4b33ad3750da77b2e85f2f8b724)
set(PYTHON_SOURCE_FILE Python-${PYTHON_VERSION}.tgz)
set(PYTHON_SOURCE_SOURCE_URL https://www.python.org/ftp/python/${PYTHON_VERSION}/${PYTHON_SOURCE_FILE})
set(PYTHON_SOURCE_TARGET_FILE ${SOURCE_DOWNLOAD_DIR}/${PYTHON_SOURCE_FILE})
set(PYTHON_SOURCE_TARGET_DIR ${CMAKE_BINARY_DIR}/src/${PYTHON_SOURCE_FILE})

if(BUILD_WINDOWS)
    message(FATAL_ERROR "Not implemented")
    set(PYTHON_INSTALL_TARGET_DIR ${TARGET_INSTALL_DIR}/python2/${PYTHON_VERSION}/windows)
elseif(BUILD_LINUX)
    set(PYTHON_INSTALL_TARGET_DIR ${TARGET_INSTALL_DIR}/python2/${PYTHON_VERSION}/linux)
elseif (BUILD_OSX)
    set(PYTHON_INSTALL_TARGET_DIR ${TARGET_INSTALL_DIR}/python2/${PYTHON_VERSION}/osx)
    set(openssl_LOCATION /usr/local/opt/openssl)
    set(readline_LOCATION /usr/local/Cellar/readline/7.0.3_1)
else()
    message(FATAL_ERROR "Unspecified OS type")
endif()

# Build python and package it up
#
#

# gpg --verify cmake-3.5.2-SHA-256.txt.asc

add_custom_target(download_python
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SOURCE_DOWNLOAD_DIR}
    COMMAND ${CMAKE_COMMAND} -DSOURCE=${PYTHON_SOURCE_SOURCE_URL} -DTARGET=${PYTHON_SOURCE_TARGET_FILE} -P ${download_COMMAND}
)

add_custom_target(unpack_python
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SOURCE_SRC_DIR}
    COMMAND ${CMAKE_COMMAND} -DSOURCE=${PYTHON_SOURCE_TARGET_FILE} -DTARGET=${SOURCE_SRC_DIR} -P ${untar_COMMAND}
    DEPENDS download_python
)

add_custom_target(build_python
    COMMAND ${CMAKE_COMMAND} -E make_directory ${PYTHON_INSTALL_TARGET_DIR}
    COMMAND ${CMAKE_COMMAND} -E env CPPFLAGS='-I${openssl_LOCATION}/include -I${readline_LOCATION}/include'
                                    LDFLAGS='-L${openssl_LOCATION}/lib -L${readline_LOCATION}/lib'
                                    ./configure --prefix=${PYTHON_INSTALL_TARGET_DIR} --enable-optimizations
    COMMAND make
    COMMAND make install prefix=${PYTHON_INSTALL_TARGET_DIR}
    COMMAND ${CMAKE_COMMAND} -DSOURCE="https://bootstrap.pypa.io/get-pip.py" -DTARGET="${PYTHON_INSTALL_TARGET_DIR}/get-pip.py" -P ${download_COMMAND}
    COMMAND ${PYTHON_INSTALL_TARGET_DIR}/bin/python ${PYTHON_INSTALL_TARGET_DIR}/get-pip.py
    COMMAND ${PYTHON_INSTALL_TARGET_DIR}/bin/pip install -v -r ${CMAKE_CURRENT_SOURCE_DIR}/requirements.txt
    COMMAND ${CMAKE_COMMAND} -E remove ${PYTHON_INSTALL_TARGET_DIR}/get-pip.py
    WORKING_DIRECTORY ${SOURCE_SRC_DIR}/Python-${PYTHON_VERSION}
    DEPENDS unpack_python
)

add_custom_target(package_python
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SOURCE_UPLOAD_DIR}
    COMMAND ${CMAKE_COMMAND} -E tar czvf ${SOURCE_UPLOAD_DIR}/python2-${PYTHON_VERSION}-osx.tgz python2
    WORKING_DIRECTORY ${TARGET_INSTALL_DIR}
    DEPENDS build_python
)
