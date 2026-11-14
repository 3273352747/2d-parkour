# 2D跑酷开发与发布说明

[返回项目首页](../README.md)

## 1. 文档范围

本仓库是WebGL发布包。原Unity工程使用2022.3.62f3c1，主要脚本位于 `Assets/Scripts/`。下文说明原工程设计，不表示仓库中包含可重新编辑和构建的完整源码。

## 2. 模块划分

| 模块 | 职责 |
| --- | --- |
| LoginSceneController | 玩家名输入、开始标记和场景跳转 |
| GameManager | 游戏状态、难度、计分、暂停、结算数据与UI |
| PlayerControl | 跳跃、接地动画、危险碰撞与死亡 |
| GroundControl | 地形滚动、有效预制体筛选、随机选择与高度约束 |
| CoinControl | 金币触发、收集反馈与计分通知 |
| BgControl | 背景运动 |
| AudioManager | 音频管理与播放 |
| GameOverSceneController | 本局结果、最高分展示及重新开始 |

```text
LoginScene -> SampleScene -> GameOverScene
                   |
                Playing <-> Paused
                   |
                GameOver

键盘 / UI -> PlayerControl.Jump -> 刚体与动画
GameManager.SpeedMultiplier -> 地形滚动与距离分
金币 -> AddCoin -> HUD -> 结束时保存结算数据
```

## 3. 核心实现

### 状态管理

GameManager明确区分Ready、Playing、Paused、GameOver。Update只在Playing累计时间、计算速度倍率与距离分。暂停同时修改状态与timeScale；重新开始和场景跳转恢复timeScale，避免把暂停状态带入下一场景。

启动输入门禁基于unscaledTime，默认约0.18秒，仅用于场景启动边界。它不是每次跳跃的固定延迟。

### 跳跃与死亡

键盘和UI调用同一Jump方法，检查游戏输入许可、接地和生命后施力。接触地面更新动画，碰到Enemy或Die触发死亡；生命归零后再次进入死亡入口会返回，避免重复切换结算。

目前接地基于碰撞进入和退出。后续应在接受跳跃时立即锁定离地，并处理跨两个相邻地块时的接触数量，防止短时间重复施力或错误离地。

### 随机地形

生成前筛选带有效地面标记和碰撞的预制体；选择时有限次数重抽，减少连续重复。高度限制在全局范围和相邻变化范围内。出屏地块触发补位，然后销毁旧实例。

这套实现控制了随机范围，但没有完成严格可达性证明。下一步需要把速度、跳跃轨迹、平台宽度和间隙一起加入候选筛选，并提供安全回退地块。

### 分数与本地记录

距离分由游玩时间和速度倍率累积，金币额外计分，结束时生成整数总分。PlayerPrefs保存PlayerName、BestScore、LastScore、LastCoins等字段，入口、游玩和结算按各自职责读取。

这些数据服务于单机Demo，不具备服务端排行榜的可信性。浏览器存储受来源、用户清理操作和隐私模式影响，不能承诺跨设备或永久保存。

## 4. 开发中关注的问题

| 问题 | 处理方向 |
| --- | --- |
| 点击开始后立即跳跃 | 用统一状态与短时输入门禁隔离场景切换输入 |
| 键盘与UI同时操作 | 共用Jump入口；继续完善同帧请求去重与焦点检查 |
| 随机生成无效地块 | 在候选列表中先过滤标记与碰撞条件 |
| 高度变化过大 | 限制全局高度和相邻高度差，继续补可达性判断 |
| 暂停后重开仍冻结 | 场景切换与重开显式恢复timeScale |
| 编译后按钮中文缺失 | 核对字体覆盖、资源引用和真实构建中的显示结果 |

这里区分已经存在的处理与后续改进，不将单个局部修正描述为彻底解决所有边界问题。

## 5. 原工程构建

1. 用Unity 2022.3.62f3c1打开完整原工程，并安装WebGL Build Support。
2. 先确认Console无编译错误，检查Player、地形预制体、UI事件和音频引用。
3. Build Settings顺序为 `Assets/Scenes/LoginScene.unity`、`Assets/Scenes/SampleScene.unity`、`Assets/Scenes/GameOverScene.unity`。
4. 将WebGL构建输出到独立目录，避免覆盖工程文件。
5. 当前仓库包含 `runner-responsive.*.unityweb` 与匹配的loader，使用其解压支持。不要把其他未压缩构建的loader单独混进来。
6. 使用HTTP服务验证玩家名输入、开始、跳跃、暂停、失败、结算与重开。

## 6. 更新GitHub发布包

- 四个关联构建文件必须来自同一次构建，一起更新至 `Build/`。
- 文件名或压缩方式改变时，同步更新 `web/build-config.json`，不要仅修改文件扩展名。
- 保持 `index.html` 和 `.nojekyll` 位于根目录，保持原试玩URL。
- 配置中的companyName、productName与productVersion保留构建身份；网页显示标题独立维护。
- 运行静态检查、本地预览和 [测试清单](TESTING.md)，再提交推送。
- 等待GitHub Pages发布后，线上再次检查标题、资源请求、进入游戏和结算。
- 整理README和启动层不需要重编译；改变C#玩法或Unity资源必须重新构建。

本次目录整理不变更Unity工程或现有游戏二进制，不自动清理旧文件。

## 7. 本地预览与故障定位

```bash
node tools/check-site.mjs
node --check web/boot.js
node tools/serve.mjs 8081
```

预览服务只监听127.0.0.1，不是对外发布服务，不依赖npm包。

| 现象 | 检查顺序 |
| --- | --- |
| 一直显示加载 | 配置请求、loader、各构建文件状态与Console |
| 解压或WASM失败 | loader与unityweb是否成套、服务器响应头 |
| 按键无响应 | 页面焦点、Playing状态、输入解锁时间和接地状态 |
| 暂停后界面可点但玩法异常 | 状态门禁与timeScale是否同时恢复 |
| 分数未保存 | 结算是否执行、网站数据权限、是否跨来源或设备 |
| 线上旧版本仍可见 | Pages发布状态、HTML与构建缓存是否一致 |
