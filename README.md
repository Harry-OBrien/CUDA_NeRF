# Instant Video Nerf
It's like mipnerf 360, instant ngp, and a recurrent network had a fucked up baby.

This is a **WIP**. Treat it as such.

### Setup
```
sudo apt-get install xxx

git clone --recursive https://path/to/this/repo
```

Also! Install [CUDA](https://developer.nvidia.com/cuda-toolkit) and [OptiX](https://developer.nvidia.com/optix) in `/usr/local/` and adding the installations to your PATH.

For example, if you have CUDA 11.4, add the following to your `~/.bashrc`
```sh
export PATH="/usr/local/cuda-11.4/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/cuda-11.4/lib64:$LD_LIBRARY_PATH"

export OptiX_INSTALL_DIR=/usr/local/NVIDIA-OptiX-SDK-7.6.0-linux64-x86_64
```

### Compilation (Linux only)
```sh
cmake . -B build
cmake --build build
```