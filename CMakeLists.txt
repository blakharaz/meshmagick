project(meshmagick)

cmake_minimum_required(VERSION 2.8)
cmake_policy(SET CMP0003 NEW)
cmake_policy(SET CMP0005 OLD)

include(FeatureSummary)

set(MESHMAGICK_MAJOR_VERSION 0)
set(MESHMAGICK_MINOR_VERSION 5)
set(MESHMAGICK_PATCH_VERSION 2)

get_directory_property(hasParent PARENT_DIRECTORY)
if(hasParent)
  message(STATUS "Has a parent scope.")
else()
  message(STATUS "Doesn't have a parent scope.")
    # include our custom cmake scripts
    set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)
    find_package(OGRE REQUIRED)

    # Find Boost
    # Prefer static linking in all cases
    if (NOT OGRE_BUILD_PLATFORM_IPHONE)
        if (WIN32 OR APPLE)
            set(Boost_USE_STATIC_LIBS TRUE)
        else ()
            # Statically linking boost to a dynamic Ogre build doesn't work on Linux 64bit
            set(Boost_USE_STATIC_LIBS ${OGRE_STATIC})
        endif ()

        if (APPLE AND OGRE_BUILD_PLATFORM_APPLE_IOS)
            set(Boost_USE_MULTITHREADED OFF)
        endif()

        set(Boost_ADDITIONAL_VERSIONS "1.57" "1.57.0" "1.56" "1.56.0" "1.55" "1.55.0" "1.54" "1.54.0" "1.53" "1.53.0" "1.52" "1.52.0" "1.51" "1.51.0" "1.50" "1.50.0" "1.49" "1.49.0" "1.48" "1.48.0" "1.47" "1.47.0" "1.46" "1.46.0" "1.45" "1.45.0" "1.44" "1.44.0" "1.42" "1.42.0" "1.41.0" "1.41" "1.40.0" "1.40")

        # Components that need linking (NB does not include header-only components like bind)
        set(OGRE_BOOST_COMPONENTS thread date_time system regex filesystem)

        find_package(Boost COMPONENTS ${OGRE_BOOST_COMPONENTS} QUIET)

        if (NOT Boost_FOUND)
            # Try again with the other type of libs
            if(Boost_USE_STATIC_LIBS)
                set(Boost_USE_STATIC_LIBS OFF)
            else()
                set(Boost_USE_STATIC_LIBS ON)
            endif()

            find_package(Boost COMPONENTS ${OGRE_BOOST_COMPONENTS} QUIET)
        endif()

        if(Boost_FOUND AND Boost_VERSION GREATER 104900)
            if(Boost_VERSION GREATER 105300)
                set(OGRE_BOOST_COMPONENTS thread date_time system atomic chrono regex filesystem)
            else()
                set(OGRE_BOOST_COMPONENTS thread date_time system chrono regex filesystem)
            endif()
            find_package(Boost COMPONENTS ${OGRE_BOOST_COMPONENTS} QUIET)
        endif()

        if(Boost_FOUND AND NOT WIN32)
          list(REMOVE_DUPLICATES Boost_LIBRARIES)
        endif()

        if (Boost_FOUND)
            set(OGRE_LIBRARIES ${OGRE_LIBRARIES} ${Boost_LIBRARIES})
            set(OGRE_INCLUDE_DIRS ${OGRE_INCLUDE_DIRS} ${Boost_INCLUDE_DIRS})
        endif (Boost_FOUND)
    endif(NOT OGRE_BUILD_PLATFORM_IPHONE)
endif()

# dependencies
#find_package(PkgConfig)

set(MESHMAGICK_SOURCE
	src/MeshMagick.cpp
	src/MmEditableBone.cpp
	src/MmEditableMesh.cpp
	src/MmEditableSkeleton.cpp
	src/MmInfoTool.cpp
	src/MmInfoToolFactory.cpp
	src/MmMeshMergeTool.cpp
	src/MmMeshMergeToolFactory.cpp
	src/MmMeshUtils.cpp
	src/MmOgreEnvironment.cpp
	src/MmOptimiseTool.cpp
	src/MmOptimiseToolFactory.cpp
	src/MmOptionsParser.cpp
	src/MmRenameTool.cpp
	src/MmRenameToolFactory.cpp
	src/MmStatefulMeshSerializer.cpp
	src/MmStatefulSkeletonSerializer.cpp
	src/MmTool.cpp
	src/MmToolManager.cpp
	src/MmToolsUtils.cpp
	src/MmTransformTool.cpp
	src/MmTransformToolFactory.cpp
)

set(MESHMAGICK_HEADERS
	include/MeshMagick.h
	include/MeshMagickPrerequisites.h
	include/MmEditableBone.h
	include/MmEditableMesh.h
	include/MmEditableSkeleton.h
	include/MmInfoToolFactory.h
	include/MmInfoTool.h
	include/MmMeshMergeToolFactory.h
	include/MmMeshMergeTool.h
	include/MmMeshUtils.h
	include/MmOgreEnvironment.h
	include/MmOptimiseToolFactory.h
	include/MmOptimiseTool.h
	include/MmOptionsParser.h
	include/MmRenameToolFactory.h
	include/MmRenameTool.h
	include/MmStatefulMeshSerializer.h
	include/MmStatefulSkeletonSerializer.h
	include/MmToolFactory.h
	include/MmTool.h
	include/MmToolManager.h
	include/MmToolUtils.h
	include/MmTransformToolFactory.h
	include/MmTransformTool.h
)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include ${OGRE_INCLUDE_DIRS})

if(hasParent)
    ogitor_add_library(meshmagick_shared_lib SHARED ${MESHMAGICK_SOURCE} ${MESHMAGICK_HEADERS})
else()
    add_library(meshmagick_shared_lib SHARED ${MESHMAGICK_SOURCE} ${MESHMAGICK_HEADERS})
endif()
set(LIBRARY_OUTPUT_PATH  ../lib)
set_target_properties(meshmagick_shared_lib PROPERTIES
	OUTPUT_NAME meshmagick
	VERSION ${MESHMAGICK_MAJOR_VERSION}.${MESHMAGICK_MINOR_VERSION}.${MESHMAGICK_PATCH_VERSION}
	SOVERSION ${MESHMAGICK_MAJOR_VERSION}.${MESHMAGICK_MINOR_VERSION}
	DEFINE_SYMBOL MESHMAGICK_EXPORTS)
target_link_libraries(meshmagick_shared_lib ${OGRE_LIBRARIES})

if(NOT APPLE)
	add_executable(meshmagick_bin src/main.cpp)
else()
	set(COPY_FRAMEWORKS ${OGRE_LIBRARIES})

	add_executable(meshmagick_bin src/main.cpp ${COPY_FRAMEWORKS})

	set_source_files_properties(${COPY_FRAMEWORKS} PROPERTIES MACOSX_PACKAGE_LOCATION Frameworks)
endif(NOT APPLE)

set_target_properties(meshmagick_bin PROPERTIES
	OUTPUT_NAME meshmagick
	DEFINE_SYMBOL MESHMAGICK_IMPORTS)
target_link_libraries(meshmagick_bin meshmagick_shared_lib ${OGRE_LIBRARIES})

#configure_file(${CMAKE_CURRENT_SOURCE_DIR}/meshmagick.pc.cmake ${CMAKE_CURRENT_BINARY_DIR}/meshmagick.pc)

if(WIN32)
	add_custom_command(TARGET meshmagick_shared_lib PRE_BUILD
		COMMAND if not exist ..\\..\\RunPath\\bin mkdir ..\\..\\RunPath\\bin)
	add_custom_command(TARGET meshmagick_shared_lib POST_BUILD
		COMMAND copy \"$(TargetPath)\" ..\\..\\RunPath\\bin)

	install(TARGETS meshmagick_bin meshmagick_shared_lib
		RUNTIME DESTINATION bin
		CONFIGURATIONS Debug Release RelWithDebInfo MinSizeRel)
endif(WIN32)

#install(TARGETS meshmagick_bin meshmagick_shared_lib
#	RUNTIME DESTINATION bin
#	LIBRARY DESTINATION lib)

#if(NOT WIN32)
#	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/meshmagick.pc
#					DESTINATION ${CMAKE_INSTALL_PREFIX}/lib/pkgconfig)
#endif(NOT WIN32)

#install(FILES
#include/MeshMagick.h
#include/MeshMagickPrerequisites.h
#include/MmEditableBone.h
#include/MmEditableMesh.h
#include/MmEditableSkeleton.h
#include/MmInfoToolFactory.h
#include/MmInfoTool.h
#include/MmMeshMergeToolFactory.h
#include/MmMeshMergeTool.h
#include/MmMeshUtils.h
#include/MmOgreEnvironment.h
#include/MmOptimiseToolFactory.h
#include/MmOptimiseTool.h
#include/MmOptionsParser.h
#include/MmRenameToolFactory.h
#include/MmRenameTool.h
#include/MmStatefulMeshSerializer.h
#include/MmStatefulSkeletonSerializer.h
#include/MmToolFactory.h
#include/MmTool.h
#include/MmToolManager.h
#include/MmToolUtils.h
#include/MmTransformToolFactory.h
#include/MmTransformTool.h
#DESTINATION ${CMAKE_INSTALL_PREFIX}/include/meshmagick)

#include(CPack)

# vim: set sw=2 ts=2 noet:
