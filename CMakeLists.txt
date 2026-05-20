cmake_minimum_required(VERSION 3.16)
project(Application VERSION 0.1 LANGUAGES CXX)

set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 让 AUTOUIC 搜索 ui/ 目录
set(CMAKE_AUTOUIC_SEARCH_PATHS ${CMAKE_CURRENT_SOURCE_DIR}/ui)

# ====== 查找 Qt6 ======
find_package(Qt6 REQUIRED COMPONENTS Core Gui Network Widgets OpenGLWidgets)
find_package(Qt6Mqtt REQUIRED)

# ====== Protobuf 配置 ======
set(Protobuf_USE_STATIC_LIBS ON)
find_package(Protobuf REQUIRED)
find_package(absl CONFIG REQUIRED)

# 获取 protoc 路径
if(NOT Protobuf_PROTOC_EXECUTABLE)
    find_program(Protobuf_PROTOC_EXECUTABLE protoc REQUIRED)
endif()
message(STATUS "Using protoc: ${Protobuf_PROTOC_EXECUTABLE}")

# ====== 源文件列表 ======
set(PROJECT_SOURCES
    src/main.cpp
    src/play.cpp
    src/videowidget.cpp
    src/widget.cpp
    src/hevcnalparser.cpp
)

# ====== 头文件列表（用于 IDE 显示，不是必须） ======
set(PROJECT_HEADERS
    include/play.h
    include/videowidget.h
    include/widget.h
    include/hevcnalparser.h
)

# ====== UI 文件列表（qt_add_executable 会自动处理） ======
set(PROJECT_UIS
    ui/play.ui
    ui/widget.ui
)

# ====== Protobuf 生成 ======
set(ROBOMASTER_PROTO_FILE ${CMAKE_CURRENT_SOURCE_DIR}/proto/robomaster_protocol.proto)
set(GENERATED_PROTO_CC ${CMAKE_CURRENT_BINARY_DIR}/robomaster_protocol.pb.cc)
set(GENERATED_PROTO_H  ${CMAKE_CURRENT_BINARY_DIR}/robomaster_protocol.pb.h)

if(NOT EXISTS "${ROBOMASTER_PROTO_FILE}")
    message(FATAL_ERROR "Proto file not found: ${ROBOMASTER_PROTO_FILE}")
endif()

add_custom_command(
    OUTPUT ${GENERATED_PROTO_CC} ${GENERATED_PROTO_H}
    COMMAND ${Protobuf_PROTOC_EXECUTABLE}
        --cpp_out=${CMAKE_CURRENT_BINARY_DIR}
        -I${CMAKE_CURRENT_SOURCE_DIR}/proto
        ${ROBOMASTER_PROTO_FILE}
    DEPENDS ${ROBOMASTER_PROTO_FILE}
    COMMENT "Generating robomaster_protocol protobuf"
)
list(APPEND PROJECT_SOURCES ${GENERATED_PROTO_CC})

# ====== FFmpeg 配置 (pkg-config) ======
find_package(PkgConfig REQUIRED)
pkg_check_modules(AVCODEC REQUIRED libavcodec)
pkg_check_modules(AVFORMAT REQUIRED libavformat)
pkg_check_modules(AVUTIL REQUIRED libavutil)
pkg_check_modules(SWSCALE REQUIRED libswscale)
pkg_check_modules(SWRESAMPLE REQUIRED libswresample)
pkg_check_modules(AVFILTER REQUIRED libavfilter)
pkg_check_modules(AVDEVICE REQUIRED libavdevice)

# ====== 创建可执行文件 ======
qt_add_executable(Application
    MANUAL_FINALIZATION
    ${PROJECT_SOURCES}
    ${PROJECT_UIS}
)

# 让编译器能找到 include/ 目录
target_include_directories(Application PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR}      # 生成的头文件（protobuf）
    ${AVCODEC_INCLUDE_DIRS}
    ${AVFORMAT_INCLUDE_DIRS}
    ${AVUTIL_INCLUDE_DIRS}
    ${SWSCALE_INCLUDE_DIRS}
    ${SWRESAMPLE_INCLUDE_DIRS}
    ${AVFILTER_INCLUDE_DIRS}
    ${AVDEVICE_INCLUDE_DIRS}
)

# ====== 链接库 ======
target_link_libraries(Application PRIVATE
    Qt6::Widgets
    Qt6::Network
    Qt6::Mqtt
    Qt6::OpenGLWidgets
    protobuf::libprotobuf
    absl::log_internal_nullguard
    absl::strings
    absl::synchronization
    absl::time
    ${AVCODEC_LIBRARIES}
    ${AVFORMAT_LIBRARIES}
    ${AVUTIL_LIBRARIES}
    ${SWSCALE_LIBRARIES}
    ${SWRESAMPLE_LIBRARIES}
    ${AVFILTER_LIBRARIES}
    ${AVDEVICE_LIBRARIES}
)

# ====== 平台特定配置 ======
if(WIN32)
    if(NOT DEFINED FFMPEG_DIR)
        set(FFMPEG_DIR ${CMAKE_CURRENT_SOURCE_DIR}/ffmpeg)
    endif()
    add_custom_command(TARGET Application POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${FFMPEG_DIR}/bin
        $<TARGET_FILE_DIR:Application>
        COMMENT "Copying FFmpeg DLLs"
    )
else()
    # Linux 下复制 QSS
    add_custom_command(TARGET Application POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_CURRENT_SOURCE_DIR}/style.qss
            $<TARGET_FILE_DIR:Application>
        COMMENT "Copying style.qss"
    )
endif()

set_target_properties(Application PROPERTIES
    WIN32_EXECUTABLE TRUE
)

include(GNUInstallDirs)
install(TARGETS Application
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

qt_finalize_executable(Application)