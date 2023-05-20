cmake_minimum_required(VERSION 3.16.0 FATAL_ERROR)

set(CMAKE_SYSTEM_VERSION 10.0 CACHE STRING "" FORCE)

project(soh LANGUAGES C CXX)
set(CMAKE_CXX_STANDARD 20 CACHE STRING "The C++ standard to use")

if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    enable_language(OBJCXX)
    set(CMAKE_OBJC_FLAGS "${CMAKE_OBJC_FLAGS} -fobjc-arc")
    set(CMAKE_OBJCXX_FLAGS "${CMAKE_OBJCXX_FLAGS} -fobjc-arc")
endif()

set (BUILD_UTILS OFF CACHE STRING "no utilities")
set (BUILD_SHARED_LIBS OFF CACHE STRING "install/link shared instead of static libs")

################################################################################
# Set target arch type if empty. Visual studio solution generator provides it.
################################################################################
if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
	if(NOT CMAKE_VS_PLATFORM_NAME)
		set(CMAKE_VS_PLATFORM_NAME "x64")
	endif()
	message("${CMAKE_VS_PLATFORM_NAME} architecture in use")

	if(NOT ("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x64"
		OR "${CMAKE_VS_PLATFORM_NAME}" STREQUAL "Win32"))
		message(FATAL_ERROR "${CMAKE_VS_PLATFORM_NAME} arch is not supported!")
	endif()
endif()

################################################################################
# Global configuration types
################################################################################
set(CMAKE_CONFIGURATION_TYPES
    "Debug"
    "Release"
    CACHE STRING "" FORCE
)

################################################################################
# Global compiler options
################################################################################
if(MSVC)
    # remove default flags provided with CMake for MSVC
    set(CMAKE_C_FLAGS "")
    set(CMAKE_C_FLAGS_DEBUG "")
    set(CMAKE_C_FLAGS_RELEASE "")
    set(CMAKE_CXX_FLAGS "")
    set(CMAKE_CXX_FLAGS_DEBUG "")
    set(CMAKE_CXX_FLAGS_RELEASE "")
endif()

################################################################################
# Global linker options
################################################################################
if(MSVC)
    # remove default flags provided with CMake for MSVC
    set(CMAKE_EXE_LINKER_FLAGS "")
    set(CMAKE_MODULE_LINKER_FLAGS "")
    set(CMAKE_SHARED_LINKER_FLAGS "")
    set(CMAKE_STATIC_LINKER_FLAGS "")
    set(CMAKE_EXE_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS}")
    set(CMAKE_MODULE_LINKER_FLAGS_DEBUG "${CMAKE_MODULE_LINKER_FLAGS}")
    set(CMAKE_SHARED_LINKER_FLAGS_DEBUG "${CMAKE_SHARED_LINKER_FLAGS}")
    set(CMAKE_STATIC_LINKER_FLAGS_DEBUG "${CMAKE_STATIC_LINKER_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS}")
    set(CMAKE_MODULE_LINKER_FLAGS_RELEASE "${CMAKE_MODULE_LINKER_FLAGS}")
    set(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS}")
    set(CMAKE_STATIC_LINKER_FLAGS_RELEASE "${CMAKE_STATIC_LINKER_FLAGS}")
endif()

################################################################################
# Common utils
################################################################################
include(CMake/Utils.cmake)

################################################################################
# Additional Global Settings(add specific info there)
################################################################################
include(CMake/GlobalSettingsInclude.cmake OPTIONAL)

################################################################################
# Use solution folders feature
################################################################################
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

################################################################################
# Sub-projects
################################################################################
if (NOT TARGET libultraship)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../libultraship ${CMAKE_BINARY_DIR}/libultraship)
endif()

if (NOT TARGET ZAPDUtils)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../ZAPDTR/ZAPDUtils ${CMAKE_BINARY_DIR}/ZAPDUtils)
endif()

if (NOT TARGET ZAPDLib)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/../ZAPDTR/ZAPD ${CMAKE_BINARY_DIR}/ZAPD)
endif()

set(PROJECT_NAME soh)

################################################################################
# Sources
################################################################################
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/src/boot/build.c.in ${CMAKE_BINARY_DIR}/build.c @ONLY)
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/src/boot/properties.h.in ${CMAKE_CURRENT_SOURCE_DIR}/properties.h @ONLY)

set(Header_Files "resource.h")
source_group("headers" FILES ${Header_Files})

# include {{{
file(GLOB Header_Files__include "include/*.h" "include/*.inc")
list(APPEND Header_Files__include ${CMAKE_CURRENT_SOURCE_DIR}/include/libc/stdarg.h)
list(REMOVE_ITEM Header_Files__include ${CMAKE_CURRENT_SOURCE_DIR}/include/bgm.h)
list(REMOVE_ITEM Header_Files__include ${CMAKE_CURRENT_SOURCE_DIR}/include/math_n64.h)
list(REMOVE_ITEM Header_Files__include ${CMAKE_CURRENT_SOURCE_DIR}/include/stdbool_n64.h)
list(REMOVE_ITEM Header_Files__include ${CMAKE_CURRENT_SOURCE_DIR}/include/stddef_n64.h)
list(REMOVE_ITEM Header_Files__include ${CMAKE_CURRENT_SOURCE_DIR}/include/stdlib_n64.h)
list(REMOVE_ITEM Header_Files__include ${CMAKE_CURRENT_SOURCE_DIR}/include/ultra64.h)
source_group("include" FILES ${Header_Files__include})
# }}}

# soh (root)
file(GLOB soh__ RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "soh/*.c" "soh/*.cpp" "soh/*.h")
source_group("soh" FILES ${soh__})

if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    set_source_files_properties(soh/OTRGlobals.cpp PROPERTIES COMPILE_FLAGS "/utf-8")
endif()

# soh/enhancements {{{
file(GLOB_RECURSE soh__Enhancements RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}
    "soh/Enhancements/*.c"
    "soh/Enhancements/*.cpp"
    "soh/Enhancements/*.h"
    "soh/Enhancements/*.hpp"
    "soh/Enhancements/*_extern.inc"
    "soh/Enhancements/*.mm"
)

list(REMOVE_ITEM soh__Enhancements "soh/Enhancements/gamecommand.h")
list(FILTER soh__Enhancements EXCLUDE REGEX "soh/Enhancements/gfx.*")

# handle crowd control removals
list(REMOVE_ITEM soh__Enhancements "soh/Enhancements/crowd-control/soh.cs")
list(REMOVE_ITEM soh__Enhancements "soh/Enhancements/crowd-control/soh.ccpak")
if (!BUILD_CROWD_CONTROL)
    list(FILTER soh__Enhancements EXCLUDE REGEX "soh/Enhancements/crowd-control/*")
endif()

# handle speechsynthesizer removals
if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
    list(FILTER soh__Enhancements EXCLUDE REGEX "soh/Enhancements/speechsynthesizer/Darwin*")
elseif (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    list(FILTER soh__Enhancements EXCLUDE REGEX "soh/Enhancements/speechsynthesizer/SAPI*")
else()
    list(FILTER soh__Enhancements EXCLUDE REGEX "soh/Enhancements/speechsynthesizer/(Darwin|SAPI).*")
endif()

source_group("soh\\Enhancements" REGULAR_EXPRESSION "soh/Enhancements/*")
source_group("soh\\Enhancements\\audio" REGULAR_EXPRESSION "soh/Enhancements/audio/*")
source_group("soh\\Enhancements\\controls" REGULAR_EXPRESSION "soh/Enhancements/controls/*")
source_group("soh\\Enhancements\\cosmetics" REGULAR_EXPRESSION "soh/Enhancements/cosmetics/*")
source_group("soh\\Enhancements\\crowd-control" REGULAR_EXPRESSION "soh/Enhancements/crowd-control/*")
source_group("soh\\Enhancements\\custom-message" REGULAR_EXPRESSION "soh/Enhancements/custom-message/*")
source_group("soh\\Enhancements\\debugger" REGULAR_EXPRESSION "soh/Enhancements/debugger/*")
source_group("soh\\Enhancements\\game-interactor" REGULAR_EXPRESSION "soh/Enhancements/game-interactor/*")
source_group("soh\\Enhancements\\item-tables" REGULAR_EXPRESSION "soh/Enhancements/item-tables/*")
source_group("soh\\Enhancements\\randomizer" REGULAR_EXPRESSION "soh/Enhancements/randomizer/*")
source_group("soh\\Enhancements\\randomizer\\3drando" REGULAR_EXPRESSION "soh/Enhancements/randomizer/3drando/*")
source_group("soh\\Enhancements\\randomizer\\3drando\\hint_list" REGULAR_EXPRESSION "soh/Enhancements/randomizer/3drando/hint_list/*")
source_group("soh\\Enhancements\\randomizer\\3drando\\location_access" REGULAR_EXPRESSION "soh/Enhancements/randomizer/3drando/location_access/*")
source_group("soh\\Enhancements\\speechsynthesizer" REGULAR_EXPRESSION "soh/Enhancements/speechsynthesizer/*")
source_group("soh\\Enhancements\\tts" REGULAR_EXPRESSION "soh/Enhancements/tts/*")

if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    set_source_files_properties(soh/Enhancements/tts/tts.cpp PROPERTIES COMPILE_FLAGS "/utf-8")
endif()
# }}}

if(NOT CMAKE_SYSTEM_NAME MATCHES "NintendoSwitch|CafeOS")
    # soh/Extractor {{{
    file(GLOB_RECURSE soh__Extractor RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}
        "soh/Extractor/*.c"
        "soh/Extractor/*.cpp"
        "soh/Extractor/*.h"
        "soh/Extractor/*.hpp"
    )
    # }}}
else()
    file(GLOB_RECURSE soh__Extractor RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}
        "soh/Extractor/*.h"
        "soh/Extractor/*.hpp"
    )
# }}}
endif()

set(Source_Files__src__overlays__actors
    "src/overlays/actors/ovl_Arms_Hook/z_arms_hook.c"
    "src/overlays/actors/ovl_Arms_Hook/z_arms_hook.h"
    "src/overlays/actors/ovl_Arrow_Fire/z_arrow_fire.c"
    "src/overlays/actors/ovl_Arrow_Fire/z_arrow_fire.h"
    "src/overlays/actors/ovl_Arrow_Ice/z_arrow_ice.c"
    "src/overlays/actors/ovl_Arrow_Ice/z_arrow_ice.h"
    "src/overlays/actors/ovl_Arrow_Light/z_arrow_light.c"
    "src/overlays/actors/ovl_Arrow_Light/z_arrow_light.h"
    "src/overlays/actors/ovl_Bg_Bdan_Objects/z_bg_bdan_objects.c"
    "src/overlays/actors/ovl_Bg_Bdan_Objects/z_bg_bdan_objects.h"
    "src/overlays/actors/ovl_Bg_Bdan_Switch/z_bg_bdan_switch.c"
    "src/overlays/actors/ovl_Bg_Bdan_Switch/z_bg_bdan_switch.h"
    "src/overlays/actors/ovl_Bg_Bom_Guard/z_bg_bom_guard.c"
    "src/overlays/actors/ovl_Bg_Bom_Guard/z_bg_bom_guard.h"
    "src/overlays/actors/ovl_Bg_Bombwall/z_bg_bombwall.c"
    "src/overlays/actors/ovl_Bg_Bombwall/z_bg_bombwall.h"
    "src/overlays/actors/ovl_Bg_Bowl_Wall/z_bg_bowl_wall.c"
    "src/overlays/actors/ovl_Bg_Bowl_Wall/z_bg_bowl_wall.h"
    "src/overlays/actors/ovl_Bg_Breakwall/z_bg_breakwall.c"
    "src/overlays/actors/ovl_Bg_Breakwall/z_bg_breakwall.h"
    "src/overlays/actors/ovl_Bg_Ddan_Jd/z_bg_ddan_jd.c"
    "src/overlays/actors/ovl_Bg_Ddan_Jd/z_bg_ddan_jd.h"
    "src/overlays/actors/ovl_Bg_Ddan_Kd/z_bg_ddan_kd.c"
    "src/overlays/actors/ovl_Bg_Ddan_Kd/z_bg_ddan_kd.h"
    "src/overlays/actors/ovl_Bg_Dodoago/z_bg_dodoago.c"
    "src/overlays/actors/ovl_Bg_Dodoago/z_bg_dodoago.h"
    "src/overlays/actors/ovl_Bg_Dy_Yoseizo/z_bg_dy_yoseizo.c"
    "src/overlays/actors/ovl_Bg_Dy_Yoseizo/z_bg_dy_yoseizo.h"
    "src/overlays/actors/ovl_Bg_Ganon_Otyuka/z_bg_ganon_otyuka.c"
    "src/overlays/actors/ovl_Bg_Ganon_Otyuka/z_bg_ganon_otyuka.h"
    "src/overlays/actors/ovl_Bg_Gate_Shutter/z_bg_gate_shutter.c"
    "src/overlays/actors/ovl_Bg_Gate_Shutter/z_bg_gate_shutter.h"
    "src/overlays/actors/ovl_Bg_Gjyo_Bridge/z_bg_gjyo_bridge.c"
    "src/overlays/actors/ovl_Bg_Gjyo_Bridge/z_bg_gjyo_bridge.h"
    "src/overlays/actors/ovl_Bg_Gnd_Darkmeiro/z_bg_gnd_darkmeiro.c"
    "src/overlays/actors/ovl_Bg_Gnd_Darkmeiro/z_bg_gnd_darkmeiro.h"
    "src/overlays/actors/ovl_Bg_Gnd_Firemeiro/z_bg_gnd_firemeiro.c"
    "src/overlays/actors/ovl_Bg_Gnd_Firemeiro/z_bg_gnd_firemeiro.h"
    "src/overlays/actors/ovl_Bg_Gnd_Iceblock/z_bg_gnd_iceblock.c"
    "src/overlays/actors/ovl_Bg_Gnd_Iceblock/z_bg_gnd_iceblock.h"
    "src/overlays/actors/ovl_Bg_Gnd_Nisekabe/z_bg_gnd_nisekabe.c"
    "src/overlays/actors/ovl_Bg_Gnd_Nisekabe/z_bg_gnd_nisekabe.h"
    "src/overlays/actors/ovl_Bg_Gnd_Soulmeiro/z_bg_gnd_soulmeiro.c"
    "src/overlays/actors/ovl_Bg_Gnd_Soulmeiro/z_bg_gnd_soulmeiro.h"
    "src/overlays/actors/ovl_Bg_Haka/z_bg_haka.c"
    "src/overlays/actors/ovl_Bg_Haka/z_bg_haka.h"
    "src/overlays/actors/ovl_Bg_Haka_Gate/z_bg_haka_gate.c"
    "src/overlays/actors/ovl_Bg_Haka_Gate/z_bg_haka_gate.h"
    "src/overlays/actors/ovl_Bg_Haka_Huta/z_bg_haka_huta.c"
    "src/overlays/actors/ovl_Bg_Haka_Huta/z_bg_haka_huta.h"
    "src/overlays/actors/ovl_Bg_Haka_Megane/z_bg_haka_megane.c"
    "src/overlays/actors/ovl_Bg_Haka_Megane/z_bg_haka_megane.h"
    "src/overlays/actors/ovl_Bg_Haka_MeganeBG/z_bg_haka_meganebg.c"
    "src/overlays/actors/ovl_Bg_Haka_MeganeBG/z_bg_haka_meganebg.h"
    "src/overlays/actors/ovl_Bg_Haka_Sgami/z_bg_haka_sgami.c"
    "src/overlays/actors/ovl_Bg_Haka_Sgami/z_bg_haka_sgami.h"
    "src/overlays/actors/ovl_Bg_Haka_Ship/z_bg_haka_ship.c"
    "src/overlays/actors/ovl_Bg_Haka_Ship/z_bg_haka_ship.h"
    "src/overlays/actors/ovl_Bg_Haka_Trap/z_bg_haka_trap.c"
    "src/overlays/actors/ovl_Bg_Haka_Trap/z_bg_haka_trap.h"
    "src/overlays/actors/ovl_Bg_Haka_Tubo/z_bg_haka_tubo.c"
    "src/overlays/actors/ovl_Bg_Haka_Tubo/z_bg_haka_tubo.h"
    "src/overlays/actors/ovl_Bg_Haka_Water/z_bg_haka_water.c"
    "src/overlays/actors/ovl_Bg_Haka_Water/z_bg_haka_water.h"
    "src/overlays/actors/ovl_Bg_Haka_Zou/z_bg_haka_zou.c"
    "src/overlays/actors/ovl_Bg_Haka_Zou/z_bg_haka_zou.h"
    "src/overlays/actors/ovl_Bg_Heavy_Block/z_bg_heavy_block.c"
    "src/overlays/actors/ovl_Bg_Heavy_Block/z_bg_heavy_block.h"
    "src/overlays/actors/ovl_Bg_Hidan_Curtain/z_bg_hidan_curtain.c"
    "src/overlays/actors/ovl_Bg_Hidan_Curtain/z_bg_hidan_curtain.h"
    "src/overlays/actors/ovl_Bg_Hidan_Dalm/z_bg_hidan_dalm.c"
    "src/overlays/actors/ovl_Bg_Hidan_Dalm/z_bg_hidan_dalm.h"
    "src/overlays/actors/ovl_Bg_Hidan_Firewall/z_bg_hidan_firewall.c"
    "src/overlays/actors/ovl_Bg_Hidan_Firewall/z_bg_hidan_firewall.h"
    "src/overlays/actors/ovl_Bg_Hidan_Fslift/z_bg_hidan_fslift.c"
    "src/overlays/actors/ovl_Bg_Hidan_Fslift/z_bg_hidan_fslift.h"
    "src/overlays/actors/ovl_Bg_Hidan_Fwbig/z_bg_hidan_fwbig.c"
    "src/overlays/actors/ovl_Bg_Hidan_Fwbig/z_bg_hidan_fwbig.h"
    "src/overlays/actors/ovl_Bg_Hidan_Hamstep/z_bg_hidan_hamstep.c"
    "src/overlays/actors/ovl_Bg_Hidan_Hamstep/z_bg_hidan_hamstep.h"
    "src/overlays/actors/ovl_Bg_Hidan_Hrock/z_bg_hidan_hrock.c"
    "src/overlays/actors/ovl_Bg_Hidan_Hrock/z_bg_hidan_hrock.h"
    "src/overlays/actors/ovl_Bg_Hidan_Kousi/z_bg_hidan_kousi.c"
    "src/overlays/actors/ovl_Bg_Hidan_Kousi/z_bg_hidan_kousi.h"
    "src/overlays/actors/ovl_Bg_Hidan_Kowarerukabe/z_bg_hidan_kowarerukabe.c"
    "src/overlays/actors/ovl_Bg_Hidan_Kowarerukabe/z_bg_hidan_kowarerukabe.h"
    "src/overlays/actors/ovl_Bg_Hidan_Rock/z_bg_hidan_rock.c"
    "src/overlays/actors/ovl_Bg_Hidan_Rock/z_bg_hidan_rock.h"
    "src/overlays/actors/ovl_Bg_Hidan_Rsekizou/z_bg_hidan_rsekizou.c"
    "src/overlays/actors/ovl_Bg_Hidan_Rsekizou/z_bg_hidan_rsekizou.h"
    "src/overlays/actors/ovl_Bg_Hidan_Sekizou/z_bg_hidan_sekizou.c"
    "src/overlays/actors/ovl_Bg_Hidan_Sekizou/z_bg_hidan_sekizou.h"
    "src/overlays/actors/ovl_Bg_Hidan_Sima/z_bg_hidan_sima.c"
    "src/overlays/actors/ovl_Bg_Hidan_Sima/z_bg_hidan_sima.h"
    "src/overlays/actors/ovl_Bg_Hidan_Syoku/z_bg_hidan_syoku.c"
    "src/overlays/actors/ovl_Bg_Hidan_Syoku/z_bg_hidan_syoku.h"
    "src/overlays/actors/ovl_Bg_Ice_Objects/z_bg_ice_objects.c"
    "src/overlays/actors/ovl_Bg_Ice_Objects/z_bg_ice_objects.h"
    "src/overlays/actors/ovl_Bg_Ice_Shelter/z_bg_ice_shelter.c"
    "src/overlays/actors/ovl_Bg_Ice_Shelter/z_bg_ice_shelter.h"
    "src/overlays/actors/ovl_Bg_Ice_Shutter/z_bg_ice_shutter.c"
    "src/overlays/actors/ovl_Bg_Ice_Shutter/z_bg_ice_shutter.h"
    "src/overlays/actors/ovl_Bg_Ice_Turara/z_bg_ice_turara.c"
    "src/overlays/actors/ovl_Bg_Ice_Turara/z_bg_ice_turara.h"
    "src/overlays/actors/ovl_Bg_Ingate/z_bg_ingate.c"
    "src/overlays/actors/ovl_Bg_Ingate/z_bg_ingate.h"
    "src/overlays/actors/ovl_Bg_Jya_1flift/z_bg_jya_1flift.c"
    "src/overlays/actors/ovl_Bg_Jya_1flift/z_bg_jya_1flift.h"
    "src/overlays/actors/ovl_Bg_Jya_Amishutter/z_bg_jya_amishutter.c"
    "src/overlays/actors/ovl_Bg_Jya_Amishutter/z_bg_jya_amishutter.h"
    "src/overlays/actors/ovl_Bg_Jya_Bigmirror/z_bg_jya_bigmirror.c"
    "src/overlays/actors/ovl_Bg_Jya_Bigmirror/z_bg_jya_bigmirror.h"
    "src/overlays/actors/ovl_Bg_Jya_Block/z_bg_jya_block.c"
    "src/overlays/actors/ovl_Bg_Jya_Block/z_bg_jya_block.h"
    "src/overlays/actors/ovl_Bg_Jya_Bombchuiwa/z_bg_jya_bombchuiwa.c"
    "src/overlays/actors/ovl_Bg_Jya_Bombchuiwa/z_bg_jya_bombchuiwa.h"
    "src/overlays/actors/ovl_Bg_Jya_Bombiwa/z_bg_jya_bombiwa.c"
    "src/overlays/actors/ovl_Bg_Jya_Bombiwa/z_bg_jya_bombiwa.h"
    "src/overlays/actors/ovl_Bg_Jya_Cobra/z_bg_jya_cobra.c"
    "src/overlays/actors/ovl_Bg_Jya_Cobra/z_bg_jya_cobra.h"
    "src/overlays/actors/ovl_Bg_Jya_Goroiwa/z_bg_jya_goroiwa.c"
    "src/overlays/actors/ovl_Bg_Jya_Goroiwa/z_bg_jya_goroiwa.h"
    "src/overlays/actors/ovl_Bg_Jya_Haheniron/z_bg_jya_haheniron.c"
    "src/overlays/actors/ovl_Bg_Jya_Haheniron/z_bg_jya_haheniron.h"
    "src/overlays/actors/ovl_Bg_Jya_Ironobj/z_bg_jya_ironobj.c"
    "src/overlays/actors/ovl_Bg_Jya_Ironobj/z_bg_jya_ironobj.h"
    "src/overlays/actors/ovl_Bg_Jya_Kanaami/z_bg_jya_kanaami.c"
    "src/overlays/actors/ovl_Bg_Jya_Kanaami/z_bg_jya_kanaami.h"
    "src/overlays/actors/ovl_Bg_Jya_Lift/z_bg_jya_lift.c"
    "src/overlays/actors/ovl_Bg_Jya_Lift/z_bg_jya_lift.h"
    "src/overlays/actors/ovl_Bg_Jya_Megami/z_bg_jya_megami.c"
    "src/overlays/actors/ovl_Bg_Jya_Megami/z_bg_jya_megami.h"
    "src/overlays/actors/ovl_Bg_Jya_Zurerukabe/z_bg_jya_zurerukabe.c"
    "src/overlays/actors/ovl_Bg_Jya_Zurerukabe/z_bg_jya_zurerukabe.h"
    "src/overlays/actors/ovl_Bg_Menkuri_Eye/z_bg_menkuri_eye.c"
    "src/overlays/actors/ovl_Bg_Menkuri_Eye/z_bg_menkuri_eye.h"
    "src/overlays/actors/ovl_Bg_Menkuri_Kaiten/z_bg_menkuri_kaiten.c"
    "src/overlays/actors/ovl_Bg_Menkuri_Kaiten/z_bg_menkuri_kaiten.h"
    "src/overlays/actors/ovl_Bg_Menkuri_Nisekabe/z_bg_menkuri_nisekabe.c"
    "src/overlays/actors/ovl_Bg_Menkuri_Nisekabe/z_bg_menkuri_nisekabe.h"
    "src/overlays/actors/ovl_Bg_Mizu_Bwall/z_bg_mizu_bwall.c"
    "src/overlays/actors/ovl_Bg_Mizu_Bwall/z_bg_mizu_bwall.h"
    "src/overlays/actors/ovl_Bg_Mizu_Movebg/z_bg_mizu_movebg.c"
    "src/overlays/actors/ovl_Bg_Mizu_Movebg/z_bg_mizu_movebg.h"
    "src/overlays/actors/ovl_Bg_Mizu_Shutter/z_bg_mizu_shutter.c"
    "src/overlays/actors/ovl_Bg_Mizu_Shutter/z_bg_mizu_shutter.h"
    "src/overlays/actors/ovl_Bg_Mizu_Uzu/z_bg_mizu_uzu.c"
    "src/overlays/actors/ovl_Bg_Mizu_Uzu/z_bg_mizu_uzu.h"
    "src/overlays/actors/ovl_Bg_Mizu_Water/z_bg_mizu_water.c"
    "src/overlays/actors/ovl_Bg_Mizu_Water/z_bg_mizu_water.h"
    "src/overlays/actors/ovl_Bg_Mjin/z_bg_mjin.c"
    "src/overlays/actors/ovl_Bg_Mjin/z_bg_mjin.h"
    "src/overlays/actors/ovl_Bg_Mori_Bigst/z_bg_mori_bigst.c"
    "src/overlays/actors/ovl_Bg_Mori_Bigst/z_bg_mori_bigst.h"
    "src/overlays/actors/ovl_Bg_Mori_Elevator/z_bg_mori_elevator.c"
    "src/overlays/actors/ovl_Bg_Mori_Elevator/z_bg_mori_elevator.h"
    "src/overlays/actors/ovl_Bg_Mori_Hashigo/z_bg_mori_hashigo.c"
    "src/overlays/actors/ovl_Bg_Mori_Hashigo/z_bg_mori_hashigo.h"
    "src/overlays/actors/ovl_Bg_Mori_Hashira4/z_bg_mori_hashira4.c"
    "src/overlays/actors/ovl_Bg_Mori_Hashira4/z_bg_mori_hashira4.h"
    "src/overlays/actors/ovl_Bg_Mori_Hineri/z_bg_mori_hineri.c"
    "src/overlays/actors/ovl_Bg_Mori_Hineri/z_bg_mori_hineri.h"
    "src/overlays/actors/ovl_Bg_Mori_Idomizu/z_bg_mori_idomizu.c"
    "src/overlays/actors/ovl_Bg_Mori_Idomizu/z_bg_mori_idomizu.h"
    "src/overlays/actors/ovl_Bg_Mori_Kaitenkabe/z_bg_mori_kaitenkabe.c"
    "src/overlays/actors/ovl_Bg_Mori_Kaitenkabe/z_bg_mori_kaitenkabe.h"
    "src/overlays/actors/ovl_Bg_Mori_Rakkatenjo/z_bg_mori_rakkatenjo.c"
    "src/overlays/actors/ovl_Bg_Mori_Rakkatenjo/z_bg_mori_rakkatenjo.h"
    "src/overlays/actors/ovl_Bg_Po_Event/z_bg_po_event.c"
    "src/overlays/actors/ovl_Bg_Po_Event/z_bg_po_event.h"
    "src/overlays/actors/ovl_Bg_Po_Syokudai/z_bg_po_syokudai.c"
    "src/overlays/actors/ovl_Bg_Po_Syokudai/z_bg_po_syokudai.h"
    "src/overlays/actors/ovl_Bg_Pushbox/z_bg_pushbox.c"
    "src/overlays/actors/ovl_Bg_Pushbox/z_bg_pushbox.h"
    "src/overlays/actors/ovl_Bg_Relay_Objects/z_bg_relay_objects.c"
    "src/overlays/actors/ovl_Bg_Relay_Objects/z_bg_relay_objects.h"
    "src/overlays/actors/ovl_Bg_Spot00_Break/z_bg_spot00_break.c"
    "src/overlays/actors/ovl_Bg_Spot00_Break/z_bg_spot00_break.h"
    "src/overlays/actors/ovl_Bg_Spot00_Hanebasi/z_bg_spot00_hanebasi.c"
    "src/overlays/actors/ovl_Bg_Spot00_Hanebasi/z_bg_spot00_hanebasi.h"
    "src/overlays/actors/ovl_Bg_Spot01_Fusya/z_bg_spot01_fusya.c"
    "src/overlays/actors/ovl_Bg_Spot01_Fusya/z_bg_spot01_fusya.h"
    "src/overlays/actors/ovl_Bg_Spot01_Idohashira/z_bg_spot01_idohashira.c"
    "src/overlays/actors/ovl_Bg_Spot01_Idohashira/z_bg_spot01_idohashira.h"
    "src/overlays/actors/ovl_Bg_Spot01_Idomizu/z_bg_spot01_idomizu.c"
    "src/overlays/actors/ovl_Bg_Spot01_Idomizu/z_bg_spot01_idomizu.h"
    "src/overlays/actors/ovl_Bg_Spot01_Idosoko/z_bg_spot01_idosoko.c"
    "src/overlays/actors/ovl_Bg_Spot01_Idosoko/z_bg_spot01_idosoko.h"
    "src/overlays/actors/ovl_Bg_Spot01_Objects2/z_bg_spot01_objects2.c"
    "src/overlays/actors/ovl_Bg_Spot01_Objects2/z_bg_spot01_objects2.h"
    "src/overlays/actors/ovl_Bg_Spot02_Objects/z_bg_spot02_objects.c"
    "src/overlays/actors/ovl_Bg_Spot02_Objects/z_bg_spot02_objects.h"
    "src/overlays/actors/ovl_Bg_Spot03_Taki/z_bg_spot03_taki.c"
    "src/overlays/actors/ovl_Bg_Spot03_Taki/z_bg_spot03_taki.h"
    "src/overlays/actors/ovl_Bg_Spot05_Soko/z_bg_spot05_soko.c"
    "src/overlays/actors/ovl_Bg_Spot05_Soko/z_bg_spot05_soko.h"
    "src/overlays/actors/ovl_Bg_Spot06_Objects/z_bg_spot06_objects.c"
    "src/overlays/actors/ovl_Bg_Spot06_Objects/z_bg_spot06_objects.h"
    "src/overlays/actors/ovl_Bg_Spot07_Taki/z_bg_spot07_taki.c"
    "src/overlays/actors/ovl_Bg_Spot07_Taki/z_bg_spot07_taki.h"
    "src/overlays/actors/ovl_Bg_Spot08_Bakudankabe/z_bg_spot08_bakudankabe.c"
    "src/overlays/actors/ovl_Bg_Spot08_Bakudankabe/z_bg_spot08_bakudankabe.h"
    "src/overlays/actors/ovl_Bg_Spot08_Iceblock/z_bg_spot08_iceblock.c"
    "src/overlays/actors/ovl_Bg_Spot08_Iceblock/z_bg_spot08_iceblock.h"
    "src/overlays/actors/ovl_Bg_Spot09_Obj/z_bg_spot09_obj.c"
    "src/overlays/actors/ovl_Bg_Spot09_Obj/z_bg_spot09_obj.h"
    "src/overlays/actors/ovl_Bg_Spot11_Bakudankabe/z_bg_spot11_bakudankabe.c"
    "src/overlays/actors/ovl_Bg_Spot11_Bakudankabe/z_bg_spot11_bakudankabe.h"
    "src/overlays/actors/ovl_Bg_Spot11_Oasis/z_bg_spot11_oasis.c"
    "src/overlays/actors/ovl_Bg_Spot11_Oasis/z_bg_spot11_oasis.h"
    "src/overlays/actors/ovl_Bg_Spot12_Gate/z_bg_spot12_gate.c"
    "src/overlays/actors/ovl_Bg_Spot12_Gate/z_bg_spot12_gate.h"
    "src/overlays/actors/ovl_Bg_Spot12_Saku/z_bg_spot12_saku.c"
    "src/overlays/actors/ovl_Bg_Spot12_Saku/z_bg_spot12_saku.h"
    "src/overlays/actors/ovl_Bg_Spot15_Rrbox/z_bg_spot15_rrbox.c"
    "src/overlays/actors/ovl_Bg_Spot15_Rrbox/z_bg_spot15_rrbox.h"
    "src/overlays/actors/ovl_Bg_Spot15_Saku/z_bg_spot15_saku.c"
    "src/overlays/actors/ovl_Bg_Spot15_Saku/z_bg_spot15_saku.h"
    "src/overlays/actors/ovl_Bg_Spot16_Bombstone/z_bg_spot16_bombstone.c"
    "src/overlays/actors/ovl_Bg_Spot16_Bombstone/z_bg_spot16_bombstone.h"
    "src/overlays/actors/ovl_Bg_Spot16_Doughnut/z_bg_spot16_doughnut.c"
    "src/overlays/actors/ovl_Bg_Spot16_Doughnut/z_bg_spot16_doughnut.h"
    "src/overlays/actors/ovl_Bg_Spot17_Bakudankabe/z_bg_spot17_bakudankabe.c"
    "src/overlays/actors/ovl_Bg_Spot17_Bakudankabe/z_bg_spot17_bakudankabe.h"
    "src/overlays/actors/ovl_Bg_Spot17_Funen/z_bg_spot17_funen.c"
    "src/overlays/actors/ovl_Bg_Spot17_Funen/z_bg_spot17_funen.h"
    "src/overlays/actors/ovl_Bg_Spot18_Basket/z_bg_spot18_basket.c"
    "src/overlays/actors/ovl_Bg_Spot18_Basket/z_bg_spot18_basket.h"
    "src/overlays/actors/ovl_Bg_Spot18_Futa/z_bg_spot18_futa.c"
    "src/overlays/actors/ovl_Bg_Spot18_Futa/z_bg_spot18_futa.h"
    "src/overlays/actors/ovl_Bg_Spot18_Obj/z_bg_spot18_obj.c"
    "src/overlays/actors/ovl_Bg_Spot18_Obj/z_bg_spot18_obj.h"
    "src/overlays/actors/ovl_Bg_Spot18_Shutter/z_bg_spot18_shutter.c"
    "src/overlays/actors/ovl_Bg_Spot18_Shutter/z_bg_spot18_shutter.h"
    "src/overlays/actors/ovl_Bg_Sst_Floor/z_bg_sst_floor.c"
    "src/overlays/actors/ovl_Bg_Sst_Floor/z_bg_sst_floor.h"
    "src/overlays/actors/ovl_Bg_Toki_Hikari/z_bg_toki_hikari.c"
    "src/overlays/actors/ovl_Bg_Toki_Hikari/z_bg_toki_hikari.h"
    "src/overlays/actors/ovl_Bg_Toki_Swd/z_bg_toki_swd.c"
    "src/overlays/actors/ovl_Bg_Toki_Swd/z_bg_toki_swd.h"
    "src/overlays/actors/ovl_Bg_Toki_Swd/z_bg_toki_swd_cutscene_data_1.c"
    "src/overlays/actors/ovl_Bg_Toki_Swd/z_bg_toki_swd_cutscene_data_2.c"
    "src/overlays/actors/ovl_Bg_Toki_Swd/z_bg_toki_swd_cutscene_data_3.c"
    "src/overlays/actors/ovl_Bg_Treemouth/z_bg_treemouth.c"
    "src/overlays/actors/ovl_Bg_Treemouth/z_bg_treemouth.h"
    "src/overlays/actors/ovl_Bg_Treemouth/z_bg_treemouth_cutscene_data.c"
    "src/overlays/actors/ovl_Bg_Umajump/z_bg_umajump.c"
    "src/overlays/actors/ovl_Bg_Umajump/z_bg_umajump.h"
    "src/overlays/actors/ovl_Bg_Vb_Sima/z_bg_vb_sima.c"
    "src/overlays/actors/ovl_Bg_Vb_Sima/z_bg_vb_sima.h"
    "src/overlays/actors/ovl_Bg_Ydan_Hasi/z_bg_ydan_hasi.c"
    "src/overlays/actors/ovl_Bg_Ydan_Hasi/z_bg_ydan_hasi.h"
    "src/overlays/actors/ovl_Bg_Ydan_Maruta/z_bg_ydan_maruta.c"
    "src/overlays/actors/ovl_Bg_Ydan_Maruta/z_bg_ydan_maruta.h"
    "src/overlays/actors/ovl_Bg_Ydan_Sp/z_bg_ydan_sp.c"
    "src/overlays/actors/ovl_Bg_Ydan_Sp/z_bg_ydan_sp.h"
    "src/overlays/actors/ovl_Bg_Zg/z_bg_zg.c"
    "src/overlays/actors/ovl_Bg_Zg/z_bg_zg.h"
    "src/overlays/actors/ovl_Boss_Dodongo/z_boss_dodongo.c"
    "src/overlays/actors/ovl_Boss_Dodongo/z_boss_dodongo.h"
    "src/overlays/actors/ovl_Boss_Dodongo/z_boss_dodongo_data.c"
    "src/overlays/actors/ovl_Boss_Fd/z_boss_fd.c"
    "src/overlays/actors/ovl_Boss_Fd/z_boss_fd.h"
    "src/overlays/actors/ovl_Boss_Fd/z_boss_fd_colchk.c"
    "src/overlays/actors/ovl_Boss_Fd2/z_boss_fd2.c"
    "src/overlays/actors/ovl_Boss_Fd2/z_boss_fd2.h"
    "src/overlays/actors/ovl_Boss_Fd2/z_boss_fd2_colchk.c"
    "src/overlays/actors/ovl_Boss_Ganon/z_boss_ganon.c"
    "src/overlays/actors/ovl_Boss_Ganon/z_boss_ganon.h"
    "src/overlays/actors/ovl_Boss_Ganon2/z_boss_ganon2.c"
    "src/overlays/actors/ovl_Boss_Ganon2/z_boss_ganon2.h"
    "src/overlays/actors/ovl_Boss_Ganon2/z_boss_ganon2_data.c"
    "src/overlays/actors/ovl_Boss_Ganondrof/z_boss_ganondrof.c"
    "src/overlays/actors/ovl_Boss_Ganondrof/z_boss_ganondrof.h"
    "src/overlays/actors/ovl_Boss_Goma/z_boss_goma.c"
    "src/overlays/actors/ovl_Boss_Goma/z_boss_goma.h"
    "src/overlays/actors/ovl_Boss_Mo/z_boss_mo.c"
    "src/overlays/actors/ovl_Boss_Mo/z_boss_mo.h"
    "src/overlays/actors/ovl_Boss_Mo/z_boss_mo_colchk.c"
    "src/overlays/actors/ovl_Boss_Sst/z_boss_sst.c"
    "src/overlays/actors/ovl_Boss_Sst/z_boss_sst.h"
    "src/overlays/actors/ovl_Boss_Sst/z_boss_sst_colchk.c"
    "src/overlays/actors/ovl_Boss_Tw/z_boss_tw.c"
    "src/overlays/actors/ovl_Boss_Tw/z_boss_tw.h"
    "src/overlays/actors/ovl_Boss_Va/z_boss_va.c"
    "src/overlays/actors/ovl_Boss_Va/z_boss_va.h"
    "src/overlays/actors/ovl_Demo_6K/z_demo_6k.c"
    "src/overlays/actors/ovl_Demo_6K/z_demo_6k.h"
    "src/overlays/actors/ovl_Demo_Du/z_demo_du.c"
    "src/overlays/actors/ovl_Demo_Du/z_demo_du.h"
    "src/overlays/actors/ovl_Demo_Du/z_demo_du_cutscene_data.c"
    "src/overlays/actors/ovl_Demo_Ec/z_demo_ec.c"
    "src/overlays/actors/ovl_Demo_Ec/z_demo_ec.h"
    "src/overlays/actors/ovl_Demo_Effect/z_demo_effect.c"
    "src/overlays/actors/ovl_Demo_Effect/z_demo_effect.h"
    "src/overlays/actors/ovl_Demo_Ext/z_demo_ext.c"
    "src/overlays/actors/ovl_Demo_Ext/z_demo_ext.h"
    "src/overlays/actors/ovl_Demo_Geff/z_demo_geff.c"
    "src/overlays/actors/ovl_Demo_Geff/z_demo_geff.h"
    "src/overlays/actors/ovl_Demo_Gj/z_demo_gj.c"
    "src/overlays/actors/ovl_Demo_Gj/z_demo_gj.h"
    "src/overlays/actors/ovl_Demo_Go/z_demo_go.c"
    "src/overlays/actors/ovl_Demo_Go/z_demo_go.h"
    "src/overlays/actors/ovl_Demo_Gt/z_demo_gt.c"
    "src/overlays/actors/ovl_Demo_Gt/z_demo_gt.h"
    "src/overlays/actors/ovl_Demo_Ik/z_demo_ik.c"
    "src/overlays/actors/ovl_Demo_Ik/z_demo_ik.h"
    "src/overlays/actors/ovl_Demo_Im/z_demo_im.c"
    "src/overlays/actors/ovl_Demo_Im/z_demo_im.h"
    "src/overlays/actors/ovl_Demo_Im/z_demo_im_cutscene_data.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo.h"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo_cutscene_data1.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo_cutscene_data2.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo_cutscene_data3.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo_cutscene_data4.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo_cutscene_data5.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo_cutscene_data6.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo_cutscene_data7.c"
    "src/overlays/actors/ovl_Demo_Kankyo/z_demo_kankyo_cutscene_data8.c"
    "src/overlays/actors/ovl_Demo_Kekkai/z_demo_kekkai.c"
    "src/overlays/actors/ovl_Demo_Kekkai/z_demo_kekkai.h"
    "src/overlays/actors/ovl_Demo_Sa/z_demo_sa.c"
    "src/overlays/actors/ovl_Demo_Sa/z_demo_sa.h"
    "src/overlays/actors/ovl_Demo_Sa/z_demo_sa_cutscene_data.c"
    "src/overlays/actors/ovl_Demo_Shd/z_demo_shd.c"
    "src/overlays/actors/ovl_Demo_Shd/z_demo_shd.h"
    "src/overlays/actors/ovl_Demo_Tre_Lgt/z_demo_tre_lgt.c"
    "src/overlays/actors/ovl_Demo_Tre_Lgt/z_demo_tre_lgt.h"
    "src/overlays/actors/ovl_Door_Ana/z_door_ana.c"
    "src/overlays/actors/ovl_Door_Ana/z_door_ana.h"
    "src/overlays/actors/ovl_Door_Gerudo/z_door_gerudo.c"
    "src/overlays/actors/ovl_Door_Gerudo/z_door_gerudo.h"
    "src/overlays/actors/ovl_Door_Killer/z_door_killer.c"
    "src/overlays/actors/ovl_Door_Killer/z_door_killer.h"
    "src/overlays/actors/ovl_Door_Shutter/z_door_shutter.c"
    "src/overlays/actors/ovl_Door_Shutter/z_door_shutter.h"
    "src/overlays/actors/ovl_Door_Toki/z_door_toki.c"
    "src/overlays/actors/ovl_Door_Toki/z_door_toki.h"
    "src/overlays/actors/ovl_Door_Warp1/z_door_warp1.c"
    "src/overlays/actors/ovl_Door_Warp1/z_door_warp1.h"
    "src/overlays/actors/ovl_Efc_Erupc/z_efc_erupc.c"
    "src/overlays/actors/ovl_Efc_Erupc/z_efc_erupc.h"
    "src/overlays/actors/ovl_Eff_Dust/z_eff_dust.c"
    "src/overlays/actors/ovl_Eff_Dust/z_eff_dust.h"
    "src/overlays/actors/ovl_Elf_Msg/z_elf_msg.c"
    "src/overlays/actors/ovl_Elf_Msg/z_elf_msg.h"
    "src/overlays/actors/ovl_Elf_Msg2/z_elf_msg2.c"
    "src/overlays/actors/ovl_Elf_Msg2/z_elf_msg2.h"
    "src/overlays/actors/ovl_En_Am/z_en_am.c"
    "src/overlays/actors/ovl_En_Am/z_en_am.h"
    "src/overlays/actors/ovl_En_Ani/z_en_ani.c"
    "src/overlays/actors/ovl_En_Ani/z_en_ani.h"
    "src/overlays/actors/ovl_En_Anubice/z_en_anubice.c"
    "src/overlays/actors/ovl_En_Anubice/z_en_anubice.h"
    "src/overlays/actors/ovl_En_Anubice_Fire/z_en_anubice_fire.c"
    "src/overlays/actors/ovl_En_Anubice_Fire/z_en_anubice_fire.h"
    "src/overlays/actors/ovl_En_Anubice_Tag/z_en_anubice_tag.c"
    "src/overlays/actors/ovl_En_Anubice_Tag/z_en_anubice_tag.h"
    "src/overlays/actors/ovl_En_Arow_Trap/z_en_arow_trap.c"
    "src/overlays/actors/ovl_En_Arow_Trap/z_en_arow_trap.h"
    "src/overlays/actors/ovl_En_Arrow/z_en_arrow.c"
    "src/overlays/actors/ovl_En_Arrow/z_en_arrow.h"
    "src/overlays/actors/ovl_En_Attack_Niw/z_en_attack_niw.c"
    "src/overlays/actors/ovl_En_Attack_Niw/z_en_attack_niw.h"
    "src/overlays/actors/ovl_En_Ba/z_en_ba.c"
    "src/overlays/actors/ovl_En_Ba/z_en_ba.h"
    "src/overlays/actors/ovl_En_Bb/z_en_bb.c"
    "src/overlays/actors/ovl_En_Bb/z_en_bb.h"
    "src/overlays/actors/ovl_En_Bdfire/z_en_bdfire.c"
    "src/overlays/actors/ovl_En_Bdfire/z_en_bdfire.h"
    "src/overlays/actors/ovl_En_Bigokuta/z_en_bigokuta.c"
    "src/overlays/actors/ovl_En_Bigokuta/z_en_bigokuta.h"
    "src/overlays/actors/ovl_En_Bili/z_en_bili.c"
    "src/overlays/actors/ovl_En_Bili/z_en_bili.h"
    "src/overlays/actors/ovl_En_Bird/z_en_bird.c"
    "src/overlays/actors/ovl_En_Bird/z_en_bird.h"
    "src/overlays/actors/ovl_En_Blkobj/z_en_blkobj.c"
    "src/overlays/actors/ovl_En_Blkobj/z_en_blkobj.h"
    "src/overlays/actors/ovl_En_Bom/z_en_bom.c"
    "src/overlays/actors/ovl_En_Bom/z_en_bom.h"
    "src/overlays/actors/ovl_En_Bom_Bowl_Man/z_en_bom_bowl_man.c"
    "src/overlays/actors/ovl_En_Bom_Bowl_Man/z_en_bom_bowl_man.h"
    "src/overlays/actors/ovl_En_Bom_Bowl_Pit/z_en_bom_bowl_pit.c"
    "src/overlays/actors/ovl_En_Bom_Bowl_Pit/z_en_bom_bowl_pit.h"
    "src/overlays/actors/ovl_En_Bom_Chu/z_en_bom_chu.c"
    "src/overlays/actors/ovl_En_Bom_Chu/z_en_bom_chu.h"
    "src/overlays/actors/ovl_En_Bombf/z_en_bombf.c"
    "src/overlays/actors/ovl_En_Bombf/z_en_bombf.h"
    "src/overlays/actors/ovl_En_Boom/z_en_boom.c"
    "src/overlays/actors/ovl_En_Boom/z_en_boom.h"
    "src/overlays/actors/ovl_En_Box/z_en_box.c"
    "src/overlays/actors/ovl_En_Box/z_en_box.h"
    "src/overlays/actors/ovl_En_Brob/z_en_brob.c"
    "src/overlays/actors/ovl_En_Brob/z_en_brob.h"
    "src/overlays/actors/ovl_En_Bubble/z_en_bubble.c"
    "src/overlays/actors/ovl_En_Bubble/z_en_bubble.h"
    "src/overlays/actors/ovl_En_Butte/z_en_butte.c"
    "src/overlays/actors/ovl_En_Butte/z_en_butte.h"
    "src/overlays/actors/ovl_En_Bw/z_en_bw.c"
    "src/overlays/actors/ovl_En_Bw/z_en_bw.h"
    "src/overlays/actors/ovl_En_Bx/z_en_bx.c"
    "src/overlays/actors/ovl_En_Bx/z_en_bx.h"
    "src/overlays/actors/ovl_En_Changer/z_en_changer.c"
    "src/overlays/actors/ovl_En_Changer/z_en_changer.h"
    "src/overlays/actors/ovl_En_Clear_Tag/z_en_clear_tag.c"
    "src/overlays/actors/ovl_En_Clear_Tag/z_en_clear_tag.h"
    "src/overlays/actors/ovl_En_Cow/z_en_cow.c"
    "src/overlays/actors/ovl_En_Cow/z_en_cow.h"
    "src/overlays/actors/ovl_En_Crow/z_en_crow.c"
    "src/overlays/actors/ovl_En_Crow/z_en_crow.h"
    "src/overlays/actors/ovl_En_Cs/z_en_cs.c"
    "src/overlays/actors/ovl_En_Cs/z_en_cs.h"
    "src/overlays/actors/ovl_En_Daiku/z_en_daiku.c"
    "src/overlays/actors/ovl_En_Daiku/z_en_daiku.h"
    "src/overlays/actors/ovl_En_Daiku_Kakariko/z_en_daiku_kakariko.c"
    "src/overlays/actors/ovl_En_Daiku_Kakariko/z_en_daiku_kakariko.h"
    "src/overlays/actors/ovl_En_Dekubaba/z_en_dekubaba.c"
    "src/overlays/actors/ovl_En_Dekubaba/z_en_dekubaba.h"
    "src/overlays/actors/ovl_En_Dekunuts/z_en_dekunuts.c"
    "src/overlays/actors/ovl_En_Dekunuts/z_en_dekunuts.h"
    "src/overlays/actors/ovl_En_Dh/z_en_dh.c"
    "src/overlays/actors/ovl_En_Dh/z_en_dh.h"
    "src/overlays/actors/ovl_En_Dha/z_en_dha.c"
    "src/overlays/actors/ovl_En_Dha/z_en_dha.h"
    "src/overlays/actors/ovl_En_Diving_Game/z_en_diving_game.c"
    "src/overlays/actors/ovl_En_Diving_Game/z_en_diving_game.h"
    "src/overlays/actors/ovl_En_Dns/z_en_dns.c"
    "src/overlays/actors/ovl_En_Dns/z_en_dns.h"
    "src/overlays/actors/ovl_En_Dnt_Demo/z_en_dnt_demo.c"
    "src/overlays/actors/ovl_En_Dnt_Demo/z_en_dnt_demo.h"
    "src/overlays/actors/ovl_En_Dnt_Jiji/z_en_dnt_jiji.c"
    "src/overlays/actors/ovl_En_Dnt_Jiji/z_en_dnt_jiji.h"
    "src/overlays/actors/ovl_En_Dnt_Nomal/z_en_dnt_nomal.c"
    "src/overlays/actors/ovl_En_Dnt_Nomal/z_en_dnt_nomal.h"
    "src/overlays/actors/ovl_En_Dodojr/z_en_dodojr.c"
    "src/overlays/actors/ovl_En_Dodojr/z_en_dodojr.h"
    "src/overlays/actors/ovl_En_Dodongo/z_en_dodongo.c"
    "src/overlays/actors/ovl_En_Dodongo/z_en_dodongo.h"
    "src/overlays/actors/ovl_En_Dog/z_en_dog.c"
    "src/overlays/actors/ovl_En_Dog/z_en_dog.h"
    "src/overlays/actors/ovl_En_Door/z_en_door.c"
    "src/overlays/actors/ovl_En_Door/z_en_door.h"
    "src/overlays/actors/ovl_En_Ds/z_en_ds.c"
    "src/overlays/actors/ovl_En_Ds/z_en_ds.h"
    "src/overlays/actors/ovl_En_Du/z_en_du.c"
    "src/overlays/actors/ovl_En_Du/z_en_du.h"
    "src/overlays/actors/ovl_En_Dy_Extra/z_en_dy_extra.c"
    "src/overlays/actors/ovl_En_Dy_Extra/z_en_dy_extra.h"
    "src/overlays/actors/ovl_En_Eg/z_en_eg.c"
    "src/overlays/actors/ovl_En_Eg/z_en_eg.h"
    "src/overlays/actors/ovl_En_Eiyer/z_en_eiyer.c"
    "src/overlays/actors/ovl_En_Eiyer/z_en_eiyer.h"
    "src/overlays/actors/ovl_En_Elf/z_en_elf.c"
    "src/overlays/actors/ovl_En_Elf/z_en_elf.h"
    "src/overlays/actors/ovl_En_Encount1/z_en_encount1.c"
    "src/overlays/actors/ovl_En_Encount1/z_en_encount1.h"
    "src/overlays/actors/ovl_En_Encount2/z_en_encount2.c"
    "src/overlays/actors/ovl_En_Encount2/z_en_encount2.h"
    "src/overlays/actors/ovl_En_Ex_Item/z_en_ex_item.c"
    "src/overlays/actors/ovl_En_Ex_Item/z_en_ex_item.h"
    "src/overlays/actors/ovl_En_Ex_Ruppy/z_en_ex_ruppy.c"
    "src/overlays/actors/ovl_En_Ex_Ruppy/z_en_ex_ruppy.h"
    "src/overlays/actors/ovl_En_Fd/z_en_fd.c"
    "src/overlays/actors/ovl_En_Fd/z_en_fd.h"
    "src/overlays/actors/ovl_En_Fd_Fire/z_en_fd_fire.c"
    "src/overlays/actors/ovl_En_Fd_Fire/z_en_fd_fire.h"
    "src/overlays/actors/ovl_En_fHG/z_en_fhg.c"
    "src/overlays/actors/ovl_En_fHG/z_en_fhg.h"
    "src/overlays/actors/ovl_En_Fhg_Fire/z_en_fhg_fire.c"
    "src/overlays/actors/ovl_En_Fhg_Fire/z_en_fhg_fire.h"
    "src/overlays/actors/ovl_En_Fire_Rock/z_en_fire_rock.c"
    "src/overlays/actors/ovl_En_Fire_Rock/z_en_fire_rock.h"
    "src/overlays/actors/ovl_En_Firefly/z_en_firefly.c"
    "src/overlays/actors/ovl_En_Firefly/z_en_firefly.h"
    "src/overlays/actors/ovl_En_Fish/z_en_fish.c"
    "src/overlays/actors/ovl_En_Fish/z_en_fish.h"
    "src/overlays/actors/ovl_En_Floormas/z_en_floormas.c"
    "src/overlays/actors/ovl_En_Floormas/z_en_floormas.h"
    "src/overlays/actors/ovl_En_Fr/z_en_fr.c"
    "src/overlays/actors/ovl_En_Fr/z_en_fr.h"
    "src/overlays/actors/ovl_En_Fu/z_en_fu.c"
    "src/overlays/actors/ovl_En_Fu/z_en_fu.h"
    "src/overlays/actors/ovl_En_Fw/z_en_fw.c"
    "src/overlays/actors/ovl_En_Fw/z_en_fw.h"
    "src/overlays/actors/ovl_En_Fz/z_en_fz.c"
    "src/overlays/actors/ovl_En_Fz/z_en_fz.h"
    "src/overlays/actors/ovl_En_G_Switch/z_en_g_switch.c"
    "src/overlays/actors/ovl_En_G_Switch/z_en_g_switch.h"
    "src/overlays/actors/ovl_En_Ganon_Mant/z_en_ganon_mant.c"
    "src/overlays/actors/ovl_En_Ganon_Mant/z_en_ganon_mant.h"
    "src/overlays/actors/ovl_En_Ganon_Organ/z_en_ganon_organ.c"
    "src/overlays/actors/ovl_En_Ganon_Organ/z_en_ganon_organ.h"
    "src/overlays/actors/ovl_En_Gb/z_en_gb.c"
    "src/overlays/actors/ovl_En_Gb/z_en_gb.h"
    "src/overlays/actors/ovl_En_Ge1/z_en_ge1.c"
    "src/overlays/actors/ovl_En_Ge1/z_en_ge1.h"
    "src/overlays/actors/ovl_En_Ge2/z_en_ge2.c"
    "src/overlays/actors/ovl_En_Ge2/z_en_ge2.h"
    "src/overlays/actors/ovl_En_Ge3/z_en_ge3.c"
    "src/overlays/actors/ovl_En_Ge3/z_en_ge3.h"
    "src/overlays/actors/ovl_En_GeldB/z_en_geldb.c"
    "src/overlays/actors/ovl_En_GeldB/z_en_geldb.h"
    "src/overlays/actors/ovl_En_GirlA/z_en_girla.c"
    "src/overlays/actors/ovl_En_GirlA/z_en_girla.h"
    "src/overlays/actors/ovl_En_Gm/z_en_gm.c"
    "src/overlays/actors/ovl_En_Gm/z_en_gm.h"
    "src/overlays/actors/ovl_En_Go/z_en_go.c"
    "src/overlays/actors/ovl_En_Go/z_en_go.h"
    "src/overlays/actors/ovl_En_Go2/z_en_go2.c"
    "src/overlays/actors/ovl_En_Go2/z_en_go2.h"
    "src/overlays/actors/ovl_En_Goma/z_en_goma.c"
    "src/overlays/actors/ovl_En_Goma/z_en_goma.h"
    "src/overlays/actors/ovl_En_Goroiwa/z_en_goroiwa.c"
    "src/overlays/actors/ovl_En_Goroiwa/z_en_goroiwa.h"
    "src/overlays/actors/ovl_En_Gs/z_en_gs.c"
    "src/overlays/actors/ovl_En_Gs/z_en_gs.h"
    "src/overlays/actors/ovl_En_Guest/z_en_guest.c"
    "src/overlays/actors/ovl_En_Guest/z_en_guest.h"
    "src/overlays/actors/ovl_En_Hata/z_en_hata.c"
    "src/overlays/actors/ovl_En_Hata/z_en_hata.h"
    "src/overlays/actors/ovl_En_Heishi1/z_en_heishi1.c"
    "src/overlays/actors/ovl_En_Heishi1/z_en_heishi1.h"
    "src/overlays/actors/ovl_En_Heishi2/z_en_heishi2.c"
    "src/overlays/actors/ovl_En_Heishi2/z_en_heishi2.h"
    "src/overlays/actors/ovl_En_Heishi3/z_en_heishi3.c"
    "src/overlays/actors/ovl_En_Heishi3/z_en_heishi3.h"
    "src/overlays/actors/ovl_En_Heishi4/z_en_heishi4.c"
    "src/overlays/actors/ovl_En_Heishi4/z_en_heishi4.h"
    "src/overlays/actors/ovl_En_Hintnuts/z_en_hintnuts.c"
    "src/overlays/actors/ovl_En_Hintnuts/z_en_hintnuts.h"
    "src/overlays/actors/ovl_En_Holl/z_en_holl.c"
    "src/overlays/actors/ovl_En_Holl/z_en_holl.h"
    "src/overlays/actors/ovl_En_Honotrap/z_en_honotrap.c"
    "src/overlays/actors/ovl_En_Honotrap/z_en_honotrap.h"
    "src/overlays/actors/ovl_En_Horse/z_en_horse.c"
    "src/overlays/actors/ovl_En_Horse/z_en_horse.h"
    "src/overlays/actors/ovl_En_Horse_Game_Check/z_en_horse_game_check.c"
    "src/overlays/actors/ovl_En_Horse_Game_Check/z_en_horse_game_check.h"
    "src/overlays/actors/ovl_En_Horse_Ganon/z_en_horse_ganon.c"
    "src/overlays/actors/ovl_En_Horse_Ganon/z_en_horse_ganon.h"
    "src/overlays/actors/ovl_En_Horse_Link_Child/z_en_horse_link_child.c"
    "src/overlays/actors/ovl_En_Horse_Link_Child/z_en_horse_link_child.h"
    "src/overlays/actors/ovl_En_Horse_Normal/z_en_horse_normal.c"
    "src/overlays/actors/ovl_En_Horse_Normal/z_en_horse_normal.h"
    "src/overlays/actors/ovl_En_Horse_Zelda/z_en_horse_zelda.c"
    "src/overlays/actors/ovl_En_Horse_Zelda/z_en_horse_zelda.h"
    "src/overlays/actors/ovl_En_Hs/z_en_hs.c"
    "src/overlays/actors/ovl_En_Hs/z_en_hs.h"
    "src/overlays/actors/ovl_En_Hs2/z_en_hs2.c"
    "src/overlays/actors/ovl_En_Hs2/z_en_hs2.h"
    "src/overlays/actors/ovl_En_Hy/z_en_hy.c"
    "src/overlays/actors/ovl_En_Hy/z_en_hy.h"
    "src/overlays/actors/ovl_En_Ice_Hono/z_en_ice_hono.c"
    "src/overlays/actors/ovl_En_Ice_Hono/z_en_ice_hono.h"
    "src/overlays/actors/ovl_En_Ik/z_en_ik.c"
    "src/overlays/actors/ovl_En_Ik/z_en_ik.h"
    "src/overlays/actors/ovl_En_In/z_en_in.c"
    "src/overlays/actors/ovl_En_In/z_en_in.h"
    "src/overlays/actors/ovl_En_Insect/z_en_insect.c"
    "src/overlays/actors/ovl_En_Insect/z_en_insect.h"
    "src/overlays/actors/ovl_En_Ishi/z_en_ishi.c"
    "src/overlays/actors/ovl_En_Ishi/z_en_ishi.h"
    "src/overlays/actors/ovl_En_It/z_en_it.c"
    "src/overlays/actors/ovl_En_It/z_en_it.h"
    "src/overlays/actors/ovl_En_Jj/z_en_jj.c"
    "src/overlays/actors/ovl_En_Jj/z_en_jj.h"
    "src/overlays/actors/ovl_En_Jj/z_en_jj_cutscene_data.c"
    "src/overlays/actors/ovl_En_Js/z_en_js.c"
    "src/overlays/actors/ovl_En_Js/z_en_js.h"
    "src/overlays/actors/ovl_En_Jsjutan/z_en_jsjutan.c"
    "src/overlays/actors/ovl_En_Jsjutan/z_en_jsjutan.h"
    "src/overlays/actors/ovl_En_Kakasi/z_en_kakasi.c"
    "src/overlays/actors/ovl_En_Kakasi/z_en_kakasi.h"
    "src/overlays/actors/ovl_En_Kakasi2/z_en_kakasi2.c"
    "src/overlays/actors/ovl_En_Kakasi2/z_en_kakasi2.h"
    "src/overlays/actors/ovl_En_Kakasi3/z_en_kakasi3.c"
    "src/overlays/actors/ovl_En_Kakasi3/z_en_kakasi3.h"
    "src/overlays/actors/ovl_En_Kanban/z_en_kanban.c"
    "src/overlays/actors/ovl_En_Kanban/z_en_kanban.h"
    "src/overlays/actors/ovl_En_Kanban/z_en_kanban_gfx.c"
    "src/overlays/actors/ovl_En_Karebaba/z_en_karebaba.c"
    "src/overlays/actors/ovl_En_Karebaba/z_en_karebaba.h"
    "src/overlays/actors/ovl_En_Ko/z_en_ko.c"
    "src/overlays/actors/ovl_En_Ko/z_en_ko.h"
    "src/overlays/actors/ovl_En_Kusa/z_en_kusa.c"
    "src/overlays/actors/ovl_En_Kusa/z_en_kusa.h"
    "src/overlays/actors/ovl_En_Kz/z_en_kz.c"
    "src/overlays/actors/ovl_En_Kz/z_en_kz.h"
    "src/overlays/actors/ovl_En_Light/z_en_light.c"
    "src/overlays/actors/ovl_En_Light/z_en_light.h"
    "src/overlays/actors/ovl_En_Lightbox/z_en_lightbox.c"
    "src/overlays/actors/ovl_En_Lightbox/z_en_lightbox.h"
    "src/overlays/actors/ovl_En_M_Fire1/z_en_m_fire1.c"
    "src/overlays/actors/ovl_En_M_Fire1/z_en_m_fire1.h"
    "src/overlays/actors/ovl_En_M_Thunder/z_en_m_thunder.c"
    "src/overlays/actors/ovl_En_M_Thunder/z_en_m_thunder.h"
    "src/overlays/actors/ovl_En_Ma1/z_en_ma1.c"
    "src/overlays/actors/ovl_En_Ma1/z_en_ma1.h"
    "src/overlays/actors/ovl_En_Ma2/z_en_ma2.c"
    "src/overlays/actors/ovl_En_Ma2/z_en_ma2.h"
    "src/overlays/actors/ovl_En_Ma3/z_en_ma3.c"
    "src/overlays/actors/ovl_En_Ma3/z_en_ma3.h"
    "src/overlays/actors/ovl_En_Mag/z_en_mag.c"
    "src/overlays/actors/ovl_En_Mag/z_en_mag.h"
    "src/overlays/actors/ovl_En_Mb/z_en_mb.c"
    "src/overlays/actors/ovl_En_Mb/z_en_mb.h"
    "src/overlays/actors/ovl_En_Md/z_en_md.c"
    "src/overlays/actors/ovl_En_Md/z_en_md.h"
    "src/overlays/actors/ovl_En_Mk/z_en_mk.c"
    "src/overlays/actors/ovl_En_Mk/z_en_mk.h"
    "src/overlays/actors/ovl_En_Mm/z_en_mm.c"
    "src/overlays/actors/ovl_En_Mm/z_en_mm.h"
    "src/overlays/actors/ovl_En_Mm2/z_en_mm2.c"
    "src/overlays/actors/ovl_En_Mm2/z_en_mm2.h"
    "src/overlays/actors/ovl_En_Ms/z_en_ms.c"
    "src/overlays/actors/ovl_En_Ms/z_en_ms.h"
    "src/overlays/actors/ovl_En_Mu/z_en_mu.c"
    "src/overlays/actors/ovl_En_Mu/z_en_mu.h"
    "src/overlays/actors/ovl_En_Nb/z_en_nb.c"
    "src/overlays/actors/ovl_En_Nb/z_en_nb.h"
    "src/overlays/actors/ovl_En_Nb/z_en_nb_cutscene_data.c"
    "src/overlays/actors/ovl_En_Niw/z_en_niw.c"
    "src/overlays/actors/ovl_En_Niw/z_en_niw.h"
    "src/overlays/actors/ovl_En_Niw_Girl/z_en_niw_girl.c"
    "src/overlays/actors/ovl_En_Niw_Girl/z_en_niw_girl.h"
    "src/overlays/actors/ovl_En_Niw_Lady/z_en_niw_lady.c"
    "src/overlays/actors/ovl_En_Niw_Lady/z_en_niw_lady.h"
    "src/overlays/actors/ovl_En_Nutsball/z_en_nutsball.c"
    "src/overlays/actors/ovl_En_Nutsball/z_en_nutsball.h"
    "src/overlays/actors/ovl_En_Nwc/z_en_nwc.c"
    "src/overlays/actors/ovl_En_Nwc/z_en_nwc.h"
    "src/overlays/actors/ovl_En_Ny/z_en_ny.c"
    "src/overlays/actors/ovl_En_Ny/z_en_ny.h"
    "src/overlays/actors/ovl_En_OE2/z_en_oe2.c"
    "src/overlays/actors/ovl_En_OE2/z_en_oe2.h"
    "src/overlays/actors/ovl_En_Okarina_Effect/z_en_okarina_effect.c"
    "src/overlays/actors/ovl_En_Okarina_Effect/z_en_okarina_effect.h"
    "src/overlays/actors/ovl_En_Okarina_Tag/z_en_okarina_tag.c"
    "src/overlays/actors/ovl_En_Okarina_Tag/z_en_okarina_tag.h"
    "src/overlays/actors/ovl_En_Okarina_Tag/z_en_okarina_tag_cutscene_data.c"
    "src/overlays/actors/ovl_En_Okuta/z_en_okuta.c"
    "src/overlays/actors/ovl_En_Okuta/z_en_okuta.h"
    "src/overlays/actors/ovl_En_Ossan/z_en_ossan.c"
    "src/overlays/actors/ovl_En_Ossan/z_en_ossan.h"
    "src/overlays/actors/ovl_En_Owl/z_en_owl.c"
    "src/overlays/actors/ovl_En_Owl/z_en_owl.h"
    "src/overlays/actors/ovl_En_Part/z_en_part.c"
    "src/overlays/actors/ovl_En_Part/z_en_part.h"
    "src/overlays/actors/ovl_En_Peehat/z_en_peehat.c"
    "src/overlays/actors/ovl_En_Peehat/z_en_peehat.h"
    "src/overlays/actors/ovl_En_Po_Desert/z_en_po_desert.c"
    "src/overlays/actors/ovl_En_Po_Desert/z_en_po_desert.h"
    "src/overlays/actors/ovl_En_Po_Field/z_en_po_field.c"
    "src/overlays/actors/ovl_En_Po_Field/z_en_po_field.h"
    "src/overlays/actors/ovl_En_Po_Relay/z_en_po_relay.c"
    "src/overlays/actors/ovl_En_Po_Relay/z_en_po_relay.h"
    "src/overlays/actors/ovl_En_Po_Sisters/z_en_po_sisters.c"
    "src/overlays/actors/ovl_En_Po_Sisters/z_en_po_sisters.h"
    "src/overlays/actors/ovl_En_Poh/z_en_poh.c"
    "src/overlays/actors/ovl_En_Poh/z_en_poh.h"
    "src/overlays/actors/ovl_En_Pu_box/z_en_pu_box.c"
    "src/overlays/actors/ovl_En_Pu_box/z_en_pu_box.h"
    "src/overlays/actors/ovl_En_Rd/z_en_rd.c"
    "src/overlays/actors/ovl_En_Rd/z_en_rd.h"
    "src/overlays/actors/ovl_En_Reeba/z_en_reeba.c"
    "src/overlays/actors/ovl_En_Reeba/z_en_reeba.h"
    "src/overlays/actors/ovl_En_River_Sound/z_en_river_sound.c"
    "src/overlays/actors/ovl_En_River_Sound/z_en_river_sound.h"
    "src/overlays/actors/ovl_En_Rl/z_en_rl.c"
    "src/overlays/actors/ovl_En_Rl/z_en_rl.h"
    "src/overlays/actors/ovl_En_Rr/z_en_rr.c"
    "src/overlays/actors/ovl_En_Rr/z_en_rr.h"
    "src/overlays/actors/ovl_En_Ru1/z_en_ru1.c"
    "src/overlays/actors/ovl_En_Ru1/z_en_ru1.h"
    "src/overlays/actors/ovl_En_Ru1/z_en_ru1_cutscene_data.c"
    "src/overlays/actors/ovl_En_Ru2/z_en_ru2.c"
    "src/overlays/actors/ovl_En_Ru2/z_en_ru2.h"
    "src/overlays/actors/ovl_En_Ru2/z_en_ru2_cutscene_data.c"
    "src/overlays/actors/ovl_En_Sa/z_en_sa.c"
    "src/overlays/actors/ovl_En_Sa/z_en_sa.h"
    "src/overlays/actors/ovl_En_Sb/z_en_sb.c"
    "src/overlays/actors/ovl_En_Sb/z_en_sb.h"
    "src/overlays/actors/ovl_En_Scene_Change/z_en_scene_change.c"
    "src/overlays/actors/ovl_En_Scene_Change/z_en_scene_change.h"
    "src/overlays/actors/ovl_En_Sda/z_en_sda.c"
    "src/overlays/actors/ovl_En_Sda/z_en_sda.h"
    "src/overlays/actors/ovl_En_Shopnuts/z_en_shopnuts.c"
    "src/overlays/actors/ovl_En_Shopnuts/z_en_shopnuts.h"
    "src/overlays/actors/ovl_En_Si/z_en_si.c"
    "src/overlays/actors/ovl_En_Si/z_en_si.h"
    "src/overlays/actors/ovl_En_Siofuki/z_en_siofuki.c"
    "src/overlays/actors/ovl_En_Siofuki/z_en_siofuki.h"
    "src/overlays/actors/ovl_En_Skb/z_en_skb.c"
    "src/overlays/actors/ovl_En_Skb/z_en_skb.h"
    "src/overlays/actors/ovl_En_Skj/z_en_skj.c"
    "src/overlays/actors/ovl_En_Skj/z_en_skj.h"
    "src/overlays/actors/ovl_En_Skjneedle/z_en_skjneedle.c"
    "src/overlays/actors/ovl_En_Skjneedle/z_en_skjneedle.h"
    "src/overlays/actors/ovl_En_Ssh/z_en_ssh.c"
    "src/overlays/actors/ovl_En_Ssh/z_en_ssh.h"
    "src/overlays/actors/ovl_En_St/z_en_st.c"
    "src/overlays/actors/ovl_En_St/z_en_st.h"
    "src/overlays/actors/ovl_En_Sth/z_en_sth.c"
    "src/overlays/actors/ovl_En_Sth/z_en_sth.h"
    "src/overlays/actors/ovl_En_Stream/z_en_stream.c"
    "src/overlays/actors/ovl_En_Stream/z_en_stream.h"
    "src/overlays/actors/ovl_En_Sw/z_en_sw.c"
    "src/overlays/actors/ovl_En_Sw/z_en_sw.h"
    "src/overlays/actors/ovl_En_Syateki_Itm/z_en_syateki_itm.c"
    "src/overlays/actors/ovl_En_Syateki_Itm/z_en_syateki_itm.h"
    "src/overlays/actors/ovl_En_Syateki_Man/z_en_syateki_man.c"
    "src/overlays/actors/ovl_En_Syateki_Man/z_en_syateki_man.h"
    "src/overlays/actors/ovl_En_Syateki_Niw/z_en_syateki_niw.c"
    "src/overlays/actors/ovl_En_Syateki_Niw/z_en_syateki_niw.h"
    "src/overlays/actors/ovl_En_Ta/z_en_ta.c"
    "src/overlays/actors/ovl_En_Ta/z_en_ta.h"
    "src/overlays/actors/ovl_En_Takara_Man/z_en_takara_man.c"
    "src/overlays/actors/ovl_En_Takara_Man/z_en_takara_man.h"
    "src/overlays/actors/ovl_En_Tana/z_en_tana.c"
    "src/overlays/actors/ovl_En_Tana/z_en_tana.h"
    "src/overlays/actors/ovl_En_Test/z_en_test.c"
    "src/overlays/actors/ovl_En_Test/z_en_test.h"
    "src/overlays/actors/ovl_En_Tg/z_en_tg.c"
    "src/overlays/actors/ovl_En_Tg/z_en_tg.h"
    "src/overlays/actors/ovl_En_Tite/z_en_tite.c"
    "src/overlays/actors/ovl_En_Tite/z_en_tite.h"
    "src/overlays/actors/ovl_En_Tk/z_en_tk.c"
    "src/overlays/actors/ovl_En_Tk/z_en_tk.h"
    "src/overlays/actors/ovl_En_Torch/z_en_torch.c"
    "src/overlays/actors/ovl_En_Torch/z_en_torch.h"
    "src/overlays/actors/ovl_En_Torch2/z_en_torch2.c"
    "src/overlays/actors/ovl_En_Torch2/z_en_torch2.h"
    "src/overlays/actors/ovl_En_Toryo/z_en_toryo.c"
    "src/overlays/actors/ovl_En_Toryo/z_en_toryo.h"
    "src/overlays/actors/ovl_En_Tp/z_en_tp.c"
    "src/overlays/actors/ovl_En_Tp/z_en_tp.h"
    "src/overlays/actors/ovl_En_Tr/z_en_tr.c"
    "src/overlays/actors/ovl_En_Tr/z_en_tr.h"
    "src/overlays/actors/ovl_En_Trap/z_en_trap.c"
    "src/overlays/actors/ovl_En_Trap/z_en_trap.h"
    "src/overlays/actors/ovl_En_Tubo_Trap/z_en_tubo_trap.c"
    "src/overlays/actors/ovl_En_Tubo_Trap/z_en_tubo_trap.h"
    "src/overlays/actors/ovl_En_Vali/z_en_vali.c"
    "src/overlays/actors/ovl_En_Vali/z_en_vali.h"
    "src/overlays/actors/ovl_En_Vase/z_en_vase.c"
    "src/overlays/actors/ovl_En_Vase/z_en_vase.h"
    "src/overlays/actors/ovl_En_Vb_Ball/z_en_vb_ball.c"
    "src/overlays/actors/ovl_En_Vb_Ball/z_en_vb_ball.h"
    "src/overlays/actors/ovl_En_Viewer/z_en_viewer.c"
    "src/overlays/actors/ovl_En_Viewer/z_en_viewer.h"
    "src/overlays/actors/ovl_En_Vm/z_en_vm.c"
    "src/overlays/actors/ovl_En_Vm/z_en_vm.h"
    "src/overlays/actors/ovl_En_Wall_Tubo/z_en_wall_tubo.c"
    "src/overlays/actors/ovl_En_Wall_Tubo/z_en_wall_tubo.h"
    "src/overlays/actors/ovl_En_Wallmas/z_en_wallmas.c"
    "src/overlays/actors/ovl_En_Wallmas/z_en_wallmas.h"
    "src/overlays/actors/ovl_En_Weather_Tag/z_en_weather_tag.c"
    "src/overlays/actors/ovl_En_Weather_Tag/z_en_weather_tag.h"
    "src/overlays/actors/ovl_En_Weiyer/z_en_weiyer.c"
    "src/overlays/actors/ovl_En_Weiyer/z_en_weiyer.h"
    "src/overlays/actors/ovl_En_Wf/z_en_wf.c"
    "src/overlays/actors/ovl_En_Wf/z_en_wf.h"
    "src/overlays/actors/ovl_En_Wonder_Item/z_en_wonder_item.c"
    "src/overlays/actors/ovl_En_Wonder_Item/z_en_wonder_item.h"
    "src/overlays/actors/ovl_En_Wonder_Talk/z_en_wonder_talk.c"
    "src/overlays/actors/ovl_En_Wonder_Talk/z_en_wonder_talk.h"
    "src/overlays/actors/ovl_En_Wonder_Talk2/z_en_wonder_talk2.c"
    "src/overlays/actors/ovl_En_Wonder_Talk2/z_en_wonder_talk2.h"
    "src/overlays/actors/ovl_En_Wood02/z_en_wood02.c"
    "src/overlays/actors/ovl_En_Wood02/z_en_wood02.h"
    "src/overlays/actors/ovl_En_Xc/z_en_xc.c"
    "src/overlays/actors/ovl_En_Xc/z_en_xc.h"
    "src/overlays/actors/ovl_En_Yabusame_Mark/z_en_yabusame_mark.c"
    "src/overlays/actors/ovl_En_Yabusame_Mark/z_en_yabusame_mark.h"
    "src/overlays/actors/ovl_En_Yukabyun/z_en_yukabyun.c"
    "src/overlays/actors/ovl_En_Yukabyun/z_en_yukabyun.h"
    "src/overlays/actors/ovl_En_Zf/z_en_zf.c"
    "src/overlays/actors/ovl_En_Zf/z_en_zf.h"
    "src/overlays/actors/ovl_En_Zl1/z_en_zl1.c"
    "src/overlays/actors/ovl_En_Zl1/z_en_zl1.h"
    "src/overlays/actors/ovl_En_Zl1/z_en_zl1_camera_data.c"
    "src/overlays/actors/ovl_En_Zl1/z_en_zl1_cutscene_data.c"
    "src/overlays/actors/ovl_En_Zl2/z_en_zl2.c"
    "src/overlays/actors/ovl_En_Zl2/z_en_zl2.h"
    "src/overlays/actors/ovl_En_Zl3/z_en_zl3.c"
    "src/overlays/actors/ovl_En_Zl3/z_en_zl3.h"
    "src/overlays/actors/ovl_En_Zl4/z_en_zl4.c"
    "src/overlays/actors/ovl_En_Zl4/z_en_zl4.h"
    "src/overlays/actors/ovl_En_Zl4/z_en_zl4_cutscene_data.c"
    "src/overlays/actors/ovl_En_Zo/z_en_zo.c"
    "src/overlays/actors/ovl_En_Zo/z_en_zo.h"
    "src/overlays/actors/ovl_End_Title/z_end_title.c"
    "src/overlays/actors/ovl_End_Title/z_end_title.h"
    "src/overlays/actors/ovl_Fishing/z_fishing.c"
    "src/overlays/actors/ovl_Fishing/z_fishing.h"
    "src/overlays/actors/ovl_Item_B_Heart/z_item_b_heart.c"
    "src/overlays/actors/ovl_Item_B_Heart/z_item_b_heart.h"
    "src/overlays/actors/ovl_Item_Etcetera/z_item_etcetera.c"
    "src/overlays/actors/ovl_Item_Etcetera/z_item_etcetera.h"
    "src/overlays/actors/ovl_Item_Inbox/z_item_inbox.c"
    "src/overlays/actors/ovl_Item_Inbox/z_item_inbox.h"
    "src/overlays/actors/ovl_Item_Ocarina/z_item_ocarina.c"
    "src/overlays/actors/ovl_Item_Ocarina/z_item_ocarina.h"
    "src/overlays/actors/ovl_Item_Shield/z_item_shield.c"
    "src/overlays/actors/ovl_Item_Shield/z_item_shield.h"
    "src/overlays/actors/ovl_Magic_Dark/z_magic_dark.c"
    "src/overlays/actors/ovl_Magic_Dark/z_magic_dark.h"
    "src/overlays/actors/ovl_Magic_Fire/z_magic_fire.c"
    "src/overlays/actors/ovl_Magic_Fire/z_magic_fire.h"
    "src/overlays/actors/ovl_Magic_Wind/z_magic_wind.c"
    "src/overlays/actors/ovl_Magic_Wind/z_magic_wind.h"
    "src/overlays/actors/ovl_Mir_Ray/z_mir_ray.c"
    "src/overlays/actors/ovl_Mir_Ray/z_mir_ray.h"
    "src/overlays/actors/ovl_Obj_Bean/z_obj_bean.c"
    "src/overlays/actors/ovl_Obj_Bean/z_obj_bean.h"
    "src/overlays/actors/ovl_Obj_Blockstop/z_obj_blockstop.c"
    "src/overlays/actors/ovl_Obj_Blockstop/z_obj_blockstop.h"
    "src/overlays/actors/ovl_Obj_Bombiwa/z_obj_bombiwa.c"
    "src/overlays/actors/ovl_Obj_Bombiwa/z_obj_bombiwa.h"
    "src/overlays/actors/ovl_Obj_Comb/z_obj_comb.c"
    "src/overlays/actors/ovl_Obj_Comb/z_obj_comb.h"
    "src/overlays/actors/ovl_Obj_Dekujr/z_obj_dekujr.c"
    "src/overlays/actors/ovl_Obj_Dekujr/z_obj_dekujr.h"
    "src/overlays/actors/ovl_Obj_Elevator/z_obj_elevator.c"
    "src/overlays/actors/ovl_Obj_Elevator/z_obj_elevator.h"
    "src/overlays/actors/ovl_Obj_Hamishi/z_obj_hamishi.c"
    "src/overlays/actors/ovl_Obj_Hamishi/z_obj_hamishi.h"
    "src/overlays/actors/ovl_Obj_Hana/z_obj_hana.c"
    "src/overlays/actors/ovl_Obj_Hana/z_obj_hana.h"
    "src/overlays/actors/ovl_Obj_Hsblock/z_obj_hsblock.c"
    "src/overlays/actors/ovl_Obj_Hsblock/z_obj_hsblock.h"
    "src/overlays/actors/ovl_Obj_Ice_Poly/z_obj_ice_poly.c"
    "src/overlays/actors/ovl_Obj_Ice_Poly/z_obj_ice_poly.h"
    "src/overlays/actors/ovl_Obj_Kibako/z_obj_kibako.c"
    "src/overlays/actors/ovl_Obj_Kibako/z_obj_kibako.h"
    "src/overlays/actors/ovl_Obj_Kibako2/z_obj_kibako2.c"
    "src/overlays/actors/ovl_Obj_Kibako2/z_obj_kibako2.h"
    "src/overlays/actors/ovl_Obj_Lift/z_obj_lift.c"
    "src/overlays/actors/ovl_Obj_Lift/z_obj_lift.h"
    "src/overlays/actors/ovl_Obj_Lightswitch/z_obj_lightswitch.c"
    "src/overlays/actors/ovl_Obj_Lightswitch/z_obj_lightswitch.h"
    "src/overlays/actors/ovl_Obj_Makekinsuta/z_obj_makekinsuta.c"
    "src/overlays/actors/ovl_Obj_Makekinsuta/z_obj_makekinsuta.h"
    "src/overlays/actors/ovl_Obj_Makeoshihiki/z_obj_makeoshihiki.c"
    "src/overlays/actors/ovl_Obj_Makeoshihiki/z_obj_makeoshihiki.h"
    "src/overlays/actors/ovl_Obj_Mure/z_obj_mure.c"
    "src/overlays/actors/ovl_Obj_Mure/z_obj_mure.h"
    "src/overlays/actors/ovl_Obj_Mure2/z_obj_mure2.c"
    "src/overlays/actors/ovl_Obj_Mure2/z_obj_mure2.h"
    "src/overlays/actors/ovl_Obj_Mure3/z_obj_mure3.c"
    "src/overlays/actors/ovl_Obj_Mure3/z_obj_mure3.h"
    "src/overlays/actors/ovl_Obj_Oshihiki/z_obj_oshihiki.c"
    "src/overlays/actors/ovl_Obj_Oshihiki/z_obj_oshihiki.h"
    "src/overlays/actors/ovl_Obj_Roomtimer/z_obj_roomtimer.c"
    "src/overlays/actors/ovl_Obj_Roomtimer/z_obj_roomtimer.h"
    "src/overlays/actors/ovl_Obj_Switch/z_obj_switch.c"
    "src/overlays/actors/ovl_Obj_Switch/z_obj_switch.h"
    "src/overlays/actors/ovl_Obj_Syokudai/z_obj_syokudai.c"
    "src/overlays/actors/ovl_Obj_Syokudai/z_obj_syokudai.h"
    "src/overlays/actors/ovl_Obj_Timeblock/z_obj_timeblock.c"
    "src/overlays/actors/ovl_Obj_Timeblock/z_obj_timeblock.h"
    "src/overlays/actors/ovl_Obj_Tsubo/z_obj_tsubo.c"
    "src/overlays/actors/ovl_Obj_Tsubo/z_obj_tsubo.h"
    "src/overlays/actors/ovl_Obj_Warp2block/z_obj_warp2block.c"
    "src/overlays/actors/ovl_Obj_Warp2block/z_obj_warp2block.h"
    "src/overlays/actors/ovl_Object_Kankyo/z_object_kankyo.c"
    "src/overlays/actors/ovl_Object_Kankyo/z_object_kankyo.h"
    "src/overlays/actors/ovl_Oceff_Spot/z_oceff_spot.c"
    "src/overlays/actors/ovl_Oceff_Spot/z_oceff_spot.h"
    "src/overlays/actors/ovl_Oceff_Storm/z_oceff_storm.c"
    "src/overlays/actors/ovl_Oceff_Storm/z_oceff_storm.h"
    "src/overlays/actors/ovl_Oceff_Wipe/z_oceff_wipe.c"
    "src/overlays/actors/ovl_Oceff_Wipe/z_oceff_wipe.h"
    "src/overlays/actors/ovl_Oceff_Wipe2/z_oceff_wipe2.c"
    "src/overlays/actors/ovl_Oceff_Wipe2/z_oceff_wipe2.h"
    "src/overlays/actors/ovl_Oceff_Wipe3/z_oceff_wipe3.c"
    "src/overlays/actors/ovl_Oceff_Wipe3/z_oceff_wipe3.h"
    "src/overlays/actors/ovl_Oceff_Wipe4/z_oceff_wipe4.c"
    "src/overlays/actors/ovl_Oceff_Wipe4/z_oceff_wipe4.h"
    "src/overlays/actors/ovl_player_actor/z_player.c"
    "src/overlays/actors/ovl_Shot_Sun/z_shot_sun.c"
    "src/overlays/actors/ovl_Shot_Sun/z_shot_sun.h"
    "src/overlays/actors/ovl_Scene_Exit/z_scene_exit.c"
)
source_group("Source Files\\src\\overlays\\actors" FILES ${Source_Files__src__overlays__actors})

source_group("soh\\resource\\type" REGULAR_EXPRESSION "soh/resource/type/*")
source_group("soh\\resource\\type\\scenecommand" REGULAR_EXPRESSION "soh/resource/type/scenecommand/*")
source_group("soh\\resource\\importer" REGULAR_EXPRESSION "soh/resource/importer/*")
source_group("soh\\resource\\importer\\scenecommand" REGULAR_EXPRESSION "soh/resource/importer/scenecommand/*")
# }}}

# src (decomp) {{{
file(GLOB_RECURSE src__ RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "src/*.c" "src/*.h")

list(APPEND src__ ${CMAKE_BINARY_DIR}/build.c)
list(APPEND src__ ${CMAKE_CURRENT_SOURCE_DIR}/properties.h)
list(APPEND src__ ${CMAKE_CURRENT_SOURCE_DIR}/Resource.rc)
list(FILTER src__ EXCLUDE REGEX "src/dmadata/*")
list(FILTER src__ EXCLUDE REGEX "src/elf_message/*")
list(FILTER src__ EXCLUDE REGEX "src/libultra/io/*")
list(FILTER src__ EXCLUDE REGEX "src/libultra/libc/*")
list(FILTER src__ EXCLUDE REGEX "src/libultra/os/*")
list(FILTER src__ EXCLUDE REGEX "src/libultra/rmon/*")
list(APPEND src__ "src/libultra/libc/sprintf.c")
list(REMOVE_ITEM src__ "src/libultra/gu/cosf.c")
list(REMOVE_ITEM src__ "src/libultra/gu/lookat.c")
list(REMOVE_ITEM src__ "src/libultra/gu/lookathil.c")
list(REMOVE_ITEM src__ "src/libultra/gu/perspective.c")
list(REMOVE_ITEM src__ "src/libultra/gu/position.c")
list(REMOVE_ITEM src__ "src/libultra/gu/sinf.c")
list(REMOVE_ITEM src__ "src/libultra/gu/sinf.c")
list(REMOVE_ITEM src__ "src/libultra/gu/sqrtf.c")
list(REMOVE_ITEM src__ "src/libultra/gu/us2dex.c")

source_group("src" REGULAR_EXPRESSION "src/*")
source_group("src\\build" FILES ${CMAKE_BINARY_DIR}/build.c ${CMAKE_CURRENT_SOURCE_DIR}/properties.h ${CMAKE_CURRENT_SOURCE_DIR}/Resource.rc)
source_group("src\\boot" REGULAR_EXPRESSION "src/boot/*")
source_group("src\\buffers" REGULAR_EXPRESSION "src/buffers/*")
source_group("src\\code" REGULAR_EXPRESSION "src/code/*")
source_group("src\\libultra" REGULAR_EXPRESSION "src/libultra/*")
source_group("src\\overlays\\actors" REGULAR_EXPRESSION "src/overlays/actors/*")
source_group("src\\overlays\\effects" REGULAR_EXPRESSION "src/overlays/effects/*")
source_group("src\\overlays\\gamestates" REGULAR_EXPRESSION  "src/overlays/gamestates/*")
source_group("src\\overlays\\misc" REGULAR_EXPRESSION  "src/overlays/misc/*")
# }}}

set(ALL_FILES
    ${Header_Files}
    ${Header_Files__include}
    ${soh__}
    ${soh__Enhancements}
    ${soh__Extractor}
    ${soh__Resource}
    ${src__}
)

################################################################################
# Target
################################################################################
add_executable(${PROJECT_NAME} ${ALL_FILES})

if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
use_props(${PROJECT_NAME} "${CMAKE_CONFIGURATION_TYPES}" "${DEFAULT_CXX_PROPS}")
endif()

set(ROOT_NAMESPACE soh)

if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
	set_target_properties(${PROJECT_NAME} PROPERTIES
		VS_GLOBAL_KEYWORD "Win32Proj"
	)
	if("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x64")
		set_target_properties(${PROJECT_NAME} PROPERTIES
			INTERPROCEDURAL_OPTIMIZATION_RELEASE "TRUE"
		)
	elseif("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "Win32")
		set_target_properties(${PROJECT_NAME} PROPERTIES
			INTERPROCEDURAL_OPTIMIZATION_RELEASE "TRUE"
		)
	endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set_target_properties(${PROJECT_NAME} PROPERTIES
        XCODE_ATTRIBUTE_CLANG_ENABLE_OBJC_ARC YES 
        OUTPUT_NAME "soh-macos"
    )
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
	set_target_properties(${PROJECT_NAME} PROPERTIES
	OUTPUT_NAME "soh.elf"
	)
endif()
################################################################################
# MSVC runtime library
################################################################################
if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
	get_property(MSVC_RUNTIME_LIBRARY_DEFAULT TARGET ${PROJECT_NAME} PROPERTY MSVC_RUNTIME_LIBRARY)
	if("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x64")
		string(CONCAT "MSVC_RUNTIME_LIBRARY_STR"
			$<$<CONFIG:Debug>:
				MultiThreadedDebug
			>
			$<$<CONFIG:Release>:
				MultiThreaded
			>
			$<$<NOT:$<OR:$<CONFIG:Debug>,$<CONFIG:Release>>>:${MSVC_RUNTIME_LIBRARY_DEFAULT}>
		)
	elseif("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "Win32")
		string(CONCAT "MSVC_RUNTIME_LIBRARY_STR"
			$<$<CONFIG:Debug>:
				MultiThreadedDebug
			>
			$<$<CONFIG:Release>:
				MultiThreaded
			>
			$<$<NOT:$<OR:$<CONFIG:Debug>,$<CONFIG:Release>>>:${MSVC_RUNTIME_LIBRARY_DEFAULT}>
		)
	endif()
	set_target_properties(${PROJECT_NAME} PROPERTIES MSVC_RUNTIME_LIBRARY ${MSVC_RUNTIME_LIBRARY_STR})
endif()
################################################################################
# Find/download Boost
################################################################################
include(FetchContent)
FetchContent_Declare(
    Boost
    URL      https://boostorg.jfrog.io/artifactory/main/release/1.81.0/source/boost_1_81_0.tar.gz
    URL_HASH SHA256=205666dea9f6a7cfed87c7a6dfbeb52a2c1b9de55712c9c1a87735d7181452b6
    SOURCE_SUBDIR "null" # Set to a nonexistent directory so boost is not built (we don't need to build it)
    DOWNLOAD_EXTRACT_TIMESTAMP false # supress timestamp warning, not needed since the url wont change
)

set(Boost_NO_BOOST_CMAKE false)
set(BOOST_INCLUDEDIR ${FETCHCONTENT_BASE_DIR}/boost-src) # Location where FetchContent stores the source
message("Searching for Boost installation")
find_package(Boost)

if (NOT ${Boost_FOUND})
    message("Boost not found. Downloading now...")
    FetchContent_MakeAvailable(Boost)
    message("Boost downloaded to " ${FETCHCONTENT_BASE_DIR}/boost-src)
    set(BOOST-INCLUDE ${FETCHCONTENT_BASE_DIR}/boost-src)
else()
    message("Boost found in " ${Boost_INCLUDE_DIRS})
    set(BOOST-INCLUDE ${Boost_INCLUDE_DIRS})
endif()
################################################################################
# Compile definitions
################################################################################
find_package(SDL2)
set(SDL2-INCLUDE ${SDL2_INCLUDE_DIRS})

if (BUILD_CROWD_CONTROL)
    find_package(SDL2_net)
    set(SDL2-NET-INCLUDE ${SDL_NET_INCLUDE_DIRS})
endif()

target_include_directories(${PROJECT_NAME} PRIVATE assets
	${CMAKE_CURRENT_SOURCE_DIR}/include/
	${CMAKE_CURRENT_SOURCE_DIR}/src/
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/include
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/binarytools
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/log
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/debug
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/menu
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/misc
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/core
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/resource
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/resource/type
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/audio
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/extern
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/extern/Mercury
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/extern/tinyxml2
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/libultraship/Lib/
	${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/libultraship/Lib/libjpeg/include/
	${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/libultraship/Lib/spdlog/include/
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/graphic/Fast3D/U64/PR
    ${CMAKE_CURRENT_SOURCE_DIR}/../libultraship/src/graphic
	${CMAKE_CURRENT_SOURCE_DIR}/../ZAPDTR/ZAPDUtils
    ${CMAKE_CURRENT_SOURCE_DIR}/../ZAPDTR/ZAPD/resource/type
	${SDL2-INCLUDE}
    ${SDL2-NET-INCLUDE}
    ${BOOST-INCLUDE}
	${CMAKE_CURRENT_SOURCE_DIR}/assets/
	.
)

if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
	if("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x64")
		target_compile_definitions(${PROJECT_NAME} PRIVATE
			"$<$<CONFIG:Debug>:"
				"_DEBUG;"
				"_CRT_SECURE_NO_WARNINGS;"
				"ENABLE_DX11;"
			">"
			"$<$<CONFIG:Release>:"
				"NDEBUG"
			">"
            "$<$<BOOL:${BUILD_CROWD_CONTROL}>:ENABLE_CROWD_CONTROL>"
			"INCLUDE_GAME_PRINTF;"
            "ENABLE_CROWD_CONTROL;"
			"UNICODE;"
			"_UNICODE"
			STORMLIB_NO_AUTO_LINK
			"_CRT_SECURE_NO_WARNINGS;"
		)
	elseif("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "Win32")
		target_compile_definitions(${PROJECT_NAME} PRIVATE
			"$<$<CONFIG:Debug>:"
				"NOINCLUDE_GAME_PRINTF;"
				"_DEBUG;"
				"_CRT_SECURE_NO_WARNINGS;"
				"ENABLE_OPENGL"
			">"
			"$<$<CONFIG:Release>:"
				"NDEBUG;"
			">"
			"INCLUDE_GAME_PRINTF;"
			"WIN32;"
			"UNICODE;"
			"_UNICODE"
			STORMLIB_NO_AUTO_LINK
		)
	endif()
elseif (CMAKE_SYSTEM_NAME STREQUAL "CafeOS")
	target_compile_definitions(${PROJECT_NAME} PRIVATE
		"$<$<CONFIG:Debug>:"
			"_DEBUG"
		">"
		"$<$<CONFIG:Release>:"
			"NDEBUG"
		">"
        "SPDLOG_ACTIVE_LEVEL=3;"
		"SPDLOG_NO_THREAD_ID;"
        "SPDLOG_NO_TLS;"
		"STBI_NO_THREAD_LOCALS;"
	)
elseif ("${CMAKE_CXX_COMPILER_ID}" MATCHES "GNU|Clang|AppleClang")
	target_compile_definitions(${PROJECT_NAME} PRIVATE
		"$<$<CONFIG:Debug>:"
			"_DEBUG"
		">"
		"$<$<CONFIG:Release>:"
			"NDEBUG"
		">"
        "$<$<BOOL:${BUILD_CROWD_CONTROL}>:ENABLE_CROWD_CONTROL>"
		"SPDLOG_ACTIVE_LEVEL=0;"
		"_CONSOLE;"
		"_CRT_SECURE_NO_WARNINGS;"
		"ENABLE_OPENGL;"
		"UNICODE;"
		"_UNICODE"
	)
endif()
################################################################################
# Compile and link options
################################################################################
if(MSVC)
    if("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x64")
        target_compile_options(${PROJECT_NAME} PRIVATE
            $<$<CONFIG:Debug>:
                /w;
                /Od
            >
            $<$<CONFIG:Release>:
                /Oi;
                /Gy;
                /W3
            >
            /sdl-;
            /permissive-;
            /MP;
            ${DEFAULT_CXX_DEBUG_INFORMATION_FORMAT};
            ${DEFAULT_CXX_EXCEPTION_HANDLING}
        )
        target_compile_options(${PROJECT_NAME} PRIVATE  $<$<CONFIG:Debug>:/ZI;>)
    elseif("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "Win32")
        target_compile_options(${PROJECT_NAME} PRIVATE
            $<$<CONFIG:Debug>:
                /RTCs
            >
            $<$<CONFIG:Release>:
                /O2;
                /Oi;
                /Gy
            >
            /permissive-;
            /MP;
            /sdl-;
            /w;
            ${DEFAULT_CXX_DEBUG_INFORMATION_FORMAT};
            ${DEFAULT_CXX_EXCEPTION_HANDLING}
        )
    endif()
    if("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x64")
        target_link_options(${PROJECT_NAME} PRIVATE
            $<$<CONFIG:Debug>:
                /INCREMENTAL
            >
            $<$<CONFIG:Release>:
                /OPT:REF;
                /OPT:ICF;
                /INCREMENTAL:NO;
                /FORCE:MULTIPLE
            >
            /DEBUG;
            /SUBSYSTEM:WINDOWS
        )
    elseif("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "Win32")
        target_link_options(${PROJECT_NAME} PRIVATE
            $<$<CONFIG:Debug>:
                /STACK:8777216
            >
            $<$<CONFIG:Release>:
                /OPT:REF;
                /OPT:ICF;
                /INCREMENTAL:NO;
                /FORCE:MULTIPLE
            >
            /DEBUG;
            /SUBSYSTEM:WINDOWS
        )
    endif()
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
        target_compile_options(${PROJECT_NAME} PRIVATE
            -Wall -Wextra -Wno-error
            -Wno-return-type
            -Wno-unused-parameter
            -Wno-unused-function
            -Wno-unused-variable
            -Wno-missing-field-initializers
            -Wno-parentheses
            -Wno-narrowing
            -Wno-c++11-narrowing
            -Wno-implicit-function-declaration
            -Wno-missing-braces
            -Wno-incompatible-pointer-types
            $<$<COMPILE_LANGUAGE:CXX>:-fpermissive>
            $<$<COMPILE_LANGUAGE:CXX>:-Wno-deprecated-enum-enum-conversion>
            -pthread
        )

        target_link_options(${PROJECT_NAME} PRIVATE
            -pthread
        )
    elseif (CMAKE_SYSTEM_NAME STREQUAL "NintendoSwitch")
        target_compile_options(${PROJECT_NAME} PRIVATE
            -Wall -Wextra -Wno-error
            -Wno-return-type
            -Wno-unused-parameter
            -Wno-unused-function
            -Wno-unused-variable
            -Wno-missing-field-initializers
            -Wno-parentheses
            -Wno-narrowing
            -Wno-c++11-narrowing
            -Wno-implicit-function-declaration
            -Wno-missing-braces
            -Wno-incompatible-pointer-types
            $<$<COMPILE_LANGUAGE:CXX>:-fpermissive>
            $<$<COMPILE_LANGUAGE:CXX>:-Wno-deprecated-enum-enum-conversion>
            -pthread
        )

        target_link_options(${PROJECT_NAME} PRIVATE
            -pthread
        )
    elseif (CMAKE_SYSTEM_NAME STREQUAL "CafeOS")
        target_compile_options(${PROJECT_NAME} PRIVATE
            -O2

            # disable some warnings to not clutter output
            -Wno-multichar
            -Wno-return-type
            -Wno-narrowing
            -Wno-switch-outside-range
            $<$<COMPILE_LANGUAGE:C>:
                -Wno-incompatible-pointer-types
                -Wno-discarded-array-qualifiers
                -Wno-discarded-qualifiers
                -Wno-int-conversion
                -Wno-implicit-function-declaration
                -Wno-builtin-declaration-mismatch
                -Wno-switch-unreachable
                -Wno-stringop-overflow
            >
        )
    else()
        if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64")
		set(CPU_OPTION -msse2 -mfpmath=sse)
        endif()

        target_compile_options(${PROJECT_NAME} PRIVATE
            -Wall -Wextra -Wno-error
            -Wno-unused-parameter
            -Wno-unused-function
            -Wno-unused-variable
            -Wno-missing-field-initializers
            -Wno-parentheses
            -Wno-narrowing
            -Wno-missing-braces
            -Wno-incompatible-pointer-types
            $<$<COMPILE_LANGUAGE:CXX>:-fpermissive>
            $<$<COMPILE_LANGUAGE:CXX>:-Wno-deprecated-enum-enum-conversion>
            -pthread
	    ${CPU_OPTION}
        )

        target_link_options(${PROJECT_NAME} PRIVATE
            -pthread
            -Wl,-export-dynamic
        )
    endif()
endif()
################################################################################
# Pre build events
################################################################################
if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
	add_custom_command_if(
		TARGET ${PROJECT_NAME}
		PRE_BUILD
		COMMANDS
		COMMAND   $<CONFIG:Debug> copy /b $<SHELL_PATH:${CMAKE_BINARY_DIR}/>build.c +,,
	)
endif()

if(NOT CMAKE_SYSTEM_NAME MATCHES "NintendoSwitch|CafeOS")
    add_custom_command(
		TARGET ${PROJECT_NAME}
		POST_BUILD
		COMMENT "Copying asset xmls..."
		COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/OTRGui/assets $<TARGET_FILE_DIR:soh>/assets
		COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/soh/assets/xml $<TARGET_FILE_DIR:soh>/assets/extractor/xmls
		COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/OTRExporter/assets $<TARGET_FILE_DIR:soh>/Extract/assets
	)
endif()
################################################################################
# Dependencies
################################################################################
add_dependencies(${PROJECT_NAME}
    ZAPDUtils
    libultraship
)
if(NOT CMAKE_SYSTEM_NAME MATCHES "NintendoSwitch|CafeOS")
add_dependencies(${PROJECT_NAME}
    ZAPDLib
)
endif()

if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
	if("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "x64")
		set(ADDITIONAL_LIBRARY_DEPENDENCIES
			"libultraship;"
			"ZAPDUtils;"
			"ZAPDLib;"
			"glu32;"
			"SDL2::SDL2;"
			"SDL2::SDL2main;"
            "$<$<BOOL:${BUILD_CROWD_CONTROL}>:SDL2_net::SDL2_net-static>"
			"glfw;"
			"winmm;"
			"imm32;"
			"version;"
			"setupapi"
		)
	elseif("${CMAKE_VS_PLATFORM_NAME}" STREQUAL "Win32") 
		set(ADDITIONAL_LIBRARY_DEPENDENCIES
			"libultraship;"
			"ZAPDUtils;"
			"ZAPDLib;"
			"glu32;"
			"SDL2::SDL2;"
			"SDL2::SDL2main;"
			"glfw;"
			"winmm;"
			"imm32;"
			"version;"
			"setupapi"
		)
	endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "NintendoSwitch")
    find_package(SDL2)
    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads REQUIRED)
    set(ADDITIONAL_LIBRARY_DEPENDENCIES
        "libultraship;"
        "ZAPDUtils;"
        SDL2::SDL2
        -lglad
        Threads::Threads
    )
elseif(CMAKE_SYSTEM_NAME STREQUAL "CafeOS")
    find_package(SDL2 REQUIRED)
    set(ADDITIONAL_LIBRARY_DEPENDENCIES
        "libultraship;"
        SDL2::SDL2-static

        "$<$<CONFIG:Debug>:-Wl,--wrap=abort>"
    )
    target_include_directories(${PROJECT_NAME} PRIVATE
        ${DEVKITPRO}/portlibs/wiiu/include/
    )
else()
    find_package(SDL2)
    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads REQUIRED)
 	set(ADDITIONAL_LIBRARY_DEPENDENCIES
        "libultraship;"
        "ZAPDUtils;"
		"ZAPDLib;"
		SDL2::SDL2
        "$<$<BOOL:${BUILD_CROWD_CONTROL}>:SDL2_net::SDL2_net>"
		${CMAKE_DL_LIBS}
		Threads::Threads
	)
endif()

if(NOT CMAKE_SYSTEM_NAME MATCHES "NintendoSwitch|CafeOS")
INSTALL(TARGETS soh DESTINATION . COMPONENT ship)
endif()

if (CMAKE_SYSTEM_NAME STREQUAL "Windows")
INSTALL(FILES $<TARGET_PDB_FILE:soh> DESTINATION ./debug COMPONENT ship)
endif()

find_program(CURL NAMES curl DOC "Path to the curl program.  Used to download files.")
execute_process(COMMAND ${CURL} -sSfL https://raw.githubusercontent.com/gabomdq/SDL_GameControllerDB/master/gamecontrollerdb.txt -o ${CMAKE_BINARY_DIR}/gamecontrollerdb.txt OUTPUT_VARIABLE RESULT)

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Darwin")
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/macosx/Info.plist.in ${CMAKE_BINARY_DIR}/macosx/Info.plist @ONLY)
INSTALL(FILES ${CMAKE_BINARY_DIR}/gamecontrollerdb.txt DESTINATION ../MacOS COMPONENT ship)
INSTALL(FILES ${CMAKE_BINARY_DIR}/soh/soh.otr DESTINATION ../Resources COMPONENT ship)
elseif(NOT "${CMAKE_SYSTEM_NAME}" MATCHES "NintendoSwitch|CafeOS")
INSTALL(FILES ${CMAKE_BINARY_DIR}/gamecontrollerdb.txt DESTINATION . COMPONENT ship)
endif()

if(CMAKE_SYSTEM_NAME MATCHES "NintendoSwitch|CafeOS")
    if (NOT TARGET pathconf)
        add_library(pathconf OBJECT platform/pathconf.c)
    endif()
    target_link_libraries(${PROJECT_NAME} PRIVATE "${ADDITIONAL_LIBRARY_DEPENDENCIES}" $<TARGET_OBJECTS:pathconf> )
else()
    target_link_libraries(${PROJECT_NAME} PRIVATE "${ADDITIONAL_LIBRARY_DEPENDENCIES}")
endif()

if(CMAKE_SYSTEM_NAME MATCHES "NintendoSwitch")

nx_generate_nacp(Ship.nacp
   NAME "Ship of Harkinian"
   AUTHOR "${PROJECT_TEAM}"
   VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}"
)
 
nx_create_nro(soh
    NACP Ship.nacp
    ICON ${CMAKE_CURRENT_SOURCE_DIR}/icon.jpg
)

INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/soh.nro DESTINATION . COMPONENT ship)

elseif(CMAKE_SYSTEM_NAME MATCHES "CafeOS")

wut_create_rpx(${PROJECT_NAME})

wut_create_wuhb(${PROJECT_NAME}
	NAME       "Ship of Harkinian"
	SHORTNAME  "SoH"
	AUTHOR     "${PROJECT_TEAM}"
	ICON       ${CMAKE_CURRENT_SOURCE_DIR}/icon.jpg
)

INSTALL(FILES ${CMAKE_CURRENT_BINARY_DIR}/soh.rpx ${CMAKE_CURRENT_BINARY_DIR}/soh.wuhb DESTINATION . COMPONENT ship)

endif()
