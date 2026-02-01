# OpenClaw 桌面自动化能力扩展开发指南

> 基于 macOS 平台的计算机使用 Agent 实现方案

## 一、概述

### 1.1 目标

扩展 OpenClaw，使其具备「看屏幕」和「操作鼠标/键盘」的能力，能够自动完成：
- 软件安装（如微信开发者工具）
- GUI 应用交互
- 开发环境配置

### 1.2 技术栈

| 模块 | 技术选型 | 说明 |
|------|----------|------|
| 模拟输入 | `@nut-tree/nut-js` | Node.js 桌面自动化库，跨平台支持 |
| 视觉分析 | GPT-4o / Claude 3.5 | 截屏 -> 分析 -> 获取坐标 |
| 系统调用 | `child_process` | 应用检测、下载、安装 |

---

## 二、开发环境准备

### 2.1 系统依赖

```bash
# 使用 Homebrew 安装必要依赖
brew install node@22
brew install python3
brew install opencv@4  # nut-js 的核心依赖
```

### 2.2 权限配置

macOS 需要给终端.app 添加「辅助功能」权限：
1. 系统设置 -> 隐私与安全性 -> 辅助功能
2. 勾选 终端.app

### 2.3 项目依赖

```bash
# 安装 nut-js 核心库
pnpm add @nut-tree/nut-js

# 安装图像处理辅助库
pnpm add sharp jimp
```

### 2.4 环境配置

创建 `.env.macOS.local`：

```bash
NODE_ENV=development
DESKTOP_PLATFORM=macOS
NUTJS_RESOURCES_PATH=./src/desktop/resources
SCREENSHOT_DIR=./tmp/screenshots
```

---

## 三、目录结构

```
src/desktop/
├── driver/
│   ├── index.ts           # 驱动入口，封装 nut-js 实例
│   ├── mouse.ts           # 鼠标操作（点击、移动、拖拽）
│   ├── keyboard.ts        # 键盘操作
│   └── screen.ts          # 截图、坐标获取
├── vision/
│   ├── index.ts           # 视觉分析入口
│   ├── analyzer.ts        # 图像分析逻辑
│   └── prompt.ts          # Vision Model Prompt 模板
├── apps/
│   ├── detector.ts        # 应用检测（通过 bundle id）
│   ├── installer.ts       # 应用下载与安装流程
│   └── wechat-devtools.ts # 微信开发者工具专用逻辑
├── resources/
│   ├── templates/         # 图像匹配模板
│   └── icons/             # 常用图标缓存
└── types.ts               # 类型定义
```

---

## 四、核心实现

### 4.1 桌面驱动层 (Driver Layer)

**文件**: `src/desktop/driver/index.ts`

```typescript
import { mouse, right, left, up, down, Point, Button } from "@nut-tree/nut-js";
import { screen } from "@nut-tree/nut-js";

export class DesktopDriver {
  private screenSize: { width: number; height: number };

  constructor() {
    this.init();
  }

  private async init() {
    const { width, height } = await screen.getScreenSize();
    this.screenSize = { width, height };
  }

  async moveMouse(x: number, y: number): Promise<void> {
    await mouse.setPosition(new Point(x, y));
  }

  async click(x: number, y: number, button: 'left' | 'right' = 'left'): Promise<void> {
    await this.moveMouse(x, y);
    const nutButton = button === 'left' ? Button.LEFT : Button.RIGHT;
    await mouse.click(nutButton);
  }

  async doubleClick(x: number, y: number): Promise<void> {
    await this.moveMouse(x, y);
    await mouse.doubleClick(Button.LEFT);
  }

  async drag(from: { x: number; y: number }, to: { x: number; y: number }): Promise<void> {
    await mouse.setPosition(new Point(from.x, from.y));
    await mouse.press(Button.LEFT);
    await mouse.setPosition(new Point(to.x, to.y));
    await mouse.release(Button.LEFT);
  }

  async typeText(text: string): Promise<void> {
    await keyboard.type(text);
  }

  async takeScreenshot(path: string): Promise<string> {
    const screenshot = await screen.capture();
    await screenshot.toFile(path);
    return path;
  }
}

export const desktopDriver = new DesktopDriver();
```

### 4.2 应用检测模块

**文件**: `src/desktop/apps/detector.ts`

```typescript
import { exec } from "node:child_process";
import { promisify } from "node:util";

const execAsync = promisify(exec);

export async function checkAppInstalled(bundleId: string): Promise<boolean> {
  try {
    await execAsync(`osascript -e 'id of application "${bundleId}"'`);
    return true;
  } catch {
    return false;
  }
}

export async function launchApp(bundleId: string): Promise<void> {
  await execAsync(`open -a "${bundleId}"`);
}

export const KNOWN_APPS = {
  WECHAT_DEVTOOLS: "com.tencent.wechat.devtools",
  WECHAT: "com.tencent.xinWeChat",
};
```

### 4.3 视觉分析模块

**文件**: `src/desktop/vision/analyzer.ts`

```typescript
import fs from "node:fs";
import path from "node:path";
import { desktopDriver } from "../driver/index.js";
import { visionProvider } from "../providers/vision.js";

const SCREENSHOT_DIR = "./tmp/screenshots";

export interface VisionResult {
  action: "click" | "type" | "wait" | "complete" | "error";
  target?: string;
  coordinate?: { x: number; y: number };
  text?: string;
  reasoning: string;
}

export async function analyzeScreen(
  prompt: string,
  screenshotPath?: string
): Promise<VisionResult> {
  const targetPath = screenshotPath || path.join(SCREENSHOT_DIR, `screen_${Date.now()}.png`);
  
  if (!screenshotPath) {
    await desktopDriver.takeScreenshot(targetPath);
  }

  const imageBase64 = fs.readFileSync(targetPath, { encoding: "base64" });

  const response = await visionProvider.analyze({
    image: imageBase64,
    prompt: `
      你是一个自动化助手。当前任务：${prompt}
      
      请以 JSON 格式返回：
      {
        "action": "click" | "type" | "wait" | "complete" | "error",
        "coordinate": {"x": 0, "y": 0},
        "reasoning": "决策说明"
      }
    `,
  });

  return JSON.parse(response);
}
```

### 4.4 微信开发者工具安装流程

**文件**: `src/desktop/apps/wechat-devtools.ts`

```typescript
import { desktopDriver } from "../driver/index.js";
import { analyzeScreen } from "../vision/analyzer.js";
import { checkAppInstalled, launchApp, KNOWN_APPS } from "./detector.js";
import { mountDmg, installFromDmg, unmountDmg } from "./installer.js";

export async function installWechatDevtools(): Promise<void> {
  // Step 1: 检查是否已安装
  if (await checkAppInstalled(KNOWN_APPS.WECHAT_DEVTOOLS)) {
    console.log("微信开发者工具已安装");
    return;
  }

  // Step 2: 下载安装包 (略)
  const dmgPath = "./tmp/downloads/wechat_devtools.dmg";

  // Step 3: 挂载 dmg
  const mountPoint = await mountDmg(dmgPath);

  // Step 4: 视觉引导安装
  let completed = false;
  let attempts = 0;
  const maxAttempts = 60;

  while (!completed && attempts < maxAttempts) {
    attempts++;
    const result = await analyzeScreen(
      "检查是否出现安装完成的提示，或 Applications 文件夹中的微信开发者工具图标"
    );

    if (result.action === "click" && result.coordinate) {
      await desktopDriver.click(result.coordinate.x, result.coordinate.y);
    } else if (result.action === "complete") {
      completed = true;
    } else if (result.action === "wait") {
      await new Promise((resolve) => setTimeout(resolve, 5000));
    }
  }

  await unmountDmg(mountPoint);
}
```

---

## 五、Agent 工具集成

**文件**: `src/agents/tools/computer_use.ts`

```typescript
import { desktopDriver } from "../../desktop/driver/index.js";
import { analyzeScreen } from "../../desktop/vision/analyzer.js";

export const computerUseTool = {
  name: "computer_use",
  description: "通过视觉引导操作计算机桌面",
  parameters: {
    type: "object",
    properties: {
      instruction: { type: "string", description: "操作指令" },
      screenshot: { type: "boolean", description: "是否截屏分析", default: false },
    },
    required: ["instruction"],
  },

  async handler(params: { instruction: string; screenshot?: boolean }) {
    if (params.screenshot) {
      return await analyzeScreen(params.instruction);
    }
    // 执行具体操作...
  },
};
```

---

## 六、提交历史规划

| 提交 | 说明 |
|------|------|
| `feat(desktop): init desktop automation module structure` | 初始化目录结构 |
| `feat(desktop): implement mouse control with nut-js` | 鼠标控制 |
| `feat(desktop): add screenshot and keyboard support` | 截图和键盘 |
| `feat(desktop): add app detection via macOS bundle id` | 应用检测 |
| `feat(vision): implement screen capture for vision analysis` | 截屏功能 |
| `feat(vision): integrate LLM for coordinate prediction` | LLM 坐标分析 |
| `feat(apps): implement wechat devtools installation workflow` | 安装流程 |
| `feat(agents): add computer_use tool definition` | Agent 工具 |
| `test(desktop): add unit tests for driver module` | 单元测试 |

---

## 七、注意事项

1. **WSL 兼容性**: 本方案针对 macOS 原生环境，WSL 下无法直接控制鼠标
2. **权限**: 首次运行需在「隐私设置」中授权终端
3. **安全**: 建议增加操作确认机制，避免误删文件
4. **DPI**: 不同缩放比例可能导致坐标偏移，需做适配
