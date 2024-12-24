---
title: OpenEnclave机密计算开发环境教程
author: Axellance
date: 2024-12-24
layout: post
category: TEE
tags: [TEE, SGX, TrustZone]
mermaid: true
---

OpenEnclave（OE）是微软开发的TEE软件栈，它支持Intel SGX和OP-TEE on Arm TrustZone，值得注意的是，OE和linux-sgx两者互为替代关系。同时OE也支持C/C++。OpenEnclave的一个主要用途是在微软Azure云上保护目标安全。

该SDK旨在推广来自不同硬件供应商的tee之间的enclave应用程序的开发。当前的实现提供了对Intel SGX的支持，以及对ARM TrustZone上OP-TEE操作系统的预览支持。作为一个开源项目，该SDK还努力提供一个透明的解决方案，不受特定供应商、服务提供商和操作系统选择的影响。

但是目前在网上没有完整的入门教程，所以本文整理了github以及其它社区的一些OpenEnclave教程，和各位同行分享。

### 1. OpenEnclave简介

OpenEnclave是一个用于在C和c++中构建Enclave应用程序的SDK。每个enclave应用程序将自己划分为两个组件：

1. 不受信任的程序组件（host）；
2. 可信组件（enclave）

enclave是一个受保护的内存区域，为数据和代码执行提供机密性。它是可信执行环境（TEE）的一个实例，TEE通常由硬件保护，例如Intel Software Guard Extensions （SGX）。

这里值得注意的是，虽然TrustZone的底层架构与SGX有较大差异，但是这里我们统一将他们的TEE称为enclave

### 2. 安装OpenEnclave SDK

我们目前暂时整理了在Intel SGX上的OE SDK安装教程，基于OP-TEE on Arm TrustZone的教程后续也会整理。为了保证后续安装过程顺利执行，在进行这一步操作之前最好先安装Intel SGX驱动（注意，是驱动而不是linux sgx, linux sgx是sdk），安装教程之前可以参考之前的文章。

**预备阶段**，由于OE SDK安装过程中需要使用clang编译器，建议先安装clang

```shell
sudo apt-get install clang
```

同时，如果设备上没有安装libstdc++，最好提前安装，不然后面链接过程中会报错

```shell
sudo apt-get install libstdc++-dev
```

**第一步**，从远程仓库拉取代码，一定要注意递归拉取（加上`--recursive` 参数）

```shell
git clone --recursive https://github.com/openenclave/openenclave.git
```

这一过程可能会花费很长时间，具体看网络情况。

**第二步**，安装项目需求

```shell
cd openenclave
# 1.安装Ansible
sudo scripts/ansible/install-ansible.sh
ansible-playbook scripts/ansible/oe-contributors-setup-sgx1.yml

# 2.build
mkdir build && cd build
cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX=/opt/openenclave ..

# 3.安装sdk
make install
```

这里的 `CMAKE_INSTALL_PREFIX` 可以自由设置，也可以安装之前安装linux-sgx sdk的习惯，将其安装在`/opt`目录下。

接下来就可以等待安装过程了。

**第三步**， 运行单元测试，检查sdk是否可以按照预期运行，这里路径不用变，直接在第二步的文件夹下执行即可

```shell
cmake .. && make
ctest 
```

运行测试程序，这个过程很长，可以去干点别的。

```shell
99% tests passed, 6 tests failed out of 1173

Total Test time (real) = 1086.57 sec

The following tests did not run:
        100 - tests/mbedtls_tls_e2e (Skipped)
        101 - tests/openssl_tls_e2e (Skipped)
        102 - tests/openssl_3_tls_e2e (Skipped)
        109 - tests/attestation_cert_api_mbedtls (Skipped)
        110 - tests/attestation_cert_api_openssl (Skipped)
        111 - tests/attestation_cert_api_openssl_3 (Skipped)
        112 - tests/attestation_plugin_mbedtls (Skipped)
        113 - tests/attestation_plugin_openssl (Skipped)
        114 - tests/attestation_plugin_openssl_3 (Skipped)
        115 - tests/attestation_plugin_cert_mbedtls (Skipped)
        116 - tests/attestation_plugin_cert_openssl (Skipped)
        117 - tests/attestation_plugin_cert_openssl_3 (Skipped)
        119 - tests/bigmalloc (Skipped)
        292 - tests/pf_gp_exceptions (Skipped)
        300 - tests/report_attestation_without_enclave (Skipped)

The following tests FAILED:
         82 - tests/host_verify (Child aborted)
        306 - tests/secure_verify_tdx_openssl (Failed)
        307 - tests/secure_verify_tdx_mbedtls (Failed)
        308 - tests/secure_verify_tdx_v5_openssl (Failed)
        309 - tests/secure_verify_tdx_v5_mbedtls (Failed)
        310 - tests/intel_qve_thread_test (Failed)
Errors while running CTest
```

大部分测试过了就可以，如果有测试没过，就需要去检查之前的安装过程。如果错误模块中没有后续开发要使用的模块，建议跳过这一步。

### 3.运行sample代码

```shell
# 1.进入工作区
cd openenclave/samples/helloworld
```

这里我们先对项目文件结构做简单说明，以helloworld为例，结构如下：

```
helloworld
	|_enclave
	|	|_CMakeLists.txt
	|	|_Makefile
	|	|_enc.c
	|	|_helloworld.conf
	|
	|_host
	|	|_CMakeLists.txt
	|	|_Makefile
	|	|_host.c
	|	|_CMakeLists.txt
	|
	|_Makefile
	|_README.md
	|_helloworld.edl
```

看起来结构还是比较清晰的，分成enclave，host和helloworld.edl三个部分去看，分别是安全区、非安全区和边界，目录结构上类似linux-sgx（实际上大家如果后续想深入学习的话，会发现开发流程也是类似的）

构建并运行应用程序：

```shell
# 2.构建应用程序
make debug && cd debug && cmake .. && make
./host/helloworld_host ./enclave/enclave.signed
```

结果如下，好一个HelloWorld：

```shell
Hello world from the enclave
Enclave called into host to print: Hello World!
```

程序中包含一个OCall和一个ECall，感兴趣的话可以去github上拉源码看看。其实不难可以看出，在文件结构、开发流程以及运行方式上，OE和linux-sgx其实没有太大差异。

### 4.说明

我们目前暂时整理了在Intel SGX上的OE SDK安装教程，基于OP-TEE on Arm TrustZone的教程后续也会整理。

本文参考以下内容：

1. https://github.com/openenclave/openenclave

   
