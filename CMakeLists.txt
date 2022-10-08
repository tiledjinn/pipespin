cmake_minimum_required(VERSION 3.23)
project(pipespin C)

set(CMAKE_C_STANDARD 99)

file(GLOB pipespin_SRC
        "source/*.c"
        "include/*.h"
        )

if (GBA)
    set(CMAKE_C_FLAGS -DGBA)
    gba_add_library_subdirectory(rom)

    include_directories(BEFORE SYSTEM "platform/gba/sys_inc")
    include_directories(BEFORE SYSTEM "C:/devkitPro/libgba/include")
    include_directories(BEFORE SYSTEM "C:/devkitPro/devkitARM/arm-none-eabi/include")
endif(GBA)


add_custom_command(OUTPUT "generated/soundbank.h" "generated/soundbank.bin"
        COMMAND "c:/devkitPro/tools/bin/mmutil.exe" "${CMAKE_CURRENT_SOURCE_DIR}/maxmod_data/*" "-ogenerated/soundbank.bin" "-hgenerated/soundbank.h"
        )
add_custom_command(OUTPUT "generated/soundbank_bin.h" "generated/soundbank.s"
        COMMAND "c:/devkitPro/tools/bin/bin2s.exe" -a 4 -H ${CMAKE_CURRENT_BINARY_DIR}/generated/soundbank_bin.h ${CMAKE_CURRENT_BINARY_DIR}/generated/soundbank.bin > ${CMAKE_CURRENT_BINARY_DIR}/generated/soundbank.s
        DEPENDS "generated/soundbank.bin"
        )
add_custom_command(OUTPUT "generated/Ruby_pcx.h" "generated/Ruby_pcx.s"
        COMMAND "c:/devkitPro/tools/bin/bin2s.exe" -a 4 -H ${CMAKE_CURRENT_BINARY_DIR}/generated/Ruby_pcx.h ${CMAKE_CURRENT_SOURCE_DIR}/data/Ruby.pcx > ${CMAKE_CURRENT_BINARY_DIR}/generated/Ruby_pcx.s
        )
add_custom_command(OUTPUT "generated/Ruby_pcx.o"
        COMMAND "c:/devkitPro/devkitARM/bin/arm-none-eabi-as" -o generated/Ruby_pcx.o ${CMAKE_CURRENT_BINARY_DIR}/generated/Ruby_pcx.s
        DEPENDS "generated/Ruby_pcx.s"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        )
add_custom_command(OUTPUT "generated/soundbank.o"
        COMMAND "c:/devkitPro/devkitARM/bin/arm-none-eabi-as" -o generated/soundbank.o ${CMAKE_CURRENT_BINARY_DIR}/generated/soundbank.s
        DEPENDS "generated/soundbank.s"
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        )


file(GLOB grit_files "gfx/*.bmp")
foreach(var IN LISTS grit_files)
    file(RELATIVE_PATH PATH_VAR ${CMAKE_CURRENT_SOURCE_DIR}/gfx ${var})

    string(REPLACE ".bmp" ".h" OUT_VAR ${PATH_VAR})
    string(REPLACE ".bmp" ".c" SRC_VAR ${PATH_VAR})

    list(APPEND generated_SRC "${CMAKE_CURRENT_BINARY_DIR}/generated/${OUT_VAR}")
    list(APPEND generated_SRC "${CMAKE_CURRENT_BINARY_DIR}/generated/${SRC_VAR}")

    add_custom_command(OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/generated/${OUT_VAR}" "${CMAKE_CURRENT_BINARY_DIR}/generated/${SRC_VAR}"
            COMMAND    "c:/devkitPro/tools/bin/grit.exe" ${var}
            WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/generated
            )
endforeach()

add_executable(pipespin
        ${pipespin_SRC} ${platform_SRC} ${generated_SRC}
        generated/soundbank.h generated/soundbank_bin.h generated/soundbank.o
        generated/Ruby_pcx.h generated/Ruby_pcx.o)
target_include_directories(pipespin PUBLIC ${platform_INC} PUBLIC "include")
target_include_directories(pipespin PUBLIC "${CMAKE_CURRENT_BINARY_DIR}/generated")

if (GBA)
    target_link_directories(pipespin PUBLIC "C:/devkitPro/libgba/lib" "C:/devkitPro/devkitARM/arm-none-eabi/lib")
    target_link_libraries(pipespin PUBLIC mm)
    target_link_libraries(pipespin PUBLIC gba)
    target_link_libraries(pipespin PRIVATE rom)
    target_link_libraries(pipespin PRIVATE sysbase)

    set_target_properties(pipespin PROPERTIES SUFFIX ".elf")
    target_compile_options(pipespin PRIVATE -mthumb -Wall)


    gba_target_objcopy(pipespin FIX_HEADER)
endif(GBA)

