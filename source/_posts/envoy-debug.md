---
title: Envoy 编译调试
tags:
  - Backend
  - Middleware
categories:
  - Middleware
date: 2021.07.16 20:09:00
updated: 2021.07.16 20:09:00
---


## Debian9 上编译调试

主要参考Envoy官方的[Bazel编译文档](https://github.com/envoyproxy/envoy/tree/main/bazel)

1. 下载bazelisk-linux-amd64

		sudo wget -O /usr/local/bin/bazel https://github.com/bazelbuild/bazelisk/releases/latest/download/bazelisk-linux-amd64
		sudo chmod +x /usr/local/bin/bazel

2. 安装依赖

		sudo apt-get install \
		   autoconf \
		   automake \
		   cmake \
		   curl \
		   libtool \
		   make \
		   ninja-build \
		   patch \
		   python3-pip \
		   unzip \
		   virtualenv

3. 下载llvm编译器

		md llvm // 新建目录
		wget https://github.com/llvm/llvm-project/releases/download/llvmorg-12.0.0/clang+llvm-12.0.0-x86_64-linux-gnu-ubuntu-16.04.tar.xz
		
		tar xf clang+llvm-12.0.0-x86_64-linux-gnu-ubuntu-16.04.tar.xz
		
		mv clang+llvm-12.0.0-x86_64-linux-gnu-ubuntu-16.04 src // 修改名字

		#保存到环境变量 .zshrc （不保存也可以，下面脚本会写的user.bazelrc中去）
		export PATH=$PATH:/home/fanlv/llvm/src/bin
		export PATH=$PATH:/home/fanlv/llvm/src/include
		export PATH=$PATH:/home/fanlv/llvm/src/lib
		export PATH=$PATH:/home/fanlv/llvm/src/libexec
		export PATH=$PATH:/home/fanlv/llvm/src/share

4. `Debian9.x `默认是`python3.5`，build过程需要用到`Jinja2 - 3.0.1` 必须要`python 3.6`以上版本，如果是3.6以上可以忽略这一步。

		apt-get autoremove python3.5 python3.5-dev
		sudo apt-get install dirmngr sudo gcc
		vim /etc/apt/sources.list
		deb http://mirrors.163.com/ubuntu/ bionic main
		sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3B4FE6ACC0B21F32
		echo 'APT::Default-Release "stable";' | sudo tee -a /etc/apt/apt.conf.d/00local
		sudo apt-get update
		sudo apt-get -t bionic install python3.6 python3.6-dev python3-distutils python3-pip
		ln -s /usr/bin/python3.6 /usr/bin/python3

5. clone envoy源码
		
		//当前commit b62dae29a5dd06b7f689899b26974d9567a98f0e
		git clone git@github.com:envoyproxy/envoy.git

6. 配置Bazel使用llvm编译器

		cd envoy
		bazel/setup_clang.sh /home/fanlv/llvm/src // 这个是llvm 文件位置
		echo "build --config=libc++" >> user.bazelrc
	
		#--config=libc++ means using clang + libc++
		#--config=clang means using clang + libstdc++
		#no config flag means using gcc + libstdc++

7. 开始编译

		cd envoy
		bazel build -c dbg --verbose_failures --verbose_explanations  --config=libc++ //source/exe:envoy-static



8. 动态库找不到错误报错
	
		python3 ../../tools/run.py ./bytecode_builtins_list_generator gen/builtins-generated/bytecodes-builtins-list.h
		./bytecode_builtins_list_generator: error while loading shared libraries: libc++abi.so.1: cannot open shared object file: No such file or directory

	解决方式，更多参考[这里](https://github.com/envoyproxy/envoy/pull/9024)

		ln -s /data00/home/fanlv/llvm/src/lib/* /usr/lib/


![调试效果](https://upload-images.jianshu.io/upload_images/12321605-3ad5af16e2f5f0ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## Mac 上编译调试

1. bazel安装同上一
2. 安装依赖

		brew install coreutils wget cmake libtool automake ninja clang-format autoconf aspell
		
3. 编译
	
		bazel build --explain=file.txt --verbose_explanations --verbose_failures //source/exe:envoy-static
		// 带符号表的
		bazel build -c dbg //source/exe:envoy-static --copt=-Wno-inconsistent-missing-override --spawn_strategy=standalone --genrule_strategy=standalone
		
