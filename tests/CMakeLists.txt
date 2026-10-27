# -----------------------------------------------------------------------------
# Catch2 Dependency (vcpkg or Automated FetchContent Fallback)
# -----------------------------------------------------------------------------
find_package(Catch2 3 CONFIG QUIET)

if(NOT Catch2_FOUND)
    message(STATUS "Catch2 not found via vcpkg/system. Fetching Catch2 v3.7.1 from GitHub...")
    include(FetchContent)
    FetchContent_Declare(
        Catch2
        URL https://github.com/catchorg/Catch2/archive/refs/tags/v3.7.1.tar.gz
        DOWNLOAD_EXTRACT_TIMESTAMP TRUE
    )
    FetchContent_MakeAvailable(Catch2)
endif()

# -----------------------------------------------------------------------------
# Test Target
# -----------------------------------------------------------------------------
file(GLOB TEST_SOURCES CONFIGURE_DEPENDS "*.cpp")

if(TEST_SOURCES)
    add_executable(unit_tests
        ${TEST_SOURCES}
    )

    target_link_libraries(unit_tests PRIVATE
        Catch2::Catch2WithMain
        project_warnings
        metal_impl
    )

    # Automated test discovery for CTest and CLion/Xcode runners
    include(CTest)
    include(Catch)
    catch_discover_tests(unit_tests)
else()
    message(STATUS "No test source files found in tests/. Target 'unit_tests' will be generated when .cpp files are added.")
endif()
