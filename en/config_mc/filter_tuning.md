# 前言

滤波器可用于权衡控制延迟和噪声过滤的效果，这两者都会影响飞行性能以及电机的健康状况。本主题提供有关控制延迟和PX4滤波器调优的概述。滤波器可用于权衡控制延迟和噪声过滤的效果，这两者都会影响飞行性能以及电机的健康状况。

> 注意事项
> 在调整滤波器之前，您应该先进行基本的PID调整 [( Basic MC PID tuning)](https://docs.px4.io/main/en/config_mc/pid_tuning_guide_multicopter_basic.html)。车辆需要进行欠调（P和D增益应设置得过低），这样控制器就不会产生可能被解释为噪音的振荡（默认增益可能足够好）。

# 控制延迟
控制延迟是从物理干扰产生到主体电机对变化做出反应之间的延时。

> 注意事项
> 降低延迟要求调试者增加速率P增益，从而提高飞行性能。即使是一毫秒的延迟差异也会产生重要影响。

以下因素影响控制延迟：

 - 柔软的机身以及减震器的安装会增加延迟（它们起到了滤波器的作用）。 
 - 软件和传感器芯片上的低通滤波器通过增加延迟来改善噪声滤波。
 - PX4内部程序：传感器信号需要在驱动程序中读取，然后通过控制器传递到输出驱动器。
 - 陀螺仪的最大速率（通过[ IMU_GYRO_RATEMAX ](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#IMU_GYRO_RATEMAX)配置）。较高的速率可以减少延迟，但会增加计算量并影响其他程序进程。仅建议在使用STM32H7处理器或更新版本的控制器时选择4kHz或更高的速率（2 kHz的值已接近较低性能处理器的极限）。
- IO芯片（MAIN引脚）与使用AUX引脚相比，延迟约为5.4毫秒（Pixracer或Omnibus F4不适用此规则，但Pixhawk适用）。为避免IO延迟，应将电机连接到AUX引脚。
- PWM输出信号：优先启用[ Dshot ](https://docs.px4.io/main/en/peripherals/dshot.html)以减少延迟（如果不支持DShot，则使用One-Shot）。在[执行器配置](https://docs.px4.io/main/en/config/actuators.html)期间，协议是为一组输出选择的。

以下是低通滤波器对延迟的影响
# 滤波器
这是PX4控制器的滤波通道：

 - 陀螺仪传感器片上数字低通滤波器。所有可以禁用的芯片上都已禁用此功能（否则则将截止频率设置为芯片的最高频率）。
 - 陀螺仪传感器数据上的陷波滤波器，用于过滤窄带噪声。例如转子叶片的频率振动。可以使用[ IMU_GYRO_NF0_BW ](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#IMU_GYRO_NF0_BW)和[ IMU_GYRO_NF0_FRQ ](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#IMU_GYRO_NF0_FRQ)参数来配置此滤波器。
 - 陀螺仪传感器数据上的低通滤波器。可以使用[ IMU_GYRO_CUTOFF ](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#IMU_GYRO_CUTOFF)参数进行配置。

> 注意
>采样和滤波始终以全速原始传感器速率进行（通常为8kHz，取决于IMU）。

 - 独立的D-低通滤波器。D-低通滤波器对噪声最敏感，而轻微增加的延迟不会对性能产生负面影响。因此D项具有可单独配置的低通滤波器， [IMU_DGYRO_CUTOFF](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#MOT_SLEW_MAX)。
 - 电机输出的斜率滤波器 [（MOT_SLEW_MAX）](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#MOT_SLEW_MAX)通常不被使用。

为了降低控制延迟，我们希望增加低通滤波器的截止频率。增加**IMU_GYRO_CUTOFF** 的延迟效果如下所示：	
| 截止频率（Hz） |  延迟近似值（ms）|
|--|--|
| 30  | 8 |
| 60 | 3.8 |
|120|1.9|

然而这个操作的结果是在增加 *IMU_GYRO_CUTOFF* 参数的同时馈送更多噪声信号到电机中。对电机的噪声会产生以下后果：

- 电机和电调发热导致损坏。
- 电机速率频繁的动态调整导致的飞行时间减少。
- 可见的随机小幅度震颤。

具有明显的低频噪声峰值（ 例如由于转子叶片通过频率的谐波 ）的设置可以通过使用陷波滤波器在将信号传递到低通滤波器之前清除信号来调整（这些谐波对电机的影响与其他噪声来源相似）。如果没有陷波滤波器，则必须将低通滤波器截止频率设置得更低（增加延迟），以避免将此噪声传递给电机。

> 注意
>只提供一个陷波滤波器。具有多个低频噪声峰值的机体通常使用陷波滤波器清除第一个峰值，并使用低通滤波器清除随后的峰值。

最佳的滤波器设置取决于飞行器。默认值是保守的，可以适用于质量较低的设置。
# 滤波器调参
首先确保已启用高速日志记录配置文件（[ SDLOG_PROFILE ](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#SDLOG_PROFILE)参数）。[飞行日志 ](https://docs.px4.io/main/en/getting_started/flight_reporting.html)将显示滚转、俯仰和偏航控制的FFT图。

  >注意事项
  >不要尝试通过调整滤波器来修复遭受高振动的飞行器！而应该修复飞行器的硬件配置。
确认 PID 参数，其中的参数 D 过高会导致肉眼可见的小幅度振动。

调整滤波最好通过审查飞行日志来完成。您可以连续进行多次飞行，使用不同的参数，然后检查所有日志，但请确保在两次飞行之间解除配置，以创建单独的日志文件。
无人机的飞行可以是在 [手动/自稳模式](https://docs.px4.io/main/zh/flight_modes_mc/manual_stabilized.html)下简单地悬停，同时进行一些滚动和俯仰操作，向各个方向进行，并增加一些节流阀周期。总时长不需要超过30秒。为了更好地比较，所有测试中的无人机操作动作应保持相似。
首先，通过 [ IMU_GYRO_CUTOFF ](https://docs.px4.io/main/zh/advanced_config/parameter_reference.html#IMU_GYRO_CUTOFF)以10 Hz为步长来调整陀螺仪滤波器。同时将D-滤波器值设置为较低的值（[ IMU_GYRO_CUTOFF ](https://docs.px4.io/main/zh/advanced_config/parameter_reference.html#IMU_GYRO_CUTOFF) = 30）。将日志上传到[ Flight Review ](https://logs.px4.io/) 并比较执行器控制的FFT图。将截止频率设置为在噪声开始明显增加之前（在60 Hz左右及以上频率）的一个值。
然后以同样的方式调整 D 滤波器（**IMU_DGYRO_CUTOFF**）。请注意，如果**IMU_GYRO_CUTOFF**和**IMU_DGYRO_CUTOFF**之间的差异过大，可能会对性能产生负面影响（但差异必须明显，例如D = 15，陀螺仪= 80）。

以下是三个不同**IMU_DGYRO_CUTOFF**滤波器值（40Hz，70Hz，90Hz）的示例。在频率为90Hz时，总体噪音水平开始增加（尤其是横滚），因此70Hz的截止频率是一个安全的设置。
![执行器控制FFT](https://img-blog.csdnimg.cn/direct/53683e8703cf4e28be935130154e42c2.png)
![执行器控制FFT](https://img-blog.csdnimg.cn/direct/908dc264ac4c4b2d975051eda07b3385.png)
![执行器控制FFT](https://img-blog.csdnimg.cn/direct/e0648ae99d814da895e3d654e22c2191.png)

> 注意
>  无法在不同的飞机之间进行比较，因为Y轴刻度可能不同。在同一架飞机上是一致的，与飞行时长无关。

如果飞行情况反馈图显示出显著的低频尖峰，就可以使用陷波滤波器来去除它。如下图所示，在这种情况下您可以使用以下设置：[IMU_GYRO_NF0_FRQ=32 ](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#IMU_GYRO_NF0_FRQ) 和[IMU_GYRO_NF0_BW=5 ](https://docs.px4.io/main/en/advanced_config/parameter_reference.html#IMU_GYRO_NF0_BW)（此尖峰比通常更窄）。低通滤波器和陷波滤波器可以独立调节（即您不需要在收集数据之前设置陷波滤波器来调节低通滤波器）。
![Actuator Control FFT](https://img-blog.csdnimg.cn/direct/8b8d1d45cf3e42e0ac39ed4444a83212.png)
 # 注意事项

 1.  可接受的延迟取决于车辆的大小和期望。FPV竞速无人机（ 穿越机 ）通常调校以实现最小延迟（如**IMU_GYRO_CUTOFF**约为120，**IMU_DGYRO_CUTOFF**约为50到80）。对于较大的车辆，延迟就不那么关键了，**IMU_GYRO_CUTOFF**约为80是较为合适的。
2.   您可以从更高的**IMU_GYRO_CUTOFF**值（例如100Hz）开始调校，因为默认的**IMU_GYRO_CUTOFF**调校值设置得非常低（30Hz）。但是需要注意以下风险：
         - 不要连续飞行超过20-30秒 
         - 检查电机是否过热 
         - 留意异常声音和过多噪音的症状。

