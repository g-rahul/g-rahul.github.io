cmake_minimum_required(VERSION 3.16)
project(RahulResume NONE)

set(RESUME_TEX_ENGINE "auto" CACHE STRING
    "TeX engine to use: auto, latexmk, or tectonic")
set_property(CACHE RESUME_TEX_ENGINE PROPERTY STRINGS auto latexmk tectonic)

if(NOT RESUME_TEX_ENGINE MATCHES "^(auto|latexmk|tectonic)$")
    message(FATAL_ERROR
        "Invalid RESUME_TEX_ENGINE='${RESUME_TEX_ENGINE}'. "
        "Choose auto, latexmk, or tectonic.")
endif()

# Prefer a regular TeX installation, but support a lightweight local Tectonic
# binary as well. Either executable can be supplied explicitly with
# -DLATEXMK_EXECUTABLE=/path/to/latexmk or
# -DTECTONIC_EXECUTABLE=/path/to/tectonic.
if(RESUME_TEX_ENGINE STREQUAL "auto" OR RESUME_TEX_ENGINE STREQUAL "latexmk")
    find_program(LATEXMK_EXECUTABLE NAMES latexmk)
endif()

if(RESUME_TEX_ENGINE STREQUAL "auto" OR RESUME_TEX_ENGINE STREQUAL "tectonic")
    find_program(TECTONIC_EXECUTABLE NAMES tectonic)
endif()

if(RESUME_TEX_ENGINE STREQUAL "latexmk")
    if(NOT LATEXMK_EXECUTABLE)
        message(FATAL_ERROR
            "RESUME_TEX_ENGINE=latexmk, but latexmk was not found. "
            "Set LATEXMK_EXECUTABLE to its full path.")
    endif()
    set(RESUME_TEX_BACKEND latexmk)
elseif(RESUME_TEX_ENGINE STREQUAL "tectonic")
    if(NOT TECTONIC_EXECUTABLE)
        message(FATAL_ERROR
            "RESUME_TEX_ENGINE=tectonic, but tectonic was not found. "
            "Set TECTONIC_EXECUTABLE to a local executable.")
    endif()
    set(RESUME_TEX_BACKEND tectonic)
elseif(LATEXMK_EXECUTABLE)
    set(RESUME_TEX_BACKEND latexmk)
elseif(TECTONIC_EXECUTABLE)
    set(RESUME_TEX_BACKEND tectonic)
else()
    message(FATAL_ERROR
        "No supported TeX engine found. Install latexmk or tectonic, or set "
        "LATEXMK_EXECUTABLE/TECTONIC_EXECUTABLE to a local executable.")
endif()

# One education-first resume assembled from shared section files.
set(SRC_TEX ${CMAKE_CURRENT_SOURCE_DIR}/RG.tex)
set(RESUME_TEX
    ${SRC_TEX}
    ${CMAKE_CURRENT_SOURCE_DIR}/sections/education.tex
    ${CMAKE_CURRENT_SOURCE_DIR}/sections/skills.tex
    ${CMAKE_CURRENT_SOURCE_DIR}/sections/experience.tex
    ${CMAKE_CURRENT_SOURCE_DIR}/sections/academics.tex
)

# Date code for release snapshots. Override with -DRESUME_VERSION=MM-YYYY.
if(NOT DEFINED RESUME_VERSION)
    string(TIMESTAMP RESUME_VERSION "%m-%Y")
endif()

set(FINAL_PDF ${CMAKE_CURRENT_BINARY_DIR}/Rahuls_Resume.pdf)
set(TECTONIC_PDF ${CMAKE_CURRENT_BINARY_DIR}/RG.pdf)
set(RELEASE_NAME RG-${RESUME_VERSION}.pdf)
set(RELEASE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/release)

# Path to the github.io repo's assets/files directory (override with -DSITE_ASSETS_DIR=...)
set(SITE_ASSETS_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../g-rahul.github.io/assets/files"
    CACHE PATH "Path to g-rahul.github.io/assets/files")

message(STATUS "Resume TeX backend: ${RESUME_TEX_BACKEND}")
message(STATUS "Final PDF: build/Rahuls_Resume.pdf")
message(STATUS "Release snapshot: release/${RELEASE_NAME}")

if(RESUME_TEX_BACKEND STREQUAL "latexmk")
    add_custom_command(
        OUTPUT ${FINAL_PDF}
        COMMAND ${LATEXMK_EXECUTABLE} -xelatex
                -interaction=nonstopmode -halt-on-error
                -output-directory=${CMAKE_CURRENT_BINARY_DIR}
                -jobname=Rahuls_Resume
                ${SRC_TEX}
        DEPENDS ${RESUME_TEX}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Building Rahuls_Resume.pdf with latexmk/XeLaTeX"
        VERBATIM
    )
else()
    add_custom_command(
        OUTPUT ${FINAL_PDF}
        COMMAND ${TECTONIC_EXECUTABLE}
                --outdir ${CMAKE_CURRENT_BINARY_DIR}
                ${SRC_TEX}
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
                ${TECTONIC_PDF} ${FINAL_PDF}
        COMMAND ${CMAKE_COMMAND} -E remove -f ${TECTONIC_PDF}
        DEPENDS ${RESUME_TEX}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Building Rahuls_Resume.pdf with tectonic"
        VERBATIM
    )
endif()

add_custom_target(resume ALL DEPENDS ${FINAL_PDF})

add_custom_target(release
    COMMAND ${CMAKE_COMMAND} -E make_directory ${RELEASE_DIR}
    COMMAND ${CMAKE_COMMAND} -E copy
            ${FINAL_PDF} ${RELEASE_DIR}/${RELEASE_NAME}
    DEPENDS ${FINAL_PDF}
    COMMENT "Releasing ${RELEASE_NAME} -> release/"
    VERBATIM
)

add_custom_target(publish
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SITE_ASSETS_DIR}/resume/sections
    COMMAND ${CMAKE_COMMAND} -E copy ${FINAL_PDF} ${SITE_ASSETS_DIR}/RAHUL_CV.pdf
    COMMAND ${CMAKE_COMMAND} -E copy ${SRC_TEX} ${SITE_ASSETS_DIR}/resume/RG.tex
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/CMakeLists.txt
            ${SITE_ASSETS_DIR}/resume/CMakeLists.txt
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE
            ${SITE_ASSETS_DIR}/resume/LICENSE
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/sections
            ${SITE_ASSETS_DIR}/resume/sections
    DEPENDS ${FINAL_PDF}
    COMMENT "Publishing resume -> ${SITE_ASSETS_DIR}"
    VERBATIM
)

if(RESUME_TEX_BACKEND STREQUAL "latexmk")
    add_custom_target(clean-resume
        COMMAND ${LATEXMK_EXECUTABLE} -C -output-directory=${CMAKE_CURRENT_BINARY_DIR}
                -jobname=Rahuls_Resume ${SRC_TEX}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Cleaning latexmk artifacts"
        VERBATIM
    )
else()
    add_custom_target(clean-resume
        COMMAND ${CMAKE_COMMAND} -E remove -f
                ${TECTONIC_PDF} ${FINAL_PDF}
        COMMENT "Cleaning tectonic artifacts"
        VERBATIM
    )
endif()
