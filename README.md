difuze: Fuzzer for Linux Kernel Drivers
===================

[![License](https://img.shields.io/github/license/angr/angr.svg)](https://github.com/ucsb-seclab/difuze/blob/master/LICENSE)

This repo contains all the sources (including setup scripts), you need to get `difuze` up and running.
### Tested on
Ubuntu >= 14.04.5 LTS

As explained in our [paper](https://acmccs.github.io/papers/p2123-corinaA.pdf), There are two main components of `difuze`: **Interface Recovery** and **Fuzzing Engine**

## 1. Interface Recovery
The Interface recovery mechanism is based on LLVM analysis passes. Every step of interface recovery are written as  individual passes. Follow the below instructions on how to get the *Interface Recovery* up and running.

### 1.1 Setup
This step takes care of installing LLVM and `c2xml`:

First, make sure that you have libxml (required for c2xml):
```
sudo apt-get install libxml2-dev
```

Next, We have created a single script, which downloads and builds all the required tools.
```
cd helper_scripts
python setup_difuze.py --help
usage: setup_difuze.py [-h] [-b TARGET_BRANCH] [-o OUTPUT_FOLDER]

optional arguments:
  -h, --help        show this help message and exit
  -b TARGET_BRANCH  Branch (i.e. version) of the LLVM to setup. Default:
                    release_38 e.g., release_38
  -o OUTPUT_FOLDER  Folder where everything needs to be setup.

```
Example:
```
python setup_difuze.py -o difuze_deps
```
To complete the setup you also need modifications to your local `PATH` environment variable. The setup script will give you exact changes you need to do.
### 1.2 Building
This depends on the successful completion of [Setup](#11-setup).
We have a single script that builds everything, you are welcome.
```
cd InterfaceHandlers
./build.sh
```
### 1.3 Running
This depends on the successful completion of [Build](#12-building).
To run the Interface Recovery components on kernel drivers, we need to first the drivers into llvm bitcode.
#### 1.3.1 Building kernel
First, we need to have a buildable kernel. Which means you should be able to compile the kernel using regular build setup. i.e., `make`.
We first capture the output of `make` command, from this output we extract the exact compilation command.
##### 1.3.1.1 Generating output of `make` (or `makeout.txt`)

Just pass `V=1` and redirect the output to the file.
Example:
```
make V=1 O=out ARCH=arm64 > makeout.txt 2>&1
```
**NOTE: DO NOT USE MULTIPLE PROCESSES** i.e., `-j`. Running in multi-processing mode will mess up the output file as multiple process try to write to the output file.

That's it. Next, in the following step our script takes the generated `makeout.txt` and run the Interface Recovery on all the recognized drivers.
#### 1.3.2 Running Interface Recovery analysis
All the various steps of Interface Recovery are wrapped in a single script `helper_scripts/run_all.py`
How to run:
```
cd helper_scripts
python run_all.py --help

usage: run_all.py [-h] [-l LLVM_BC_OUT] [-a CHIPSET_NUM] [-m MAKEOUT]
                  [-g COMPILER_NAME] [-n ARCH_NUM] [-o OUT]
                  [-k KERNEL_SRC_DIR] [-skb] [-skl] [-skp] [-skP] [-ske]
                  [-skI] [-ski] [-skv] [-skd] [-f IOCTL_FINDER_OUT]

optional arguments:
  -h, --help           show this help message and exit
  -l LLVM_BC_OUT       Destination directory where all the generated bitcode
                       files should be stored.
  -a CHIPSET_NUM       Chipset number. Valid chipset numbers are:
                       1(mediatek)|2(qualcomm)|3(huawei)|4(samsung)
  -m MAKEOUT           Path to the makeout.txt file.
  -g COMPILER_NAME     Name of the compiler used in the makeout.txt, This is
                       needed to filter out compilation commands. Ex: aarch64
                       -linux-android-gcc
  -n ARCH_NUM          Destination architecture, 32 bit (1) or 64 bit (2).
  -o OUT               Path to the out folder. This is the folder, which could
                       be used as output directory during compiling some
                       kernels.
  -k KERNEL_SRC_DIR    Base directory of the kernel sources.
  -skb                 Skip LLVM Build (default: not skipped).
  -skl                 Skip Dr Linker (default: not skipped).
  -skp                 Skip Parsing Headers (default: not skipped).
  -skP                 Skip Generating Preprocessed files (default: not
                       skipped).
  -ske                 Skip Entry point identification (default: not skipped).
  -skI                 Skip Generate Includes (default: not skipped).
  -ski                 Skip IoctlCmdParser run (default: not skipped).
  -skv                 Skip V4L2 ioctl processing (default: not skipped).
  -skd                 Skip Device name finder (default: not skipped).
  -f IOCTL_FINDER_OUT  Path to the output folder where the ioctl command
                       finder output should be stored.
```
The script builds, links and runs Interface Recovery on all the recognized drivers, as such it might take **considerable time(45 min-90 min)**. 

The above script performs following tasks in a multiprocessor mode to make use of all CPU cores:
##### 1.3.2.1 LLVM Build 
* Enabled by default.

All the bitcode files generated will be placed in the folder provided to the argument `-l`.
This step takes considerable time, depending on the number of cores you have. 
So, if you had already done this step, You can skip this step by passing `-skb`. 
##### 1.3.2.2 Linking all driver bitcode files in s consolidated bitcode file.
* Enabled by default

This performs linking, it goes through all the bitcode files and identifies the related bitcode files that need to be linked and links them (using `llvm-link`) in to a consolidated bitcode file (which will be stored along side corresponding bitcode file).

Similar to the above step, you can skip this step by passing `-skl`.
##### 1.3.2.3 Parsing headers to identify entry function fields.
* Enabled by default.

This step looks for the entry point declarations in the header files and stores their configuration in the file: `hdr_file_config.txt` under LLVM build directory.

To skip: `-skp`
##### 1.3.2.4 Identify entry points in all the consolidated bitcode files.
* Enabled by default

This step identifies all the entry points across all the driver consolidated bitcode files.
The output will be stored in file: `entry_point_out.txt` under LLVM build directory.

Example of contents in the file `entry_point_out.txt`:
```
IOCTL:msm_lsm_ioctl:/home/difuze/kernels/pixel/msm/sound/soc/msm/qdsp6v2/msm-lsm-client.c:msm_lsm_ioctl.txt:/home/difuze/pixel/llvm_out/sound/soc/msm/qdsp6v2/llvm_link_final/final_to_check.bc
IOCTL:msm_pcm_ioctl:/home/difuze/kernels/pixel/msm/sound/soc/msm/qdsp6v2/msm-pcm-lpa-v2.c:msm_pcm_ioctl.txt:/home/difuze/pixel/llvm_out/sound/soc/msm/qdsp6v2/llvm_link_final/final_to_check.bc

```
To skip: `-ske`
##### 1.3.2.5 Run Ioctl Cmd Finder on all the identified entry points.
* Enabled by default.

This step will run the main Interface Recovery component (`IoctlCmdParser`) on all the entry points in the file `entry_point_out.txt`. The output for each entry point will be stored in the folder provided for option `-f`.

To skip: `-ski`

### 1.4 Example:
Now, we will show an example from the point where you have kernel sources to the point of getting Interface Recovery results.

We have uploaded a mediatek kernel [33.2.A.3.123.tar.bz2](https://drive.google.com/open?id=0B4XwT5D6qkNmLXdNTk93MjU3SWM). 
First download and extract the above file.

Lets say you extracted the above file in a folder called: `~/mediatek_kernel`

#### 1.4.1 Building
```
cd ~/mediatek_kernel
source ./env.sh
cd kernel-3.18
# the following step may not be needed depending on the kernel
mkdir out
make O=out ARCH=arm64 tubads_defconfig
# this following command copies all the compilation commands to makeout.txt
make V=1 -j8 O=out ARCH=arm64 > makeout.txt 2>&1
```
#### 1.4.2 Running Interface Recovery
```
cd <repo_path>/helper_scripts

python run_all.py -l ~/mediatek_kernel/llvm_bitcode_out -a 1 -m ~/mediatek_kernel/kernel-3.18/makeout.txt -g aarch64-linux-android-gcc -n 2 -o ~/mediatek_kernel/kernel-3.18/out -k ~/mediatek_kernel/kernel-3.18 -f ~/mediatek_kernel/ioctl_finder_out
```
The above command takes quite **some time (30 min - 1hr)**.

#### 1.4.3 Understanding the output
First, all the analysis results will be in the folder: **`~/mediatek_kernel/ioctl_finder_out` (argument given to the option `-f`)**, for each entry point a `.txt` file will be created, which contains all the information about the recovered interface.

If you are interested in information about just the interface and don't care about anything else, We recommend you use the [`parse_interface_output.py`](https://github.com/ucsb-seclab/difuze/blob/master/helper_scripts/parse_interface_output.py) script. This script converts the crazy output of Interface Recovery pass into nice json files with a clean and consistent format.

```
cd <repo_path>/helper_scripts
python parse_interface_output.py <ioctl_finder_out_dir> <output_directory_for_json_files>
```
Here `<ioctl_finder_out_dir>` should be same as the folder you provided to the `-f` option and `<output_directory_for_json_files>` is the folder where the json files should be created.

You can use the corresponding json files for the interface recovery of the corresponding ioctl.

#### 1.4.4 Things to note:
##### 1.4.4.1 Value for option `-g`
To provide value for option `-g` you need to know the name of the `*-gcc` binary used to compile the kernel.
An easy way to know this would be to `grep` for `gcc` in `makeout.txt` and you will see compiler commands from which you can know the `*-gcc` binary name.

For our example above, if you do `grep gcc makeout.txt` for the example build, you will see lot of lines like below:
```
aarch64-linux-android-gcc -Wp,-MD,fs/jbd2/.transaction.o.d  -nostdinc -isystem ...
```
So, the value for `-g` should be `aarch64-linux-android-gcc`. 

If the kernel to be built is 32-bit then the binary most likely will be `arm-eabi-gcc`

For Qualcomm (or msm) chipsets, you may see `*gcc-wrapper.py` instead of `*.gcc`, in which case you should provide the `*gcc-wrapper.py`.

##### 1.4.4.2 Value for option `-a`
Depeding on the chipset type, you need to provide corresponding number.

##### 1.4.4.3 Value for option `-o`
This is the path of the folder provided to the option `O=` for `make` command during kernel build.

Not all kernels need a separate out path. You may build kernel by not providing an option `O`, in which case you SHOULD NOT provide value for that option while running `run_all.py`.

### 1.5 Post Processing
Before we can begin fuzzing we need to process the output a bit with our very much research quality (sorry) parsers.

These are found [here](helper_scripts/post_processing). The main script to run will be `run_all.py`:
```
$ python run_all.py --help
usage: run_all.py [-h] -f F -o O [-n {manual,auto,hybrid}] [-m M]

run_all options

optional arguments:
  -h, --help            show this help message and exit
  -f F                  Filename of the ioctl analysis output OR the entire
                        output directory created by the system
  -o O                  Output directory to store the results. If this
                        directory does not exist it will be created
  -n {manual,auto,hybrid}
                        Specify devname options. You can choose manual
                        (specify every name manually), auto (skip anything that
                        we don't identify a name for), or hybrid (if we
                        detected a name, we use it, else we ask the user)
  -m M                  Enable multi-device output most ioctls only have one
                        applicable device node, but some may have multiple. (0
                        to disable)
```
You'll want to pass `-f` the output directory of the ioctl analysis e.g. `~/mediatek_kernel/ioctl_finder_out`.

`-o` Is where you where to store the post-processed results. These will be easily digestible XML files (jpits).

`-n` Specifies the system to what degree you want to rely on our device name recovery.
If you don't want to do any work/name hunting, you can specify `auto`.
This of course comes at the cost of skipping any device for which we don't recover a name. If you want to be paranoid and not trust any of our recovery efforts (totally reasonable) you can use the `manual` option to name every single device yourself.
`hybrid` then is a combination of both -- we will name the device for you when we can, and fall back to you when we've failed.

`-m` Sometimes ioctls can correspond to more than one device (this is common with v4l2/subdev ioctls for example). Support for this in enabled by default, but it requires user interaction to specify the numberof devices for each device. If this is too annoying for you, you can disable the prompt by passing `-m 0` (we will assume a single device for each ioctl).

After running, you should have, in your out folder, a folder for each ioctl.

## 2 Fuzzing

### 2.1 Mango Fuzz
MangoFuzz is our simple prototype fuzzer and is based off of Peach (specifically [MozPeach](https://github.com/MozillaSecurity/peach)).

It's not a particularly sophisticated fuzzer but it does find bugs.
It was also built to be easily expandable.
There are 2 components to this fuzzer, the fuzz engine and the executor.
The executor can be found [here](MangoFuzz/executor), and the fuzz engine can be found [here](MangoFuzz/fuzzer).

#### 2.1.1 Executor
The executor runs on the phone, listening for data that the fuzz engine will send to it.

Simply compile it for your phones architecture, `adb push` it on to the phone, and execute with the port you want it to listen on!

#### 2.1.2 Fuzz Engine
Interfacing with MangoFuzz is fairly simple. You'll want an `Engine` object and a `Parser` object, which you'll feed your engine into.
From here, you parse jpits with your Parser, and then run the Engine. Easy!
We've provided some simple run scripts to get you started.

To run against specific drivers you can use `runner.py` on one of the ioctl folders in the output directory (created by our post processing scripts).

e.g. `./runner.py -f honor8/out/chb -num 1000`. This tells MangoFuzz to run for 1000 iterations against all ioctl command value pairs pertaining to the `chb` ioctl/driver.

If instead we want to run against an entire device (phone), you can use `dev_runner.py`. e.g. `./dev_runner.py -f honor8/out -num 100`.
This will continue looping over the driver files, randomly switching between them for 100 iterations each.

Note that before the fuzz engine can communicate with the phone, you'll need to use ADB to set up port forwarding e.g. `adb forward tcp:2022 tcp:2022`
