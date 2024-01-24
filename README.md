# UAFSan代码和使用说明

如有问题，请联系caoyixuan2019@email.szu.edu.cn。

## 基本信息

UAFSan基于LLVM 7.1编译器框架，分为2部分：

- 运行库（runtime library ）
- 插桩模块（instrumentation module）

搭建环境：Ubuntu 18.04

开源代码链接：https://github.com/wsong-nj/UAFSan

## 改进代码使用方法

由于LLVM太大，我们仅上传我们针对运行库修改过的代码文件，分别为改进方案1和改进方案2。源代码文件名为：

- baps.c.改进方案1
- baps.c.改进方案2

将此源代码替换文件`runtime library/cbaps/lib/baps.c`，即可使用我们的改进方案。

## 基于LLVM搭建UAFSan

1. 安装LLVM/Clang 7.1的依赖项：
   sudo apt install cmake autoconf gcc g++ git

2. 获取LLVM/Clang 7.1的源代码，有3中方法：

   第一种方法是在https://releases.llvm.org/下载源代码。

   第二种方法是使用如下脚本：

   ```shell
      export BRANCH=release_70
   	git clone http://llvm.org/git/llvm.git -b $BRANCH
   	git clone http://llvm.org/git/clang.git llvm/tools/clang -b $BRANCH
   	git clone http://llvm.org/git/clang-tools-extra.git llvm/tools/clang/tools/extra -b $BRANCH
   	git clone http://llvm.org/git/compiler-rt.git llvm/projects/compiler-rt -b $BRANCH
   ```

   但是在2023年，一些链接已经失效，可以使用llvm-mirror的github仓库的代码，如下：

   ```shell
      export BRANCH=release_70
   	git clone https://github.com/llvm-mirror/llvm.git  -b $BRANCH
   	git clone https://github.com/llvm-mirror/clang.git llvm/tools/clang -b $BRANCH
   	git clone https://github.com/llvm-mirror/clang-tools-extra.git llvm/tools/clang/tools/extra -b $BRANCH
   	git clone https://github.com/llvm-mirror/compiler-rt.git llvm/projects/compiler-rt -b $BRANCH
   ```

   第三种方法是直接使用UAFSan github仓库提供的软件包。

3. 配置LLVM/Clang-7.1并编译、安装。该步骤用于保证本地环境能够成功搭建clang编译器。

   ```sh
   mkdir build && cd build
   cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DLLVM_BUILD_TESTS=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_BUILD_EXAMPLES=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_ENABLE_ASSERTIONS=OFF ..
   make -j8
   sudo make install
   ```

4. 基于UAFSan原作者的开源代码，将UAFSan的编译插桩模块复制到LLVM/Clang的相应目录，替换的目录结构为：

   ```txt
   ----instrumentation module
   ------include  //copy to llvm/include
   ------lib    //copy to llvm/lib
   ------clang // copy to llvm/tools/clang
   ```

   删除之前创建的`build`目录，重做3。

   ```sh
   rm -rf build && mkdir build && cd build
   cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DLLVM_BUILD_TESTS=OFF -DLLVM_INCLUDE_TESTS=OFF -DLLVM_BUILD_EXAMPLES=OFF -DLLVM_INCLUDE_EXAMPLES=OFF -DLLVM_ENABLE_ASSERTIONS=OFF ..
   make -j8
   sudo make install
   ```

5. 经过上述步骤，编译插桩模块成功整合进LLVM/Clang 7.1。

6. 编译运行库，可以获得静态链接库`libbaps.a`。原代码仓库是在目录`cmake-build-debug`构建的，需要注意`cmake`的版本。

使用如下命令，即可将需要测试的源代码编译为插桩文件，编译成功即视为UAFSan搭建成功。`clang`需要用`-lm`链接数学库，`clang++`不需要。对于大型工程，我们只需修改Makefile的编译器和编译选项即可。

```sh
clang -lm -fbaps ./libbaps.a -ldl -rdynamic -g -O0 main.c && ./a.out
clang++ -fbaps ./libbaps.a -ldl -rdynamic -g -O0 main.cpp && ./a.out
```

## UAFSan使用说明

### 单命令程序使用UAFSan插桩编译

使用如下命令，即可将需要测试的源代码编译为插桩文件，`clang`需要用`-lm`链接数学库，`clang++`不需要。其中`./libbaps.a`为运行库模块输出的静态链接库。

```sh
clang -lm -fbaps ./libbaps.a -ldl -rdynamic -g -O0 main.c && ./a.out
clang++ -fbaps ./libbaps.a -ldl -rdynamic -g -O0 main.cpp && ./a.out
```

### 编写Makefile支持UAFSan插桩编译

对于较为复杂的程序，我们需要编写Makefile。以Mibench数据集的程序sha为例，可以使用以下Makefile编译。

```makefile
# By default, the code is compiled for a "big endian" machine.
# To compile on a "little endian" machine set the LITTLE_ENDIAN flag.
# To make smaller object code, but run a little slower, don't use UNROLL_LOOPS.
# To use NIST's modified SHA of 7/11/94, define USE_MODIFIED_SHA

CC = clang
CFLAGS = -O0 -Wall -g -fbaps ./libbaps.a -ldl -rdynamic -lm

sha:	sha_driver.o sha.o
	$(CC) $(CFLAGS)  -o $@ sha_driver.o sha.o 
	strip $@

clean:
	rm -rf *.o sha output*
```

