# Get sources
file(GLOB_RECURSE SOURCES_GLOB
        "src/api/*.c"
        "src/collections/*.c"
        "src/link/*.c"
        "src/net/*.c"
        "src/protocol/*.c"
        "src/session/*.c"
        "src/transport/*.c"
        "src/utils/*.c"
        "src/system/espidf/*.c"
        "src/system/common/*.c"
        "src/system/socket/*.c"
        "src/runtime/*.c"
)

# Include directories
set(INCLUDES
        "include"
        "include/zenoh-pico"
)

idf_component_register(
        SRCS
            ${SOURCES_GLOB}

        INCLUDE_DIRS
            ${INCLUDES}

        REQUIRES
            esp_driver_uart
)

target_compile_definitions(${COMPONENT_LIB}
        PUBLIC
            ZENOH_ESPIDF
)