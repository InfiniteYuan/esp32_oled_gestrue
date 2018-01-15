# OLED SCREEN DEMO 说明文档
## 实现功能
- 温湿度数据采集
- 网络更新实时时间
- OLED 分页显示温湿度、实时时间
- 手势/ TouchPad 控制 OLED 显示页面上下翻页
- TouchPad 手动进入低功耗模式

实物图:
    <br>
    <img src="screen demo.jpg" width = "500" alt="screen demo" align=center />

-------

## 硬件组成
该 DEMO 使用 ESP32_Button_Module_V2 做为开发板，包含传感器：
- 接近/环境光线传感器 (APDS9960)
- OLED 显示屏 (SSD1306)
- 温度、湿度传感器 (HTS221)
- 两个 TouchPad 按钮

完整硬件原理图:[ESP32_BUTTON_MODULE_V2.pdf](ESP32_BUTTON_MODULE_V2_20170720A.pdf)

部分硬件原理图介绍：

显示屏、传感器电源开关控制原理图:
    <br>
    <img src="peripher switch.png" width = "500" alt="apds9960" align=center />

VDD33 为 DCDC (直流电源转换器) VOUT 3.3V 端, 做为 ESP32、外设、Flash 电源; VDD33_PeriP 为显示屏、温湿度传感器、手势传感器电源. 三极管 Si2301 用做电源开关, 控制 VDD33_PeriP 端电压, 默认情况下三极管 GATE 端保持高电平, 电源开关关闭; 通过控制 Power_ON 保持低电平以打开电源开关.

## 软件设计
- 使用 esp-iot-solution 开发工具包
- 基于 FreeRTOS 实时操作系统,多任务处理
- SNTP 协议获取实时时间
- 通过 TouchPad 进入低功耗模式
- 通过 TouchPad 触摸唤醒设备

-------

## 低功耗模式说明
###  低功耗模式硬件设计
为了使在低功耗模式下的功耗达到最低,我们做了这些处理:
- 显示屏、温湿度传感器、手势传感器电源开关控制
- 选用低功耗 DCDC (直流电源转换器), 静态电流约为 1uA
- TouchPad 低功耗管理

### TouchPad 周期
TouchPad 在工作时,会有两种状态: sleep、measure 循环交替进行,在正常的工作模式下我们把 sleep 时间设置的比较短,在进入低功耗模式后,我们把 sleep 时间设置的相对较长,以尽可能的降低功耗。

在进入低功耗模式前调用 touch_pad_set_meas_time(uint16_t sleep_cycle, uint16_t meas_cycle) 接口调整 TouchPad 的 sleep 与 measure 时间;

参数说明:

- sleep_cycle: sleep_cycle 决定了两次测量的间隔时间, 间隔时间 t_sleep = sleep_cycle / (RTC_SLOW_CLK frequency).

    可以使用 rtc_clk_slow_freq_get_hz() 接口获取 RTC_SLOW_CLK frequency 值.
- meas_cycle: meas_cycle 决定了测量时间, 测量时间 t_meas = meas_cycle / 8M, 最大测量时间为 0xffff / 8M = 8.19 ms.

### 低功耗模式使用
长按 TouchPad 按钮进入低功耗模式, 在低功耗模式下, TouchPad 采样频率会降至最低, 所以, 从低功耗模式下唤醒同样需要稍长时间的触摸 TouchPad 按钮. 低功耗模式下电流采样如下:

- 低功耗模式下 LDO VOUT 3.3V 端 (包含 ESP32、显示屏、传感器) 的电流采样图如下:
    <br>
    <img src="figure_3.3V.png" width = "500" alt="figure_3.3V" align=center />

> 注: 在低功耗模式下 LDO VOUT 3.3V 端的平均电流约为 30uA, 最大电流约为 1.6mA, 处在波峰时, TouchPad 位于 measure 状态.
<br>

- 低功耗模式下 LDO VIN 5V 端 (包含 ESP32、显示屏、传感器) 的电流采样图如下:
    <br>
    <img src="figure_5V.png" width = "500" alt="figure_5V" align=center />

> 注: 在低功耗模式下 LDO VIN 5V 端的平均电流约为 45uA, 最大电流约为 2.1mA.

## OLED SCREEN DEMO 编译与运行步骤:

### 前期准备

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
首先需要确保你的电脑上已经安装好 ESP32 工具链, 工具链安装请参考 ESP-IDF 中的 [README.md](https://github.com/espressif/esp-idf/blob/master/README.md).

### 获取 IoT Solution 项目代码

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
执行指令, 下载 iot-solution 项目仓库:

* 可以直接递归获取仓库代码, 这样将会自动初始化需要的所有子模块：

    ```
    git clone --recursive https://github.com/espressif/esp-iot-solution.git

    ```

* 也可以手动进行：

    ```
    git clone https://github.com/espressif/esp-iot-solution.git
    ```

* 然后切换到项目根目录执行指令, 下载本项目依赖的一些其它子模块:

    ```
    git submodule update --init --recursive
    ```

### 编译与运行

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
子模块代码下载完成后就可以对 Iot Solution 工程中的  oled_screen_module 进行编译和测试. 切换到  esp-iot-solution/examples/oled_screen_module 目录下

* 串口参数设置

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
执行以下指令进行编译配置, 如串口号和串口下载速度可以在  `Serial flasher config` 这一菜单选项中进行配置(如果不需配置, 可跳过这一步):

```
    cd YOUR_IOT_SOLUTION_PATH/examples/oled_screen_module
    make menuconfig
```

* 编译, 烧写与运行固件

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
执行下面指令，编译 oled_screen_module, 以下命令中的 flash 是下载命令, monitor 表示开启系统打印， 可根据实际情况选择添加:

```
    make flash monitor
```

> 注: 下载程序时, 如果无法自动开始下载，可以尝试手动进入下载模式。下载固件完成后，按reset键重新运行程序，可以查看串口打印
