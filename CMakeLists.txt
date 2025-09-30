# Get sources
file(GLOB_RECURSE SOURCES_GLOB
        "src/api/*.c"
        "src/collections/*.c"
        "src/link/*.c"
        "src/net/*.c"
        "src/protocol/*.c"
        "src/session/*.c"
        "transport/*.c"
        "src/utils/*.c"
        "src/system/espidf/*.c"
        "src/system/common/*.c"

)

# Include directories
set(INCLUDES
        "include/"
        "include/zenoh-pico/"
)

# Register component
idf_component_register(
            SRCS
                ${SOURCES_GLOB}
            INCLUDE_DIRS
                ${INCLUDES}
            REQUIRES
                driver
            )

# Set compile flags
idf_build_set_property(COMPILE_OPTIONS "-DZENOH_ESPIDF" APPEND)
