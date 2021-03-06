cmake_minimum_required(VERSION 2.8)
project(Ravi)

enable_language(CXX)
enable_language(C)
enable_testing()

# Get access to CMake helpers
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

# By default LLVM JIT is ON, and GCC JIT is OFF
# Only one can be ON, and at last one of them must be ON
option(LLVM_JIT "Controls whether LLVM JIT compilation will be enabled, default is OFF" OFF)
option(GCC_JIT "Controls whether GCC JIT compilation will be enabled, default is OFF" OFF)
option(NANO_JIT "Controls whether NanoJIT compilation will be enabled, default is OFF" OFF)
option(STATIC_BUILD "Build static version of Ravi, default is OFF" OFF)
option(EMBEDDED_DMRC "Controls whether the embedded dmrC feature should be enabled, default is OFF" OFF)

# We cannot link to both LLVM and GCC JIT
if (LLVM_JIT AND GCC_JIT)
    message(FATAL_ERROR
            "Both LLVM_JIT and GCC_JIT cannot be set to ON at the same time")
elseif (LLVM_JIT AND NANO_JIT)
    message(FATAL_ERROR
            "Both LLVM_JIT and NANO_JIT cannot be set to ON at the same time")    
elseif (GCC_JIT AND NANO_JIT)
    message(FATAL_ERROR
            "Both GCC_JIT and NANO_JIT cannot be set to ON at the same time")    
endif()

if (STATIC_BUILD)
    message(STATUS "STATIC library build enabled")
else()
    message(STATUS "DYNAMIC library build enabled")

endif()

if (LLVM_JIT)
    find_package(LLVM REQUIRED CONFIG)

    message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
    message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

    include_directories(${LLVM_INCLUDE_DIRS})
    add_definitions(${LLVM_DEFINITIONS})
    add_definitions(-DUSE_LLVM)
endif()

if (NANO_JIT)
    set(EMBEDDED_DMRC ON)
endif()

if (EMBEDDED_DMRC)
    add_definitions(-DUSE_DMR_C)
endif()

if (GCC_JIT)
    find_package(GCCJIT REQUIRED)

    message(STATUS "Found GCCJIT")

    include_directories(${GCCJIT_INCLUDE_DIRS})
    add_definitions(-DUSE_GCCJIT)
endif()

if (NANO_JIT)
    find_package(NanoJIT REQUIRED)
    include_directories(${NANOJITEXTRA_INCLUDE_DIRS})
    add_definitions(-DUSE_NANOJIT)
endif()

if (NOT LLVM_JIT AND NOT GCC_JIT AND NOT NANO_JIT)
    message(WARNING "Neither LLVM, NanoJIT nor gccjit will be enabled; specify -DLLVM_JIT, -DNANO_JIT or -DGCC_JIT to enable")
endif()

if (MSVC)
    set(CMAKE_C_FLAGS_DEBUG "/Od /D_DEBUG /MDd /Zi /RTC1 /EHsc")
    set(CMAKE_C_FLAGS_RELEASE "/DNDEBUG /O2 /MD /EHsc")
    set(CMAKE_C_FLAGS_RELWITHDEBINFO "/DNDEBUG /O2 /MD /Zi /EHsc")
    set(CMAKE_C_FLAGS_MINSIZEREL "/DNDEBUG /O2 /MD /EHsc")

    set(CMAKE_CXX_FLAGS "/EHsc")
    set(CMAKE_CXX_FLAGS_DEBUG "/Od /D_DEBUG /MDd /Zi /RTC1 /EHsc")
    set(CMAKE_CXX_FLAGS_RELEASE "/DNDEBUG /O2 /MD /EHsc")
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "/DNDEBUG /O2 /MD /Zi /EHsc")
    set(CMAKE_CXX_FLAGS_MINSIZEREL "/DNDEBUG /O2 /MD /EHsc")

    # Disable signed/unsigned int conversion warnings.
    add_definitions("/wd4018")
    # Disable warning about using struct/class for the same symobl.
    add_definitions("/wd4099")
    # Disable performance warning about int-to-bool conversion.
    add_definitions("/wd4800")
    # Disable signed/unsigned int conversion warnings.
    add_definitions("/wd4018")
    # Disable warning about the insecurity of using "std::copy".
    add_definitions("/wd4996")
    add_definitions("/wd4291")
    add_definitions("/wd4624")
    add_definitions("/wd4141")
    add_definitions("/DLUA_COMPAT_5_2")
endif ()


if (CMAKE_COMPILER_IS_GNUCXX AND NOT APPLE)
    if (NOT WIN32)
        # assume Linux
        set(CXX_OPTIONS -DLUA_USE_LINUX)
    endif ()
    # -fsanitize=bounds -fsanitize=alignment -fsanitize=object-size
    set(SANITIZER_FLAGS "-fsanitize=address")
    set(CMAKE_C_FLAGS "-std=c99 -O2 -fomit-frame-pointer -Wall -Wextra -Winline -DLUA_COMPAT_5_2 ${CXX_OPTIONS}")
    set(CMAKE_C_FLAGS_DEBUG "${SANITIZER_FLAGS} -fno-omit-frame-pointer -std=c99 -O0 -g3 -Wall -Wextra -DLUA_COMPAT_5_2 ${CXX_OPTIONS}")
    set(CMAKE_C_FLAGS_RELEASE "-std=c99 -O2 -fomit-frame-pointer -Wall -Wextra -Winline -DLUA_COMPAT_5_2 ${CXX_OPTIONS}")
    set(CMAKE_C_FLAGS_RELWITHDEBINFO "${SANITIZER_FLAGS} -std=c99 -O1 -g3 -Wall -Wextra -DLUA_COMPAT_5_2 ${CXX_OPTIONS}")
    set(CMAKE_CXX_FLAGS "-fno-rtti -O2 -fomit-frame-pointer -Wall -Wno-sign-compare -Winline -std=c++14 -fno-exceptions -DLUA_COMPAT_5_2 ${CXX_OPTIONS}")
    set(CMAKE_CXX_FLAGS_RELEASE "-fno-rtti -O2 -fomit-frame-pointer -Wall -Wno-sign-compare -Winline -std=c++14 -fno-exceptions -DLUA_COMPAT_5_2 ${CXX_OPTIONS}")
    set(CMAKE_CXX_FLAGS_DEBUG "${SANITIZER_FLAGS} -fno-omit-frame-pointer -fno-rtti -O0 -g3 -Wall -Wno-sign-compare -std=c++14 -fno-exceptions -DLUA_COMPAT_5_2 ${CXX_OPTIONS}")
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${SANITIZER_FLAGS} -fno-rtti -O1 -g3 -Wall -Wno-sign-compare -std=c++14 -fno-exceptions -DLUA_COMPAT_5_2 ${CXX_OPTIONS}")
    # In case we are using gcc 5.1 set ABI version
    # add_definitions("-D_GLIBCXX_USE_CXX11_ABI=0")
endif ()

if (APPLE)
    set(CMAKE_C_FLAGS "-std=c99 -O3 -Wall -Wextra -DLUA_COMPAT_5_2 -DLUA_USE_MACOSX")
    set(CMAKE_C_FLAGS_DEBUG "-std=c99 -O0 -g3 -Wall -Wextra -DLUA_COMPAT_5_2 -DLUA_USE_MACOSX")
    set(CMAKE_C_FLAGS_RELEASE "-std=c99 -O3 -Wall -Wextra -DLUA_COMPAT_5_2 -DLUA_USE_MACOSX")
    set(CMAKE_C_FLAGS_RELWITHDEBINFO "-std=c99 -O1 -g3 -Wall -Wextra -DLUA_COMPAT_5_2 -DLUA_USE_MACOSX")
    set(CMAKE_CXX_FLAGS "-O3 -Wall -fno-rtti -Wno-sign-compare -std=c++14 -fno-exceptions -DLUA_COMPAT_5_2 -DLUA_USE_MACOSX")
    set(CMAKE_CXX_FLAGS_RELEASE "-fno-rtti -O3 -Wall -Wno-sign-compare -std=c++14 -fno-exceptions -DLUA_COMPAT_5_2 -DLUA_USE_MACOSX")
    set(CMAKE_CXX_FLAGS_DEBUG "-fno-rtti -O0 -g3 -Wall -Wno-sign-compare -std=c++14 -fno-exceptions -DLUA_COMPAT_5_2 -DLUA_USE_MACOSX")
    set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-fno-rtti -O1 -g3 -Wall -Wno-sign-compare -std=c++14 -fno-exceptions -DLUA_COMPAT_5_2 -DLUA_USE_MACOSX")
endif ()

include_directories("${PROJECT_SOURCE_DIR}/include")
add_definitions(-DLUA_COMPAT_MODULE)
if (CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "DEBUG")
    message(STATUS "Enabling Lua extended test harness 'ltests'")
    add_definitions(-DLUA_USER_H="ltests.h")
endif ()

# define LLVM JIT compiler sources
if (LLVM_JIT)
    set(LLVM_JIT_SRCS src/ravi_llvmjit.cpp src/ravi_llvmtypes.cpp
            src/ravi_llvmcodegen.cpp src/ravi_llvmforprep.cpp src/ravi_llvmcomp.cpp
            src/ravi_llvmreturn.cpp src/ravi_llvmload.cpp src/ravi_llvmforloop.cpp
            src/ravi_llvmarith1.cpp src/ravi_llvmcall.cpp src/ravi_llvmtable.cpp
            src/ravi_llvmarith2.cpp src/ravi_llvmtforcall.cpp src/ravi_llvmrest.cpp
            src/ravi_llvmluaapi.cpp)
endif ()
if (GCC_JIT)
    set(GCC_JIT_SRCS src/ravi_gccjit.c src/ravi_gcctypes.c
            src/ravi_gcccodegen.c src/ravi_gccforprep.c src/ravi_gcccomp.c
            src/ravi_gccreturn.c src/ravi_gccload.c src/ravi_gccforloop.c
            src/ravi_gccarith1.c src/ravi_gcccall.c src/ravi_gcctable.c
            src/ravi_gccarith2.c src/ravi_gcctforcall.c src/ravi_gccrest.c)
endif ()
if (NANO_JIT)
    set(NANO_JIT_SRCS src/ravi_nanojit.c)
endif()
if (NOT LLVM_JIT AND NOT GCC_JIT AND NOT NANO_JIT)
    set(NO_JIT_SRCS src/ravi_nojit.c)
endif()
# define the lua core source files
set(LUA_CORE_SRCS src/lapi.c src/lcode.c src/lctype.c src/ldebug.c src/ldo.c src/ldump.c
        src/lfunc.c src/lgc.c src/llex.c src/lmem.c src/lobject.c src/lopcodes.c
        src/lparser.c src/lstate.c src/lstring.c src/ltable.c src/ltm.c src/lundump.c
        src/lvm.c src/lzio.c src/ravijit.cpp src/ltests.c src/ravi_profile.c src/ravi_membuf.c
        src/ravi_jitshared.c src/bit.c)
# define the lua lib source files
set(LUA_LIB_SRCS src/lauxlib.c src/lbaselib.c src/lbitlib.c src/lcorolib.c src/ldblib.c src/liolib.c
        src/lmathlib.c src/loslib.c src/ltablib.c src/lstrlib.c src/loadlib.c src/linit.c src/lutf8lib.c)

file(GLOB RAVI_HEADERS "${PROJECT_SOURCE_DIR}/include/*.h")
set(LUA_HEADERS include/lua.h include/luaconf.h include/lualib.h include/lauxlib.h)

if (MSVC OR APPLE)
    source_group("Ravi Headers" FILES ${RAVI_HEADERS})
    source_group("Ravi Source Files" FILES ${LUA_CORE_SRCS} ${LUA_LIB_SRCS}
            ${LLVM_JIT_SRCS} ${GCC_JIT_SRCS})
    if (APPLE)
        set(EXTRA_LIBRARIES m readline)
    endif ()
elseif (NOT WIN32)
    # On Linux we need to link libdl to get access to
    # functions like dlopen()
    # ubsan can be added to get -fsanitize=undefined
    set(EXTRA_LIBRARIES m dl readline)
endif ()

if (EMBEDDED_DMRC)
    if (CMAKE_COMPILER_IS_GNUCC)
        execute_process(COMMAND ${CMAKE_C_COMPILER} --print-file-name=
            OUTPUT_VARIABLE GCC_BASE OUTPUT_STRIP_TRAILING_WHITESPACE)
        execute_process(COMMAND ${CMAKE_C_COMPILER} -print-multiarch
            OUTPUT_VARIABLE MULTIARCH_TRIPLET ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)

        add_definitions(-DGCC_BASE="${GCC_BASE}")
        add_definitions(-DMULTIARCH_TRIPLET="${MULTIARCH_TRIPLET}")
    endif()

    message( STATUS "GCC_BASE_DIR      : " ${GCC_BASE})
    message( STATUS "MULTIARCH_TRIPLET : " ${MULTIARCH_TRIPLET} )

    include_directories("${PROJECT_SOURCE_DIR}/dmr_c/src")

    set(DMR_C_HEADERS 
        dmr_c/src/allocate.h
        dmr_c/src/char.h
        dmr_c/src/expression.h
        dmr_c/src/flow.h
        dmr_c/src/ident-list.h
        dmr_c/src/linearize.h
        dmr_c/src/lib.h        
        dmr_c/src/parse.h
        dmr_c/src/port.h 
        dmr_c/src/ptrlist.h
        dmr_c/src/ptrmap.h        
        dmr_c/src/scope.h
        dmr_c/src/symbol.h
        dmr_c/src/target.h
        dmr_c/src/token.h 
        dmr_c/src/ssa.h 
        dmr_c/src/walksymbol.h         
        )

    set(DMR_C_SRCS 
        dmr_c/src/allocate.c
        dmr_c/src/builtin.c
        dmr_c/src/char.c
        dmr_c/src/cse.c
        dmr_c/src/expression.c
        dmr_c/src/evaluate.c
        dmr_c/src/expand.c
        dmr_c/src/flow.c
        dmr_c/src/inline.c
        dmr_c/src/lib.c
        dmr_c/src/linearize.c
        dmr_c/src/liveness.c
        dmr_c/src/memops.c
        dmr_c/src/parse.c
        dmr_c/src/target.c
        dmr_c/src/tokenize.c
        dmr_c/src/pre-process.c
        dmr_c/src/ptrlist.c
        dmr_c/src/ptrmap.c
        dmr_c/src/scope.c
        dmr_c/src/show-parse.c
        dmr_c/src/simplify.c
        dmr_c/src/symbol.c
        dmr_c/src/unssa.c
        dmr_c/src/ssa.c
        dmr_c/src/walksymbol.c
        src/ravi_dmrc_parsesymbols.c
        )

    if (LLVM_JIT)
        set(DMR_C_JIT_SRCS
            dmr_c/llvm-backend/sparse-llvm.c
            )
        set(DMR_C_JIT_HEADERS
            dmr_c/llvm-backend/dmr_c.h            
            )

        include_directories("${PROJECT_SOURCE_DIR}/dmr_c/llvm-backend")
    endif()

    if (NANO_JIT)
        set(DMR_C_JIT_SRCS
            dmr_c/nanojit-backend/sparse-nanojit.c
            )
        set(DMR_C_JIT_HEADERS
            dmr_c/nanojit-backend/dmr_c.h            
            )

        include_directories("${PROJECT_SOURCE_DIR}/dmr_c/nanojit-backend")
    endif()    
    
    if (MSVC OR APPLE)
        source_group("dmrC Headers" FILES ${DMR_C_HEADERS} ${DMR_C_JIT_HEADERS})
        source_group("dmrC Source Files" FILES ${DMR_C_SRCS} ${DMR_C_JIT_SRCS})
    endif()
endif()


# Enable minimal required LLVM components so that the
# the size of the resulting binary is manageable
if (LLVM_JIT)
    if (${LLVM_PACKAGE_VERSION} VERSION_LESS "3.8")
        set(LLVM_EXTRA_LIBS ipa)
    endif()
	if (NOT ${LLVM_PACKAGE_VERSION} VERSION_LESS "5.0.0")
		set(LLVM_EXTRA_LIBS orcjit)
	endif()
    message(STATUS "SYSTEM_PROCESSOR ${CMAKE_SYSTEM_PROCESSOR}")
    if (CMAKE_SYSTEM_PROCESSOR MATCHES "(x86)|(X86)|(amd64)|(AMD64)")
	set(LLVM_LIBS_PROCESSOR
		X86CodeGen
		X86AsmParser
		X86Disassembler
		X86AsmPrinter
		X86Desc
		X86Info
		X86Utils
	    )
    elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "arm")
	set(LLVM_LIBS_PROCESSOR
		ARMCodeGen
		ARMAsmParser
		ARMDisassembler
		ARMAsmPrinter
		ARMDesc
		ARMInfo
	    )
    endif()
    llvm_map_components_to_libnames(LLVM_LIBS
            Analysis
            AsmParser
            AsmPrinter
            BitReader
            Core
            CodeGen
            ExecutionEngine
            InstCombine
            ${LLVM_EXTRA_LIBS}
            ipo
            MC
            MCJIT
            MCParser
            Object
            RuntimeDyld
            ScalarOpts
            Support
            Target
            TransformUtils
            ${LLVM_LIBS_PROCESSOR}
            )
    message(STATUS "LLVM_LIBS ${LLVM_LIBS}")
endif ()

if (NOT STATIC_BUILD)
    set(LIBRAVI_BUILD_TYPE SHARED)
endif()

if (LLVM_JIT)
    set (LIBRAVI_NAME libravillvm)
elseif (GCC_JIT)
    set (LIBRAVI_NAME libravigccjit)
elseif (NANO_JIT)
    set (LIBRAVI_NAME libravinanojit)    
else()
    set (LIBRAVI_NAME libravinojit)
endif()

#Main library
add_library(${LIBRAVI_NAME} ${LIBRAVI_BUILD_TYPE} 
        ${RAVI_HEADERS}
        ${LUA_LIB_SRCS} 
        ${LUA_CORE_SRCS}
        ${LLVM_JIT_SRCS}
        ${GCC_JIT_SRCS}
        ${NANO_JIT_SRCS}        
        ${NO_JIT_SRCS}
        ${DMR_C_HEADERS}
        ${DMR_C_SRCS}
        ${DMR_C_JIT_SRCS}
        )
if (NOT STATIC_BUILD)
    if (WIN32)
      # enable DLL export
      set_target_properties(${LIBRAVI_NAME} PROPERTIES DEFINE_SYMBOL "LUA_BUILD_AS_DLL")
  else()
      set_target_properties(${LIBRAVI_NAME} PROPERTIES PREFIX "")
  endif ()
endif()
target_link_libraries(${LIBRAVI_NAME} ${EXTRA_LIBRARIES} ${LLVM_LIBS} ${GCCJIT_LIBRARIES} ${NANOJITEXTRA_LIBRARIES})

# Ravi executable
add_executable(ravi src/lua.c)
target_link_libraries(ravi ${LIBRAVI_NAME})

# Simple VM tests
add_executable(test_vm tests/test_vm.c)
target_link_libraries(test_vm ${LIBRAVI_NAME})

if (LLVM_JIT)
    # LLVM playground
    add_executable(test_llvm tests/test_llvm.cpp)
    target_link_libraries(test_llvm ${LIBRAVI_NAME})
endif ()

add_executable(test_misc tests/test_misc.c)

if (LLVM_JIT)
    add_test(TestLLVM test_llvm)
endif ()
add_test(TestVM test_vm)
add_test(TestMisc test_misc)

# Build VSCode Debug Adapter for Ravi  
if (STATIC_BUILD AND NOT GCC_JIT AND NOT NANO_JIT)
  add_executable(testravidebug vscode-debugger/src/testravidebug.c vscode-debugger/src/json.c vscode-debugger/src/protocol.c)
  target_link_libraries(testravidebug ${LIBRAVI_NAME})

  add_test(TestRaviDebug testravidebug)

  if (LLVM_JIT)
    set(RAVI_DEBUGGER_TARGET ravidebugllvm)
  else()  
    set(RAVI_DEBUGGER_TARGET ravidebug)
  endif()  

  # Ravi VSCode Debug adapter
  add_executable(${RAVI_DEBUGGER_TARGET} vscode-debugger/src/ravidebug.c vscode-debugger/src/json.c vscode-debugger/src/protocol.c)
  target_link_libraries(${RAVI_DEBUGGER_TARGET} ${LIBRAVI_NAME})
endif()

install(FILES ${LUA_HEADERS}
        DESTINATION include/ravi)
install(TARGETS ${LIBRAVI_NAME} ravi ${RAVI_DEBUGGER_TARGET}
        RUNTIME DESTINATION bin
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib)

