set(TARGET_NAME gzip)

# High bandwidth variant
set(SOURCE_FILE gzip.cpp crc32.cpp WriteGzip.cpp CompareGzip.cpp)
set(EMULATOR_TARGET ${TARGET_NAME}.fpga_emu)
set(FPGA_TARGET ${TARGET_NAME}.fpga)
set(DEVICE_SOURCE_FILE gzipkernel.cpp)
set(DEVICE_HEADER_FILE gzipkernel.hpp)
set(HIGH_BW_SEED "-Xsseed=8")

# Low latency variant
set(SOURCE_FILE_LL gzip_ll.cpp crc32.cpp WriteGzip.cpp CompareGzip.cpp)
set(EMULATOR_TARGET_LL ${TARGET_NAME}_ll.fpga_emu)
set(FPGA_TARGET_LL ${TARGET_NAME}_ll.fpga)
set(DEVICE_SOURCE_FILE_LL gzipkernel_ll.cpp)
set(DEVICE_HEADER_FILE_LL gzipkernel_ll.hpp)
set(LL_SEED "-Xsseed=1")

# FPGA board selection
if(NOT DEFINED FPGA_BOARD)
    set(FPGA_BOARD "intel_a10gx_pac:pac_a10")
    message(STATUS "FPGA_BOARD was not specified.\
                    \nConfiguring the design to run on the default FPGA board ${FPGA_BOARD} (Intel(R) PAC with Intel Arria(R) 10 GX FPGA). \
                    \nPlease refer to the README for information on board selection.")
else()
    message(STATUS "Configuring the design to run on FPGA board ${FPGA_BOARD}")
endif()

# This is a Windows-specific flag that enables error handling in host code
if(WIN32)
    set(WIN_FLAG "/EHsc")
endif()

# Set design parameters according to the selected chip
if(FPGA_BOARD MATCHES ".*a10.*")
    # A10 parameters
    set(NUM_ENGINES 1)
    set(LL_SEED "-Xsseed=1")
    set(HIGH_BW_SEED "-Xsseed=8")
    set(NUM_REORDER "")
elseif(FPGA_BOARD MATCHES ".*s10.*")
    # S10 parameters
    set(NUM_ENGINES 2)
    set(LL_SEED "-Xsseed=1")
    set(HIGH_BW_SEED "-Xsseed=8")
    set(NUM_REORDER "-Xsnum-reorder=6")
elseif(FPGA_BOARD MATCHES ".*agilex.*")
    # Agilex
    set(NUM_ENGINES 2)
    set(LL_SEED "-Xsseed=1")
    set(HIGH_BW_SEED "-Xsseed=8")
    # For the High Bandwidth variant, specify 6 reordering units to improve global memory read bandwidth across 4 channels of DDR.
    # For Low Latency variant this is not necessary since only one channel of global memory is used (host memory).
    set(NUM_REORDER "-Xsnum-reorder=6")
else()
    message(FATAL_ERROR "Unknown board!")
endif()

# Enable low-latency variant
# Presence of USM host allocations (and whether to turn on enable the low-latency target) is detected automatically by
# looking at the name of the BSP, or manually by the user when running CMake.
# E.g., cmake .. -DUSER_ENABLE_USM=1
if(FPGA_BOARD MATCHES ".*usm.*" OR DEFINED USER_ENABLE_USM)
    set(USM_ENABLED 1)
    set(NUM_REORDER_LL "")
endif()

message(STATUS "NUM_ENGINES=${NUM_ENGINES}")
message(STATUS "LL_SEED=${LL_SEED}")
message(STATUS "HIGH_BW_SEED=${HIGH_BW_SEED}")
message(STATUS "NUM_REORDER=${NUM_REORDER}")
message(STATUS "NUM_REORDER_LL=${NUM_REORDER_LL}")

# A DPC++ ahead-of-time (AoT) compile processes the device code in two stages.
# 1. The "compile" stage compiles the device code to an intermediate representation (SPIR-V).
# 2. The "link" stage invokes the compiler's FPGA backend before linking.
#    For this reason, FPGA backend flags must be passed as link flags in CMake.
set(EMULATOR_COMPILE_FLAGS "-Wall ${WIN_FLAG} -fintelfpga -g0 -DNUM_ENGINES=${NUM_ENGINES} -DFPGA_EMULATOR")
set(EMULATOR_LINK_FLAGS "-fintelfpga -DNUM_ENGINES=${NUM_ENGINES}")
set(HARDWARE_COMPILE_FLAGS "-Wall ${WIN_FLAG} -MMD -fintelfpga -DNUM_ENGINES=${NUM_ENGINES}")
set(HARDWARE_LINK_FLAGS "-fintelfpga -Xshardware -Xsparallel=2 -Xsopt-arg=\"-nocaching\" -Xsboard=${FPGA_BOARD} -DNUM_ENGINES=${NUM_ENGINES} ${USER_HARDWARE_FLAGS}")
# use cmake -D USER_HARDWARE_FLAGS=<flags> to set extra flags for FPGA backend compilation

#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#
#                           High Bandwidth Variant                             #
#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#
###############################################################################
### FPGA Emulator
###############################################################################
add_executable(${EMULATOR_TARGET} ${SOURCE_FILE} ${DEVICE_SOURCE_FILE})
set_target_properties(${EMULATOR_TARGET} PROPERTIES COMPILE_FLAGS "${EMULATOR_COMPILE_FLAGS}")
set_target_properties(${EMULATOR_TARGET} PROPERTIES LINK_FLAGS "${EMULATOR_LINK_FLAGS}")
add_custom_target(fpga_emu DEPENDS ${EMULATOR_TARGET})

###############################################################################
### Generate Report
###############################################################################
set(FPGA_EARLY_IMAGE ${TARGET_NAME}_report.a)
# The compile output is not an executable, but an intermediate compilation result unique to DPC++.
add_executable(${FPGA_EARLY_IMAGE} ${SOURCE_FILE} ${DEVICE_SOURCE_FILE})
add_custom_target(report DEPENDS ${FPGA_EARLY_IMAGE})
set_target_properties(${FPGA_EARLY_IMAGE} PROPERTIES COMPILE_FLAGS "${HARDWARE_COMPILE_FLAGS}")
set_target_properties(${FPGA_EARLY_IMAGE} PROPERTIES LINK_FLAGS "${HARDWARE_LINK_FLAGS} ${NUM_REORDER} ${HIGH_BW_SEED} -fsycl-link=early")
# fsycl-link=early stops the compiler after RTL generation, before invoking Quartus

###############################################################################
### FPGA Hardware
###############################################################################
add_executable(${FPGA_TARGET} EXCLUDE_FROM_ALL ${SOURCE_FILE} ${DEVICE_SOURCE_FILE})
add_custom_target(fpga DEPENDS ${FPGA_TARGET})
set_target_properties(${FPGA_TARGET} PROPERTIES COMPILE_FLAGS "${HARDWARE_COMPILE_FLAGS}")
set_target_properties(${FPGA_TARGET} PROPERTIES LINK_FLAGS "${HARDWARE_LINK_FLAGS} ${NUM_REORDER} ${HIGH_BW_SEED} -reuse-exe=${CMAKE_BINARY_DIR}/${FPGA_TARGET}")
# The -reuse-exe flag enables rapid recompilation of host-only code changes.
# See DPC++FPGA/GettingStarted/fast_recompile for details.
#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#



#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#
#                            Low Latency Variant                               #
#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#
if (${USM_ENABLED})
###############################################################################
### FPGA Emulator
###############################################################################
add_executable(${EMULATOR_TARGET_LL} ${SOURCE_FILE_LL} ${DEVICE_SOURCE_FILE_LL})
set_target_properties(${EMULATOR_TARGET_LL} PROPERTIES COMPILE_FLAGS "${EMULATOR_COMPILE_FLAGS}")
set_target_properties(${EMULATOR_TARGET_LL} PROPERTIES LINK_FLAGS "${EMULATOR_LINK_FLAGS}")
add_custom_target(fpga_emu_ll DEPENDS ${EMULATOR_TARGET_LL})

###############################################################################
### Generate Report
###############################################################################
set(FPGA_EARLY_IMAGE_LL ${TARGET_NAME}_ll_report.a)
# The compile output is not an executable, but an intermediate compilation result unique to DPC++.
add_executable(${FPGA_EARLY_IMAGE_LL} ${SOURCE_FILE_LL} ${DEVICE_SOURCE_FILE_LL})
add_custom_target(report_ll DEPENDS ${FPGA_EARLY_IMAGE_LL})
set_target_properties(${FPGA_EARLY_IMAGE_LL} PROPERTIES COMPILE_FLAGS "${HARDWARE_COMPILE_FLAGS}")
set_target_properties(${FPGA_EARLY_IMAGE_LL} PROPERTIES LINK_FLAGS "${HARDWARE_LINK_FLAGS} ${NUM_REORDER_LL} ${LL_SEED} -fsycl-link=early")
# fsycl-link=early stops the compiler after RTL generation, before invoking Quartus

###############################################################################
### FPGA Hardware
###############################################################################
add_executable(${FPGA_TARGET_LL} EXCLUDE_FROM_ALL ${SOURCE_FILE_LL} ${DEVICE_SOURCE_FILE_LL})
add_custom_target(fpga_ll DEPENDS ${FPGA_TARGET_LL})
set_target_properties(${FPGA_TARGET_LL} PROPERTIES COMPILE_FLAGS "${HARDWARE_COMPILE_FLAGS}")
set_target_properties(${FPGA_TARGET_LL} PROPERTIES LINK_FLAGS "${HARDWARE_LINK_FLAGS} ${NUM_REORDER_LL} ${LL_SEED} -reuse-exe=${CMAKE_BINARY_DIR}/${FPGA_TARGET_LL}")
# The -reuse-exe flag enables rapid recompilation of host-only code changes.
# See DPC++FPGA/GettingStarted/fast_recompile for details.
endif()
#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#
#//////////////////////////////////////////////////////////////////////////////#