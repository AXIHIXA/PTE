cmake_minimum_required(VERSION 3.27)

project(PTE CUDA CXX)

# Ensure correct CUDA architecture
# NVIDIA Geforce RTX 2080 Ti has Compute Capability 7.5
# https://developer.nvidia.com/cuda-gpus
# https://stackoverflow.com/questions/67794606/cmake-cuda-architecture-flags
set(CMAKE_CUDA_ARCHITECTURES 75)
set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)

# Set to root directory of your virtual environment. 
set(Python_ROOT_DIR "$ENV{HOME}/opt/anaconda3/envs/py3")
find_package(Python REQUIRED COMPONENTS Interpreter Development)

# CMAKE_PREFIX_PATH can be gotten from `python -m "import torch;print(torch.utils.cmake_prefix_path)"`
# libTorch conda build conflicts with OpenCV, so download compiled library directly from pytorch.org.
# libtorch REQUIRES CMAKE_CUDA_STANDARD <= 17 and CMAKE_CXX_STANDARD <= 17.
set(CAFFE2_USE_CUDNN ON)
set(Torch_ROOT_DIR "${Python_ROOT_DIR}/lib/python${Python_VERSION_MAJOR}.${Python_VERSION_MINOR}/site-packages/torch")
find_package(Torch REQUIRED CONFIG HINTS "${Torch_ROOT_DIR}/share/cmake")



# Hard-coded from ninja outputs by PyTorch's offical setuptools.
# libTorch's Find module lacks essential components for CUDA/C++ extensions!
set(TORCH_EXTENSION pte)
add_library(${TORCH_EXTENSION} SHARED
        src/main.cu
)
set_target_properties(${TORCH_EXTENSION} PROPERTIES 
        PREFIX "" 
        OUTPUT_NAME ${TORCH_EXTENSION} 
        SUFFIX ".so"
        LIBRARY_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/${TORCH_EXTENSION}"
)
target_include_directories(${TORCH_EXTENSION} PUBLIC
        ${CMAKE_SOURCE_DIR}/include
        ${Torch_ROOT_DIR}/include
        ${Torch_ROOT_DIR}/include/torch/csrc/api/include
        ${Torch_ROOT_DIR}/include/TH
        ${Torch_ROOT_DIR}/include/THC
        /usr/local/cuda/include
        ${Python_INCLUDE_DIRS}
)
target_compile_definitions(${TORCH_EXTENSION} PUBLIC
        -D__CUDA_NO_HALF_OPERATORS__
        -D__CUDA_NO_HALF_CONVERSIONS__
        -D__CUDA_NO_HALF2_OPERATORS__
        -D__CUDA_NO_HALF2_CONVERSIONS__
        -D__CUDA_NO_BFLOAT16_OPERATORS__
        -D__CUDA_NO_BFLOAT16_CONVERSIONS__
        -D__CUDA_NO_BFLOAT162_OPERATORS__
        -D__CUDA_NO_BFLOAT162_CONVERSIONS__
        -DTORCH_API_INCLUDE_EXTENSION_H
        -DPYBIND11_COMPILER_TYPE="_gcc"
        -DPYBIND11_STDLIB="_libstdcpp"
        -DPYBIND11_BUILD_ABI="_cxxabi1011"
        -D_GLIBCXX_USE_CXX11_ABI=0
        -DTORCH_EXTENSION_NAME=${TORCH_EXTENSION}
)
target_compile_options(${TORCH_EXTENSION} PUBLIC
        --compiler-options -fPIC        
        --expt-extended-lambda
        --expt-relaxed-constexpr
        --use_fast_math
        -Xptxas -v
        -lineinfo
)
target_link_directories(${TORCH_EXTENSION} PUBLIC
        ${Python_LIBRARY_DIRS}
        ${Torch_ROOT_DIR}/lib
        /usr/local/cuda/lib64
)
target_link_libraries(${TORCH_EXTENSION} PUBLIC
        c10
        c10_cuda
        cudart
        curand
        torch
        torch_cpu
        torch_cuda
        torch_python
)
target_link_options(${TORCH_EXTENSION} PUBLIC
        -pthread
        -B ${Python_ROOT_DIR}/compiler_compat
        -Wl,-rpath,${Python_LIBRARY_DIRS}
        -Wl,-rpath-link,${Python_LIBRARY_DIRS}
)
