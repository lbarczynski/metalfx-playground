if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/main.cpp")
    file(GLOB APP_SOURCES CONFIGURE_DEPENDS "*.cpp" "*.mm" "platform/*.mm")
    add_executable(metalfx_app
        ${APP_SOURCES}
    )

    target_include_directories(metalfx_app PRIVATE
        "${CMAKE_CURRENT_SOURCE_DIR}"
        "${CMAKE_CURRENT_SOURCE_DIR}/platform"
    )

    target_link_libraries(metalfx_app PRIVATE
        project_warnings
        metal_cpp_interface
    )

    if(TARGET metal_impl)
        target_link_libraries(metalfx_app PRIVATE metal_impl)
    endif()
else()
    message(STATUS "src/main.cpp not found. Target 'metalfx_app' will be configured when created in TASK-01.")
endif()
