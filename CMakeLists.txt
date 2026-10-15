cmake_minimum_required(VERSION 3.29)
project(VulkanEffects LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find modules copied from the Vulkan Tutorial (attachments/CMake):
# Findtinygltf.cmake, FindKTX.cmake, Findstb.cmake, FindVulkan.cmake
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/CMake")

# Dependencies (installed via vcpkg, Vulkan SDK from LunarG)
find_package(Vulkan 1.4.335 REQUIRED)
find_package(glfw3 REQUIRED)
find_package(glm REQUIRED)
find_package(tinygltf REQUIRED)
find_package(KTX REQUIRED)
find_package(stb REQUIRED)

# Slang shader compiler from the Vulkan SDK
find_program(SLANGC_EXECUTABLE slangc HINTS $ENV{VULKAN_SDK}/bin REQUIRED)

add_executable(app src/main.cpp)

# Visual Studio: start `app` by default and run it from the exe folder
# (the program loads shaders/, models/ and textures/ relative to it)
set_target_properties(app PROPERTIES VS_DEBUGGER_WORKING_DIRECTORY "$<TARGET_FILE_DIR:app>")
set_property(DIRECTORY PROPERTY VS_STARTUP_PROJECT app)

target_link_libraries(app PRIVATE
    Vulkan::Vulkan
    glfw
    glm::glm
    tinygltf::tinygltf
    KTX::ktx)

target_include_directories(app PRIVATE ${stb_INCLUDE_DIRS})

target_compile_definitions(app PRIVATE
    VULKAN_HPP_DISPATCH_LOADER_DYNAMIC=1
    VULKAN_HPP_NO_STRUCT_CONSTRUCTORS=1
    VULKAN_HPP_HANDLE_ERROR_OUT_OF_DATE_AS_SUCCESS)

# --- Shaders: compiled on every build into <exe dir>/shaders/slang.spv ---
set(SHADER_SRC ${CMAKE_CURRENT_SOURCE_DIR}/shaders/shader.slang)
add_custom_target(shaders
    COMMAND ${CMAKE_COMMAND} -E make_directory $<TARGET_FILE_DIR:app>/shaders
    COMMAND ${SLANGC_EXECUTABLE} ${SHADER_SRC}
            -target spirv -profile spirv_1_4+spvRayQueryKHR
            -emit-spirv-directly -fvk-use-entrypoint-name
            -entry vertMain -entry fragMain
            -o $<TARGET_FILE_DIR:app>/shaders/slang.spv
    DEPENDS ${SHADER_SRC}
    COMMENT "Compiling Slang shader"
    VERBATIM)
add_dependencies(app shaders)

# --- Assets: models and textures are copied next to the executable ---
add_custom_target(assets
    COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different
            ${CMAKE_CURRENT_SOURCE_DIR}/assets/models $<TARGET_FILE_DIR:app>/models
    COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different
            ${CMAKE_CURRENT_SOURCE_DIR}/assets/textures $<TARGET_FILE_DIR:app>/textures
    COMMENT "Copying assets"
    VERBATIM)
add_dependencies(app assets)