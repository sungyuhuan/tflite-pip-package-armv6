# tflite-pip-package-armv6
## Modifications to make tensorflow lite standalone pip package script work for ARMv6

```
cd tensorflow/lite/tools/pip_package
make BASE_IMAGE=debian:stretch PYTHON=python3 TENSORFLOW_TARGET=rpi TENSORFLOW_TARGET_ARCH=armv6 docker-build
```

TENSORFLOW_TARGET 設 rpi, 只會走 armv7l (詳情可以看 build_pip_package.sh 裡面的判斷)
開始進行以下修改:

### Fix-1 fatal error: arm-linux-gnueabi/python3.5m/pyconfig.h: No such file or directory
```sudo nano Dockerfile```

Add ```RUN dpkg --add-architecture armel```

Add ```libpython3.5-dev:armel``` in the list of apt install

### Fix-2 fatal error: pybind11/pybind11.h: No such file or directory
```sudo nano Dockerfile```

Add ```pybind11-dev``` in the list of apt install

### Fix-3 補上 build armv6 需要的 dependency
```
sudo nano Dockerfile
```
Add ```crossbuild-essential-armel``` in the list of apt install

### Fix-4 讓 Makefile 可以把 ```TARGET_ARCH=armv6``` 傳下去
```sudo nano Makefile```

```
TENSORFLOW_TARGET ?= native
TENSORFLOW_TARGET_ARCH ?=
…
--env "TENSORFLOW_TARGET=$(TENSORFLOW_TARGET)" \
--env "TENSORFLOW_TARGET_ARCH=$(TENSORFLOW_TARGET_ARCH)" \
```

### Fix-5 permission denied : /bin/bash: tensorflow/tensorflow/lite/tools/pip_package/build_pip_package.sh
```sudo nano Makefile```

```
/bin/bash -c "bash tensorflow/tensorflow/lite/tools/pip_package/build_pip_package.sh
```

### Fix-6 讓 package 名稱正確顯示 armv6
```sudo nano build_pip_package.sh```

```
# Build python wheel.
cd "${BUILD_DIR}"
case "${TENSORFLOW_TARGET}" in
  rpi)
    if [[ -n "${TENSORFLOW_TARGET_ARCH}" ]]; then
      ${PYTHON} setup.py bdist --plat-name=linux-${TENSORFLOW_TARGET_ARCH}l \
                         bdist_wheel --plat-name=linux-${TENSORFLOW_TARGET_ARCH}l
    else
      ${PYTHON} setup.py bdist --plat-name=linux-armv7l \
                         bdist_wheel --plat-name=linux-armv7l
    fi
    ;;
```

### Fix-7 讓 dpkg-buildpackage 使用 armel
```sudo nano build_pip_package.sh```

```
case "${TENSORFLOW_TARGET}" in
  rpi)
    if [[ "${TENSORFLOW_TARGET_ARCH}" == "armv6" ]]; then
      dpkg-buildpackage -b -rfakeroot -us -uc -tc -d -a armel
    else
      dpkg-buildpackage -b -rfakeroot -us -uc -tc -d -a armhf
    fi
    ;;
```

### Fix-8 讓 tensorflow kernel build 成 armv6
```sudo nano ../make/target/rpi_makefile.inc```

```
  # Default to the architecture used on the Pi Two/Three (ArmV7), but override this
  # with TARGET_ARCH=armv6 to build for the Pi Zero or One.
  #TARGET_ARCH := armv7l 
…
ifeq ($(TARGET_ARCH), armv6)
    TARGET_TOOLCHAIN_PREFIX := arm-linux-gnueabi-
```    

### Fix-9 讓 interpreter wrapper 也 build 成 armv6
```sudo nano setup.py```

```
# Setup cross compiling
TARGET = os.environ.get('TENSORFLOW_TARGET', None)
TARGET_ARCH = os.environ.get('TENSORFLOW_TARGET_ARCH')
if TARGET == 'rpi':
  if TARGET_ARCH  == 'armv6':
    os.environ['CXX'] = 'arm-linux-gnueabi-g++'
    os.environ['CC'] = 'arm-linux-gnueabi-gcc'
  else:
    os.environ['CXX'] = 'arm-linux-gnueabihf-g++'
    os.environ['CC'] = 'arm-linux-gnueabihf-gcc'
elif TARGET == 'aarch64':
  os.environ['CXX'] = 'aarch64-linux-gnu-g++'
  os.environ['CC'] = 'aarch64-linux-gnu-gcc'
```

```
(for TF2.1)
MAKE_CROSS_OPTIONS = ['TARGET=%s' % TARGET]  if TARGET else []
MAKE_CROSS_OPTIONS += ['TARGET_ARCH=%s' % TARGET_ARCH]  if TARGET_ARCH else []
```

### Fix-10 permission denied : tensorflow/tensorflow/lite/tools/make/download_dependencies.sh
```sudo nano setup.py```

```subprocess.check_call(DOWNLOAD_SCRIPT_PATH)=> subprocess.check_call((['bash', DOWNLOAD_SCRIPT_PATH]))```

### Fix-11 修改tools/make/target/rpi_makefile.inc 裡的設定跟 tools/pip_package/setup.py 一致 (避免發生 compile failed with "sorry, unimplemented: Thumb-1 hard-float VFP ABI"(ref))
```sudo nano setup.py```

```extra_compile_args=['--std=c++11', '-march=armv6', '-mfpu=vfp'],```

### Fix-12 修改版號
```sudo nano ../../../../tensorflow/tools/pip_package/setup.py```

```_VERSION = 'XXXX.2.2.0-rc2'```

