cmake_minimum_required(VERSION 3.9 FATAL_ERROR)
project(rubberband VERSION 2.1.1)

option(RUBBERBAND_EXAMPLE "Build the Rubberband example app" OFF)
option(RUBBERBAND_OPTIMIZED "Build Rubberband optimized for the build machine's CPU" OFF)
option(RUBBERBAND_KISSFFT "Enable KissFFT" OFF)
option(RUBBERBAND_FFTW "Enable FFTW (if available)" ON)
option(RUBBERBAND_BUILTIN_FFT "Enable built-in FFT" ON)

### Dependencies ###

# ARM cross-compilers do not support march=native, only mtune=native
set(MARCH_NATIVE_FLAG)

include(CheckCXXCompilerFlag)

if(RUBBERBAND_OPTIMIZED)
  check_cxx_compiler_flag("-march=native" MARCH_NATIVE_SUPPORTED)
  if(MARCH_NATIVE_SUPPORTED)
    set(MARCH_NATIVE_FLAG "-march=native")
  else()
  endif()
endif()

find_package(Threads REQUIRED)

# FFTW3
if(RUBBERBAND_FFTW)
  find_path(FFTW3_INCLUDEDIR fftw3.h)
  find_library(FFTW3_LIBRARY fftw3)
endif()

# libsamplerate
if(NOT TARGET samplerate)
  find_path(SAMPLERATE_INCLUDEDIR samplerate.h)
  find_library(SAMPLERATE_LIBRARY samplerate)
endif()

# Accelerate framework
if(APPLE)
  find_library(ACCELERATE_FRAMEWORK NAMES Accelerate)
endif()

### Create our target ###
add_library(rubberband)
target_compile_features(rubberband PRIVATE cxx_std_20)

### Handle general flags, include paths, etc ###
target_compile_options(rubberband
  PRIVATE
    $<$<BOOL:${APPLE}>:-mmacosx-version-min=10.11>
    $<$<AND:$<NOT:$<CXX_COMPILER_ID:MSVC>>,$<CONFIG:Release>>:-ffast-math ${MARCH_NATIVE_FLAG} -O3 -ftree-vectorize>
)

target_compile_definitions(rubberband
  PRIVATE
    $<$<BOOL:${UNIX}>:USE_PTHREADS>
    $<$<BOOL:${APPLE}>:HAVE_VDSP>
    $<$<BOOL:$<CXX_COMPILER_ID:MSVC>>:__MSVC__ WIN32 _LIB NOMINMAX _USE_MATH_DEFINES USE_KISSFFT>
    $<$<AND:$<CXX_COMPILER_ID:MSVC>,$<CONFIG:Release>>:NDEBUG>
    $<$<AND:$<CXX_COMPILER_ID:MSVC>,$<CONFIG:Debug>>:_DEBUG NO_TIMING NO_THREAD_CHECKS>
    MALLOC_IS_ALIGNED
    NO_THREAD_CHECKS
    NO_TIMING
)

target_include_directories(rubberband
  PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/rubberband>
      $<INSTALL_INTERFACE:rubberband>
  PRIVATE
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/>
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
)

target_link_libraries(rubberband
  PRIVATE
    Threads::Threads
)

# Handle FFT library
if(RUBBERBAND_FFTW AND FFTW3_INCLUDEDIR AND FFTW3_LIBRARY)
  target_include_directories(rubberband
    PRIVATE
      ${FFTW3_INCLUDEDIR}
  )
  target_compile_definitions(rubberband
    PRIVATE
      HAVE_FFTW3
      FFTW_DOUBLE_ONLY
  )
  target_link_libraries(rubberband
    PUBLIC
      ${FFTW3_LIBRARY}
  )
endif()

if(RUBBERBAND_KISSFFT)
  target_compile_definitions(rubberband
    PRIVATE
      HAVE_KISSFFT
  )
  target_sources(rubberband
    PRIVATE
      src/kissfft/kiss_fft.c
      src/kissfft/kiss_fftr.c
  )
endif()

if(RUBBERBAND_BUILTIN_FFT)
  target_compile_definitions(rubberband
    PRIVATE
      USE_BUILTIN_FFT
  )
endif()

# Apple framework
if(APPLE)
  target_link_libraries(rubberband
    PRIVATE
      "${ACCELERATE_FRAMEWORK}"
  )
endif()

# Handle resampling library
if(TARGET samplerate)
  target_compile_definitions(rubberband
    PRIVATE
      HAVE_LIBSAMPLERATE
  )

  target_link_libraries(rubberband
    PRIVATE
      samplerate
  )
elseif(SAMPLERATE_INCLUDEDIR AND SAMPLERATE_LIBRARY)
  target_include_directories(rubberband
    PRIVATE
      ${SAMPLERATE_INCLUDEDIR}
  )
  target_compile_definitions(rubberband
    PRIVATE
      HAVE_LIBSAMPLERATE
  )
  target_link_libraries(rubberband
    PUBLIC
      ${SAMPLERATE_LIBRARY}
  )
else()
  target_compile_definitions(rubberband
    PRIVATE
      USE_SPEEX
  )
  target_sources(rubberband
    PRIVATE
      src/speex/resample.c
  )
endif()

### Source files ###
target_sources(rubberband
  PRIVATE
    src/RubberBandStretcher.cpp
    src/common/Allocators.cpp
    src/common/BQResampler.cpp
    src/common/FFT.cpp
    src/common/Log.cpp
    src/common/Profiler.cpp
    src/common/Resampler.cpp
    src/common/StretchCalculator.cpp
    src/common/Thread.cpp
    src/common/mathmisc.cpp
    src/common/sysutils.cpp
    src/faster/AudioCurveCalculator.cpp
    src/faster/CompoundAudioCurve.cpp
    src/faster/HighFrequencyAudioCurve.cpp
    src/faster/PercussiveAudioCurve.cpp
    src/faster/R2Stretcher.cpp
    src/faster/SilentAudioCurve.cpp
    src/faster/StretcherChannelData.cpp
    src/faster/StretcherProcess.cpp
    src/finer/R3Stretcher.cpp
#     src/rubberband-c.cpp
)

### Example ###
if(RUBBERBAND_EXAMPLE)
  add_executable(rubberband_test main/main.cpp)
  target_include_directories(rubberband_test PRIVATE src Threads::Threads)
  target_link_libraries(rubberband_test PRIVATE rubberband -lsndfile)
endif()

### Installation ###

install(
  TARGETS rubberband
  EXPORT rubberband-exports
  RUNTIME DESTINATION lib
  ARCHIVE DESTINATION lib)

install(EXPORT rubberband-exports
        DESTINATION lib/cmake/rubberband)
export(EXPORT rubberband-exports)
