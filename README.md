# Redmi K90 Pro Max：GitHub 自动编译 whisper.cpp OpenCL

这个仓库模板会通过 GitHub Actions 交叉编译 Android `arm64-v8a` 版 `whisper.cpp`，启用针对 Adreno 的 OpenCL 后端。产物可直接下载到 Termux 安装。

本配置专门针对 Redmi K90 Pro Max 的 SM8850。实机 `/proc/cpuinfo` 已确认提供 `SVE2`、`I8MM` 和 `SME`，因此 CPU 编译目标设为：

```text
armv9.2-a+dotprod+fp16+i8mm+sve2+sme
```

这里的 `arm64-v8a` 是 Android ABI 名称，与 Armv9.2-A CPU 指令集并不冲突。

## 使用方法

1. 在 GitHub 新建一个空仓库，把本模板中的 `.github/workflows/build-whisper-opencl-android.yml` 上传到同一路径。
2. 打开仓库的 **Actions** 页面，选择 **Build whisper.cpp OpenCL for Termux**。
3. 点击 **Run workflow**。`whisper_ref` 保持 `master`，或填入指定 tag/commit。
4. 构建完成后，在该次运行页面的 **Artifacts** 下载 `whisper-opencl-android-arm64-*`。
5. 解压 GitHub 下载的 ZIP，得到其中的 `.tar.gz`，再把 `.tar.gz` 放到手机 Termux 的主目录。

Termux 中执行：

```bash
tar -xzf whisper-opencl-android-arm64-*.tar.gz
cd whisper-opencl-android-arm64
bash install.sh
bash download-model.sh base
```

准备音频并转写：

```bash
pkg install ffmpeg
ffmpeg -i input.m4a -ar 16000 -ac 1 -c:a pcm_s16le input.wav
whisper-opencl -m ~/models/ggml-base.bin -f input.wav -l zh -otxt -osrt -t 8
```

`base` 是多语言模型，速度快；要提高中文识别准确率可改用 `small`：

```bash
bash download-model.sh small
whisper-opencl -m ~/models/ggml-small.bin -f input.wav -l zh -otxt -osrt -t 8
```

## 自动构建规则

- 手动：Actions 页面点击 **Run workflow**。
- 修改工作流并推送到 `main` 或 `master`：自动运行。
- 每周一 UTC 03:17：自动跟随 `whisper.cpp` 的 `master` 构建。

## 为什么不用普通 OpenCL ICD

Qualcomm Adreno 的 Android 驱动通常不提供 `cl_khr_icd`，因此 Termux 的通用 ICD 加载器可能让 `clinfo` 显示 0 个平台。本模板把可执行文件改为直接依赖手机厂商的 `libOpenCL.so`，并由 Termux 的 `opencl-vendor-driver` 复制和修补厂商驱动依赖。

工作流还把 `GGML_OPENCL_TARGET_VERSION` 固定为 `200`，避免部分 Qualcomm OpenCL 2.0 驱动因缺少 OpenCL 2.1 的 `clGetKernelSubGroupInfo` 符号而无法启动。

## 故障排查

确认手机存在厂商驱动：

```bash
ls -l /vendor/lib64/libOpenCL.so
```

若安装后仍提示找不到驱动：

```bash
pkg install patchelf opencl-vendor-driver
OVERRIDE_LIBOPENCL=/vendor/lib64/libOpenCL.so pkg reinstall opencl-vendor-driver
```

系统升级后需要重新复制和修补驱动：

```bash
pkg reinstall opencl-vendor-driver
```

运行一次实际转写来验证，日志中应出现 OpenCL/Adreno 信息。由于本模板绕过 ICD，`clinfo` 的结果不能作为唯一判断依据。
