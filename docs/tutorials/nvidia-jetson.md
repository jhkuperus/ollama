# Running Ollama on NVIDIA Jetson Devices

Ollama runs on [NVIDIA Jetson Devices](https://www.nvidia.com/en-us/autonomous-machines/embedded-systems/). If the Ollama server does not find a GPU on startup, there are a couple of things you can check to make sure your system is properly configured.

The following has been tested on the [JetPack](https://developer.nvidia.com/embedded/jetpack) firmware, on versions 5.1.2 and 6 DP.

## No CUDA libraries detected during startup

### Check CUDA libraries are installed

When Ollama starts up, it outputs information about the GPU discovery process. When you are running the `ollama serve` manually in a terminal, you will see this information appear in your terminal. If it is running as a service, you can access this information through `journalctl --unit ollama`.

```
level=INFO source=payload_common.go:107 msg="Extracting dynamic libraries..."
level=INFO source=payload_common.go:146 msg="Dynamic LLM libraries [cuda_v12 cpu cpu_avx2 cpu_avx]"
level=INFO source=gpu.go:133 msg="Detecting GPU type"
level=INFO source=gpu.go:317 msg="Searching for GPU management library libcudart.so"
level=INFO source=gpu.go:363 msg="Discovered GPU libraries: []"
level=INFO source=gpu.go:317 msg="Searching for GPU management library librocm_smi64.so"
level=INFO source=gpu.go:363 msg="Discovered GPU libraries: []"
level=INFO source=cpu_common.go:18 msg="CPU does not have vector extensions"
level=INFO source=routes.go:1036 msg="no GPU detected"
```

The sitaution above shows that Ollama was unable to find any CUDA libraries and goes into CPU only mode. When you see this, it is most likely the case that your CUDA libraries are not installed, or you are running an older version of Ollama.

Your Jetson device should have the CUDA libraries installed in the `/usr/local/cuda` directory. You should see either `cuda-11` or `cuda-12` folders here:

```
root@jetson:/# ll -d /usr/local/cuda*
lrwxrwxrwx  1 root root   22 Feb 15 11:26 /usr/local/cuda -> /etc/alternatives/cuda/
lrwxrwxrwx  1 root root   25 Feb 15 11:26 /usr/local/cuda-12 -> /etc/alternatives/cuda-12/
drwxr-xr-x 13 root root 4096 Feb 15 11:44 /usr/local/cuda-12.2/
```

If these files are missing, either install the JetPack SDK Components throught the SDK Manager, or install them with `sudo apt install cuda-toolkit`. For more information, you can visit the [NVIDIA CUDA Installation Instructions](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html).

## Install latest version of Ollama

Make sure you have the newest version of Ollama installed. The detection of CUDA libraries did not work on Jetson devices prior to version X. You can install Ollama via the standard Linux installation method:

```
curl https://ollama.com/install.sh | sh
```

## Permission Issues during startup

On a system that has the CUDA Toolkit installed and also has a recent version of Ollama installed, there is another situation that may cause problems. On a fresh install for Ollama, this should not be a problem. It is likely only to occur after updating from an older version of Ollama.

The symptoms for this situation is the message `NvRmMemInitNvmap failed with Permission denied` during startup:

```
level=INFO source=payload_common.go:107 msg="Extracting dynamic libraries..."
level=INFO source=payload_common.go:146 msg="Dynamic LLM libraries [cuda_v12 cpu cpu_avx2 cpu_avx]"
level=INFO source=gpu.go:133 msg="Detecting GPU type"
level=INFO source=gpu.go:317 msg="Searching for GPU management library libcudart.so"
level=INFO source=gpu.go:363 msg="Discovered GPU libraries: [/usr/local/cuda/lib64/libcudart.so.12.2.140 /usr/local/cuda-12/lib64/libcudart.so.12.2.140 /usr/local/cuda-12.2/lib64/libcudart.so.12.2.140]"
NvRmMemInitNvmap failed with Permission denied
356: Memory Manager Not supported
****NvRmMemMgrInit failed**** error type: 196626
time=2024-02-15T18:55:47.591+01:00 level=INFO source=gpu.go:375 msg="Unable to load CUDA management library /usr/local/cuda/lib64/libcudart.so.12.2.140: cudart vram init failure: 999"
NvRmMemInitNvmap failed with Permission denied
356: Memory Manager Not supported
****NvRmMemMgrInit failed**** error type: 196626
time=2024-02-15T18:55:47.593+01:00 level=INFO source=gpu.go:375 msg="Unable to load CUDA management library /usr/local/cuda-12/lib64/libcudart.so.12.2.140: cudart vram init failure: 999"
NvRmMemInitNvmap failed with Permission denied
356: Memory Manager Not supported
****NvRmMemMgrInit failed**** error type: 196626
level=INFO source=gpu.go:375 msg="Unable to load CUDA management library /usr/local/cuda-12.2/lib64/libcudart.so.12.2.140: cudart vram init failure: 999"
level=INFO source=gpu.go:317 msg="Searching for GPU management library librocm_smi64.so"
level=INFO source=gpu.go:363 msg="Discovered GPU libraries: []"
level=INFO source=cpu_common.go:18 msg="CPU does not have vector extensions"
level=INFO source=routes.go:1036 msg="no GPU detected"
```

These lines show that Ollama has been able to find CUDA libraries, but is unable to load them due to missing permissions. The user that is starting `ollama serve` needs to have permission to access the video hardware. Depending on your exact distribution, this can be restricted to the `render` or `video` group. The system service is by default configured to run under the `ollama` user. To make sure the `ollama` user has access to the these groups, you can run these commands:

```
sudo usermod -a -G ollama render
sudo usermod -a -G ollama video
```

If this does not fix your permission denied issues, make sure that the service is run under the correct user, or try to find out which group will allow access to the GPU.

