cmake_minimum_required(VERSION 3.5)
project(caffe)

add_compile_options(-std=c++11)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -Wall")
#list(APPEND CMAKE_MODULE_PATH src/cmake/ src/caffe/cmake/Modules/)

find_package(catkin REQUIRED COMPONENTS
  roscpp
  std_msgs
)
#Caffe CMake utils
include(src/caffe/cmake/Utils.cmake)

#Caffe Dependencies
find_package(Boost REQUIRED COMPONENTS system thread filesystem regex)
include_directories(${Boost_INCLUDE_DIRS})
link_directories(${Boost_LIBRARY_DIRS})

include(src/caffe/cmake/Modules/FindGFlags.cmake)
include(src/caffe/cmake/Modules/FindGlog.cmake)
include(src/caffe/cmake/ProtoBuf.cmake)
include(src/caffe/cmake/Modules/FindLevelDB.cmake)
include(src/caffe/cmake/Modules/FindSnappy.cmake)
include(src/caffe/cmake/Modules/FindOpenBLAS.cmake)

find_package(Protobuf QUIET)
find_package(HDF5 COMPONENTS HL QUIET)
find_package(OpenCV REQUIRED)

if (GFLAGS_FOUND
    AND GLOG_FOUND
    AND PROTOBUF_FOUND
    AND HDF5_FOUND
    AND LevelDB_FOUND
    AND SNAPPY_FOUND
    AND OpenBLAS_FOUND
    )

    include_directories(${GFLAGS_INCLUDE_DIRS})
    link_directories(${GFLAGS_LIBRARIES})

    include_directories(${GLOG_INCLUDE_DIRS})
    link_directories(${GLOG_LIBRARIES})

    include_directories(${PROTOBUF_INCLUDE_DIR})
    link_directories(${PROTOBUF_LIBRARIES})

    include_directories(${HDF5_INCLUDE_DIRS} ${HDF5_HL_INCLUDE_DIR})
    link_directories(${HDF5_LIBRARIES} ${HDF5_HL_LIBRARIES})

    include_directories(${LevelDB_INCLUDE})
    link_directories(${LevelDB_LIBRARIES})
    add_definitions(-DUSE_LEVELDB)

    include_directories(${Snappy_INCLUDE_DIR})
    link_directories(${Snappy_LIBRARIES})

    include_directories(${OpenBLAS_INCLUDE_DIR})
    link_directories(${OpenBLAS_LIB})
    #Required since Caffe OpenBLAS detector is not using standard namings
    set(OpenBLAS_LIBRARIES ${OpenBLAS_LIB})
    set(OpenBLAS_INCLUDE_DIRS ${OpenBLAS_INCLUDE_DIR})

    include_directories(${OpenCV_INCLUDE_DIRS})
    link_directories(${OpenCV_LIBS})
    add_definitions(-DUSE_OPENCV)

    #CUDA
    find_package(CUDA QUIET)
    if (${CUDA_FOUND})
        caffe_option(CPU_ONLY  "Build Caffe without CUDA support" OFF)
        caffe_option(USE_CUDNN "Build Caffe with cuDNN library support" ON IF NOT CPU_ONLY)
        include(src/caffe/cmake/Cuda.cmake)

        IF ("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "^arm")
            link_directories(/usr/lib/arm-linux-gnueabihf/tegra)
        endif ()
    else()
        caffe_option(CPU_ONLY  "Build Caffe without CUDA support" ON)
        add_definitions(-DCPU_ONLY)
    endif()

    ###################################
    ## catkin specific configuration ##
    ###################################
    ## The catkin_package macro generates cmake config files for your package
    ## Declare things to be passed to dependent projects
    ## INCLUDE_DIRS: uncomment this if your package contains header files
    ## LIBRARIES: libraries you create in this project that dependent projects also need
    ## CATKIN_DEPENDS: catkin_packages dependent projects also need
    ## DEPENDS: system dependencies of this project that dependent projects also need
    catkin_package(
      INCLUDE_DIRS src/caffe/include
      LIBRARIES caffe
      CATKIN_DEPENDS
      DEPENDS
            GFLAGS
            GLOG
            PROTOBUF
            HDF5
            LevelDB
            Snappy
            OpenBLAS
            OpenCV
    )

    ###########
    ## Build ##
    ###########

    include_directories(
     src/caffe/include
      ${catkin_INCLUDE_DIRS}
    )

    #Proto
    file(GLOB proto_files src/caffe/src/caffe/proto/*.proto)
    caffe_protobuf_generate_cpp_py(${proto_gen_folder} proto_srcs proto_hdrs proto_python ${proto_files})
    add_library(proto STATIC ${proto_hdrs} ${proto_srcs} ${proto_python})

    #Caffe Library
    include(src/caffe/cmake/Targets.cmake)
    caffe_pickup_caffe_sources(src/caffe/)# creates 'test_srcs', 'srcs', 'test_cuda', 'cuda' lists
    if(CUDA_FOUND)
        caffe_cuda_compile(cuda_objs ${cuda})
        list(APPEND srcs ${cuda_objs} ${cuda})
    endif()
    configure_file(src/caffe/cmake/Templates/caffe_config.h.in "${PROJECT_BINARY_DIR}/caffe_config.h")
    add_library(caffe ${srcs})
    target_link_libraries(caffe proto ${Caffe_LINKER_LIBS})


    #############
    ## Install ##
    #############

    ## Mark executables and/or libraries for installation
    install(TARGETS caffe proto
       ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
       LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
       RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
    )

    ## Mark cpp header files for installation
    install(DIRECTORY src/caffe/include/caffe/
       DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
       FILES_MATCHING PATTERN "*.h*"
       PATTERN ".svn" EXCLUDE
    )

    install(FILES ${proto_hdrs}
            DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}/proto)

    ## Mark other files for installation (e.g. launch and bag files, etc.)
    # install(FILES
    #   # myfile1
    #   # myfile2
    #   DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
    # )

endif()