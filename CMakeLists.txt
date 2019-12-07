cmake_minimum_required(VERSION 2.8)
project(Souper)

set(CMAKE_MACOSX_RPATH NEW)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING
      "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif()

set(LLVM_CONFIG_EXECUTABLE ${TOB_FOLDER}/llvm/bin/llvm-config)
if(NOT EXISTS ${LLVM_CONFIG_EXECUTABLE})
  message(FATAL_ERROR "llvm-config could not be found!")
endif()

execute_process(
  COMMAND ${LLVM_CONFIG_EXECUTABLE} --includedir
  OUTPUT_VARIABLE LLVM_INCLUDEDIR
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

execute_process(
  COMMAND ${LLVM_CONFIG_EXECUTABLE} --cppflags
  OUTPUT_VARIABLE LLVM_CXXFLAGS
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

set(LLVM_CXXFLAGS "${LLVM_CXXFLAGS} -fno-exceptions -fno-rtti")

execute_process(
  COMMAND ${LLVM_CONFIG_EXECUTABLE} --libs
  OUTPUT_VARIABLE LLVM_LIBS
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

execute_process(
  COMMAND ${LLVM_CONFIG_EXECUTABLE} --system-libs
  OUTPUT_VARIABLE LLVM_SYSTEM_LIBS
  OUTPUT_STRIP_TRAILING_WHITESPACE
)
set(LLVM_LIBS "${LLVM_LIBS} ${LLVM_SYSTEM_LIBS}")
string(STRIP ${LLVM_LIBS} LLVM_LIBS)

execute_process(
  COMMAND ${LLVM_CONFIG_EXECUTABLE} --ldflags
  OUTPUT_VARIABLE LLVM_LDFLAGS
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

set(PASS_LDFLAGS "${LLVM_LDFLAGS}")

execute_process(
  COMMAND ${LLVM_CONFIG_EXECUTABLE} --bindir
  OUTPUT_VARIABLE LLVM_BINDIR
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

execute_process(
  COMMAND ${LLVM_CONFIG_EXECUTABLE} --src-root
  OUTPUT_VARIABLE LLVM_SRC
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

execute_process(
  COMMAND ${LLVM_CONFIG_EXECUTABLE} --obj-root
  OUTPUT_VARIABLE LLVM_BUILD
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

set(CLANG_CXXFLAGS "")
set(CLANG_INCLUDEDIR "${LLVM_SRC}/tools/clang/include" "${LLVM_BUILD}/tools/clang/include")

set(ALIVE_DIR "${ALIVE2_FOLDER}")
set(ALIVE_LDFLAGS "-L ${ALIVE_DIR}/lib -lalive2")

set(CLANG_LIBS "-lclangCodeGen -lclangTooling -lclangRewrite -lclangFrontend -lclangAnalysis -lclangParse -lclangSerialization -lclangSema -lclangEdit -lclangAnalysis -lclangAST -lclangDriver -lclangLex -lclangBasic -lclangASTMatchers")

set(GTEST_CXXFLAGS "-DGTEST_HAS_RTTI=0")
set(GTEST_INCLUDEDIR "${LLVM_SRC}/utils/unittest/googletest/include")
set(GTEST_LIBS "-lgtest_main -lgtest")

set(LLVM_CXXFLAGS "${LLVM_CXXFLAGS} -std=c++17")

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
  set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LANGUAGE_STANDARD "c++17")
  set(LLVM_CXXFLAGS "${LLVM_CXXFLAGS} -fvisibility-inlines-hidden")
  set(PASS_LDFLAGS "-Wl,-undefined,dynamic_lookup")
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
  set(CMAKE_CXX_STANDARD 17)
  set(CMAKE_CXX_STANDARD_REQUIRED ON)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /std:c++17")
  set(LLVM_CXXFLAGS "${LLVM_CXXFLAGS} -march=native -D_LIBCPP_VERSION /std:c++17")
endif()

include_directories(include ${KLEE_FOLDER}/include)
include_directories(include ${ALIVE_DIR}/include)
include_directories(include ${ALIVE_DIR}/include/alive2)

set(KLEE_EXPR_FILES
  ${KLEE_FOLDER}/lib/Expr/Constraints.cpp
  ${KLEE_FOLDER}/lib/Expr/ExprBuilder.cpp
  ${KLEE_FOLDER}/lib/Expr/Expr.cpp
  ${KLEE_FOLDER}/lib/Expr/ExprEvaluator.cpp
  ${KLEE_FOLDER}/lib/Expr/ExprPPrinter.cpp
  ${KLEE_FOLDER}/lib/Expr/ExprSMTLIBPrinter.cpp
  ${KLEE_FOLDER}/lib/Expr/ExprUtil.cpp
  ${KLEE_FOLDER}/lib/Expr/ExprVisitor.cpp
  ${KLEE_FOLDER}/lib/Expr/Lexer.cpp
  ${KLEE_FOLDER}/lib/Expr/Parser.cpp
  ${KLEE_FOLDER}/lib/Expr/Updates.cpp
)

add_library(kleeExpr STATIC
  ${KLEE_EXPR_FILES}
)

find_path(HIREDIS_INCLUDE_DIR
  NAMES
  hiredis/hiredis.h
  PATHS
  ${DEPENDENCIES_FOLDER}/hiredis/include
  NO_DEFAULT_PATH)

include_directories(${HIREDIS_INCLUDE_DIR}/hiredis)

find_library(HIREDIS_LIBRARY
  NAMES
  hiredis
  PATHS
  ${DEPENDENCIES_FOLDER}/hiredis/lib
  NO_DEFAULT_PATH)

find_library(ALIVE_LIBRARY alive2 PATHS "${ALIVE_DIR}/lib" NO_DEFAULT_PATH)

find_path(Z3_INCLUDE_DIRECTORY
  NAMES
  z3.h
  z3++.h
  PATHS
  ${DEPENDENCIES_FOLDER}/tob_libraries/z3/include
  NO_DEFAULT_PATH)

include_directories(${Z3_INCLUDE_DIRECTORY})

find_library(Z3_LIBRARY
  NAMES
  libz3.a
  PATHS
  ${DEPENDENCIES_FOLDER}/tob_libraries/z3/bin
  NO_DEFAULT_PATH)

find_program(Z3
  NAMES
  z3
  PATHS
  ${DEPENDENCIES_FOLDER}/tob_libraries/z3/bin
  NO_DEFAULT_PATH)

set(TEST_SOLVER "-z3-path=${Z3}")

set(SOUPER_CLANG_TOOL_FILES
  lib/ClangTool/Actions.cpp
  include/souper/ClangTool/Actions.h
)

add_library(souperClangTool STATIC
  ${SOUPER_CLANG_TOOL_FILES}
)

set(SOUPER_EXTRACTOR_FILES
  lib/Extractor/Candidates.cpp
  lib/Extractor/ExprBuilder.cpp
  lib/Extractor/KLEEBuilder.cpp
  lib/Extractor/Solver.cpp
  include/souper/Extractor/Candidates.h
  include/souper/Extractor/ExprBuilder.h
  include/souper/Extractor/Solver.h
)

add_library(souperExtractor STATIC
  ${SOUPER_EXTRACTOR_FILES}
)

set(SOUPER_KVSTORE_FILES
  lib/KVStore/KVStore.cpp
  include/souper/KVStore/KVStore.h
)

add_library(souperKVStore STATIC
  ${SOUPER_KVSTORE_FILES}
)

set(SOUPER_INFER_FILES
  lib/Infer/InstSynthesis.cpp
  include/souper/Infer/InstSynthesis.h
  lib/Infer/ConstantSynthesis.cpp
  include/souper/Infer/ConstantSynthesis.h
  lib/Infer/EnumerativeSynthesis.cpp
  include/souper/Infer/EnumerativeSynthesis.h
  lib/Infer/AliveDriver.cpp
  include/souper/Infer/AliveDriver.h
  lib/Infer/Pruning.cpp
  include/souper/Infer/Pruning.h
  lib/Infer/Interpreter.cpp
  lib/Infer/AbstractInterpreter.cpp
  include/souper/Infer/Interpreter.h
)

add_library(souperInfer STATIC
  ${SOUPER_INFER_FILES}
)

set(SOUPER_INST_FILES
  lib/Inst/Inst.cpp
  include/souper/Inst/Inst.h
  include/souper/Inst/InstGraph.h
)

add_library(souperInst STATIC
  ${SOUPER_INST_FILES}
)

set(SOUPER_PARSER_FILES
  lib/Parser/Parser.cpp
  include/souper/Parser/Parser.h
)

add_library(souperParser STATIC
  ${SOUPER_PARSER_FILES}
)

set(SOUPER_SMTLIB2_FILES
  lib/SMTLIB2/Solver.cpp
  include/souper/SMTLIB2/Solver.h
)

add_library(souperSMTLIB2 STATIC
  ${SOUPER_SMTLIB2_FILES}
)

set(SOUPER_TOOL_FILES
  lib/Tool/CandidateMapUtils.cpp
  include/souper/Tool/CandidateMapUtils.h
  include/souper/Tool/GetSolverFromArgs.h
)

add_library(souperTool STATIC
  ${SOUPER_TOOL_FILES}
)

set(SOUPER_SOURCES
  ${SOUPER_EXTRACTOR_FILES}
  ${SOUPER_INST_FILES}
  ${SOUPER_KVSTORE_FILES}
  ${SOUPER_PARSER_FILES}
  ${SOUPER_SMTLIB2_FILES}
  ${SOUPER_TOOL_FILES}
  ${SOUPER_INFER_FILES})

add_library(souperPass STATIC
  ${KLEE_EXPR_FILES}
  ${SOUPER_SOURCES}
  lib/Pass/Pass.cpp
)

add_library(souperPassProfileAll STATIC
  ${KLEE_EXPR_FILES}
  ${SOUPER_SOURCES}
  lib/Pass/Pass.cpp
)
target_compile_definitions(souperPassProfileAll PRIVATE DYNAMIC_PROFILE_ALL=1)

set(LLVM_LDFLAGS "${LLVM_LDFLAGS} ${ALIVE_LDFLAGS}")
foreach(target souperExtractor souperInfer souperInst souperKVStore souperParser
               souperSMTLIB2 souperTool souperPass souperPassProfileAll kleeExpr)
  set_target_properties(${target} PROPERTIES COMPILE_FLAGS "${LLVM_CXXFLAGS}")
  target_include_directories(${target} PRIVATE "${LLVM_INCLUDEDIR}")
endforeach()
foreach(target souperClangTool)
  set_target_properties(${target} PROPERTIES COMPILE_FLAGS "${CLANG_CXXFLAGS} ${LLVM_CXXFLAGS}")
  target_include_directories(${target} PRIVATE "${LLVM_INCLUDEDIR}" ${CLANG_INCLUDEDIR})
endforeach()

add_library(z3 STATIC IMPORTED)
set_target_properties(z3 PROPERTIES IMPORTED_LOCATION ${Z3_LIBRARY})
set_target_properties(z3 PROPERTIES IMPORTED_IMPLIB ${Z3_LIBRARY})

# static
target_link_libraries(kleeExpr ${LLVM_LIBS} ${LLVM_LDFLAGS})
target_link_libraries(souperClangTool souperExtractor souperTool ${CLANG_LIBS} ${LLVM_LIBS} ${LLVM_LDFLAGS})
target_link_libraries(souperExtractor souperParser souperKVStore souperInfer souperInst kleeExpr)
target_link_libraries(souperInfer souperExtractor ${LLVM_LIBS} ${LLVM_LDFLAGS} z3)
target_link_libraries(souperInst ${LLVM_LIBS} ${LLVM_LDFLAGS})
target_link_libraries(souperKVStore ${HIREDIS_LIBRARY} ${LLVM_LIBS} ${LLVM_LDFLAGS})
target_link_libraries(souperParser souperInst ${LLVM_LIBS} ${LLVM_LDFLAGS} ${ALIVE_LIBRARY})
target_link_libraries(souperSMTLIB2 ${LLVM_LIBS} ${LLVM_LDFLAGS})
target_link_libraries(souperTool souperExtractor souperSMTLIB2)

# dynamic
target_link_libraries(souperPass ${PASS_LDFLAGS} ${HIREDIS_LIBRARY} ${ALIVE_LIBRARY} z3)
target_link_libraries(souperPassProfileAll ${PASS_LDFLAGS} ${HIREDIS_LIBRARY} ${ALIVE_LIBRARY} z3)

SET(BUILD_CLANG_TOOL 1 CACHE BOOL "Build the Souper Clang tool")
if (NOT BUILD_CLANG_TOOL)
  message(STATUS "Excluding Clang tool from build")
  set_target_properties(souperClangTool PROPERTIES EXCLUDE_FROM_ALL 1 EXCLUDE_FROM_DEFAULT_BUILD 1)
endif()

set(TEST_SYNTHESIS "ON" CACHE STRING "Enable additional, computationally intensive synthesis tests")
set(TEST_LONG_DURATION_SYNTHESIS "" CACHE STRING "Enable long duration (> 10 min) synthesis tests")

configure_file(
  ${CMAKE_SOURCE_DIR}/test/lit.site.cfg.in
  ${CMAKE_BINARY_DIR}/test/lit.site.cfg
)

configure_file(
  ${CMAKE_SOURCE_DIR}/utils/run_lit.in
  ${CMAKE_BINARY_DIR}/run_lit
)

configure_file(
  ${CMAKE_SOURCE_DIR}/tools/gdbprinter.py
  ${CMAKE_BINARY_DIR}/souper-check-gdb.py
)

add_custom_target(check
  COMMAND ${CMAKE_BINARY_DIR}/run_lit
  DEPENDS profileRuntime souperPass souperPassProfileAll
  USES_TERMINAL)

add_library(profileRuntime STATIC
  runtime/souperPassProfile.c)

set(SOUPER_PASS ${CMAKE_BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}souperPass${CMAKE_SHARED_LIBRARY_SUFFIX})
set(SOUPER_PASS_PROFILE_ALL ${CMAKE_BINARY_DIR}/${CMAKE_SHARED_LIBRARY_PREFIX}souperPassProfileAll${CMAKE_SHARED_LIBRARY_SUFFIX})
set(PROFILE_LIBRARY ${CMAKE_BINARY_DIR}/${CMAKE_STATIC_LIBRARY_PREFIX}profileRuntime${CMAKE_STATIC_LIBRARY_SUFFIX})
configure_file(${CMAKE_SOURCE_DIR}/utils/reduce.in ${CMAKE_BINARY_DIR}/reduce @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/utils/cache_dump.in ${CMAKE_BINARY_DIR}/cache_dump @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/utils/cache_dfa.in ${CMAKE_BINARY_DIR}/cache_dfa @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/utils/cache_import.in ${CMAKE_BINARY_DIR}/cache_import @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/utils/cache_infer.in ${CMAKE_BINARY_DIR}/cache_infer @ONLY)
configure_file(${CMAKE_SOURCE_DIR}/utils/souper2llvm.in ${CMAKE_BINARY_DIR}/souper2llvm @ONLY)

if (BUILD_CLANG_TOOL)
  configure_file(${CMAKE_SOURCE_DIR}/utils/sclang.in ${CMAKE_BINARY_DIR}/sclang @ONLY)
  configure_file(${CMAKE_SOURCE_DIR}/utils/sclang.in ${CMAKE_BINARY_DIR}/sclang++ @ONLY)
endif()

add_subdirectory(docs)
