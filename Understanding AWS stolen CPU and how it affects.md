# Understanding AWS stolen CPU and how it affects your apps

> https://www.datadoghq.com/blog/understanding-aws-stolen-cpu-and-how-it-affects-your-apps/


----------


> **Stolen CPU** is a metric that’s often looked at but can be hard to understand. It implies some malevolent intent from your virtual neighbors. **In reality, it is a relative measure of the cycles a CPU should have been able to run but could not due to the `hypervisor` diverting cycles away from the instance.** From the point of view of your application, stolen CPU cycles are cycles that your application could have used.

- 和可能（malevolent intent）有恶意的 virtual neighbors 有关；
- Stolen CPU 代表本应该能够运行却（由于 hypervisor 的缘故）无法运行的 CPU 时间；

> Some of these `diverted cycles stem` from the **hypervisor enforcing a quota based on the `ECU` you have purchased**. In other cases, such as the one shown below, the amount of diverted or stolen CPU cycles varies over time, presumably due to **other instances on the same physical hardware also requesting CPU cycles from the underlying hardware.**

导致 Stolen CPU 的两种情况：

- hypervisor 基于你所购买的 `ECU` 进行 quota 限定；
- 位于同一台物理硬件上的其他 instances 同时在请求 CPU cycles 导致；

## Seeing AWS stolen CPU

> Here is a graph of CPU usage on a host with `stolen` CPU in **yellow**. **Light blue** denotes “`idle`” or available cycles, **purple** denotes “`user`” or cycles spent executing application code, and **dark blue** denotes “`system`” or cycles spent executing kernel code. In this case we can see that the amount of “**stolen**” CPU is clearly visible.

![](https://datadog-prod.imgix.net/img/blog/understanding-aws-stolen-cpu-and-how-it-affects-your-apps/aws-stolen-cpu-seeing-it.png)

查看 CPU 各项的内容；


## Are noisy neighbors stealing from you?

> Let us now find out whether **other tenants** on the same machine can affect the amount of stolen CPU. The following graphs show the amount of **stolen CPU** (top) and the amount of **idle CPU** (bottom), both measured in percent of all CPU cycles for the same machine at the same time. CPU usage is sampled from the operating system of the instance every 15 seconds.

![](https://datadog-prod.imgix.net/img/blog/understanding-aws-stolen-cpu-and-how-it-affects-your-apps/aws-stolen-system-cpu-metric-show.png)

> The interesting part occurs when `idle` CPU reaches zero. All cycles have been accounted for, either doing useful work (not represented here) or being taken away by the hypervisor (the stolen CPU graph).

当 idle 为 0 时：

- 要么 CPU 被用于完成 useful work
- 要么 CPU 被 hypervisor 给 steal 了

> Notice that at in the highlighted sections, the amount of stolen CPU differs from **30% (red)** to **50% (purple)**. If the **ECU quota** were the only thing causing CPU to be stolen, we should expect the stolen CPU to be equal at these two points in time.

这里反证了：如果 ECU quota 是唯一导致 stolen CPU 的前提条件的话，那么应该有一致的数值输出，然并卵；

## How AWS stolen CPU affects your applications

> **Stolen CPU is particularly problematic for CPU intensive applications** which will more frequently attempt to run at or near the threshold for ECUs you’ve purchased. These applications use more cycles and have idle CPU at zero more often.

对于 CPU 密集型应用更容易收到 Stolen CPU 问题影响；

> **If the hypervisor is simply enforcing a quota based on ECUs you’ve purchased, you would expect consistent values of stolen CPU when idle CPU approaches zero.** If this is case, the limitation set by your instance quota would slow down your application in a consistent and predictable way. You are getting the ECUs you purchased and can increase application performance by upgrading your instance.

针对非实际情况的讨论（反证）；

> **If instead you’re seeing varying values for stolen CPU it’s likely that you and other tenants are requesting more CPU cycles than are available on your hardware.** Because you have no insight into other tenants CPU usage behavior, it can be difficult or impossible to predict the amount of stolen CPU you will see as your application approaches idle CPU of zero. In this case, you are not consistently getting the ECUs you purchased and the changing CPU availability means you can’t predictably forecast performance for your application.

针对实际情况的讨论：

- 无法实际看到其他租户（tenants）的 CPU 使用情况；
- 当你的应用看到 idle CPU 接近 0 时 ，很难或无法预测 stolen CPU 准确数量；
- 因此无法真正得到与购买的 ECUs 匹配的算力，以及即使升级也无法准确判定性能改善的量；

##How to resolve EBS performance issues from AWS stolen CPU

- Buy more powerful EC2 instances
- Baseline your application compute needs
- Profile your app on an EC2 instance before finalizing a deploy decision
- Re-deploy your application in another instance
