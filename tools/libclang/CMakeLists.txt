set(SOURCES
  ARCMigrate.cpp
  BuildSystem.cpp
  CIndex.cpp
  CIndexCXX.cpp
  CIndexCodeCompletion.cpp
  CIndexDiagnostic.cpp
  CIndexHigh.cpp
  CIndexInclusionStack.cpp
  CIndexUSRs.cpp
  CIndexer.cpp
  CXComment.cpp
  CXCursor.cpp
  CXIndexDataConsumer.cpp
  CXCompilationDatabase.cpp
  CXLoadedDiagnostic.cpp
  CXSourceLocation.cpp
  CXStoredDiagnostic.cpp
  CXString.cpp
  CXType.cpp
  Indexing.cpp

  ADDITIONAL_HEADERS
  CIndexDiagnostic.h
  CIndexer.h
  CXCursor.h
  CXLoadedDiagnostic.h
  CXSourceLocation.h
  CXString.h
  CXTranslationUnit.h
  CXType.h
  Index_Internal.h
  ../../include/clang-c/Index.h
  )

set(LIBS
  clangAST
  clangBasic
  clangFrontend
  clangIndex
  clangLex
  clangSema
  clangTooling
)

if (CLANG_ENABLE_ARCMT)
  list(APPEND LIBS clangARCMigrate)
endif ()

if (TARGET clangTidyPlugin)
  add_definitions(-DCLANG_TOOL_EXTRA_BUILD)
  list(APPEND LIBS clangTidyPlugin)
endif ()

find_library(DL_LIBRARY_PATH dl NO_CMAKE_SYSTEM_PATH)
if (DL_LIBRARY_PATH)
  list(APPEND LIBS dl)
endif()

option(LIBCLANG_BUILD_STATIC
  "Build libclang as a static library (in addition to a shared one)" OFF)

set(LLVM_EXPORTED_SYMBOL_FILE ${CMAKE_CURRENT_SOURCE_DIR}/libclang.exports)

if(MSVC)
  # Avoid LNK4197 not to spceify libclang.def here.
  # Each functions is exported as "dllexport" in include/clang-c.
  # KB835326
  set(LLVM_EXPORTED_SYMBOL_FILE)
endif()

if( LLVM_ENABLE_PIC )
  set(ENABLE_SHARED SHARED)
endif()

if((NOT LLVM_ENABLE_PIC OR LIBCLANG_BUILD_STATIC) AND NOT WIN32)
  set(ENABLE_STATIC STATIC)
endif()

if(WIN32)
  set(output_name "libclang")
else()
  set(output_name "clang")
endif()

add_clang_library(libclang ${ENABLE_SHARED} ${ENABLE_STATIC}
  OUTPUT_NAME ${output_name}
  ${SOURCES}
  DEPENDS clang-headers

  LINK_LIBS
  ${LIBS}

  LINK_COMPONENTS
  ${LLVM_TARGETS_TO_BUILD}
  Core
  Support
  )

if(ENABLE_SHARED)
  if(WIN32)
    set_target_properties(libclang
      PROPERTIES
      VERSION ${LIBCLANG_LIBRARY_VERSION}
      DEFINE_SYMBOL _CINDEX_LIB_)
  elseif(APPLE)
    set(LIBCLANG_LINK_FLAGS " -Wl,-compatibility_version -Wl,1")
    set(LIBCLANG_LINK_FLAGS "${LIBCLANG_LINK_FLAGS} -Wl,-current_version -Wl,${LLVM_VERSION_MAJOR}.${LLVM_VERSION_MINOR}.${LLVM_VERSION_PATCH}")

    set_property(TARGET libclang APPEND_STRING PROPERTY
                 LINK_FLAGS ${LIBCLANG_LINK_FLAGS})
  else()
    set_target_properties(libclang
      PROPERTIES
      VERSION ${LIBCLANG_LIBRARY_VERSION}
      DEFINE_SYMBOL _CINDEX_LIB_)
  endif()
endif()

if(INTERNAL_INSTALL_PREFIX)
  set(LIBCLANG_HEADERS_INSTALL_DESTINATION "${INTERNAL_INSTALL_PREFIX}/include")
else()
  set(LIBCLANG_HEADERS_INSTALL_DESTINATION include)
endif()

install(DIRECTORY ../../include/clang-c
  COMPONENT libclang-headers
  DESTINATION "${LIBCLANG_HEADERS_INSTALL_DESTINATION}"
  FILES_MATCHING
  PATTERN "*.h"
  PATTERN ".svn" EXCLUDE
  )

if (NOT CMAKE_CONFIGURATION_TYPES) # don't add this for IDE's.
  add_custom_target(install-libclang-headers
    DEPENDS
    COMMAND "${CMAKE_COMMAND}"
            -DCMAKE_INSTALL_COMPONENT=libclang-headers
            -P "${CMAKE_BINARY_DIR}/cmake_install.cmake")
endif()
