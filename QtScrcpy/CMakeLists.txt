﻿set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(QT NAMES Qt6 Qt5 COMPONENTS Widgets Network LinguistTools REQUIRED)
find_package(Qt${QT_VERSION_MAJOR} COMPONENTS Widgets Network LinguistTools REQUIRED)

if(MSVC)
    # FFmpeg cannot be compiled natively by MSVC version < 12.0 (2013)
    if(MSVC_VERSION LESS 1800)
        message(FATAL_ERROR "[QtScrcpy] FATAL ERROR: MSVC version is older than 12.0 (2013).")
    endif()

    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /utf-8")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /utf-8")
endif()


# ==================== macOS ====================
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    # QS_MAC_RESOURCES: esource file list stored in Contents/MacOS
    file(GLOB QS_MAC_RESOURCES "${PROJECT_SOURCE_DIR}/third_party/ffmpeg/lib/*.dylib")
    list(APPEND QS_MAC_RESOURCES
        "${PROJECT_SOURCE_DIR}/third_party/scrcpy-server"
        "${PROJECT_SOURCE_DIR}/adb/mac/adb"
    )

    # QS_MAC_CONFIG: Config file stored in Contents/MacOS/config
    set(QS_MAC_CONFIG "${PROJECT_SOURCE_DIR}/config/config.ini")
endif()

set(QS_TS_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/res/i18n/zh_CN.ts
    ${CMAKE_CURRENT_SOURCE_DIR}/res/i18n/en_US.ts
    )
set_source_files_properties(${QS_TS_FILES} PROPERTIES OUTPUT_LOCATION "${CMAKE_CURRENT_SOURCE_DIR}/res/i18n")

set(QS_SOURCES_MAIN
    dialog.cpp
    dialog.h
    dialog.ui
    ${QS_TS_FILES}
)

set(QS_QRC_MAIN "${CMAKE_CURRENT_SOURCE_DIR}/res/res.qrc")

if(${QT_VERSION_MAJOR} GREATER_EQUAL 6)                         # Qt version 6
    qt_create_translation(QS_QM_FILES ${CMAKE_CURRENT_SOURCE_DIR} ${QS_TS_FILES})

    if(WIN32)
        qt_add_executable(${CMAKE_PROJECT_NAME} WIN32 MANUAL_FINALIZATION
            main.cpp
            ${QS_SOURCES_MAIN}
            ${QS_QRC_MAIN}
        )

    elseif(UNIX)
        if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
            qt_add_executable(${CMAKE_PROJECT_NAME} MACOSX_BUNDLE MANUAL_FINALIZATION
                main.cpp
                ${QS_SOURCES_MAIN}
                ${QS_MAC_RESOURCES}
                ${QS_MAC_CONFIG}
                ${QS_QRC_MAIN}
            )

        else()
            qt_add_executable(${CMAKE_PROJECT_NAME} MANUAL_FINALIZATION
                main.cpp
                ${QS_SOURCES_MAIN}
                ${QS_QRC_MAIN}
            )

        endif()
    endif()

else()                                                          # Qt version 5
    qt5_create_translation(QS_QM_FILES ${CMAKE_CURRENT_SOURCE_DIR} ${QS_TS_FILES})

    if(WIN32)
        add_executable(${CMAKE_PROJECT_NAME} WIN32
            main.cpp
            ${QS_SOURCES_MAIN}
            ${QS_QRC_MAIN}
        )
    elseif(UNIX)
        if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
            add_executable(${CMAKE_PROJECT_NAME} MACOSX_BUNDLE
                main.cpp
                ${QS_SOURCES_MAIN}
                ${QS_MAC_RESOURCES}
                ${QS_MAC_CONFIG}
                ${QS_QRC_MAIN}
            )
        else()
            add_executable(${CMAKE_PROJECT_NAME}
                main.cpp
                ${QS_SOURCES_MAIN}
                ${QS_QRC_MAIN}
            )
        endif()
    endif()

endif()

# ******************** Microsoft Windows ********************
if(WIN32)
    message(STATUS "[QtScrcpy] Make for Microsoft Windows.")

    # 通过rc的方式的话，VERSION变量rc中获取不到,定义为宏方便rc中使用
    # Define macros for .rc file
    add_compile_definitions(
        VERSION_MAJOR=${VERSION_MAJOR}
        VERSION_MINOR=${VERSION_MINOR}
        VERSION_PATCH=${VERSION_PATCH}
        VERSION_RC_STR=\\\"${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}\\\"
    )

    if(CMAKE_SIZEOF_VOID_P EQUAL 8)                                     # Compiler is 64-bit
        message(STATUS "[QtScrcpy] 64-bit compiler detected.")

        set(QS_LIB_PATH "${PROJECT_SOURCE_DIR}/third_party/ffmpeg/lib/x64")
        if(CMAKE_BUILD_TYPE STREQUAL "Debug")
            message(STATUS "[QtScrcpy] In debug mode.")
            set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
                RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/output/win/x64/debug"
                )
        else()
            message(STATUS "[QtScrcpy] In release mode.")
            set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
                RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/output/win/x64/release")
        endif()

        set(QS_DLL_PATH "${PROJECT_SOURCE_DIR}/third_party/ffmpeg/bin/x64")

    elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)                                 # Compiler is 32-bit
        message(STATUS "[QtScrcpy] 32-bit compiler detected.")

        set(QS_LIB_PATH "${PROJECT_SOURCE_DIR}/third_party/ffmpeg/lib/x86")
        if(CMAKE_BUILD_TYPE STREQUAL "Debug")
            message(STATUS "[QtScrcpy] In debug mode.")
            set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
                RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/output/win/x86/debug")
        else()
            message(STATUS "[QtScrcpy] In release mode.")
            set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
                RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/output/win/x86/release")
        endif()

        set(QS_DLL_PATH "${PROJECT_SOURCE_DIR}/third_party/ffmpeg/bin/x86")
    endif()

    # 构建完成后复制DLL依赖库
    # Copy DLL dependencies after building
    get_target_property(QS_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_PROJECT_NAME} RUNTIME_OUTPUT_DIRECTORY)
    file(GLOB QS_DLL_FILES "${QS_DLL_PATH}/*.dll")
    foreach(QS_DLL_FILE ${QS_DLL_FILES})
        add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD COMMAND
            ${CMAKE_COMMAND} -E copy_if_different
            "${QS_DLL_FILE}" "${QS_RUNTIME_OUTPUT_DIRECTORY}"
        )
    endforeach()

    if(MSVC)
        message(STATUS "[QtScrcpy] Microsoft Visual C++ is used.")
        target_link_directories(${CMAKE_PROJECT_NAME} PRIVATE ${QS_LIB_PATH})
        set(QS_EXTERNAL_LIBS_FFMPEG
            avformat
            avcodec
            avutil
            swscale
        )
    # If MinGW is used, it is not appropriate to link static MSVC libs.
    # Instead, we link DLLs directly
    elseif(MINGW)
        message(STATUS "[QtScrcpy] MinGW GCC is used.")
        target_link_options(${CMAKE_PROJECT_NAME} PRIVATE
            "-static"
            ${QS_DLL_FILES}
            "-Wl,--enable-stdcall-fixup"
            )
    endif()

    set(RC_FILE "${CMAKE_CURRENT_SOURCE_DIR}/res/QtScrcpy.rc")

# ******************** Unix-like OSs ********************
elseif(UNIX)
    set(QS_LIB_PATH "${PROJECT_SOURCE_DIR}/third_party/ffmpeg/lib")

    # ==================== macOS ====================
    if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
        message(STATUS "[QtScrcpy] Make for macOS.")
        target_link_directories(${CMAKE_PROJECT_NAME} PRIVATE ${QS_LIB_PATH})

        if(CMAKE_BUILD_TYPE STREQUAL "Debug")
            set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
                RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/output/mac/debug")
        else()
            set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
                RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/output/mac/release")
        endif()

        # Icon file stored in Contents/Resources
        set(QS_MAC_ICON_NAME "QtScrcpy.icns")
        set(QS_MAC_ICON_PATH "${CMAKE_CURRENT_SOURCE_DIR}/res/${QS_MAC_ICON_NAME}")

        set_source_files_properties(${QS_MAC_RESOURCES} PROPERTIES
            MACOSX_PACKAGE_LOCATION "MacOS"
        )
        set_source_files_properties(${QS_MAC_CONFIG} PROPERTIES
            MACOSX_PACKAGE_LOCATION "MacOS/config"
        )

        set(QS_EXTERNAL_LIBS_FFMPEG
            avformat.58
            avcodec.58
            avutil.56
            swscale.5
        )

        set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
            # The base plist template file
            MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/res/Info_Mac.plist"
            # The elements to be overwritten
            MACOSX_BUNDLE_ICON_FILE "${QS_MAC_ICON_NAME}"
            MACOSX_BUNDLE_BUNDLE_VERSION "${STRING_VERSION}"
            MACOSX_BUNDLE_SHORT_VERSION_STRING "${STRING_VERSION}"
            MACOSX_BUNDLE_LONG_VERSION_STRING "${STRING_VERSION}"

            # Copy file(s) to Contents/Resources
            RESOURCE "${QS_MAC_ICON_PATH}"
        )

    # =============== Non-Mac OSs (Linux, BSD, etc.) ===============
    else()
        if(CMAKE_BUILD_TYPE STREQUAL "Debug")
            set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
                RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/output/linux/debug")
        else()
            set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES
                RUNTIME_OUTPUT_DIRECTORY "${PROJECT_SOURCE_DIR}/output/linux/release")
        endif()

        find_package(Threads REQUIRED)

        message(STATUS "[QtScrcpy] Make for non-Mac Unix-like OS.")
        set(INSTALLED_FFMPEG_FOUND false)

        find_package(PkgConfig)
        if(PkgConfig_FOUND)
            pkg_check_modules(FFmpeg libavformat>=58 libavcodec>=58 libavutil>=56 libswscale>=5)
            if(FFmpeg_FOUND)
                set(INSTALLED_FFMPEG_FOUND true)
                message(STATUS "[QtScrcpy] Development files of FFmpeg were detected in your OS and will be used.")
                target_link_options(${CMAKE_PROJECT_NAME} PRIVATE ${FFmpeg_LDFLAGS})
                target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE ${FFmpeg_CFLAGS})
                set(QS_EXTERNAL_LIBS_FFMPEG ${FFmpeg_LIBRARIES})
            endif()
        endif()

        if(NOT INSTALLED_FFMPEG_FOUND)
            message(STATUS "[QtScrcpy] Development files of FFmpeg were not detected in your OS. Files within third_party/ffmpeg/ will be used.")
            target_link_directories(${CMAKE_PROJECT_NAME} PRIVATE ${QS_LIB_PATH})
            set(QS_EXTERNAL_LIBS_FFMPEG
                avformat
                avcodec
                avutil
                swscale
                Threads::Threads
            )
        endif()
    endif()
endif()

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

set(QS_SUBDIRECTORIES_MAIN
    adb
    common
    device
    devicemanage
    fontawesome
    uibase
    util
)
foreach(QS_SUBDIRECTORY_MAIN ${QS_SUBDIRECTORIES_MAIN})
    add_subdirectory(${QS_SUBDIRECTORY_MAIN})
endforeach()

target_link_libraries(${CMAKE_PROJECT_NAME} PUBLIC
    adb
    devicemanage
    )

target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE
    Qt${QT_VERSION_MAJOR}::Widgets
    Qt${QT_VERSION_MAJOR}::Network
    device
        stream
        ui
    util
    )

if(QT_VERSION_MAJOR EQUAL 6)
    qt_finalize_executable(${CMAKE_PROJECT_NAME})
endif()
