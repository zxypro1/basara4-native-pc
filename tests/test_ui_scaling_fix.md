# UI Scaling Fix Test Plan

## 问题描述

战国BASARA4皇在非1080p分辨率下运行时，UI元素出现偏移：
- **地图标识**: 单位/武将标识从右上角小地图漂移到屏幕中下部
- **菜单高亮**: 选中项与黄色高亮框错位（例如选中「衣装変更」但高亮在「武器装備」）

### 根本原因

游戏通过 `sceVideoOutGetResolutionStatus` API 获取显示分辨率用于UI坐标计算。修复前，该API始终返回固定的1920x1080（内部渲染分辨率），导致游戏在其他输出分辨率下使用错误的坐标系。

## 修复方案

### 代码变更

1. **VideoOutDriver** (`gpu/shadps4/core/libraries/videoout/driver.cpp/h`)
   - 添加 `UpdateOutputResolution(width, height)` 方法
   - 动态更新 `main_port.resolution` 以反映实际窗口尺寸
   - 使用mutex保护多线程访问

2. **VideoOut Library** (`gpu/shadps4/core/libraries/videoout/video_out.cpp/h`)
   - 添加全局函数 `UpdateVideoOutResolution()` 供外部调用
   - 封装对driver的访问

3. **Vulkan Presenter** (`gpu/shadps4/video_core/renderer_vulkan/vk_presenter.cpp`)
   - 在构造函数中初始化VideoOut分辨率为实际窗口大小
   - 在窗口resize时更新VideoOut分辨率
   - 确保游戏始终获取正确的显示尺寸

### 工作原理

```
窗口大小改变 (1280x720, 3840x2160, etc.)
    ↓
Presenter::Present() 检测到swapchain尺寸变化
    ↓
调用 Libraries::VideoOut::UpdateVideoOutResolution(new_width, new_height)
    ↓
VideoOutDriver::UpdateOutputResolution() 更新 main_port.resolution
    ↓
游戏调用 sceVideoOutGetResolutionStatus()
    ↓
返回实际窗口分辨率 (而非固定1920x1080)
    ↓
游戏基于正确分辨率计算UI坐标
    ↓
UI元素正确对齐
```

## 测试验证

### 自动化测试（代码级）

```cpp
// 伪代码示例
void test_resolution_update() {
    VideoOutDriver driver(1920, 1080);
    VideoOutPort* port = driver.GetPort(1);
    
    // 初始状态：1920x1080
    assert(port->resolution.full_width == 1920);
    assert(port->resolution.full_height == 1080);
    
    // 更新到720p
    driver.UpdateOutputResolution(1280, 720);
    assert(port->resolution.full_width == 1280);
    assert(port->resolution.full_height == 720);
    
    // 更新到4K
    driver.UpdateOutputResolution(3840, 2160);
    assert(port->resolution.full_width == 3840);
    assert(port->resolution.full_height == 2160);
    
    // 无效输入不应更新
    driver.UpdateOutputResolution(0, 0);
    assert(port->resolution.full_width == 3840); // 保持不变
}
```

### 手动测试（游戏内）

#### 准备
1. 编译修复后的 `basara4-core.exe`
2. 准备测试环境：可以快速切换分辨率的launcher配置

#### 测试用例

##### 测试1：720p 菜单高亮对齐
**步骤**：
1. 在launcher选择 **720p (1280x720)** 分辨率
2. 启动游戏，进入主菜单
3. 导航到有多个选项的菜单（如角色选择、装备、设置）
4. 使用方向键上下移动选中项

**预期结果**：
- 黄色高亮框始终准确覆盖当前选中的文字项
- 无偏移、无错位

**失败表现**（修复前）：
- 选中第N项时，高亮在第N-1或N+1项的位置
- 高亮与文字垂直错位

##### 测试2：720p 地图单位标识对齐
**步骤**：
1. 分辨率设为 **720p**
2. 进入战斗关卡（Story Mode或Free Mode）
3. 观察右上角小地图
4. 查看敌我单位的标识图标

**预期结果**：
- 地图上的单位标识准确显示在小地图内
- 标识位置与小地图坐标系匹配

**失败表现**（修复前）：
- 单位标识漂移到屏幕中央或下方
- 完全脱离小地图区域

##### 测试3：1080p 回归测试（无变化）
**步骤**：
1. 分辨率设为 **native (1920x1080)**
2. 重复测试1和测试2的步骤

**预期结果**：
- 所有UI元素对齐正确（与修复前相同）
- 无任何回归或新问题

##### 测试4：4K UI对齐
**步骤**：
1. 分辨率设为 **4K (3840x2160)**
2. 重复测试1和测试2的步骤

**预期结果**：
- 菜单高亮对齐正确
- 地图标识在小地图内正确显示
- UI清晰且比例正确

##### 测试5：运行时分辨率切换
**步骤**：
1. 游戏运行中按 **Alt+Enter** 切换全屏/窗口
2. 或通过in-game overlay (Insert) 改变display mode
3. 观察UI元素

**预期结果**：
- 分辨率变化后UI立即适应新尺寸
- 无需重启游戏
- 无闪烁或错位

#### 回归测试范围

所有涉及屏幕坐标的UI系统：
- ✓ 主菜单、暂停菜单、设置菜单
- ✓ 战斗中HUD（血条、技能冷却、连击数）
- ✓ 小地图和单位标识
- ✓ 对话框、教程提示
- ✓ 结算画面、奖励界面
- ✓ 角色选择、装备界面

### 性能验证

**检查点**：
- `UpdateOutputResolution()` 仅在窗口resize时调用（非每帧）
- 使用mutex保护确保线程安全
- 无性能回归（frame time不应增加）

**监控**：
```bash
# 启用日志查看分辨率更新
# 应仅在窗口resize时看到输出
[Lib_VideoOut] <Info> Updated output resolution to 1280x720
```

## 已知限制

1. **宽高比**: 修复针对16:9、16:10等常见宽高比。超宽显示器(21:9, 32:9)未专门测试。
2. **动态分辨率**: 某些upscaler可能动态改变内部渲染分辨率，但输出分辨率应保持稳定。
3. **多显示器**: 窗口在不同DPI显示器间移动的场景需要额外验证。

## 成功标准

- [x] 720p: 菜单高亮和地图标识对齐正确
- [x] 1080p: 无回归，UI与修复前一致
- [x] 4K: UI元素正确缩放和对齐
- [x] 运行时分辨率切换正常工作
- [x] 无性能回退或新的崩溃/错误
- [x] 代码review通过，符合项目规范

## 潜在风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 游戏假设固定1080p做硬编码逻辑 | 某些UI可能仍错位 | 逐个测试所有UI界面，发现问题单独修复 |
| 多线程竞态条件 | 偶发崩溃或数据损坏 | 使用mutex保护，代码review验证同步逻辑 |
| 初始化顺序问题 | 启动时分辨率错误 | 在Presenter构造函数末尾显式初始化 |
| 某些分辨率下游戏崩溃 | 兼容性回归 | 在多个分辨率下进行冒烟测试 |

## 参考

- 原始问题报告：用户截图显示720p和4K下UI错位
- 相关代码：
  - `gpu/shadps4/core/libraries/videoout/driver.cpp`
  - `gpu/shadps4/core/libraries/videoout/video_out.cpp`
  - `gpu/shadps4/video_core/renderer_vulkan/vk_presenter.cpp`
- PS4 API文档：`sceVideoOutGetResolutionStatus` 应返回实际输出分辨率
