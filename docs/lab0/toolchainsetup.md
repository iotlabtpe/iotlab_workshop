---
permalink: /lab0/toolchainsetup
---

# Setup toolchain for ESP32

## In the Cloud9 Terminal window install prerequisites

```bash
sudo yum install flex gperf
```
```bash
sudo pip install pyserial
```
```bash
wget https://github.com/Kitware/CMake/releases/download/v3.16.4/cmake-3.16.4-Linux-x86_64.sh
sudo sh cmake-3.16.4-Linux-x86_64.sh
```

## Download 64-bit version of Xtensa ESP32 toolchain:

```bash
cd ~/environment
wget https://dl.espressif.com/dl/xtensa-esp32-elf-linux64-1.22.0-80-g6c4433a-5.2.0.tar.gz
```

## Create *esp* directory and unzip the tar archive there:

```bash
mkdir ~/environment/esp
cd ~/environment/esp
tar xvfz ../xtensa-esp32-elf-linux64-1.22.0-80-g6c4433a-5.2.0.tar.gz
```

## Add toolchain path to *~/.bash_profile* PATH variable

```bash
PATH=$PATH:$HOME/.local/bin:$HOME/bin:$HOME/environment/esp/xtensa-esp32-elf/bin:$HOME/environment/cmake-3.16.4-Linux-x86_64/bin
```

## Re-evaluate ~/bash_profile

```bash
source ~/.bash_profile
```

## Next Step

## [Create your Thing]({{ "/lab0/iotcoresetup.html" | absolute_url }})
