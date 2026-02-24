# Neon Breakout

A neon-style breakout game built with Svelte + TypeScript + Vite.

[中文](README.md) | English

## Game Introduction

Neon Breakout is a classic breakout game with a modern neon visual style. Players control the paddle to bounce the ball and break all the bricks to win.

## Game Features

- 🎮 **Classic Gameplay** - Control the paddle to bounce the ball and break all the bricks
- 🌟 **Neon Visuals** - Beautiful neon light effects and gradient colors
- 📱 **Responsive Design** - Adapts to different screen sizes
- 💾 **Local Storage** - Automatically saves high scores
- ⌨️ **Multiple Controls** - Supports keyboard and touch operations

## Game Controls

### Keyboard
- `A` / `D` or `←` / `→` - Move paddle
- `Space` - Start game / Pause
- `R` - Restart

### Touch
- Drag on the canvas to control paddle position

## Game Rules

- Initial lives: 3
- 100 points for each brick broken
- Ball speeds up after hitting each brick
- Losing the ball off the bottom costs one life
- Break all bricks to win
- Scores are automatically compared with high score and saved

## Tech Stack

- **Svelte 5** - Frontend framework
- **TypeScript** - Type safety
- **Vite 8** - Build tool
- **Canvas API** - Game rendering

## Quick Start

### Install Dependencies

```bash
pnpm install
```

### Development Mode

```bash
pnpm dev
```

### Build for Production

```bash
pnpm build
```

### Preview Production Build

```bash
pnpm preview
```

### Type Checking

```bash
pnpm check
```

## Project Structure

```
breakout/
├── src/
│   ├── App.svelte      # Main game component
│   ├── main.ts         # Application entry
│   └── app.css         # Global styles
├── public/             # Static assets
├── index.html          # HTML template
├── vite.config.ts      # Vite configuration
├── svelte.config.js    # Svelte configuration
└── tsconfig.json       # TypeScript configuration
```

## Game Parameters

Key parameters in the game (adjustable in `App.svelte`):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `BRICK_ROWS` | 6 | Number of brick rows |
| `BRICK_COLS` | 10 | Number of brick columns |
| `INITIAL_LIVES` | 3 | Initial lives |
| `ballBaseSpeed` | 380 | Ball initial speed |
| `ballMaxSpeed` | 760 | Ball maximum speed |

## Development Environment

Recommended to use VS Code with the following extension:

- [Svelte for VS Code](https://marketplace.visualstudio.com/items?itemName=svelte.svelte-vscode)

## License

MIT
