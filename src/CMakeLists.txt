add_executable(metalfx_app
    main.cpp
)

target_link_libraries(metalfx_app PRIVATE
    project_warnings
    metal_impl
)
