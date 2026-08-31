cmake_dependent_option(Z3_BUILD_EXECUTABLE
    "Build the z3 executable" ON
    "PROJECT_IS_TOP_LEVEL;Z3_BUILD_LIBZ3_CORE" OFF)

################################################################################
# Traverse directories each adding a Z3 component
################################################################################
# I'm duplicating the order in ``mk_project.py`` for now to help us keep
# the build systems in sync.
#
# The components in these directory explicitly declare their dependencies so
# you may be able to re-order some of these directories but an error will be
# raised if you try to declare a component is dependent on another component
# that has not yet been declared.
add_subdirectory(util)
add_subdirectory(math/polynomial)
add_subdirectory(math/dd)
add_subdirectory(math/hilbert)
add_subdirectory(math/simplex)
add_subdirectory(math/interval)
add_subdirectory(math/realclosure)
add_subdirectory(math/subpaving)
add_subdirectory(ast)
add_subdirectory(params)
add_subdirectory(ast/rewriter)
add_subdirectory(ast/rewriter/bit_blaster)
add_subdirectory(ast/normal_forms)
add_subdirectory(ast/macros)
add_subdirectory(model)
add_subdirectory(ast/euf)
add_subdirectory(ast/converters)
add_subdirectory(ast/substitution)
add_subdirectory(ast/simplifiers)
add_subdirectory(tactic)
add_subdirectory(qe/mbp)
add_subdirectory(qe/lite)
add_subdirectory(parsers/util)
add_subdirectory(math/grobner)
add_subdirectory(sat)
add_subdirectory(nlsat)
add_subdirectory(tactic/core)
add_subdirectory(math/subpaving/tactic)
add_subdirectory(tactic/aig)
add_subdirectory(tactic/arith)
add_subdirectory(solver)
add_subdirectory(cmd_context)
add_subdirectory(cmd_context/extra_cmds)
add_subdirectory(parsers/smt2)
add_subdirectory(solver/assertions)
add_subdirectory(ast/pattern)
add_subdirectory(math/lp)
add_subdirectory(ast/sls)
add_subdirectory(sat/smt)
add_subdirectory(sat/tactic)
add_subdirectory(nlsat/tactic)
add_subdirectory(ackermannization)
add_subdirectory(ast/proofs)
add_subdirectory(ast/fpa)
add_subdirectory(smt/proto_model)
add_subdirectory(smt)
add_subdirectory(tactic/bv)
add_subdirectory(smt/tactic)
add_subdirectory(tactic/sls)
add_subdirectory(qe)
add_subdirectory(muz/base)
add_subdirectory(muz/dataflow)
add_subdirectory(muz/transforms)
add_subdirectory(muz/rel)
add_subdirectory(muz/clp)
add_subdirectory(muz/tab)
add_subdirectory(muz/bmc)
add_subdirectory(muz/ddnf)
add_subdirectory(muz/spacer)
add_subdirectory(muz/fp)
add_subdirectory(tactic/ufbv)
add_subdirectory(sat/sat_solver)
add_subdirectory(tactic/smtlogics)
add_subdirectory(tactic/fpa)
add_subdirectory(tactic/fd_solver)
add_subdirectory(tactic/portfolio)
add_subdirectory(opt)
add_subdirectory(api)
add_subdirectory(api/dll)
################################################################################
# libz3
################################################################################
target_sources(libz3 PUBLIC
  FILE_SET c_headers
  TYPE HEADERS
  BASE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/api"
  FILES
    api/z3_algebraic.h
    api/z3_api.h
    api/z3_ast_containers.h
    api/z3_fixedpoint.h
    api/z3_fpa.h
    api/z3.h
    api/z3_macros.h
    api/z3_optimization.h
    api/z3_polynomial.h
    api/z3_rcf.h
    api/z3_spacer.h
    api/z3_v1.h
  FILE_SET cxx_headers
  TYPE HEADERS
  BASE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/api/c++"
  FILES api/c++/z3++.h
  FILE_SET generated_headers
  TYPE HEADERS
  BASE_DIRS "${CMAKE_CURRENT_BINARY_DIR}/util"
  FILES "${CMAKE_CURRENT_BINARY_DIR}/util/z3_version.h")
set_target_properties(libz3 PROPERTIES
  # VERSION determines the version in the filename of the shared library.
  # SOVERSION determines the value of the DT_SONAME field on ELF platforms.
  # On ELF platforms the final compiled filename will be libz3.so.W.X.Y.Z
  # but symlinks will be made to this file from libz3.so and also from
  # libz3.so.W.X.
  # This indicates that no breaking API changes will be made within a single
  # minor version.
  VERSION ${Z3_VERSION}
  SOVERSION ${Z3_VERSION_MAJOR}.${Z3_VERSION_MINOR})

# Set macOS-specific properties for proper .dylib versioning (fixes issue #6651)
if (CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  set_target_properties(libz3 PROPERTIES
    # Use @rpath for install name to make library relocatable
    INSTALL_NAME_DIR "@rpath"
    # Enable RPATH support
    MACOSX_RPATH TRUE
  )
  # Add header padding to allow install_name_tool to modify the dylib
  # This fixes issues where install_name_tool fails with "larger updated load commands do not fit"
  # See: https://github.com/Z3Prover/z3/issues/7623
  target_link_options(libz3 PRIVATE "-Wl,-headerpad_max_install_names")
endif()

if (NOT MSVC)
  # On UNIX like platforms if we don't change the OUTPUT_NAME
  # the library gets a name like ``liblibz3.so`` so we change it
  # here. We don't do a rename with MSVC because we get file naming
  # conflicts (the z3 executable also has this OUTPUT_NAME) with
  # ``.ilk``, ``.pdb``, ``.lib`` and ``.exp`` files sharing the same
  # prefix.
  if (BUILD_SHARED_LIBS)
    set_target_properties(libz3 PROPERTIES OUTPUT_NAME z3)
  else()
    # Do not overwrite the shared build's library file when co-installed,
    # and give pkg-config consumers an unambiguous `-l` name to link.
    set_target_properties(libz3 PROPERTIES OUTPUT_NAME z3-static)
  endif()
elseif (WIN32 AND NOT BUILD_SHARED_LIBS)
  # A shared build installs libz3.lib as its import library. Give the static
  # archive a distinct name so both variants can share an install prefix.
  set_target_properties(libz3 PROPERTIES OUTPUT_NAME libz3-static)
endif()

# The `PRIVATE` usage requirement is specified so that when building Z3 as a
# shared library the dependent libraries are specified on the link command line
# so that if those are also shared libraries they are referenced by `libz3.so`.
target_link_libraries(libz3 PRIVATE z3_common)

install(TARGETS libz3 z3_common
  EXPORT Z3_EXPORTED_TARGETS
  LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
  ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}" # On Windows this installs ``libz3.lib`` which CMake calls the "corresponding import library". Do we want this installed?
  RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}" # For Windows. DLLs are runtime targets for CMake
  FILE_SET c_headers DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  FILE_SET cxx_headers DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  FILE_SET generated_headers DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

if (MSVC)
  # Generate the module-definition file used to export the C API. Listing the
  # generated file as a source gives CMake both the build dependency and the
  # linker input without a separate custom target.
  set(dll_module_exports_file "${CMAKE_CURRENT_BINARY_DIR}/api_dll.def")
  add_custom_command(OUTPUT "${dll_module_exports_file}"
    COMMAND
      "${Python3_EXECUTABLE}"
      "${PROJECT_SOURCE_DIR}/scripts/mk_def_file.py"
      "${dll_module_exports_file}"
      "libz3"
      ${Z3_FULL_PATH_API_HEADER_FILES_TO_SCAN}
    DEPENDS
      "${PROJECT_SOURCE_DIR}/scripts/mk_def_file.py"
      ${Z3_GENERATED_FILE_EXTRA_DEPENDENCIES}
      ${Z3_FULL_PATH_API_HEADER_FILES_TO_SCAN}
    COMMENT "Generating \"${dll_module_exports_file}\""
    USES_TERMINAL
    VERBATIM
  )
  target_sources(libz3 PRIVATE "${dll_module_exports_file}")
endif()

if (Z3_BUILD_EXECUTABLE)
    add_subdirectory(shell)
endif()

################################################################################
# z3-test
################################################################################

cmake_dependent_option(Z3_BUILD_TEST_EXECUTABLES
    "Build test executables" ON
    "PROJECT_IS_TOP_LEVEL;Z3_BUILD_LIBZ3_CORE" OFF)


if (Z3_BUILD_TEST_EXECUTABLES)
    add_subdirectory(test)
endif()

################################################################################
# .NET bindings
################################################################################
option(Z3_BUILD_DOTNET_BINDINGS "Build .NET bindings for Z3" OFF)
if (Z3_BUILD_DOTNET_BINDINGS)
  if (NOT BUILD_SHARED_LIBS)
    message(FATAL_ERROR "The .NET bindings will not work with a static libz3. "
      "You either need to disable Z3_BUILD_DOTNET_BINDINGS or enable BUILD_SHARED_LIBS")
  endif()
  add_subdirectory(api/dotnet)
endif()

################################################################################
# Java bindings
################################################################################
option(Z3_BUILD_JAVA_BINDINGS "Build Java bindings for Z3" OFF)
if (Z3_BUILD_JAVA_BINDINGS)
  if (NOT BUILD_SHARED_LIBS)
    message(FATAL_ERROR "The Java bindings will not work with a static libz3. "
      "You either need to disable Z3_BUILD_JAVA_BINDINGS or enable BUILD_SHARED_LIBS")
  endif()
  add_subdirectory(api/java)
endif()

################################################################################
# OCaml bindings
################################################################################
option(Z3_BUILD_OCAML_BINDINGS "Build OCaml bindings for Z3" OFF)
if (Z3_BUILD_OCAML_BINDINGS)
  if (NOT BUILD_SHARED_LIBS)
    message(FATAL_ERROR "The OCaml bindings will not work with a static libz3. "
      "You either need to disable Z3_BUILD_OCAML_BINDINGS or enable BUILD_SHARED_LIBS")
  endif()
  add_subdirectory(api/ml)
endif()

################################################################################
# Julia bindings
################################################################################
option(Z3_BUILD_JULIA_BINDINGS "Build Julia bindings for Z3" OFF)
if (Z3_BUILD_JULIA_BINDINGS)
  if (NOT BUILD_SHARED_LIBS)
    message(FATAL_ERROR "The Julia bindings will not work with a static libz3."
      "You either need to disable Z3_BUILD_JULIA_BINDINGS or enable BUILD_SHARED_LIBS")
  endif()
  add_subdirectory(api/julia)
endif()

################################################################################
# Go bindings
################################################################################
option(Z3_BUILD_GO_BINDINGS "Build Go bindings for Z3" OFF)
if (Z3_BUILD_GO_BINDINGS)
  if (NOT BUILD_SHARED_LIBS)
    message(FATAL_ERROR "The Go bindings will not work with a static libz3. "
      "You either need to disable Z3_BUILD_GO_BINDINGS or enable BUILD_SHARED_LIBS")
  endif()
  add_subdirectory(api/go)
endif()

# TODO: Implement support for other bindigns
