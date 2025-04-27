# ktransformers0.2.4post1 whl使用方法


前置：安装conda, 支持avx2指令集，支持cuda12.4 runtime（驱动版本cuda在12.4及以上）。
要求显卡计算能力在8.0;8.6;8.7;8.9;9.0中

```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --set show_channel_urls yes
conda create -n kt python=3.11 -y
conda activate kt
sudo apt-get update -y
sudo apt-get install build-essential cmake ninja-build patchelf -y
sudo apt-get install --only-upgrade libstdc++6 -y
conda install -c conda-forge libstdcxx-ng -y
strings ~/miniconda3/envs/kt/lib/libstdc++.so.6 | grep GLIBCXX_3.4.32 # 确保有GLIBCXX_3.4.32输出!
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
pip install openai zmq 
pip install torch==2.4.0 --index-url https://download.pytorch.org/whl/cu124
pip install https://github.com/Dao-AILab/flash-attention/releases/download/v2.7.4.post1/flash_attn-2.7.4.post1+cu12torch2.4cxx11abiFALSE-cp311-cp311-linux_x86_64.whl
pip install https://github.com/Anionex/ktransformers-0.2.4/releases/download/0.2.4post1/ktransformers-0.2.4.post1+cu124torch24avx2-cp311-cp311-linux_x86_64.whl
pip install https://github.com/Anionex/ktransformers-0.2.4/releases/download/0.2.4post1/flashinfer_python-0.2.3-py3-none-any.whl


```

启动：
```bash
conda activate kt
export TORCH_CUDA_ARCH_LIST=8.9 # 4090 的是8.9，其他卡需要自己更换!
export CUDA_HOME=/usr/local/cuda-12.4 # 需要提前安装cuda-12.4 的toolkit！
# ... 你的后续启动命令


## 如果运行提示找不到so文件，先执行以下命令
wget https://github.com/Anionex/ktransformers-0.2.4/releases/download/0.2.4post1/kt_libs.zip -O ~/kt_libs.zip
cd ~
unzip kt_libs.zip
KT_LIB_DIR="~/.ktransformers_libs"
mkdir -p $KT_LIB_DIR
cp ~/kt_libs/*.so* $KT_LIB_DIR
export LD_LIBRARY_PATH=$KT_LIB_DIR:$LD_LIBRARY_PATH
KT_LIB_DIR="~/.ktransformers_libs"
export LD_LIBRARY_PATH=$KT_LIB_DIR:$LD_LIBRARY_PATH
# ... 你的后续启动命令
```



# ktransformers0.2.4post1构建记录

需修改install.sh内容
```bash
#!/bin/bash
set -e  

# clear build dirs
rm -rf build
rm -rf *.egg-info
rm -rf csrc/build
rm -rf csrc/ktransformers_ext/build
rm -rf csrc/ktransformers_ext/cuda/build
rm -rf csrc/ktransformers_ext/cuda/dist
rm -rf csrc/ktransformers_ext/cuda/*.egg-info
rm -rf ~/.ktransformers
echo "Installing python dependencies from requirements.txt"
# pip install -r requirements-local_chat.txt
# pip install -r ktransformers/server/requirements.txt
echo "building ktransformers"
rm -rf ./dist
mkdir -p dist
KTRANSFORMERS_FORCE_BUILD=TRUE pip wheel -v . --no-build-isolation -w dist
CPU_INSTRUCT=AVX2 USE_BALANCE_SERVE=1  pip wheel -v third_party/custom_flashinfer/ -w dist

# SITE_PACKAGES=$(python -c "import site; print(site.getsitepackages()[0])")
# echo "Copying thirdparty libs to $SITE_PACKAGES"
# cp -a csrc/balance_serve/build/third_party/prometheus-cpp/lib/libprometheus-cpp-*.so* $SITE_PACKAGES/
# patchelf --set-rpath '$ORIGIN' $SITE_PACKAGES/sched_ext.cpython*

# echo "installing ktransformers"
# pip install --no-index --find-links=./dist
ls dist/
echo "Installation completed successfully"
```

需修改ktransformers/csrc/balance_serve/CMakeLists.txt内容：
在开头增加，设置cuda 标准为17，防止遇到：`requires the language dialect "CUDA20" , but CMake does not know the compile flags to use to enable it.`
```
set(CMAKE_CUDA_STANDARD 17) 
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
```

编译源码并构建轮子

```bash
git clone https://github.com/kvcache-ai/ktransformers/
cd ktransformers 
git submodule update --init --recursive

echo 'export PATH=/usr/local/cuda-12.4/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export CUDA_HOME=/usr/local/cuda-12.4' >> ~/.bashrc
conda init
source ~/.bashrc

conda create -n kt python=3.11 -y
conda activate kt
sudo apt-get update -y
sudo apt-get install build-essential cmake ninja-build patchelf -y
sudo apt-get install --only-upgrade libstdc++6 -y

conda install -c conda-forge libstdcxx-ng -y
strings ~/miniconda3/envs/kt/lib/libstdc++.so.6 | grep GLIBCXX_3.4.32 # 确认输出是否有内容

pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

pip install torch==2.4.0 --index-url https://download.pytorch.org/whl/cu124
pip install torch packaging ninja cpufeature numpy
pip install https://github.com/Dao-AILab/flash-attention/releases/download/v2.7.4.post1/flash_attn-2.7.4.post1+cu12torch2.4cxx11abiFALSE-cp311-cp311-linux_x86_64.whl

CPU_INSTRUCT=AVX2 USE_BALANCE_SERVE=1 bash ./install.sh

```

安装构建出来的轮子
```
pip install ktransformers flashinfer_python --no-index --find-links=./dist
```

参考错误
```
遇到错误 /root/ktransformers/csrc/balance_serve/build/CMakeFiles/CMakeTmp/CMakeLists.txt: Target "cmTC_20c6c" requires the language dialect "CUDA20" (with compiler extensions), but CMake does not know the compile flags to use to enable it.

尝试解决：
方案1: set(CMAKE_CXX_STANDARD 17) 加在了scrs.balance_serve里，但是源码里直接用了std::counting_semaphore，那是20才支持的 

我刚才升级了gcc到12，会解决这个问题吗？——不行。 

后来发现 看错 了，注意区分set(CMAKE_CUDA_STANDARD 17) 和  set(CMAKE_CXX_STANDARD 17) 我一开始通过不了是因为设置了后者，但是实际上要求cxx standard 20但是 要把 cuda standard设为17



直接测试命令：
cmake /root/ktransformers/csrc/balance_serve \
    -DCMAKE_LIBRARY_OUTPUT_DIRECTORY=/root/ktransformers/build/lib.linux-x86_64-cpython-311/ \
    -DPYTHON_EXECUTABLE=/root/miniconda3/envs/kt/bin/python \
    -DCMAKE_BUILD_TYPE=Release \
    -DKTRANSFORMERS_USE_CUDA=ON \
    -D_GLIBCXX_USE_CXX11_ABI=0 \
    -DLLAMA_NATIVE=OFF \
    -DLLAMA_FMA=ON \
    -DLLAMA_F16C=ON \
    -DLLAMA_AVX=ON \
    -DLLAMA_AVX2=ON \
    -DEXAMPLE_VERSION_INFO=0.2.4.post1+cu124torch24avx2

发现加上set(CMAKE_CUDA_STANDARD 17) 之后无报错。

方案2: 升级gcc，用一个支持c20的
sudo apt install g++-12  # Ubuntu/Debian
export CXX=g++-12
```



```
启动服务器卡住：玄学bug，目前原因未知。解决方法：
看issue编号#1056
model path小文件指令本地的，并且取名要严格按照仓库名/后的文字来。
如果再不行的话，说是要指定cuda_home和cuda archlist。
export TORCH_CUDA_ARCH_LIST="8.9" # 8.9 for 4090
```
<img src="./assets/Pasted image 20250424151652.png">