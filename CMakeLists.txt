# Include directories
zephyr_include_directories(include)

# ZEN shield support (sensor drivers and trackpad initialization).
if(CONFIG_SHIELD_ZEN_LEFT OR CONFIG_SHIELD_ZEN_RIGHT)
    zephyr_include_directories(${APPLICATION_SOURCE_DIR}/include)
    zephyr_include_directories(${CMAKE_CURRENT_SOURCE_DIR}/drivers/iqs7211e/include)
    target_sources(app PRIVATE src/mini_trackpad_init_reg.c)

    if(CONFIG_PMW3610_ALT)
        target_sources(app PRIVATE drivers/pmw3610_alt/src/pmw3610.c)
    endif()

    if(CONFIG_IQS7211E)
        target_sources(app PRIVATE
            drivers/iqs7211e/src/iqs7211e.c
            drivers/iqs7211e/src/iqs7211e_reg.c
        )
    endif()
endif()

if(CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR)
    # Add runtime input processor source
    target_sources(app PRIVATE src/pointing/input_processor_runtime.c)
    target_sources(app PRIVATE src/behaviors/behavior_input_processor_temp_config.c)
    target_sources(app PRIVATE src/behaviors/behavior_input_processor_temp_layer_keep_active.c)
    target_sources(app PRIVATE src/behaviors/behavior_input_processor_axis_snap.c)
    target_sources(app PRIVATE src/events/input_processor_state_changed.c)

    if(CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR_TEST)
        target_sources(app PRIVATE src/test/rip_settings_test.c)
    endif()

    if(CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR_STUDIO_RPC)
        file(GLOB_RECURSE C_FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/studio/*.c)
        target_sources(app PRIVATE ${C_FILES})

        list(APPEND CMAKE_MODULE_PATH ${ZEPHYR_BASE}/modules/nanopb)
        include(nanopb)
        set(NANOPB_GENERATE_CPP_APPEND_PATH TRUE)
        set(NANOPB_GENERATE_CPP_STANDALONE OFF)

        # NOTE: adding to app target directly causes build issues, so we create a library instead
        zephyr_library()
        file(GLOB_RECURSE PROTO_FILES ${CMAKE_CURRENT_SOURCE_DIR}/proto/*.proto)

        nanopb_generate_cpp(proto_srcs proto_hdrs RELPATH ${CMAKE_CURRENT_SOURCE_DIR} ${PROTO_FILES})
        target_include_directories(${ZEPHYR_CURRENT_LIBRARY} PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
        target_sources(${ZEPHYR_CURRENT_LIBRARY} PRIVATE ${proto_srcs} ${proto_hdrs})

        # app should depend on the generated proto files
        target_include_directories(app PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/proto)
        add_dependencies(app ${ZEPHYR_CURRENT_LIBRARY})
    endif()
endif()
