cmake_minimum_required(VERSION 3.18)

project(instant-video-nerf
    VERSION 0.1.0
    LANGUAGES C CXX CUDA
    DESCRIPTION "Instant rendering of Mip-Nerf 360 algorithm"    
)

set(IVN_VERSION "${CMAKE_PROJECT_VERSION}")

if (NOT IVN_DEPLOY)
	set(IVN_VERSION "${IVN_VERSION}dev")
endif()

option(IVN_BUILD_WITH_GUI "Build with GUI support (requires GLFW and GLEW)?" ON)
option(IVN_BUILD_WITH_VULKAN "Build with Vulkan to enable DLSS support?" ON)
option(IVN_BUILD_WITH_OPTIX "Build with OptiX to enable hardware ray tracing?" ON)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

###############################################################################
# Build type and C++ compiler setup
###############################################################################

# Set a default configuration if none was specified
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	message(STATUS "No release type specified. Setting to 'Release'.")
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo")
endif()

if (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/glfw/CMakeLists.txt")
	message(FATAL_ERROR
		"Some dependencies are missing. "
		"If you forgot the \"--recursive\" flag when cloning this project, "
		"this can be fixed by calling \"git submodule update --init --recursive\"."
	)
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_EXTENSIONS OFF)

###############################################################################
# CUDA compiler setup
###############################################################################

set(CMAKE_CUDA_STANDARD 14)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_EXTENSIONS OFF)
set(CUDA_LINK_LIBRARIES_KEYWORD PUBLIC)
set(CMAKE_CUDA_RUNTIME_LIBRARY Shared)

if (MSVC)
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=-bigobj")
else()
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=-mf16c")
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=-Wno-float-conversion")
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=-fno-strict-aliasing")
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=-fPIC")
endif()
list(APPEND CUDA_NVCC_FLAGS "--extended-lambda")
list(APPEND CUDA_NVCC_FLAGS "--expt-relaxed-constexpr")

###############################################################################
# Dependencies
###############################################################################

set(TCNN_BUILD_BENCHMARK OFF)
set(TCNN_BUILD_EXAMPLES OFF)
add_subdirectory(dependencies/tiny-cuda-nn)

set(CMAKE_CUDA_ARCHITECTURES ${TCNN_CUDA_ARCHITECTURES})

if (IVN_BUILD_WITH_GUI)
	find_package(Vulkan)
	if (Vulkan_FOUND AND IVN_BUILD_WITH_VULKAN)
		set(IVN_VULKAN ON)
		list(APPEND IVN_DEFINITIONS -DIVN_VULKAN -DGLFW_INCLUDE_VULKAN)
		list(APPEND IVN_INCLUDE_DIRECTORIES "${Vulkan_INCLUDE_DIRS}")
		list(APPEND IVN_LIBRARIES ${Vulkan_LIBRARIES})
	else()
		set(IVN_VULKAN OFF)
		if (IVN_BUILD_WITH_VULKAN)
			message(WARNING
				"Vulkan was not found. Instant neural video core will still compile "
				"and run correctly, but DLSS will not be supported."
			)
		endif()
	endif()

	set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL " " FORCE)
	set(GLFW_BUILD_TESTS OFF CACHE BOOL " " FORCE)
	set(GLFW_BUILD_DOCS OFF CACHE BOOL " " FORCE)
	set(GLFW_BUILD_INSTALL OFF CACHE BOOL " " FORCE)
	set(GLFW_INSTALL OFF CACHE BOOL " " FORCE)
	set(GLFW_USE_CHDIR OFF CACHE BOOL " " FORCE)
	set(GLFW_VULKAN_STATIC OFF CACHE BOOL " " FORCE)
	set(BUILD_SHARED_LIBS ON CACHE BOOL " " FORCE)

	add_subdirectory(dependencies/glfw)

	set_target_properties(glfw PROPERTIES EXCLUDE_FROM_ALL 1 EXCLUDE_FROM_DEFAULT_BUILD 1)

	mark_as_advanced(
		GLFW_BUILD_DOCS GLFW_BUILD_EXAMPLES GLFW_BUILD_INSTALL GLFW_BUILD_TESTS
		GLFW_DOCUMENT_INTERNALS GLFW_INSTALL GLFW_USE_CHDIR GLFW_USE_MENUBAR
		GLFW_USE_OSMESA GLFW_VULKAN_STATIC GLFW_USE_RETINA GLFW_USE_MIR
		BUILD_SHARED_LIBS USE_MSVC_RUNTIME_LIBRARY_DLL
	)

	list(APPEND IVN_INCLUDE_DIRECTORIES "dependencies/glfw/include" "dependencies/imgui")

    find_package(GLEW REQUIRED)
    list(APPEND IVN_INCLUDE_DIRECTORIES ${GLEW_INCLUDE_DIRS})
    list(APPEND IVN_LIBRARIES GL ${GLEW_LIBRARIES} $<TARGET_OBJECTS:glfw>)

	list(APPEND GUI_SOURCES
		dependencies/imguizmo/ImGuizmo.cpp
		dependencies/imgui/imgui.cpp
		dependencies/imgui/backends/imgui_impl_glfw.cpp
		dependencies/imgui/backends/imgui_impl_opengl3.cpp
		dependencies/imgui/imgui_draw.cpp
		dependencies/imgui/imgui_tables.cpp
		dependencies/imgui/imgui_widgets.cpp
	)

	list(APPEND IVN_DEFINITIONS -DIVN_GUI)
endif(IVN_BUILD_WITH_GUI)


list(APPEND IVN_INCLUDE_DIRECTORIES
	"dependencies"
	"dependencies/eigen"
	"dependencies/filesystem"
)

find_package(OpenMP)
if (OPENMP_FOUND)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()


find_package(OptiX)
# set OptiX_INSTALL_DIR via your environment if it's not found automatically

if ((OptiX_FOUND OR OptiX_INCLUDE) AND IVN_BUILD_WITH_OPTIX)
	set(IVN_OPTIX ON)
	list(APPEND IVN_INCLUDE_DIRECTORIES "${OptiX_INCLUDE}")
	list(APPEND IVN_DEFINITIONS -DIVN_OPTIX)
else()
	set(IVN_OPTIX OFF)
	if (IVN_BUILD_WITH_OPTIX)
		message(WARNING
			"OptiX was not found. Neural graphics primitives will still compile "
			"and run correctly. However, SDF training in 'raystab' and 'pathescape' "
			"modes will be significantly slower."
		)
	endif()
endif()

###############################################################################
# Program
###############################################################################


list(APPEND IVN_DEFINITIONS -DIVN_VERSION="${IVN_VERSION}")
list(APPEND IVN_INCLUDE_DIRECTORIES "include")
# target_include_directories(${PROJECT_NAME} PUBLIC "include")

if (NOT MSVC)
	list(APPEND IVN_LIBRARIES ${CMAKE_DL_LIBS})
endif()

# List sources here
list(APPEND IVN_SOURCES
	${GUI_SOURCES}
	src/main.cu
)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR})

get_filename_component(CUDA_COMPILER_BIN "${CMAKE_CUDA_COMPILER}" DIRECTORY)

if (IVN_OPTIX)
	# add_library(optix_program OBJECT
	# 	src/optix/pathescape.cu
	# 	src/optix/raystab.cu
	# 	src/optix/raytrace.cu
	# )

	# set_target_properties(optix_program PROPERTIES CUDA_PTX_COMPILATION ON CUDA_ARCHITECTURES OFF)
	# target_compile_definitions(optix_program PUBLIC ${IVN_DEFINITIONS} -DTCNN_MIN_GPU_ARCH=0)
	# target_compile_options(optix_program PUBLIC "--expt-relaxed-constexpr")

	get_target_property(TCNN_INCLUDE_DIRECTORIES tiny-cuda-nn INCLUDE_DIRECTORIES)
	# target_include_directories(optix_program PUBLIC ${IVN_INCLUDE_DIRECTORIES} ${TCNN_INCLUDE_DIRECTORIES})

	# OptiX programs will be compiled as PTX and packaged
	# as headers to be included from the binary dir.
	list(APPEND IVN_INCLUDE_DIRECTORIES "${CMAKE_CURRENT_BINARY_DIR}")
	set(OPTIX_PTX_HEADER ${CMAKE_CURRENT_BINARY_DIR}/optix_ptx.h)

	find_program(bin_to_c NAMES bin2c PATHS ${CUDA_COMPILER_BIN})
	if (NOT bin_to_c)
		message(FATAL_ERROR
			"bin2c not found:\n"
			"  CMAKE_CUDA_COMPILER='${CMAKE_CUDA_COMPILER}'\n"
			"  CUDA_COMPILER_BIN='${CUDA_COMPILER_BIN}'\n"
		)
	endif()

	add_custom_command(
		OUTPUT "${OPTIX_PTX_HEADER}"
		COMMAND ${CMAKE_COMMAND}
		"-DBIN_TO_C_COMMAND=${bin_to_c}"
		# "-DOBJECTS=$<TARGET_OBJECTS:optix_program>"
		"-DOUTPUT=${OPTIX_PTX_HEADER}"
		-P ${PROJECT_SOURCE_DIR}/cmake/bin2c_wrapper.cmake
		VERBATIM
		# DEPENDS optix_program
		COMMENT "Converting PTX files to a C header"
	)

	list(APPEND IVN_SOURCES ${OPTIX_PTX_HEADER})
endif()


add_library(ivn STATIC ${IVN_SOURCES})
set_target_properties(ivn PROPERTIES CUDA_RESOLVE_DEVICE_SYMBOLS ON CUDA_SEPARABLE_COMPILATION ON)
target_compile_definitions(ivn PUBLIC ${IVN_DEFINITIONS})
target_compile_options(ivn PUBLIC $<$<COMPILE_LANGUAGE:CUDA>:${CUDA_NVCC_FLAGS}>)
target_include_directories(ivn PUBLIC ${IVN_INCLUDE_DIRECTORIES})
target_link_directories(ivn PUBLIC ${IVN_LINK_DIRECTORIES})
target_link_libraries(ivn PUBLIC ${IVN_LIBRARIES} tiny-cuda-nn)

add_executable(testbed src/main.cu)
target_link_libraries(testbed PRIVATE ivn)