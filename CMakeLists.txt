cmake_minimum_required(VERSION 3.29)
project(VulkanEffects LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Dependencies (installed via vcpkg, Vulkan SDK from LunarG)
find_package(Vulkan 1.4.335 REQUIRED)
find_package(glfw3 CONFIG REQUIRED)
find_package(glm CONFIG REQUIRED)
find_package(Ktx CONFIG REQUIRED)

# Header only
find_path(TINYGLTF_INCLUDE_DIR "tiny_gltf.h" REQUIRED)
find_path(STB_INCLUDE_DIR "stb_image.h" REQUIRED)

add_executable(app
    src/main.cpp
    src/tinygltf_stb_impl.cpp
)

target_link_libraries(app PRIVATE
    Vulkan::Vulkan
    glfw
    glm::glm
    KTX::ktx
)

target_include_directories(app PRIVATE
    ${TINYGLTF_INCLUDE_DIR}
    ${STB_INCLUDE_DIR}
)

# Allow designated initializers
target_compile_definitions(app PRIVATE VULKAN_HPP_NO_STRUCT_CONSTRUCTORS)

# Shaders: compile shaders/*.slang -> shaders/<name>.slang.spv (only when the source changed)
find_program(SLANGC slangc HINTS "$ENV{VULKAN_SDK}/Bin" REQUIRED)
file(GLOB SHADER_SOURCES CONFIGURE_DEPENDS ${CMAKE_SOURCE_DIR}/shaders/*.slang)

foreach(shader ${SHADER_SOURCES})
    get_filename_component(shader_name ${shader} NAME)
    set(spv ${CMAKE_SOURCE_DIR}/shaders/${shader_name}.spv)
    add_custom_command(
        OUTPUT ${spv}
        COMMAND ${SLANGC} ${shader} -target spirv -o ${spv}
        DEPENDS ${shader}
        COMMENT "Slang: ${shader_name}"
        VERBATIM)
    list(APPEND SHADER_SPVS ${spv})
endforeach()

# ...then copy the .spv files next to the exe (only files that differ get copied)
if(SHADER_SPVS)
    add_custom_target(shaders ALL
        DEPENDS ${SHADER_SPVS}
        COMMAND ${CMAKE_COMMAND} -E make_directory $<TARGET_FILE_DIR:app>/shaders
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${SHADER_SPVS} $<TARGET_FILE_DIR:app>/shaders
        COMMENT "Copying shaders"
        VERBATIM)
    add_dependencies(app shaders)
endif()

# Assets: models and textures are copied next to the executable
add_custom_target(copy_assets ALL
    COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different
        ${CMAKE_SOURCE_DIR}/assets/models   $<TARGET_FILE_DIR:app>/assets/models
    COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different
        ${CMAKE_SOURCE_DIR}/assets/textures $<TARGET_FILE_DIR:app>/assets/textures
    COMMENT "Copying assets")
add_dependencies(app copy_assets)
