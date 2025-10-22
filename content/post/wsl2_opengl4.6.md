---
title: WSL2 + Ubuntu20.04 使用OpenGL4.6
description: 在WSL2环境中，通过配置Mesa和D3D12后实现Ubuntu 20.04使用OpenGL 4.6进行硬件加速渲染。
date: 2025-10-22T20:12:52+08:00
lastmod: 2022-10-22T20:12:52+08:00
tags:
  - wsl2
categories:
  - wsl2
  - ubuntu20.04
---

## 💡 背景

在使用 **Habitat-Sim** 时，发现默认的 OpenGL 版本仅为 3.3，无法支持更高版本的 GLSL shader。

错误信息示例：

```
GLSL 4.10 is not supported … Supported versions are: … 3.30 …
Assertion … PbrShader.cpp …
```

当时的渲染环境如下：

```
Renderer: D3D12 (NVIDIA GeForce MX450) by Microsoft Corporation
OpenGL version: 3.3 (Core Profile) Mesa 21.2.6
```

## 🧩 系统环境

| 项目 | 版本 |
|------|------|
| 设备 | Intel + Nvidia MX450 |
| Ubuntu | 20.04 |
| WSL | 2 |
| Windows | 10.0.22631.4751 |

### `nvidia-smi` 输出

```
Mon Oct 20 16:36:44 2025
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.230.02             Driver Version: 576.52       CUDA Version: 12.9     |
|-----------------------------------------+----------------------+----------------------|
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce MX450           On  | 00000000:01:00.0 Off |                  N/A |
| N/A   61C    P0              N/A / 5... |    211MiB /  2048MiB |      0%      Default |
|                                         |                      |                  N/A |
+---------------------------------------------------------------------------------------+
```

### `wsl --version` 输出

```
WSL 版本: 2.6.1.0
内核版本: 6.6.87.2-1
WSLg 版本: 1.0.66
MSRDC 版本: 1.2.6353
Direct3D 版本: 1.611.1-81528511
DXCore 版本: 10.0.26100.1-240331-1435.ge-release
Windows: 10.0.22631.4751
```

---

## 🧱 配置步骤

为了解决 OpenGL 版本过低问题，我们可以使用 **Mesa 的 D3D12 驱动** 来启用硬件加速并支持 OpenGL 4.6。

### 1️⃣ 添加 PPA 源并更新 Mesa

```bash
sudo add-apt-repository ppa:kisak/turtle
sudo add-apt-repository ppa:kisak/kisak-mesa
sudo apt update && sudo apt upgrade
```

更新完成后，执行以下命令查看当前 OpenGL 状态：

```bash
glxinfo -B
```

此时输出可能显示为 LLVM 软件渲染：

```
Vendor: Mesa (0xffffffff)
Device: llvmpipe (LLVM 15.0.7, 256 bits)
Version: 25.0.7
Accelerated: no
OpenGL renderer string: llvmpipe (LLVM 15.0.7, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.0.7
```

这意味着当前仍在使用 CPU 软件渲染，而非 GPU 加速。

---

## ⚙️ 启用 D3D12 驱动

使用以下环境变量切换到 D3D12 驱动：

```bash
export GALLIUM_DRIVER=d3d12
export MESA_D3D12_DEFAULT_ADAPTER_NAME=NVIDIA
```

再执行：

```bash
glxinfo -B
```

你会看到输出变为：

```
Vendor: Microsoft Corporation (0xffffffff)
Device: D3D12 (NVIDIA GeForce MX450) (0xffffffff)
Version: 25.0.7
Accelerated: yes
OpenGL renderer string: D3D12 (NVIDIA GeForce MX450)
OpenGL core profile version string: 4.6 (Core Profile) Mesa 25.0.7 - kisak-mesa PPA
OpenGL shading language version string: 4.60
```

✅ 此时 OpenGL 已经升级为 **4.6**，并且启用了 **硬件加速**。

---

## 🧾 验证结果

执行 `glxgears` 或运行 OpenGL 程序（如 habitat-sim）后，应能观察到显卡活动。

`nvidia-smi` 中也会显示 GPU 占用。

---

## 🧠 说明

- WSLg 提供的 D3D12 驱动允许在 Linux 应用中使用 Windows 的显卡驱动进行 GPU 渲染。
- Mesa 的 `d3d12` Gallium 驱动通过 DirectX 12 提供 OpenGL 接口层。
- 不建议混用 LLVM 渲染与 D3D12 驱动，否则可能导致冲突。

---

## 📚 参考资料

- [在WSL2中开启OpenGL直接渲染以及升级OpenGL — CSDN 博客](https://blog.csdn.net/GodNotAMen/article/details/125123186)  
- [Can WSL2 support a higher version of OpenGL? — Microsoft/WSL Discussion #6154](https://github.com/microsoft/WSL/discussions/6154)

---

## 🎉 总结

通过添加 `kisak-mesa` PPA 并启用 `d3d12` 驱动，可以在 **WSL2 + Ubuntu 20.04** 环境中成功使用 **OpenGL 4.6**。  
这不仅提升了图形渲染性能，也使得依赖高版本 GLSL 的程序（如 Habitat-Sim、Blender、Gazebo 等）能够顺利运行。
