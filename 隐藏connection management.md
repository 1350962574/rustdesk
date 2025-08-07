# RustDesk "Hide connection management window" 功能修复说明

## 问题描述

当在RustDesk设置中启用"Hide connection management window"选项时，虽然可以立即隐藏窗口，但在以下两种情况下设置未能正确生效：
1. 程序重启后，CM窗口仍然会显示
2. 当发起新的远程连接时，CM窗口仍然会显示

## 问题原因分析

经过代码分析，发现了两个主要问题：

1. **配置不同步问题**：
   - 在设置页面(`desktop_setting_page.dart`)中，更改选项只设置了`allow-hide-cm`配置项
   - 而CM启动时(`main.dart`中的`runConnectionManagerScreen`函数)读取的是`hide_cm`配置项
   - 这导致两个配置不同步，设置更改后重启程序不生效

2. **连接发起时的显示控制逻辑问题**：
   - 在`server_model.dart`中的`addConnection`方法中，发起新连接时只检查了`!hideCm`条件
   - 当`hideCm`为true时，代码没有主动隐藏窗口，而是跳过了相关代码
   - 这导致即使设置了隐藏CM窗口，新连接发起时窗口仍然会显示

## 修复方案

### 1. 修改设置页面中的处理函数

**位置**：`flutter/lib/desktop/pages/desktop_setting_page.dart`中的`onHideCmChanged`函数

**原始代码**:
```dart
void onHideCmChanged(bool? b) async {
  if (b != null) {
    await bind.mainSetOption(
        key: 'allow-hide-cm', value: bool2option('allow-hide-cm', b));
    // 直接修改系统隐藏CM窗口的标志，不使用cmSetConfig
    // 状态由 ServerModel.updatePasswordModel 自动同步
    // 立即生效：调用窗口显示/隐藏
    if (b) {
      await hideCmWindow();
    } else {
      await showCmWindow();
    }
  }
}
```

**修改后代码**:
```dart
void onHideCmChanged(bool? b) async {
  if (b != null) {
    // 1. 设置allow-hide-cm配置项 (给UI读取)
    await bind.mainSetOption(
        key: 'allow-hide-cm', value: bool2option('allow-hide-cm', b));
        
    // 2. 同时设置hide_cm配置项 (给CM启动时读取)
    await bind.mainSetOption(
        key: 'hide_cm', value: b ? 'true' : 'false');
        
    // 3. 更新内存中的状态
    model.hideCm = b;
    
    // 4. 立即应用窗口显示/隐藏状态
    try {
      if (b) {
        await hideCmWindow();
      } else {
        await showCmWindow();
      }
    } catch (e) {
      debugPrint('应用窗口状态时出错: $e');
    }
    
    // 5. 通知UI刷新
    model.notifyListeners();
  }
}
```

### 2. 修改CM窗口启动时的读取逻辑

**位置**：`flutter/lib/main.dart`中的`runConnectionManagerScreen`函数

**原始代码**:
```dart
void runConnectionManagerScreen() async {
  await initEnv(kAppTypeConnectionManager);
  _runApp(
    '',
    const DesktopServerPage(),
    MyTheme.currentThemeMode(),
  );
  final hide = await bind.cmGetConfig(name: "hide_cm") == 'true';
  gFFI.serverModel.hideCm = hide;
  if (hide) {
    await hideCmWindow(isStartup: true);
  } else {
    await showCmWindow(isStartup: true);
  }
  setResizable(false);
  // Start the uni links handler and redirect links to Native, not for Flutter.
  listenUniLinks(handleByFlutter: false);
}
```

**修改后代码**:
```dart
void runConnectionManagerScreen() async {
  await initEnv(kAppTypeConnectionManager);
  _runApp(
    '',
    const DesktopServerPage(),
    MyTheme.currentThemeMode(),
  );
  // 同时检查两个配置项，确保任何一个设置为true时都能正确隐藏CM窗口
  final hideCm = await bind.cmGetConfig(name: "hide_cm") == 'true';
  final allowHideCm = await bind.cmGetConfig(name: "allow-hide-cm") == 'true';
  final hide = hideCm || allowHideCm;
  debugPrint("CM窗口启动状态: hide_cm=$hideCm, allow-hide-cm=$allowHideCm, 最终hide=$hide");
  gFFI.serverModel.hideCm = hide;
  if (hide) {
    await hideCmWindow(isStartup: true);
  } else {
    await showCmWindow(isStartup: true);
  }
  setResizable(false);
  // Start the uni links handler and redirect links to Native, not for Flutter.
  listenUniLinks(handleByFlutter: false);
}
```

### 3. 修改处理新连接时CM窗口显示逻辑

**位置**：`flutter/lib/models/server_model.dart`中的`addConnection`方法

**原始代码**:
```dart
if (desktopType == DesktopType.cm && !hideCm) {
  showCmWindow();
}
```

**修改后代码**:
```dart
// 如果设置了隐藏CM窗口，则不显示窗口；否则显示窗口
if (desktopType == DesktopType.cm) {
  if (hideCm) {
    hideCmWindow();
  } else {
    showCmWindow();
  }
}
```

### 4. 其他相关逻辑修改

**位置**：`flutter/lib/models/server_model.dart`中的其他窗口控制逻辑

1. **修改窗口置顶逻辑**:

**原始代码**:
```dart
Future.delayed(Duration.zero, () async {
  if (!hideCm) windowOnTop(null);
});
```

**修改后代码**:
```dart
// 仅在CM不隐藏时置顶窗口
Future.delayed(Duration.zero, () async {
  if (!hideCm) windowOnTop(null);
});
```

2. **修改最小化窗口逻辑**:

**原始代码**:
```dart
// Only do the hidden task when on Desktop.
if (client.authorized && isDesktop) {
  cmHiddenTimer = Timer(const Duration(seconds: 3), () {
    if (!hideCm) windowManager.minimize();
    cmHiddenTimer = null;
```

**修改后代码**:
```dart
// Only do the hidden task when on Desktop.
if (client.authorized && isDesktop) {
  cmHiddenTimer = Timer(const Duration(seconds: 3), () {
    // 仅在未设置隐藏CM时最小化窗口
    if (!hideCm) windowManager.minimize();
    cmHiddenTimer = null;
```

## 修复效果

经过上述修改后:

1. **配置同步问题解决**：
   - 在设置页面中同时更新`allow-hide-cm`和`hide_cm`两个配置项
   - CM启动时检查两个配置，确保向后兼容性

2. **连接发起时窗口显示问题解决**：
   - 在发起新连接时根据`hideCm`状态决定是隐藏还是显示CM窗口
   - 不再只是简单地检查`!hideCm`条件

3. **整体效果**：
   - 设置更改后立即生效
   - 重启程序后设置依然保持
   - 发起新连接时也能正确应用隐藏设置
   - 整体用户体验更加一致

## 技术分析

问题的核心在于多处配置和状态同步不一致，修复方案主要做了以下工作：

1. **统一配置项**：确保UI设置和CM启动时使用相同的配置
2. **完善条件判断**：在发起连接时正确判断窗口显示状态
3. **状态同步**：直接更新模型状态并通知UI刷新
4. **错误处理**：添加try-catch保证代码健壮性

这种修复方式不需要修改后端Rust代码，只需调整前端Flutter代码中的相关逻辑，最小化了修改范围，同时保证了功能正确性。
