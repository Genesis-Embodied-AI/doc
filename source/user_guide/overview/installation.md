# 🛠️ Installation
## Prerequisites
* **Python**: 3.9+
* **OS**: Linux (*recommended*) / MacOS / Windows

:::{note}
Genesis is designed to be ***cross-platform***, supporting backend devices including *CPU*, *CUDA GPU* and *non-CUDA GPU*. That said, it is recommended to use **Linux** platform with **CUDA-compatible GPU** to achieve the best performance.
:::

Supported features on various systems are as follows:
<div style="text-align: center;">

| OS  | GPU Device        | GPU Simulation | CPU Simulation | Interactive Viewer | Headless Rendering |
| ------- | ----------------- | -------------- | -------------- | ---------------- | ------------------ |
| Linux   | Nvidia            | ✅             | ✅             | ✅               | ✅                 |
|         | AMD               | ✅             | ✅             | ✅               | ✅                 |
|         | Intel             | ✅             | ✅             | ✅               | ✅                 |
| Windows | Nvidia            | ✅             | ✅             | ❌               | ❌                 |
|         | AMD               | ✅             | ✅             | ❌               | ❌                 |
|         | Intel             | ✅             | ✅             | ❌               | ❌                 |
| MacOS   | Apple Silicon     | ✅             | ✅             | ✅               | ✅                 |

</div>

## Installation
1. Genesis is available via PyPI:
    ```bash
    pip install genesis-world
    ```

2. Install **PyTorch** following the [official instructions](https://pytorch.org/get-started/locally/).


## (Optional) Motion planning
Genesis integrated OMPL's motion planning functionalities and wraps it using a intuitive API for effortless motion planning. If you need the built-in motion planning capability, download pre-compiled OMPL wheel [here](https://github.com/ompl/ompl/releases/tag/prerelease), and then `pip install` it.

## (Optional) Surface reconstruction
If you need fancy visuals for visualizing particle-based entities (fluids, deformables, etc.), you typically need to reconstruct the mesh surface using the internal particle-based representation. We provide two options for this purpose:

1. [splashsurf](https://github.com/InteractiveComputerGraphics/splashsurf), a state-of-the-art surface reconstruction method for achieving this:
    ```bash
    cargo install splashsurf
    ```
2. ParticleMesher, our own openVDB-based surface reconstruction tool (faster but with not as smooth):
    ```bash
    echo "export LD_LIBRARY_PATH=${PWD}/ext/ParticleMesher/ParticleMesherPy:$LD_LIBRARY_PATH" >> ~/.bashrc
    source ~/.bashrc
    ```


## (Optional) Ray Tracing Renderer

If you need photo-realistic visuals, Genesis has a built-in a ray-tracing (path-tracing) based renderer developped using [LuisaCompute](https://github.com/LuisaGroup/LuisaCompute), a high-performance domain specific language designed for rendering.

### 1. Get LuisaRender
The submodule LuisaRender is under `ext/LuisaRender`:
```
git submodule update --init --recursive
```
### 2. Dependencies 

#### 2.A: If you have sudo access. Preferred.
**NB**: It seems compilation only works on Ubuntu 20.04+, As vulkan 1.2+ is needed and 18.04 only supports 1.1, but I haven't fully checked this...

- upgrade `g++` and `gcc` to version 11
    ```
    sudo apt install build-essential manpages-dev software-properties-common
    sudo add-apt-repository ppa:ubuntu-toolchain-r/test
    sudo apt update && sudo apt install gcc-11 g++-11
    sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 110
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 110

    # verify
    g++ --version
    gcc --version
    ```
- cmake
    ```
    # if you system's cmake version is under 3.18, uninstall that and reinstall via snap
    sudo snap install cmake --classic
    ```
- CUDA
    - You need to install a system-wide cuda (Now 12.0+).
        - download https://developer.nvidia.com/cuda-11-7-0-download-archive
        - Install cuda toolkit.
        - reboot
- rust
    ```
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    sudo apt-get install patchelf
    # if the above gives downloader error, make sure your curl was installed via apt, not snap
    ```
- Vulkan
    ```
    sudo apt install libvulkan-dev
    ```
- zlib
    ```
    sudo apt-get install zlib1g-dev
    ```
- RandR headers
    ```
    sudo apt-get install xorg-dev libglu1-mesa-dev
    ```
- pybind
    ```
    pip install "pybind11[global]"
    ```
- libsnappy
    ```
    sudo apt-get install libsnappy-dev
    ```
#### 2.B: If you have no sudo.
- conda dependencies
    ```
    conda install -c conda-forge gcc=11.4 gxx=11.4 cmake=3.26.1 minizip zlib libuuid patchelf vulkan-tools vulkan-headers
    ```
- rust
    ```
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    ```
- pybind
    ```
    pip install "pybind11[global]"
    ```

### 3. Compile
- Build LuisaRender and its python binding:
    - If you used system dependencies (2.A)
        ```
        cd genesis/ext/LuisaRender
        cmake -S . -B build -D CMAKE_BUILD_TYPE=Release -D PYTHON_VERSIONS=3.9 -D LUISA_COMPUTE_DOWNLOAD_NVCOMP=ON -D LUISA_COMPUTE_ENABLE_GUI=OFF 
        cmake --build build -j $(nproc)
        ```
        By default, we use optix deoniser. If you need OIDN, append `-D LUISA_COMPUTE_DOWNLOAD_OIDN=ON`.
    - If you used conda dependencies (2.B)
        ```
        export CONDA_INCLUDE_PATH=path/to/anaconda/include
        cd ./ext/LuisaRender
        cmake -S . -B build -D CMAKE_BUILD_TYPE=Release -D PYTHON_VERSIONS=3.9 -D LUISA_COMPUTE_DOWNLOAD_NVCOMP=ON -D LUISA_COMPUTE_ENABLE_GUI=OFF -D ZLIB_INCLUDE_DIR=$CONDA_INCLUDE_PATH
        cmake --build build -j $(nproc)
        ```
        The `CONDA_INCLUDE_PATH` typically looks like: `/home/user/anaconda3/envs/genesis/include`

### 4. Install
- Link the build files into the `ext` folder of your `genesis` install, so that python can import it
    ```
    PYTHON_PKG_PATH=$(python3 -c 'import sysconfig; print(sysconfig.get_paths()["purelib"])')
    cd .. # you should now be in `genesis/ext`
    ln -s $PWD/LuisaRender $PYTHON_PKG_PATH/genesis/ext
    ```

### 5. FAQs
- Assertion 'lerror’ failed: Failed to write to the process: Broken pipe:
  You may need to use CUDA of the same version as compiled.
- if you followed 2.A and see "`GLIBCXX_3.4.30` not found"
    ```
    cd ~/anaconda3/envs/genesis/lib
    mv libstdc++.so.6 libstdc++.so.6.old
    ln -s /usr/lib/x86_64-linux-gnu/libstdc++.so.6 libstdc++.so.6
    ```
