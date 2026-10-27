# 瞄准作业

[中文](README.md) · [English](README.en.md)

这是两条可选路线之一。**检测与瞄准任选其一即可，不要求同时完成。**
阅读[主 README](../README.md)了解云台教具联调的故事背景。本路线使用仿真器，
不需要先完成检测路线，也不需要真实云台、相机或电机。

## 任务

机器人会平移和自旋，你需要瞄准装甲板并决定何时开火。仿真器提供精确的目标模型，
但观测描述的是过去的状态。你需要预测弹丸到达时装甲板的位置。

只需修改 [src/lib.rs](src/lib.rs)，其中包含 `aim` 和你会使用的类型。
绘图接口位于 [src/ui.rs](src/ui.rs)。请勿修改 `src/sim/`，其中是评分、界面和 MCAP 功能。
模板默认不开火，并在 `aim` 中提供可以自行修改的装甲板绘图和命中计数示例。

## 参考效果

https://github.com/user-attachments/assets/2a08b028-4c0a-400b-9102-b1cc2989b7e8

[观看参考视频](assets/tracking-reference.mp4)（4K、60 FPS、22 秒）。
视频展示了完整实现跟踪并射击平移、自旋目标的效果。绿色轮廓表示预计命中时的装甲板姿态，
橙色曲线表示规划弹道。视频供实现时对照，仓库中不包含完整瞄准解答。

## 环境与硬件

- 安装 [Rust / rustup](https://www.rust-lang.org/tools/install)，Rust 版本至少为 **1.98.1**。
  首次构建需要联网下载依赖。本次验证使用 rustc 1.99.1。
- 需要本机 C/C++ 编译器和链接器。macOS 使用 Xcode Command Line Tools；
  Windows 使用 MSVC C++ Build Tools；Linux 需要编译工具和窗口后端的开发库。
- 3D 界面需要桌面会话，以及支持 wgpu 的图形驱动，例如 Metal、Vulkan 或 DX12。
  不需要 CUDA 或独立显卡；本次使用 Apple 集成 GPU 验证。
- 建议电脑有 **8 GB 或以上内存，16 GB 更宽裕**，预留 **5 GB 磁盘空间**用于依赖和构建。
  这是使用建议，不是已验证的最低配置。4K 视频导出、长录制和更多并行编译会增加资源需求。
- 只有导出 MP4 才需要 `ffmpeg`。普通界面、离线评测和 MCAP 不需要它。

内存较少时，可以限制为两个编译任务，并使用与下面测量相同的 release 构建：

```sh
cargo build --release --locked --jobs 2
cargo run --release --locked --jobs 2 -- --config scenarios/moving.json
```

没有可用图形环境时，可以不编译界面，离线运行并输出 MCAP。文件名需要是尚不存在的新路径：

```sh
cargo run --no-default-features --locked --jobs 2 -- --offline --record target/headless.mcap
```

### 本次实测

测量于 2026-10-28，机器为 Apple M5 Max、128 GiB 统一内存、macOS 27.0.1（arm64），
工具链为 rustc 1.99.1。运行的是默认学生模板，没有加入完整瞄准解答。

| 项目 | 实测值 | 条件 |
| --- | --- | --- |
| release 构建峰值 | 1,198 MiB（约 1.17 GiB） | `cargo build --release --locked --jobs 2` 的进程树 RSS 之和 |
| 3D 界面峰值 | 161 MiB RSS | release 二进制、`scenarios/moving.json`、默认暂停状态打开 35 秒 |
| 离线仿真峰值 | 15 MiB RSS | 同一 release 二进制，完成默认 20 秒任务并将完整过程写入 MCAP |

构建通过本机 mbx 执行，部分依赖命中缓存，**不是无缓存全量构建的峰值**。
构建进程树每 0.1 秒采样一次，不包括监测程序或独立缓存服务；采样可能漏掉短暂峰值。
运行峰值使用进程退出时的 macOS `ru_maxrss`，界面使用默认窗口及 1920 × 1100 场景缓冲。
界面测量是暂停状态的基线；离线测量运行了完整仿真，但没有图形渲染。
RSS 不等于完整 GPU/驱动内存，也不代表整机内存需求。其他操作系统、无缓存构建、
debug 构建和 4K 导出的资源需求没有在本次测量中验证。

## 运行

在 `aiming` 目录执行：

```sh
cargo run --locked
cargo run --locked -- --config scenarios/stationary.json
cargo run --locked -- --config scenarios/spinning.json
cargo run --locked -- --config scenarios/moving.json
```

先做静止目标，再做自旋，最后处理平移与自旋同时发生的情况。
这个 crate 独立编译，不依赖 OpenCV 或相机 SDK；运行器始终调用你实现的 `aim`。

## 需要实现的接口

```rust
pub fn aim(input: &Input, debug: &mut Gizmos) -> Option<Command>
```

仿真器每 10 ms 调用一次 `aim`。`Input` 包含当前仿真时间、延迟送达的观测、已知几何尺寸，
以及已经结算的射击计数。每次观测包含采集时间和
`x_m, y_m, theta_rad, vx_m_s, vy_m_s, omega_rad_s`。
最初的观测延迟尚未结束时没有观测；接口不会提供目标当前时刻的真值。

返回 `Some(Command { yaw_rad, pitch_rad, fire })`。`fire: false` 表示只瞄准不开火；
`None` 表示不开火且不提交瞄准指令。被接受的射击指令会锁定角度，在发射延迟后沿固定弹道飞行，
后续指令不能改变已经发射的弹丸。持续请求开火会受到射速限制。
命中率按 `hits / (hits + misses)` 计算，尚在飞行的弹丸未结算。

你需要处理缺失观测和不可达弹道，补偿延迟，预测平移和旋转，并选择能够从正面命中的装甲板。

## 固定物理参数

所有发射物理量和时序都在 [src/lib.rs](src/lib.rs) 的 `physics` 中定义为常量。
`Input` 和场景 JSON 不含可配置物理参数。弹道工具、绘图、界面和评分统一使用这些常量：

| 常量 | 数值 |
| --- | --- |
| `MUZZLE_M` | `[0.0, 0.0, 0.3]` m |
| `MUZZLE_SPEED_M_S` | 25 m/s |
| `GRAVITY_M_S2` | 9.81 m/s² |
| `OBSERVATION_DELAY_S` | 0.05 s |
| `LAUNCH_DELAY_S` | 0.02 s |
| `FIRE_PERIOD_S` | 0.25 s |
| `MAX_FLIGHT_S` | 发射后 2 s |
| `CONTROL_PERIOD_S` | 0.01 s |

调用工具函数时不需要额外传入物理参数。

## 第一步：确定延迟

先检查 `input.observation` 是否存在。观测在采集时刻是精确的，但并不是当前状态。
用 `input.now_s - observation.time_s` 求出观测年龄，再考虑指令被接受到实际发射之间的
`physics::LAUNCH_DELAY_S`。

目标在这两段延迟和弹丸飞行期间都会运动。把观测年龄、发射延迟和飞行时间分别处理，
避免漏算或重复相加。缺少观测时不开火；本题不要求跟踪器或状态估计。

## 第二步：选择装甲板

先选装甲板，再解弹道。可以先用距离除以弹速粗估飞行时间，预测
“观测年龄 + 发射延迟 + 粗估飞行时间”后的四块装甲板姿态，
比较外法线与炮口来向，选择朝向来弹的装甲板。现在最近的装甲板可能在命中时转到背面。

第四步迭代时固定选中的装甲板索引，避免为每个候选都完整求解弹道，或在迭代中反复换板。
初选只是近似，发射前仍需检查最终朝向；所选装甲板不可达时不开火。

装甲板顶部向内倾斜 15°，请使用完整的三维法线。命中必须从正面进入
0.135 × 0.056 m 的计分矩形。外壳比计分区域更大，击中外壳不算命中。
`Plate::point` 使用装甲板的右、上、外法线三个方向。

## 第三步：求解弹道

世界坐标为 +X 向前、+Y 向左、+Z 向上。yaw 从 +X 向 +Y 为正，pitch 向上为正。
角度用弧度、距离用米、时间用秒。

设炮口位置为 `m`、弹速为 `v`、yaw 为 `ψ`、pitch 为 `φ`、重力加速度为 `g`。
弹丸在**发射后**经过 `t` 的位置为：

$$
\begin{aligned}
x(t) &= m_x + v\cos\phi\cos\psi\;t,\\
y(t) &= m_y + v\cos\phi\sin\psi\;t,\\
z(t) &= m_z + v\sin\phi\;t - \tfrac12 gt^2.
\end{aligned}
$$

水平速度不变，竖直速度随时间减少 `g * t`。在射击所在的竖直平面内，以水平距离 `r` 表示：

$$
z(r) = m_z + r\tan\phi - \frac{g r^2}{2v^2\cos^2\phi}.
$$

这是一条抛物线，重力为零时退化为直线。仿真没有空气阻力或云台转动耗时，不需要更复杂的弹道模型。

`Trajectory` 包含 `yaw_rad`、`pitch_rad` 和 `flight_time_s`。
`Trajectory::solve(point_m)` 按已知弹速求解到**固定世界坐标点的低弹道**。
目标不可达、坐标非有限值或飞行超过寿命时返回 `None`。它不预测运动，也不添加延迟。

```rust
let trajectory = Trajectory::solve(aim_point_m)?;
debug.trajectory(trajectory, Color::Orange);
let impact_m = trajectory.position(trajectory.flight_time_s);

Some(Command {
    yaw_rad: trajectory.yaw_rad,
    pitch_rad: trajectory.pitch_rad,
    fire: true, // Decide whether this is a good shot before setting this.
})
```

`position(t)` 和 `velocity(t)` 中的时间都从发射开始计。飞行时间不是观测年龄，也不是绝对命中时刻。

## 第四步：瞄准未来位置

弹道工具瞄准固定点，但选中的装甲板仍在运动。用观测年龄、发射延迟和飞行时间预测命中时的姿态。
未来位置取决于飞行时间，而飞行时间又取决于未来位置，可以用不动点迭代处理：

1. 用距离除以弹速得到初始飞行时间。
2. 预测“观测年龄 + 发射延迟 + 估计飞行时间”后的目标状态。
3. 取预测目标上选中装甲板的位置，保持第二步选定的索引。
4. 求解到该位置的弹道，用新飞行时间更新估计。
5. 变化足够小时停止，并设置最大迭代次数；失败时不开火。

从观测采集时刻算起的预测间隔是：

$$
\Delta t = (t_{\mathrm{now}} - t_{\mathrm{capture}})
           + t_{\mathrm{launch\ delay}} + t_{\mathrm{flight}}.
$$

绝对命中时刻是 `now + launch_delay + flight_time`，计算它时不要再加观测年龄。
`Target::at(dt)` 预测匀速平移和自旋，`Target::plates(size)` 返回该时刻的四块装甲板姿态。
索引随目标一起转动；只预测机器人中心会漏掉装甲板的旋转运动。

用秒作为收敛容差的单位，并限制迭代次数。收敛后弹道终点应接近预测装甲板中心，可以同时画出检查。

还要考虑预计外法线和弹丸相对装甲板的入射速度。
`trajectory.velocity(trajectory.flight_time_s)` 给出命中时弹丸速度，
`Plate::velocity_m_s` 同时包含平移和旋转速度。可以参考下面的朝向评分：

$$
s = -\frac{\mathbf n\cdot(\mathbf v_{\mathrm{projectile}}-
                         \mathbf v_{\mathrm{plate}})}
               {\lVert\mathbf v_{\mathrm{projectile}}-\mathbf v_{\mathrm{plate}}\rVert}.
$$

较大的正值表示更接近正面入射，负值表示从背面接近。排除不可达目标，擦边角度时可以不开火。
开火阈值决定射击频率和对预测误差的敏感程度。

## 调试

在自己的 `aim` 中调用绘图接口。绘图只持续一个 tick，并与产生它的指令一起存入 MCAP：

```rust
use aiming_sim::ui::{Color, Gizmos};

debug.plate(plate, input.size, Color::Cyan);
debug.plate(predicted_plate, input.size, Color::Green);
debug.trajectory(trajectory, Color::Orange);
debug.line(from_m, to_m, Color::Purple);
debug.point("Impact", impact_m, Color::Green);
debug.overlay("Flight", format!("{:.0} ms", trajectory.flight_time_s * 1000.0));
```

`debug.trajectory()` 自动采样炮口到命中时刻的抛物线，不需要自己写绘图循环。
`debug.plate()` 画计分矩形和外法线，不添加标签；需要文字标记时使用 `debug.point()`。
工具会忽略非有限坐标并限制绘图数量，调试输出不影响评分。

左键拖动环绕视角；右键、中键或 Shift+左键拖动平移；滚轮缩放。
底部工具栏支持播放/暂停、单步、重置、时间轴和显示/隐藏叠加信息。
Tools 包含场景、绘图图层、速度、相机、初始状态、射击检查和 MCAP 保存/加载。
Inspect shot 会固定该次射击原始指令和绘图，便于逐步检查飞行过程；
Current tick 返回当前 tick 的诊断信息。

## 录制、回放与导出

```sh
cargo run --no-default-features --locked -- --offline --record target/run.mcap
cargo run --locked -- --replay target/run.mcap --at 3.65
```

录制包含 `/sim/config` 和 `/sim/frame`，保存精确输入、指令、状态、弹丸、计数和绘图。
回放直接恢复这些内容，不重新运行你的代码。新录制不会覆盖已有文件。
仿真时间以固定 10 ms 步长推进，不受界面播放速度影响。

PNG 和 MP4 使用同一个 GPU 场景渲染器。MP4 需要 `ffmpeg`。
导出视角跟随目标，视频为 3840 × 2160、60 FPS，不包含原生界面的文字标签、叠加信息和控件。

```sh
cargo run --locked -- --replay target/run.mcap --at 3.65 --preview target/preview.png
cargo run --locked -- --replay target/run.mcap --export-video target/run.mp4
```

默认目标半径为 0.25 m，装甲板中心高度为 0.3 m。
碰撞检测先在 1 ms 内包围穿越，再二分细化；相切或同一小区间内多次穿越可能被漏掉。
本题在同一进程内执行学生代码，不是代码沙箱。装甲图案沿用视觉仿真器的素材。

## 可选阅读

- [平动-自旋耦合目标下的火控拦截时间t解析与数值混合寻优策略](https://bbs.herkules.dev/articles/01M0Z7CK2QE1GT8XY1GNQK53YA)：
  介绍平移、自旋目标的解析与数值拦截时间求解，包括泰勒展开和牛顿迭代。
- [弹道解算及云台角度求解算法](https://bbs.herkules.dev/articles/01M2N3NVQSC21BAGE585ZF8DY1)：
  介绍带二次空气阻力的弹道与云台角度求解。

这些是基础作业以外的扩展。评分器的真值遵循上面的仅重力抛物线，模型更复杂不一定更适合本题。

## 提交与检查

提交自己的代码仓库链接、运行方式、简要思路，以及能展示静止、自旋、平移加自旋场景的录制或效果说明。
解释延迟如何相加、如何选板和判断可以开火。只选择本路线时不需要提交检测结果。

```sh
cargo fmt -- --check
cargo clippy --all-targets --all-features --locked -- -D warnings
cargo test --all-targets --locked
cargo test --no-default-features --locked
cargo doc --no-deps --locked
```
