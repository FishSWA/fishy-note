### 学习本教程的前置条件

本教程默认读者已经具备以下能力：

- 能使用 STM32CubeMX 生成基于 HAL 库的项目。
- 能使用 Keil 对单片机进行编程，并进行基本的断点调试。
- 能使用 HAL 库完成一些非常基础的小项目。

如果尚未具备上述能力，可以先查阅相关互联网教程，在掌握这些能力后再阅读本教程。
### 为什么抛弃 Keil？

目前网络上的大部分 STM32 开发教程都基于 Keil，这是一款设计风格非常陈旧的软件。其上世纪风格的用户界面使代码编写变得相当痛苦。此外，Keil 作为一个与 Windows 绑定的 IDE，不支持自由选择代码编辑器，限制了在 macOS 和 Linux 等其他平台上开发单片机固件的可能性。

同时，Keil 的项目配置散落在复杂的图形界面菜单中，这种设计对项目管理极为不利。当然，作为一个开箱即用的 IDE，Keil 的入门门槛较低，且网络上的相关教程丰富，用 Keil 入门是一个合理的选择。

在通过 Keil 入门单片机开发后，可以尝试更高效的开发框架。目前，GKD 战队的单片机开发流程采用以下工具链：

- 使用任意纯代码编辑器编写代码。
- 使用 Makefile 管理项目构建。
- 使用 OpenOCD 进行调试和烧录。
- 采用 arm-none-eabi-gcc 作为编译工具链。

该开发链可以在所有主流平台（如 Windows、macOS 和 Linux）上实现单片机开发，并且允许开发者自由选择喜爱的代码编辑器，从而打造一个更加灵活、高效的开发环境。
## 基于makefile的工程构建过程
#### 工程是如何被编译的
在使用keil这一类的IDE的时候，我们并不用关心工程代码具体是如何被编译成二进制文件（通常是.bin）再被烧录进单片机的，因为这些操作已经被keil包装成一个傻瓜式的按钮。但对于要抛弃臃肿IDE的我们，知晓并且理解这些底层构建过程是必要的。
现在，我们假设有一个简单的单片机项目，包含 `main.c`、`uart.c` 和 `spi.c` 三个文件，主函数位于 `main.c` 中，并调用了 `uart.c` 和 `spi.c` 中的代码。它们的编译过程如下图所示：
![[Pasted image 20241210173217.png]]
首先，像 `main.c` 这样的源文件会被 **编译器** 编译成二进制目标文件（例如 `main.o`）。每个源文件会生成一个对应的 `.o` 文件。在此例中，经过编译，我们得到了三个目标文件：`main.o`、`uart.o` 和 `spi.o`。
然后，**链接器** 会分析这些 `.o` 文件之间的引用关系，并将它们链接成最终的单片机固件，通常为 `.bin` 格式。
最后，**调试器** 会将生成的固件烧录到单片机中。

在这个过程中，我们将使用 `arm-none-eabi-gcc` 工具链。工具链是一个包含了构建单片机固件所需的全部工具（如编译器、链接器等）的工具集。
以下是手动调用工具链编译项目的过程：
``` bash
#编译.c文件为.o文件
arm-none-eabi-gcc -c main.c -o main.o -mcpu=cortex-m4 -mthumb -std=c99 -Wall -I./
arm-none-eabi-gcc -c uart.c -o uart.o -mcpu=cortex-m4 -mthumb -std=c99 -Wall -I./
arm-none-eabi-gcc -c spi.c -o spi.o -mcpu=cortex-m4 -mthumb -std=c99 -Wall -I./
#链接.o为.elf文件
arm-none-eabi-gcc -o firmware.elf main.o uart.o spi.o -mcpu=cortex-m4 -mthumb -T stm32f4.ld -nostdlib
#通过.elf文件生成.bin文件
arm-none-eabi-objcopy -O binary firmware.elf firmware.bin
```

#### 使用 Makefile 自动化构建
手动调用工具链进行编译是一个繁琐的过程，因此我们通常会使用构建工具来简化这一流程。通过构建工具，我们可以通过配置文件来定义和规范化整个构建过程。我们常用的构建工具是 **Make**，其配置文件为 **Makefile**。
通过 Makefile，我们可以完全控制从源代码到最终二进制文件（如 `.bin` 文件）的构建过程。在项目根目录下创建一个 **Makefile** 文件来定义编译和链接规则：
``` Makefile
# 设置编译器和工具链
CC = arm-none-eabi-gcc
AS = arm-none-eabi-as
LD = arm-none-eabi-ld
OBJCOPY = arm-none-eabi-objcopy

# 编译器和链接选项
CFLAGS = -mcpu=cortex-m4 -mthumb -std=c99 -Wall -I./
LDFLAGS = -mcpu=cortex-m4 -mthumb -T stm32f4.ld -nostdlib

# 定义源文件
SRC = main.c uart.c spi.c
OBJ = $(SRC:.c=.o)

# 默认目标
all: firmware.bin

# 编译源代码
%.o: %.c
	$(CC) -c $< -o $@ $(CFLAGS)

# 链接生成 ELF 文件
firmware.elf: $(OBJ)
	$(CC) -o $@ $^ $(LDFLAGS)

# 生成二进制文件
firmware.bin: output.elf
	$(OBJCOPY) -O binary $< $@

# 清理生成的文件
clean:
	rm -f *.o firmware.elf firmware.bin

```
随后，只需要在终端中输入：
``` bash
make  #单线程编译
```
或者
``` bash
make -j4   #多线程编译，"-j4"代表使用四个线程，"-j"代表使用全部线程
```
就可以得到最终的固件`firmware`.bin。

对于实际的单片机项目，`Makefile` 可能会变得非常复杂，因为可能涉及多个源文件、库文件以及复杂的依赖关系。但好消息是，STM32CubeMX 可以自动生成基于 Makefile 的项目，我们只需对生成的 `Makefile` 进行少量修改即可。
接下来，我们将一步步演示如何创建、编译和烧录一个基于 Makefile 构建的 STM32 单片机项目。

## 必要的软件安装教程
为了开发基于makefile的stm32工程，我们需要安装这些软件或者工具：
- vscode
- arm-none-eabi-gcc
- openocd
- make
并且值得注意的是，不建议使用**百度**去寻找软件资源和教程，因为百度现在只能说全是垃圾信息（
建议使用**google**或者**bing国际版**。这里的安装教程是针对windows的，macos或者linux可以很轻松的通过包管理器安装这些软件。
#### vscode安装
在本教程我们使用vscode作为**编辑器**，当然您也可以根据喜好选择其他的编辑器，比如clang或者nvim等等。
vscode的安装教程很容易找到，在此省略具体的安装教程。需要注意的是，也要一并安装可以帮助更好编辑项目的**makefile和c/c++扩展**
#### arm-none-eabi-gcc安装
- 在[arm_developer](https://developer.arm.com/downloads/-/gnu-rm)官网下载安装包
- 运行安装包安装，记录下安装的位置
- 将安装目录的/bin目录设置环境变量，具体的步骤如下：
![[Pasted image 20241210182008.png]]
![[Pasted image 20241210182141.png]]
或者在安装结束后勾选自动添加环境变量
![[Pasted image 20241210181848.png]]
- 终端输入`arm-none-eabi-gcc --version`查看是否安装成功
![[Pasted image 20241210182518.png]]
#### openocd安装
* 去[openocd官网](https://gnutoolchains.com/arm-eabi/openocd/)下载openocd的二进制文件，会是一个**压缩包**。
* 将压缩包解压到一个合适的文件夹，路径中不能包含中文。
* 把openocd文件夹中的`/bin`目录设置环境变量
* 终端输入`openocd -v`验证是否安装成功
![[Pasted image 20241210182552.png]]
#### make安装
- 在[gunwin官网](https://gnuwin32.sourceforge.net/packages/make.htm)下载适用于windows的make二进制包，即这个选项：
![[Pasted image 20241210183552.png]]
- 将压缩包解压到一个合适的文件夹，路径中不能包含中文。
- 把make文件夹中的`/bin`目录设置环境变量
- 终端输入`make -v`验证是否安装成功
![[Pasted image 20241210183702.png]]

## 构建第一个单片机项目
#### 使用cubemx生成makefile项目
makefile项目的配置过程与其他hal库工程没有任何区别，只需要在项目配置里做这一些修改：
- Toolchain选择makefile
![[Pasted image 20241210184619.png]]
- 并且建议勾选
![[Pasted image 20241210184651.png]]
- 之后就可以生成一个makefile项目，可以看到位于**根目录**的**Makefile文件**
![[Pasted image 20241210184809.png]]
#### 用vscode打开项目
- 直接使用vscode的`File -> Open Folder`打开cubemx生成的项目文件夹。
- 在`Terminal -> New 在Terminal`直接在vscode界面里打开终端
- 随后会看到这样的**现代界面**
![[Pasted image 20241210185150.png]]
#### 体验一下现代编辑器的功能
对比keil这种上世纪老东西（划掉），vscode有一些非常人性化的新功能，包括但不限于：
- 代码补全
![[Pasted image 20241210185517.png]]
- 鼠标悬停显示各种信息
![[Pasted image 20241210185535.png]]
![[Pasted image 20241210185610.png]]
- 按住control跳转到函数或者定义原型
![[Pasted image 20241210185637.png]]
![[Pasted image 20241210185650.png]]
- 模板填充
![[Pasted image 20241210185721.png]]
![[Pasted image 20241210185737.png]]
等等等等，可以后面慢慢自己去发现，总之就是非常好用（
#### 编译项目
简单写了一个点灯程序之后，我们可以直接在vscode编辑器中打开的终端输入
``` bash
make -j
```
等待片刻之后，便会出现编译报告：
![[Pasted image 20241210185909.png]]

随后，便可以在`/build`文件夹中找到编译出来的单片机固件
![[Pasted image 20241210185941.png]]

#### 烧录固件
在编译生成 `makefile_tutorial.bin` 固件后，需要使用 OpenOCD 将其烧录到单片机中。具体方法是，在终端中输入以下命令：
``` bash
openocd -f interface/stlink.cfg -f target/stm32g4x.cfg -c "program build/makefile_tutorial.bin 0x08000000 verify" -c "reset run" -c "exit"
```
下面是对该命令的逐步解析：
- `openocd -f interface/stlink.cfg -f target/stm32g4x.cfg`：启动 OpenOCD 调试器，使用 ST-Link 连接到 STM32 G4 系列单片机，并进入调试模式。
    - 对于其他硬件或芯片型号，需要修改 `stlink.cfg` 和 `stm32g4x.cfg` 配置文件的路径。
- `-c "program build/makefile_tutorial.bin 0x08000000 verify"`：执行烧录操作，具体含义如下：
    - **`program`**：表示将程序烧录到目标设备。
    - **`build/makefile_tutorial.bin`**：待烧录的程序文件路径，这里是编译后生成的 `.bin` 文件。
    - **`0x08000000`**：指定烧录的起始地址。对于 STM32 系列微控制器，通常程序从 `0x08000000` 地址开始烧录。
    - **`verify`**：烧录后进行验证，确保烧录的数据与目标文件一致。
- `-c "reset run"`：重启目标设备，并让它开始运行刚刚烧录的程序。
- `-c "exit"`：退出调试会话。

当然，也可以直接运行 `openocd -f interface/stlink.cfg -f target/stm32g4x.cfg` 启动一个交互式终端，然后依次输入 `program build/makefile_tutorial.bin 0x08000000 verify`、`reset run` 和 `exit`，也能实现相同的效果。
如果烧录成功，终端会显示相关信息，并且开发板上的 LED 灯会开始闪烁。
<TODO 补充图片>

#### 自动化编译以及烧录
每次改完代码都手动跑一遍make和openocd烧录是一个很麻烦的事情。我们可以在项目根目录下创建一个 `flash.sh` 脚本文件，内容如下：
``` bash
make -j
openocd -f interface/stlink.cfg -f target/stm32g4x.cfg -c "program build/makefile_tutorial.bin 0x08000000 verify" -c "reset run" -c "exit"
```
这样，每次修改完代码后，只需运行 `bash flash.sh`，就能自动编译并将更改烧录到单片机中。


#### 使用vscode调用openocd来可视化调试
对于需要**断点调试**的情况，虽然可以在终端中使用 OpenOCD 输入命令来调试，但这种方式不太直观且效率较低。下面介绍如何在 VSCode 中使用图形界面进行调试：
- 在vscode中，安装`Cortex-Debug`扩展插件
- 在 `.vscode` 目录下创建 `launch.json` 文件，用于配置调试器，内容如下：
``` json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Microcontroller",
            "type": "cortex-debug",
            "request": "launch",
            "servertype": "openocd",
            "cwd": "${workspaceFolder}",
            "executable": "./build/makefile_tutorial.elf",   //elf文件路径
            "configFiles": [
                "interface/stlink.cfg",                      //调试器
                "target/stm32g4x.cfg"                        //芯片配置文件
            ],
            "preLaunchTask": "build project"
        }
    ]
}
```
- 在 `.vscode` 目录下创建 `tasks.json`文件，用于自动编译，内容如下：
``` json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build project",
            "type": "shell",
            "command": "make -j",
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}
```
- 然后找到debug页面，选择刚刚创建的`Debug Microcontroller`项，再按下红框框柱的按钮即可开始调试。
![[Pasted image 20241210192425.png]]

<TODO 补充截图>

#### 结语
恭喜你，现在你可以成功把你的开发环境从这样：
![[Pasted image 20241210192725.png]]
变成了这样：
![[Pasted image 20241210192837.png]]

**欢迎来到21世纪（划掉）**

现在，所有项目的配置都集中在一个 `Makefile` 文件中，您不再需要在图形界面的多层菜单中查找配置项。

接下来，您可以开始学习 FreeRTOS 的基本使用，不需要深入了解，简单了解如何启动线程并在 STM32 单片机上进行多线程编程即可。我们将在第二篇教程中进一步探讨固件项目管理，教您如何编写低耦合的大型单片机项目，并使用 Git 进行代码管理。此外，我们还会介绍如何编写简单的项目文档（如 `README.md`），并讲解如何修改 `Makefile` 以支持使用 C++ 编写 STM32 固件。

---

TODO：
## 使用git进行版本管理
## 使用c++编程
## 大型单片机固件的项目结构