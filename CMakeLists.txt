cmake_minimum_required(VERSION 3.16.0)
# Disable release-builds without debug info
set(CMAKE_CONFIGURATION_TYPES Debug RelWithDebInfo CACHE STRING INTERNAL FORCE)
project(Ingnomia
	VERSION 0.0.0
	DESCRIPTION ""
	HOMEPAGE_URL "https://github.com/rschurade/Ingnomia"
	LANGUAGES "CXX"
)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/modules")

find_package(Qt5 REQUIRED QUIET COMPONENTS Core Xml Sql OpenGL Network)

find_package(Steam REQUIRED)
find_package(Noesis REQUIRED)

# Noesis license
set(NOESIS_LICENSE_NAME "" CACHE STRING "Name")
set(NOESIS_LICENSE_KEY "" CACHE STRING "Key")

# Without this, paths are not relative in the sources list
cmake_policy(SET CMP0076 NEW)

include(GNUInstallDirs)

# The program itself
add_executable(${PROJECT_NAME})

# C++ version
set_target_properties(${PROJECT_NAME}
	PROPERTIES
		CXX_STANDARD 17
		AUTOMOC ON
		INTERPROCEDURAL_OPTIMIZATION ON
		INSTALL_RPATH "$ORIGIN"
)
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY AUTOGEN_SOURCE_GROUP "Generated")

add_subdirectory(content) # Deployable content
# Copy content to target directory
add_custom_command(TARGET ${PROJECT_NAME}
	POST_BUILD
	COMMAND
		${CMAKE_COMMAND} -E copy_directory
		${CMAKE_SOURCE_DIR}/content $<TARGET_FILE_DIR:${PROJECT_NAME}>/content
)

if(WIN32)
	target_sources(${PROJECT_NAME}
		PRIVATE
			${PROJECT_NAME}.rc
			${PROJECT_NAME}.ico
			${PROJECT_NAME}.manifest
	)
endif()

target_sources(${PROJECT_NAME}
	PRIVATE
		Changelog.md
		README.md
		LICENSE
)

add_subdirectory(src) # Sources list
add_subdirectory(3rdparty) # 3rd party dependencies

# Linking
target_link_libraries(${PROJECT_NAME}
	PRIVATE
		Qt5::Core
		Qt5::Xml
		Qt5::Sql
		Qt5::OpenGL
		Qt5::Network
		Steam
		NoesisApp
		fastnoise
		exprtk
)

# Noesis license
configure_file(
	${CMAKE_SOURCE_DIR}/src/gui/license.h.in
	${CMAKE_SOURCE_DIR}/src/gui/license.h
	@ONLY
)

if(UNIX)
	# Qt requires position independent code
	target_compile_options(${PROJECT_NAME}
		PRIVATE
			-fPIC
	)
endif()

get_target_property(SOURCE_LIST ${PROJECT_NAME} SOURCES)
source_group(TREE "${CMAKE_CURRENT_SOURCE_DIR}" FILES ${SOURCE_LIST})

if(UNIX AND NOT (DEFINED ENV{CXXFLAGS} OR CMAKE_CXX_FLAGS))
	set(CMAKE_CXX_FLAGS "-Wall -Wextra -g")
endif()

if(MSVC)
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /Gy /Oi /MP /GS- /W3 /permissive-")
	SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /CGTHREADS:8 /PROFILE")
	SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO} /Ob2")
	set_target_properties(${PROJECT_NAME} PROPERTIES
		LINK_FLAGS /SUBSYSTEM:WINDOWS
	)
	set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT ${PROJECT_NAME})
endif()

# Install target
install(
	TARGETS ${PROJECT_NAME}
	DESTINATION ${CMAKE_INSTALL_BINDIR}
)

get_target_property(STEAM_DLL Steam LOCATION)
get_target_property(NOESIS_DLL Noesis LOCATION)
list(APPEND RUNTIME_DLLS ${STEAM_DLL})
list(APPEND RUNTIME_DLLS ${NOESIS_DLL})

install(
	FILES
		${RUNTIME_DLLS}
		Changelog.md
		LICENSE
	DESTINATION
		${CMAKE_INSTALL_BINDIR}
)

if(WIN32)
	install(
		FILES $<TARGET_PDB_FILE:${PROJECT_NAME}>
		DESTINATION ${CMAKE_INSTALL_BINDIR}
		OPTIONAL
	)
endif()

# Deployment helpers for Windows
if(WIN32)
	include(windeployqt)
	windeployqt(${PROJECT_NAME})

	add_custom_command(TARGET ${PROJECT_NAME}
		POST_BUILD
		COMMAND
			${CMAKE_COMMAND} -E copy_if_different
			${RUNTIME_DLLS}
			$<TARGET_FILE_DIR:${PROJECT_NAME}>
	)

	INCLUDE_EXTERNAL_MSPROJECT("Blend" ${CMAKE_CURRENT_SOURCE_DIR}/gui/gui.csproj)
	set_target_properties("Blend" PROPERTIES EXCLUDE_FROM_ALL TRUE)

	target_link_libraries(${PROJECT_NAME}
		PRIVATE
			Version.lib
	)
endif()
