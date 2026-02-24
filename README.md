# Neon Breakout

一个使用 Svelte + TypeScript + Vite 构建的霓虹风格打砖块游戏。

## 游戏简介

Neon Breakout 是一款经典的打砖块游戏，采用现代霓虹视觉风格设计。玩家控制挡板反弹小球，击碎所有砖块即可获胜。

## 游戏特性

- 🎮 **经典玩法** - 控制挡板反弹小球，击碎所有砖块
- 🌟 **霓虹视觉** - 精美的霓虹灯效果和渐变色彩
- 📱 **响应式设计** - 自适应不同屏幕尺寸
- 💾 **本地存储** - 自动保存最高分记录
- ⌨️ **多种控制** - 支持键盘和触摸操作

## 游戏控制

### 键盘操作
- `A` / `D` 或 `←` / `→` - 移动挡板
- `Space` - 开始游戏 / 暂停
- `R` - 重新开始

### 触摸操作
- 在画布上拖动以控制挡板位置

## 游戏规则

- 初始生命值：3 条
- 每击碎一块砖块得 100 分
- 小球每次击中砖块后会加速
- 小球掉落屏幕底部会损失一条生命
- 击碎所有砖块即可获胜
- 分数会自动与最高分比较并保存

## 技术栈

- **Svelte 5** - 前端框架
- **TypeScript** - 类型安全
- **Vite 8** - 构建工具
- **Canvas API** - 游戏渲染

## 快速开始

### 安装依赖

```bash
pnpm install
```

### 开发模式

```bash
pnpm dev
```

### 构建生产版本

```bash
pnpm build
```

### 预览生产版本

```bash
pnpm preview
```

### 类型检查

```bash
pnpm check
```

## 项目结构

```
breakout/
├── src/
│   ├── App.svelte      # 游戏主组件
│   ├── main.ts         # 应用入口
│   └── app.css         # 全局样式
├── public/             # 静态资源
├── index.html          # HTML 模板
├── vite.config.ts      # Vite 配置
├── svelte.config.js    # Svelte 配置
└── tsconfig.json       # TypeScript 配置
```

## 游戏参数

游戏中的关键参数（可在 `App.svelte` 中调整）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `BRICK_ROWS` | 6 | 砖块行数 |
| `BRICK_COLS` | 10 | 砖块列数 |
| `INITIAL_LIVES` | 3 | 初始生命值 |
| `ballBaseSpeed` | 380 | 小球初始速度 |
| `ballMaxSpeed` | 760 | 小球最大速度 |

## 开发环境

推荐使用 VS Code 并安装以下扩展：

- [Svelte for VS Code](https://marketplace.visualstudio.com/items?itemName=svelte.svelte-vscode)

## 许可证

MIT