# Breakout 豪华版 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a deluxe Breakout game with Shinkai Makoto-style visuals, powerups, level editor, leaderboard, and particle effects using Svelte 5 + Vite + Canvas.

**Architecture:** Svelte 5 SPA with Vite bundler. Game engine is pure TypeScript classes (Ball, Paddle, Brick, etc.) running in a `requestAnimationFrame` loop, rendered to HTML5 Canvas 2D. Svelte components handle UI overlays (menus, HUD, editor). Game state managed via Svelte stores. Audio via Web Audio API synthesis. Persistence via localStorage.

**Tech Stack:** Svelte 5 (runes), Vite, TypeScript, HTML5 Canvas 2D, Web Audio API, Vitest

**Design Doc:** `docs/plans/2026-02-24-breakout-design.md`

---

## Task 1: Project Scaffolding

**Files:**
- Create: entire project via `create vite` template
- Create: `vitest.config.ts`

**Step 1: Scaffold Svelte + TS + Vite project**

Run:
```bash
cd D:/github/Breakout
npm create vite@latest . -- --template svelte-ts
npm install
```

If prompted about non-empty directory, confirm overwrite (docs/ folder exists).

**Step 2: Install dev dependencies**

```bash
npm install -D vitest
```

**Step 3: Verify dev server works**

```bash
npm run dev
```

Open browser, confirm Svelte welcome page loads. Stop the server.

**Step 4: Create directory structure**

```bash
mkdir -p src/lib/engine src/lib/audio src/lib/levels src/lib/stores src/lib/renderer src/components
```

**Step 5: Add vitest config**

Create `vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config';
import { svelte } from '@sveltejs/vite-plugin-svelte';

export default defineConfig({
  plugins: [svelte()],
  test: {
    environment: 'jsdom',
    globals: true,
  },
});
```

Add to `package.json` scripts: `"test": "vitest run", "test:watch": "vitest"`

**Step 6: Commit**

```bash
git add -A
git commit -m "chore: scaffold Svelte + Vite + TypeScript project with vitest"
```

---

## Task 2: Game Constants & Types

**Files:**
- Create: `src/lib/engine/types.ts`
- Create: `src/lib/engine/constants.ts`
- Test: `src/lib/__tests__/constants.test.ts`

**Step 1: Create types file**

Create `src/lib/engine/types.ts`:
```ts
export type GameState = 'menu' | 'playing' | 'paused' | 'gameover' | 'victory' | 'editor' | 'leaderboard';

export type BrickType = 'normal' | 'hard' | 'gold' | 'unbreakable';

export type PowerupType = 'widen' | 'multiball' | 'slow' | 'pierce' | 'life' | 'magnet';

export interface Vec2 {
  x: number;
  y: number;
}

export interface Rect {
  x: number;
  y: number;
  width: number;
  height: number;
}

export interface BrickData {
  type: BrickType;
  row: number;
  col: number;
}

export interface LevelData {
  name: string;
  bricks: BrickData[];
}

export interface LeaderboardEntry {
  name: string;
  score: number;
  level: number;
  date: string;
}
```

**Step 2: Create constants file**

Create `src/lib/engine/constants.ts`:
```ts
export const CANVAS_WIDTH = 800;
export const CANVAS_HEIGHT = 600;

export const PADDLE_WIDTH = 100;
export const PADDLE_HEIGHT = 14;
export const PADDLE_Y = CANVAS_HEIGHT - 40;
export const PADDLE_SPEED = 8;

export const BALL_RADIUS = 6;
export const BALL_SPEED = 5;

export const BRICK_ROWS = 8;
export const BRICK_COLS = 10;
export const BRICK_WIDTH = 70;
export const BRICK_HEIGHT = 24;
export const BRICK_PADDING = 4;
export const BRICK_OFFSET_TOP = 60;
export const BRICK_OFFSET_LEFT = (CANVAS_WIDTH - (BRICK_COLS * (BRICK_WIDTH + BRICK_PADDING) - BRICK_PADDING)) / 2;

export const INITIAL_LIVES = 3;

export const POWERUP_WIDTH = 24;
export const POWERUP_HEIGHT = 24;
export const POWERUP_FALL_SPEED = 2;

export const BRICK_SCORES: Record<string, number> = {
  normal: 10,
  hard: 20,
  gold: 50,
  unbreakable: 0,
};

export const BRICK_HP: Record<string, number> = {
  normal: 1,
  hard: 3,
  gold: 1,
  unbreakable: Infinity,
};
```

**Step 3: Write a quick sanity test**

Create `src/lib/__tests__/constants.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { CANVAS_WIDTH, CANVAS_HEIGHT, BRICK_OFFSET_LEFT, BRICK_COLS, BRICK_WIDTH, BRICK_PADDING } from '../engine/constants';

describe('Game Constants', () => {
  it('canvas dimensions are positive', () => {
    expect(CANVAS_WIDTH).toBeGreaterThan(0);
    expect(CANVAS_HEIGHT).toBeGreaterThan(0);
  });

  it('bricks are centered horizontally', () => {
    const totalBricksWidth = BRICK_COLS * (BRICK_WIDTH + BRICK_PADDING) - BRICK_PADDING;
    expect(BRICK_OFFSET_LEFT).toBeCloseTo((CANVAS_WIDTH - totalBricksWidth) / 2);
  });
});
```

**Step 4: Run tests**

```bash
npx vitest run
```
Expected: PASS

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: add game types and constants"
```

---

## Task 3: Ball Class

**Files:**
- Create: `src/lib/engine/Ball.ts`
- Test: `src/lib/__tests__/Ball.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/Ball.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { Ball } from '../engine/Ball';
import { CANVAS_WIDTH, CANVAS_HEIGHT, BALL_RADIUS } from '../engine/constants';

describe('Ball', () => {
  it('initializes at given position', () => {
    const ball = new Ball(100, 200);
    expect(ball.x).toBe(100);
    expect(ball.y).toBe(200);
  });

  it('moves by velocity each update', () => {
    const ball = new Ball(100, 200);
    ball.vx = 3;
    ball.vy = -4;
    ball.update();
    expect(ball.x).toBe(103);
    expect(ball.y).toBe(196);
  });

  it('bounces off left wall', () => {
    const ball = new Ball(BALL_RADIUS, 200);
    ball.vx = -3;
    ball.vy = -4;
    ball.update();
    expect(ball.vx).toBe(3);
  });

  it('bounces off right wall', () => {
    const ball = new Ball(CANVAS_WIDTH - BALL_RADIUS, 200);
    ball.vx = 3;
    ball.vy = -4;
    ball.update();
    expect(ball.vx).toBe(-3);
  });

  it('bounces off top wall', () => {
    const ball = new Ball(100, BALL_RADIUS);
    ball.vx = 3;
    ball.vy = -4;
    ball.update();
    expect(ball.vy).toBe(4);
  });

  it('detects falling below screen', () => {
    const ball = new Ball(100, CANVAS_HEIGHT + 10);
    expect(ball.isOut()).toBe(true);
  });

  it('reset places ball at new position with zero velocity', () => {
    const ball = new Ball(100, 200);
    ball.vx = 5;
    ball.vy = -5;
    ball.reset(400, 300);
    expect(ball.x).toBe(400);
    expect(ball.y).toBe(300);
    expect(ball.vx).toBe(0);
    expect(ball.vy).toBe(0);
  });
});
```

**Step 2: Run tests to verify failure**

```bash
npx vitest run src/lib/__tests__/Ball.test.ts
```
Expected: FAIL — module not found

**Step 3: Implement Ball**

Create `src/lib/engine/Ball.ts`:
```ts
import { CANVAS_WIDTH, CANVAS_HEIGHT, BALL_RADIUS } from './constants';

export class Ball {
  x: number;
  y: number;
  vx = 0;
  vy = 0;
  radius = BALL_RADIUS;
  isPiercing = false;

  trail: Array<{ x: number; y: number; alpha: number }> = [];

  constructor(x: number, y: number) {
    this.x = x;
    this.y = y;
  }

  update(): void {
    this.trail.push({ x: this.x, y: this.y, alpha: 1 });
    if (this.trail.length > 12) this.trail.shift();
    for (const t of this.trail) t.alpha *= 0.85;

    this.x += this.vx;
    this.y += this.vy;

    // Wall collisions
    if (this.x - this.radius <= 0) {
      this.x = this.radius;
      this.vx = Math.abs(this.vx);
    }
    if (this.x + this.radius >= CANVAS_WIDTH) {
      this.x = CANVAS_WIDTH - this.radius;
      this.vx = -Math.abs(this.vx);
    }
    if (this.y - this.radius <= 0) {
      this.y = this.radius;
      this.vy = Math.abs(this.vy);
    }
  }

  isOut(): boolean {
    return this.y - this.radius > CANVAS_HEIGHT;
  }

  reset(x: number, y: number): void {
    this.x = x;
    this.y = y;
    this.vx = 0;
    this.vy = 0;
    this.trail = [];
    this.isPiercing = false;
  }

  launch(angle: number, speed: number): void {
    this.vx = Math.cos(angle) * speed;
    this.vy = Math.sin(angle) * speed;
  }
}
```

**Step 4: Run tests to verify pass**

```bash
npx vitest run src/lib/__tests__/Ball.test.ts
```
Expected: PASS

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: implement Ball class with wall bouncing and trail"
```

---

## Task 4: Paddle Class

**Files:**
- Create: `src/lib/engine/Paddle.ts`
- Test: `src/lib/__tests__/Paddle.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/Paddle.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { Paddle } from '../engine/Paddle';
import { CANVAS_WIDTH, PADDLE_WIDTH, PADDLE_Y, PADDLE_SPEED } from '../engine/constants';

describe('Paddle', () => {
  it('initializes centered', () => {
    const paddle = new Paddle();
    expect(paddle.x).toBe((CANVAS_WIDTH - PADDLE_WIDTH) / 2);
    expect(paddle.y).toBe(PADDLE_Y);
  });

  it('moves left within bounds', () => {
    const paddle = new Paddle();
    paddle.moveLeft();
    expect(paddle.x).toBe((CANVAS_WIDTH - PADDLE_WIDTH) / 2 - PADDLE_SPEED);
  });

  it('does not move past left wall', () => {
    const paddle = new Paddle();
    paddle.x = 2;
    paddle.moveLeft();
    expect(paddle.x).toBe(0);
  });

  it('moves right within bounds', () => {
    const paddle = new Paddle();
    paddle.moveRight();
    expect(paddle.x).toBe((CANVAS_WIDTH - PADDLE_WIDTH) / 2 + PADDLE_SPEED);
  });

  it('does not move past right wall', () => {
    const paddle = new Paddle();
    paddle.x = CANVAS_WIDTH - paddle.width - 2;
    paddle.moveRight();
    expect(paddle.x).toBe(CANVAS_WIDTH - paddle.width);
  });

  it('moveTo centers paddle on x position', () => {
    const paddle = new Paddle();
    paddle.moveTo(400);
    expect(paddle.x).toBe(400 - paddle.width / 2);
  });

  it('widen increases width by 1.5x', () => {
    const paddle = new Paddle();
    const originalWidth = paddle.width;
    paddle.widen();
    expect(paddle.width).toBe(Math.floor(originalWidth * 1.5));
  });

  it('resetWidth restores original width', () => {
    const paddle = new Paddle();
    const originalWidth = paddle.width;
    paddle.widen();
    paddle.resetWidth();
    expect(paddle.width).toBe(originalWidth);
  });
});
```

**Step 2: Run tests — expect FAIL**

```bash
npx vitest run src/lib/__tests__/Paddle.test.ts
```

**Step 3: Implement Paddle**

Create `src/lib/engine/Paddle.ts`:
```ts
import { CANVAS_WIDTH, PADDLE_WIDTH, PADDLE_HEIGHT, PADDLE_Y, PADDLE_SPEED } from './constants';

export class Paddle {
  x: number;
  y = PADDLE_Y;
  width = PADDLE_WIDTH;
  height = PADDLE_HEIGHT;
  isMagnet = false;

  private baseWidth = PADDLE_WIDTH;

  constructor() {
    this.x = (CANVAS_WIDTH - this.width) / 2;
  }

  moveLeft(): void {
    this.x = Math.max(0, this.x - PADDLE_SPEED);
  }

  moveRight(): void {
    this.x = Math.min(CANVAS_WIDTH - this.width, this.x + PADDLE_SPEED);
  }

  moveTo(mouseX: number): void {
    this.x = Math.max(0, Math.min(CANVAS_WIDTH - this.width, mouseX - this.width / 2));
  }

  widen(): void {
    const center = this.x + this.width / 2;
    this.width = Math.floor(this.baseWidth * 1.5);
    this.x = Math.max(0, Math.min(CANVAS_WIDTH - this.width, center - this.width / 2));
  }

  resetWidth(): void {
    const center = this.x + this.width / 2;
    this.width = this.baseWidth;
    this.x = Math.max(0, Math.min(CANVAS_WIDTH - this.width, center - this.width / 2));
  }

  reset(): void {
    this.width = this.baseWidth;
    this.x = (CANVAS_WIDTH - this.width) / 2;
    this.isMagnet = false;
  }
}
```

**Step 4: Run tests — expect PASS**

```bash
npx vitest run src/lib/__tests__/Paddle.test.ts
```

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: implement Paddle class with movement and widen powerup"
```

---

## Task 5: Brick Class

**Files:**
- Create: `src/lib/engine/Brick.ts`
- Test: `src/lib/__tests__/Brick.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/Brick.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { Brick } from '../engine/Brick';
import { BRICK_WIDTH, BRICK_HEIGHT, BRICK_PADDING, BRICK_OFFSET_TOP, BRICK_OFFSET_LEFT } from '../engine/constants';

describe('Brick', () => {
  it('computes position from row and col', () => {
    const brick = new Brick('normal', 0, 0);
    expect(brick.x).toBe(BRICK_OFFSET_LEFT);
    expect(brick.y).toBe(BRICK_OFFSET_TOP);
  });

  it('normal brick has 1 HP', () => {
    const brick = new Brick('normal', 0, 0);
    expect(brick.hp).toBe(1);
  });

  it('hard brick has 3 HP', () => {
    const brick = new Brick('hard', 0, 0);
    expect(brick.hp).toBe(3);
  });

  it('hit reduces HP and returns true when destroyed', () => {
    const brick = new Brick('normal', 0, 0);
    expect(brick.hit()).toBe(true);
    expect(brick.hp).toBe(0);
  });

  it('hard brick survives multiple hits', () => {
    const brick = new Brick('hard', 0, 0);
    expect(brick.hit()).toBe(false);
    expect(brick.hit()).toBe(false);
    expect(brick.hit()).toBe(true);
  });

  it('unbreakable brick is never destroyed', () => {
    const brick = new Brick('unbreakable', 0, 0);
    expect(brick.hit()).toBe(false);
    expect(brick.hp).toBe(Infinity);
  });

  it('gold brick always drops powerup', () => {
    const brick = new Brick('gold', 0, 0);
    expect(brick.dropsPowerup).toBe(true);
  });

  it('isAlive returns false when HP is 0', () => {
    const brick = new Brick('normal', 0, 0);
    brick.hit();
    expect(brick.isAlive).toBe(false);
  });
});
```

**Step 2: Run tests — expect FAIL**

**Step 3: Implement Brick**

Create `src/lib/engine/Brick.ts`:
```ts
import type { BrickType } from './types';
import { BRICK_WIDTH, BRICK_HEIGHT, BRICK_PADDING, BRICK_OFFSET_TOP, BRICK_OFFSET_LEFT, BRICK_HP, BRICK_SCORES } from './constants';

export class Brick {
  type: BrickType;
  x: number;
  y: number;
  width = BRICK_WIDTH;
  height = BRICK_HEIGHT;
  hp: number;
  maxHp: number;
  score: number;
  dropsPowerup: boolean;

  constructor(type: BrickType, row: number, col: number) {
    this.type = type;
    this.x = BRICK_OFFSET_LEFT + col * (BRICK_WIDTH + BRICK_PADDING);
    this.y = BRICK_OFFSET_TOP + row * (BRICK_HEIGHT + BRICK_PADDING);
    this.hp = BRICK_HP[type];
    this.maxHp = this.hp;
    this.score = BRICK_SCORES[type];
    this.dropsPowerup = type === 'gold';
  }

  hit(): boolean {
    if (this.type === 'unbreakable') return false;
    this.hp--;
    return this.hp <= 0;
  }

  get isAlive(): boolean {
    return this.hp > 0;
  }
}
```

**Step 4: Run tests — expect PASS**

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: implement Brick class with types and HP system"
```

---

## Task 6: Collision Detection

**Files:**
- Create: `src/lib/engine/Collision.ts`
- Test: `src/lib/__tests__/Collision.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/Collision.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { ballRectCollision, getBallPaddleReflectAngle } from '../engine/Collision';
import { Ball } from '../engine/Ball';
import { PADDLE_WIDTH } from '../engine/constants';

describe('Collision', () => {
  it('detects ball-rect collision when overlapping', () => {
    const ball = new Ball(50, 50);
    ball.radius = 6;
    const rect = { x: 44, y: 44, width: 20, height: 20 };
    expect(ballRectCollision(ball, rect)).toBe(true);
  });

  it('no collision when ball is far from rect', () => {
    const ball = new Ball(200, 200);
    ball.radius = 6;
    const rect = { x: 0, y: 0, width: 20, height: 20 };
    expect(ballRectCollision(ball, rect)).toBe(false);
  });

  it('paddle reflect angle is steeper at edges', () => {
    // Hit left edge
    const leftAngle = getBallPaddleReflectAngle(5, 0, PADDLE_WIDTH);
    // Hit center
    const centerAngle = getBallPaddleReflectAngle(PADDLE_WIDTH / 2, 0, PADDLE_WIDTH);
    // Hit right edge
    const rightAngle = getBallPaddleReflectAngle(PADDLE_WIDTH - 5, 0, PADDLE_WIDTH);

    // Center should go straight up (angle close to -PI/2)
    expect(centerAngle).toBeCloseTo(-Math.PI / 2, 1);
    // Left edge should go left (angle < -PI/2)
    expect(leftAngle).toBeLessThan(centerAngle);
    // Right edge should go right (angle > -PI/2)
    expect(rightAngle).toBeGreaterThan(centerAngle);
  });
});
```

**Step 2: Run tests — expect FAIL**

**Step 3: Implement Collision**

Create `src/lib/engine/Collision.ts`:
```ts
import type { Ball } from './Ball';
import type { Rect } from './types';

export function ballRectCollision(ball: Ball, rect: Rect): boolean {
  const closestX = Math.max(rect.x, Math.min(ball.x, rect.x + rect.width));
  const closestY = Math.max(rect.y, Math.min(ball.y, rect.y + rect.height));
  const dx = ball.x - closestX;
  const dy = ball.y - closestY;
  return (dx * dx + dy * dy) <= (ball.radius * ball.radius);
}

export function getBallPaddleReflectAngle(
  hitX: number,
  paddleX: number,
  paddleWidth: number
): number {
  // Map hit position to -1..1 range
  const relative = (hitX - paddleX) / paddleWidth;
  const clamped = Math.max(0, Math.min(1, relative));
  // Map to angle: left edge = -150deg, center = -90deg, right edge = -30deg
  const maxAngle = Math.PI / 3; // 60 degrees from vertical
  const angle = -Math.PI / 2 + (clamped - 0.5) * 2 * maxAngle;
  return angle;
}

export function resolveBallBrickCollision(ball: Ball, rect: Rect): 'top' | 'bottom' | 'left' | 'right' {
  const overlapLeft = (ball.x + ball.radius) - rect.x;
  const overlapRight = (rect.x + rect.width) - (ball.x - ball.radius);
  const overlapTop = (ball.y + ball.radius) - rect.y;
  const overlapBottom = (rect.y + rect.height) - (ball.y - ball.radius);

  const minOverlapX = Math.min(overlapLeft, overlapRight);
  const minOverlapY = Math.min(overlapTop, overlapBottom);

  if (minOverlapX < minOverlapY) {
    ball.vx = overlapLeft < overlapRight ? -Math.abs(ball.vx) : Math.abs(ball.vx);
    return overlapLeft < overlapRight ? 'left' : 'right';
  } else {
    ball.vy = overlapTop < overlapBottom ? -Math.abs(ball.vy) : Math.abs(ball.vy);
    return overlapTop < overlapBottom ? 'top' : 'bottom';
  }
}
```

**Step 4: Run tests — expect PASS**

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: implement collision detection with ball-rect and paddle reflection"
```

---

## Task 7: Powerup Class

**Files:**
- Create: `src/lib/engine/Powerup.ts`
- Test: `src/lib/__tests__/Powerup.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/Powerup.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { Powerup } from '../engine/Powerup';
import { POWERUP_FALL_SPEED, CANVAS_HEIGHT } from '../engine/constants';

describe('Powerup', () => {
  it('falls downward each update', () => {
    const p = new Powerup('widen', 100, 50);
    const startY = p.y;
    p.update();
    expect(p.y).toBe(startY + POWERUP_FALL_SPEED);
  });

  it('detects when fallen off screen', () => {
    const p = new Powerup('life', 100, CANVAS_HEIGHT + 10);
    expect(p.isOut()).toBe(true);
  });

  it('stores correct type', () => {
    const p = new Powerup('multiball', 200, 100);
    expect(p.type).toBe('multiball');
  });
});
```

**Step 2: Run tests — expect FAIL**

**Step 3: Implement Powerup**

Create `src/lib/engine/Powerup.ts`:
```ts
import type { PowerupType } from './types';
import { POWERUP_WIDTH, POWERUP_HEIGHT, POWERUP_FALL_SPEED, CANVAS_HEIGHT } from './constants';

export class Powerup {
  type: PowerupType;
  x: number;
  y: number;
  width = POWERUP_WIDTH;
  height = POWERUP_HEIGHT;
  glowPhase = Math.random() * Math.PI * 2;

  constructor(type: PowerupType, x: number, y: number) {
    this.type = type;
    this.x = x;
    this.y = y;
  }

  update(): void {
    this.y += POWERUP_FALL_SPEED;
    this.glowPhase += 0.1;
  }

  isOut(): boolean {
    return this.y > CANVAS_HEIGHT;
  }
}

const POWERUP_TYPES: PowerupType[] = ['widen', 'multiball', 'slow', 'pierce', 'life', 'magnet'];

export function randomPowerupType(): PowerupType {
  return POWERUP_TYPES[Math.floor(Math.random() * POWERUP_TYPES.length)];
}
```

**Step 4: Run tests — expect PASS**

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: implement Powerup class with fall mechanics"
```

---

## Task 8: Particle System

**Files:**
- Create: `src/lib/engine/Particle.ts`
- Test: `src/lib/__tests__/Particle.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/Particle.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { Particle, ParticleSystem } from '../engine/Particle';

describe('Particle', () => {
  it('moves by velocity each update', () => {
    const p = new Particle(100, 100, 2, -1, 1, '#fff', 5);
    p.update();
    expect(p.x).toBe(102);
    expect(p.y).toBe(99);
  });

  it('alpha decreases over time', () => {
    const p = new Particle(0, 0, 0, 0, 1, '#fff', 5);
    p.update();
    expect(p.alpha).toBeLessThan(1);
  });

  it('isDead when alpha reaches 0', () => {
    const p = new Particle(0, 0, 0, 0, 1, '#fff', 5);
    p.alpha = 0;
    expect(p.isDead()).toBe(true);
  });
});

describe('ParticleSystem', () => {
  it('emits particles at position', () => {
    const ps = new ParticleSystem();
    ps.emit(100, 100, '#ff0000', 10);
    expect(ps.particles.length).toBe(10);
  });

  it('removes dead particles on update', () => {
    const ps = new ParticleSystem();
    ps.emit(0, 0, '#fff', 5);
    // Force all particles to die
    for (const p of ps.particles) p.alpha = 0;
    ps.update();
    expect(ps.particles.length).toBe(0);
  });
});
```

**Step 2: Run tests — expect FAIL**

**Step 3: Implement Particle**

Create `src/lib/engine/Particle.ts`:
```ts
export class Particle {
  x: number;
  y: number;
  vx: number;
  vy: number;
  alpha: number;
  color: string;
  size: number;
  decay: number;

  constructor(x: number, y: number, vx: number, vy: number, alpha: number, color: string, size: number) {
    this.x = x;
    this.y = y;
    this.vx = vx;
    this.vy = vy;
    this.alpha = alpha;
    this.color = color;
    this.size = size;
    this.decay = 0.015 + Math.random() * 0.02;
  }

  update(): void {
    this.x += this.vx;
    this.y += this.vy;
    this.vy += 0.02; // slight gravity
    this.alpha -= this.decay;
    this.size *= 0.98;
  }

  isDead(): boolean {
    return this.alpha <= 0 || this.size < 0.5;
  }
}

export class ParticleSystem {
  particles: Particle[] = [];

  emit(x: number, y: number, color: string, count: number): void {
    for (let i = 0; i < count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const speed = 0.5 + Math.random() * 2;
      const vx = Math.cos(angle) * speed;
      const vy = Math.sin(angle) * speed;
      const size = 2 + Math.random() * 4;
      this.particles.push(new Particle(x, y, vx, vy, 1, color, size));
    }
  }

  emitBackground(canvasWidth: number, canvasHeight: number): void {
    if (this.particles.length < 50 && Math.random() < 0.1) {
      const x = Math.random() * canvasWidth;
      const y = Math.random() * canvasHeight;
      const vx = (Math.random() - 0.5) * 0.3;
      const vy = -0.1 - Math.random() * 0.3;
      this.particles.push(new Particle(x, y, vx, vy, 0.3 + Math.random() * 0.3, '#ffffff', 1 + Math.random() * 3));
    }
  }

  update(): void {
    for (const p of this.particles) p.update();
    this.particles = this.particles.filter(p => !p.isDead());
  }
}
```

**Step 4: Run tests — expect PASS**

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: implement particle system for visual effects"
```

---

## Task 9: Level Data

**Files:**
- Create: `src/lib/levels/levels.ts`
- Test: `src/lib/__tests__/levels.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/levels.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { LEVELS } from '../levels/levels';

describe('Levels', () => {
  it('has 5 levels', () => {
    expect(LEVELS.length).toBe(5);
  });

  it('each level has a name and bricks', () => {
    for (const level of LEVELS) {
      expect(level.name).toBeTruthy();
      expect(level.bricks.length).toBeGreaterThan(0);
    }
  });

  it('level 1 has only normal bricks', () => {
    for (const brick of LEVELS[0].bricks) {
      expect(brick.type).toBe('normal');
    }
  });

  it('level 2 includes hard bricks', () => {
    const hasHard = LEVELS[1].bricks.some(b => b.type === 'hard');
    expect(hasHard).toBe(true);
  });
});
```

**Step 2: Run tests — expect FAIL**

**Step 3: Implement level data**

Create `src/lib/levels/levels.ts`:
```ts
import type { LevelData, BrickData } from '../engine/types';

function row(rowIdx: number, types: (string | null)[]): BrickData[] {
  return types
    .map((type, col) => type ? { type: type as BrickData['type'], row: rowIdx, col } : null)
    .filter((b): b is BrickData => b !== null);
}

const N = 'normal';
const H = 'hard';
const G = 'gold';
const U = 'unbreakable';
const _ = null;

export const LEVELS: LevelData[] = [
  {
    name: '春の目覚め',
    bricks: [
      ...row(0, [N, N, N, N, N, N, N, N, N, N]),
      ...row(1, [N, N, N, N, N, N, N, N, N, N]),
      ...row(2, [N, N, N, N, N, N, N, N, N, N]),
      ...row(3, [N, N, N, N, N, N, N, N, N, N]),
    ],
  },
  {
    name: '夕暮れの影',
    bricks: [
      ...row(0, [H, N, N, H, N, N, H, N, N, H]),
      ...row(1, [N, H, N, N, H, H, N, N, H, N]),
      ...row(2, [N, N, H, N, N, N, N, H, N, N]),
      ...row(3, [H, N, N, N, N, N, N, N, N, H]),
      ...row(4, [N, N, N, N, N, N, N, N, N, N]),
    ],
  },
  {
    name: '雨の記憶',
    bricks: [
      ...row(0, [U, N, N, U, N, N, U, N, N, U]),
      ...row(1, [N, N, H, N, N, N, N, H, N, N]),
      ...row(2, [N, H, N, N, H, H, N, N, H, N]),
      ...row(3, [N, N, N, N, N, N, N, N, N, N]),
      ...row(4, [H, N, N, N, N, N, N, N, N, H]),
    ],
  },
  {
    name: '黄金の刻',
    bricks: [
      ...row(0, [G, N, G, N, G, G, N, G, N, G]),
      ...row(1, [N, G, N, N, N, N, N, N, G, N]),
      ...row(2, [H, H, H, H, H, H, H, H, H, H]),
      ...row(3, [N, N, N, N, N, N, N, N, N, N]),
      ...row(4, [N, N, H, N, G, G, N, H, N, N]),
      ...row(5, [N, N, N, N, N, N, N, N, N, N]),
    ],
  },
  {
    name: '星空の約束',
    bricks: [
      ...row(0, [U, H, G, H, U, U, H, G, H, U]),
      ...row(1, [H, H, H, H, H, H, H, H, H, H]),
      ...row(2, [N, G, N, H, N, N, H, N, G, N]),
      ...row(3, [H, N, N, N, G, G, N, N, N, H]),
      ...row(4, [U, N, H, N, N, N, N, H, N, U]),
      ...row(5, [N, N, N, H, N, N, H, N, N, N]),
      ...row(6, [N, N, N, N, N, N, N, N, N, N]),
    ],
  },
];
```

**Step 4: Run tests — expect PASS**

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: add 5 Shinkai-themed levels with progressive difficulty"
```

---

## Task 10: Audio Manager

**Files:**
- Create: `src/lib/audio/AudioManager.ts`
- Test: `src/lib/__tests__/AudioManager.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/AudioManager.test.ts`:
```ts
import { describe, it, expect, vi } from 'vitest';
import { AudioManager } from '../audio/AudioManager';

// Mock Web Audio API
const mockOscillator = {
  type: 'sine' as OscillatorType,
  frequency: { setValueAtTime: vi.fn(), exponentialRampToValueAtTime: vi.fn() },
  connect: vi.fn(),
  start: vi.fn(),
  stop: vi.fn(),
};
const mockGain = {
  gain: { setValueAtTime: vi.fn(), exponentialRampToValueAtTime: vi.fn() },
  connect: vi.fn(),
};
const mockCtx = {
  createOscillator: vi.fn(() => ({ ...mockOscillator })),
  createGain: vi.fn(() => ({ ...mockGain })),
  destination: {},
  currentTime: 0,
};

describe('AudioManager', () => {
  it('can be muted and unmuted', () => {
    const am = new AudioManager();
    am.mute();
    expect(am.isMuted).toBe(true);
    am.unmute();
    expect(am.isMuted).toBe(false);
  });

  it('has playable sound methods', () => {
    const am = new AudioManager();
    // Should not throw even without AudioContext
    expect(() => am.playBounce()).not.toThrow();
    expect(() => am.playBrickBreak()).not.toThrow();
    expect(() => am.playPowerup()).not.toThrow();
    expect(() => am.playLoseLife()).not.toThrow();
  });
});
```

**Step 2: Run tests — expect FAIL**

**Step 3: Implement AudioManager**

Create `src/lib/audio/AudioManager.ts`:
```ts
export class AudioManager {
  private ctx: AudioContext | null = null;
  isMuted = false;

  private getCtx(): AudioContext | null {
    if (this.isMuted) return null;
    if (!this.ctx) {
      try {
        this.ctx = new AudioContext();
      } catch {
        return null;
      }
    }
    return this.ctx;
  }

  mute(): void { this.isMuted = true; }
  unmute(): void { this.isMuted = false; }
  toggleMute(): void { this.isMuted = !this.isMuted; }

  private playTone(freq: number, duration: number, type: OscillatorType = 'sine', volume = 0.3): void {
    const ctx = this.getCtx();
    if (!ctx) return;
    const osc = ctx.createOscillator();
    const gain = ctx.createGain();
    osc.type = type;
    osc.frequency.setValueAtTime(freq, ctx.currentTime);
    gain.gain.setValueAtTime(volume, ctx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + duration);
    osc.connect(gain);
    gain.connect(ctx.destination);
    osc.start(ctx.currentTime);
    osc.stop(ctx.currentTime + duration);
  }

  playBounce(): void { this.playTone(440, 0.1, 'sine', 0.2); }
  playWallBounce(): void { this.playTone(330, 0.08, 'sine', 0.15); }
  playBrickBreak(): void { this.playTone(600, 0.15, 'square', 0.2); }
  playPowerup(): void {
    this.playTone(523, 0.1, 'sine', 0.25);
    setTimeout(() => this.playTone(659, 0.1, 'sine', 0.25), 80);
    setTimeout(() => this.playTone(784, 0.15, 'sine', 0.25), 160);
  }
  playLoseLife(): void { this.playTone(200, 0.4, 'sawtooth', 0.3); }
  playGameOver(): void {
    this.playTone(400, 0.3, 'sawtooth', 0.3);
    setTimeout(() => this.playTone(300, 0.3, 'sawtooth', 0.3), 250);
    setTimeout(() => this.playTone(200, 0.5, 'sawtooth', 0.3), 500);
  }
  playVictory(): void {
    this.playTone(523, 0.15, 'sine', 0.3);
    setTimeout(() => this.playTone(659, 0.15, 'sine', 0.3), 120);
    setTimeout(() => this.playTone(784, 0.15, 'sine', 0.3), 240);
    setTimeout(() => this.playTone(1047, 0.3, 'sine', 0.3), 360);
  }
  playLevelClear(): void {
    this.playTone(660, 0.2, 'sine', 0.3);
    setTimeout(() => this.playTone(880, 0.3, 'sine', 0.3), 150);
  }
}
```

**Step 4: Run tests — expect PASS**

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: implement AudioManager with synthesized sound effects"
```

---

## Task 11: Game Store (Svelte State)

**Files:**
- Create: `src/lib/stores/gameStore.ts`
- Test: `src/lib/__tests__/gameStore.test.ts`

**Step 1: Write failing tests**

Create `src/lib/__tests__/gameStore.test.ts`:
```ts
import { describe, it, expect, beforeEach } from 'vitest';
import { gameState, score, lives, level, comboMultiplier, resetGameState, addScore } from '../stores/gameStore';
import { get } from 'svelte/store';

describe('gameStore', () => {
  beforeEach(() => {
    resetGameState();
  });

  it('initial state is menu', () => {
    expect(get(gameState)).toBe('menu');
  });

  it('initial score is 0', () => {
    expect(get(score)).toBe(0);
  });

  it('initial lives is 3', () => {
    expect(get(lives)).toBe(3);
  });

  it('addScore applies combo multiplier', () => {
    comboMultiplier.set(2);
    addScore(10);
    expect(get(score)).toBe(20);
  });

  it('resetGameState resets everything', () => {
    score.set(999);
    lives.set(0);
    level.set(5);
    resetGameState();
    expect(get(score)).toBe(0);
    expect(get(lives)).toBe(3);
    expect(get(level)).toBe(1);
  });
});
```

**Step 2: Run tests — expect FAIL**

**Step 3: Implement gameStore**

Create `src/lib/stores/gameStore.ts`:
```ts
import { writable, get } from 'svelte/store';
import type { GameState, LeaderboardEntry } from '../engine/types';
import { INITIAL_LIVES } from '../engine/constants';

export const gameState = writable<GameState>('menu');
export const score = writable(0);
export const lives = writable(INITIAL_LIVES);
export const level = writable(1);
export const comboMultiplier = writable(1);

export function addScore(points: number): void {
  const mult = get(comboMultiplier);
  score.update(s => s + points * mult);
}

export function resetGameState(): void {
  gameState.set('menu');
  score.set(0);
  lives.set(INITIAL_LIVES);
  level.set(1);
  comboMultiplier.set(1);
}

// Leaderboard
const LEADERBOARD_KEY = 'breakout-leaderboard';

export function getLeaderboard(): LeaderboardEntry[] {
  try {
    const data = localStorage.getItem(LEADERBOARD_KEY);
    return data ? JSON.parse(data) : [];
  } catch {
    return [];
  }
}

export function addLeaderboardEntry(entry: LeaderboardEntry): void {
  const lb = getLeaderboard();
  lb.push(entry);
  lb.sort((a, b) => b.score - a.score);
  const top10 = lb.slice(0, 10);
  localStorage.setItem(LEADERBOARD_KEY, JSON.stringify(top10));
}

export function isHighScore(s: number): boolean {
  const lb = getLeaderboard();
  return lb.length < 10 || s > (lb[lb.length - 1]?.score ?? 0);
}
```

**Step 4: Run tests — expect PASS**

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: implement game store with score, lives, combo, and leaderboard"
```

---

## Task 12: Shinkai Renderer

**Files:**
- Create: `src/lib/renderer/ShinkaRenderer.ts`

This is visual-only, no unit tests. Verified visually in Task 14.

**Step 1: Implement renderer**

Create `src/lib/renderer/ShinkaRenderer.ts`:
```ts
import type { Ball } from '../engine/Ball';
import type { Paddle } from '../engine/Paddle';
import type { Brick } from '../engine/Brick';
import type { Powerup } from '../engine/Powerup';
import type { ParticleSystem } from '../engine/Particle';
import { CANVAS_WIDTH, CANVAS_HEIGHT } from '../engine/constants';

// Shinkai color palette
const COLORS = {
  skyTop: '#87CEEB',
  skyBottom: '#FFB6A3',
  brickNormal: ['#F4A6C0', '#A7C7E7', '#C3B1E1', '#F7C59F'],
  brickHard: ['#8B6F9E', '#5B7FA5', '#A0526B'],
  brickGold: '#FFD700',
  brickUnbreakable: '#4A5568',
  paddle: '#2D3748',
  paddleGlow: 'rgba(135, 206, 235, 0.4)',
  ball: '#FFFFFF',
  ballGlow: 'rgba(255, 255, 255, 0.6)',
};

const POWERUP_COLORS: Record<string, string> = {
  widen: '#F4A6C0',
  multiball: '#A7C7E7',
  slow: '#C3B1E1',
  pierce: '#FF6B6B',
  life: '#FF5E7A',
  magnet: '#7EC8E3',
};

export class ShinkaRenderer {
  private ctx: CanvasRenderingContext2D;
  private bgGradient: CanvasGradient;

  constructor(ctx: CanvasRenderingContext2D) {
    this.ctx = ctx;
    this.bgGradient = ctx.createLinearGradient(0, 0, 0, CANVAS_HEIGHT);
    this.bgGradient.addColorStop(0, COLORS.skyTop);
    this.bgGradient.addColorStop(1, COLORS.skyBottom);
  }

  clear(): void {
    this.ctx.fillStyle = this.bgGradient;
    this.ctx.fillRect(0, 0, CANVAS_WIDTH, CANVAS_HEIGHT);
  }

  drawBrick(brick: Brick): void {
    if (!brick.isAlive) return;
    const ctx = this.ctx;
    let color: string;

    switch (brick.type) {
      case 'normal':
        color = COLORS.brickNormal[(brick.x * 7 + brick.y * 13) % COLORS.brickNormal.length];
        break;
      case 'hard': {
        const hpRatio = brick.hp / brick.maxHp;
        color = COLORS.brickHard[Math.floor((1 - hpRatio) * (COLORS.brickHard.length - 1))];
        break;
      }
      case 'gold':
        color = COLORS.brickGold;
        break;
      case 'unbreakable':
        color = COLORS.brickUnbreakable;
        break;
    }

    // Glow
    ctx.save();
    ctx.shadowColor = color;
    ctx.shadowBlur = 8;
    ctx.fillStyle = color;
    ctx.globalAlpha = 0.9;
    ctx.beginPath();
    ctx.roundRect(brick.x, brick.y, brick.width, brick.height, 4);
    ctx.fill();
    ctx.restore();

    // Inner highlight
    ctx.save();
    ctx.globalAlpha = 0.3;
    ctx.fillStyle = '#ffffff';
    ctx.beginPath();
    ctx.roundRect(brick.x + 2, brick.y + 2, brick.width - 4, brick.height / 2 - 2, 2);
    ctx.fill();
    ctx.restore();
  }

  drawPaddle(paddle: Paddle): void {
    const ctx = this.ctx;

    // Glow under paddle
    ctx.save();
    ctx.shadowColor = COLORS.paddleGlow;
    ctx.shadowBlur = 20;
    ctx.fillStyle = COLORS.paddle;
    ctx.beginPath();
    ctx.roundRect(paddle.x, paddle.y, paddle.width, paddle.height, 7);
    ctx.fill();
    ctx.restore();

    // Edge glow
    ctx.save();
    const edgeGrad = ctx.createLinearGradient(paddle.x, paddle.y, paddle.x, paddle.y + paddle.height);
    edgeGrad.addColorStop(0, 'rgba(135, 206, 235, 0.6)');
    edgeGrad.addColorStop(1, 'rgba(135, 206, 235, 0)');
    ctx.fillStyle = edgeGrad;
    ctx.beginPath();
    ctx.roundRect(paddle.x, paddle.y, paddle.width, paddle.height, 7);
    ctx.fill();
    ctx.restore();
  }

  drawBall(ball: Ball): void {
    const ctx = this.ctx;

    // Trail
    for (const t of ball.trail) {
      ctx.save();
      ctx.globalAlpha = t.alpha * 0.4;
      ctx.fillStyle = COLORS.ballGlow;
      ctx.beginPath();
      ctx.arc(t.x, t.y, ball.radius * 0.8, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    }

    // Ball glow
    ctx.save();
    ctx.shadowColor = ball.isPiercing ? '#FF6B6B' : COLORS.ballGlow;
    ctx.shadowBlur = 15;
    ctx.fillStyle = ball.isPiercing ? '#FF6B6B' : COLORS.ball;
    ctx.beginPath();
    ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  }

  drawPowerup(powerup: Powerup): void {
    const ctx = this.ctx;
    const color = POWERUP_COLORS[powerup.type] || '#fff';
    const glow = 0.6 + Math.sin(powerup.glowPhase) * 0.3;

    ctx.save();
    ctx.globalAlpha = glow;
    ctx.shadowColor = color;
    ctx.shadowBlur = 12;
    ctx.fillStyle = color;
    ctx.beginPath();
    ctx.arc(
      powerup.x + powerup.width / 2,
      powerup.y + powerup.height / 2,
      powerup.width / 2,
      0,
      Math.PI * 2
    );
    ctx.fill();
    ctx.restore();

    // Icon text
    const icons: Record<string, string> = {
      widen: 'W', multiball: 'M', slow: 'S', pierce: 'P', life: '♥', magnet: '⊕'
    };
    ctx.save();
    ctx.fillStyle = '#fff';
    ctx.font = 'bold 12px sans-serif';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(icons[powerup.type] || '?', powerup.x + powerup.width / 2, powerup.y + powerup.height / 2);
    ctx.restore();
  }

  drawParticles(ps: ParticleSystem): void {
    const ctx = this.ctx;
    for (const p of ps.particles) {
      ctx.save();
      ctx.globalAlpha = p.alpha;
      ctx.fillStyle = p.color;
      ctx.shadowColor = p.color;
      ctx.shadowBlur = 6;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    }
  }

  drawHUD(score: number, lives: number, level: number, combo: number): void {
    const ctx = this.ctx;
    ctx.save();
    ctx.fillStyle = 'rgba(255, 255, 255, 0.85)';
    ctx.font = '16px "Segoe UI", system-ui, sans-serif';
    ctx.textAlign = 'left';
    ctx.fillText(`スコア: ${score}`, 16, 28);
    ctx.textAlign = 'center';
    ctx.fillText(`レベル ${level}`, CANVAS_WIDTH / 2, 28);
    ctx.textAlign = 'right';
    // Hearts for lives
    let heartsText = '';
    for (let i = 0; i < lives; i++) heartsText += '♥ ';
    ctx.fillStyle = '#FF5E7A';
    ctx.fillText(heartsText.trim(), CANVAS_WIDTH - 16, 28);

    if (combo > 1) {
      ctx.fillStyle = 'rgba(255, 215, 0, 0.9)';
      ctx.textAlign = 'center';
      ctx.font = 'bold 20px "Segoe UI", system-ui, sans-serif';
      ctx.fillText(`×${combo} COMBO!`, CANVAS_WIDTH / 2, 50);
    }
    ctx.restore();
  }
}
```

**Step 2: Commit**

```bash
git add -A
git commit -m "feat: implement Shinkai-style renderer with glow effects and gradients"
```

---

## Task 13: Game Engine (Main Loop)

**Files:**
- Create: `src/lib/engine/Game.ts`

This is the core orchestrator — ties all pieces together. Tested via integration (visual) in Task 14.

**Step 1: Implement Game class**

Create `src/lib/engine/Game.ts`:
```ts
import { Ball } from './Ball';
import { Paddle } from './Paddle';
import { Brick } from './Brick';
import { Powerup, randomPowerupType } from './Powerup';
import { ParticleSystem } from './Particle';
import { ballRectCollision, getBallPaddleReflectAngle, resolveBallBrickCollision } from './Collision';
import { ShinkaRenderer } from '../renderer/ShinkaRenderer';
import { AudioManager } from '../audio/AudioManager';
import { LEVELS } from '../levels/levels';
import { CANVAS_WIDTH, CANVAS_HEIGHT, BALL_SPEED, PADDLE_Y, BALL_RADIUS } from './constants';
import type { LevelData, PowerupType } from './types';
import { gameState, score, lives, level, comboMultiplier, addScore } from '../stores/gameStore';
import { get } from 'svelte/store';

export class Game {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private renderer: ShinkaRenderer;
  audio: AudioManager;

  paddle: Paddle;
  balls: Ball[] = [];
  bricks: Brick[] = [];
  powerups: Powerup[] = [];
  particles: ParticleSystem;
  bgParticles: ParticleSystem;

  private animFrameId = 0;
  private keys: Set<string> = new Set();
  private mouseX = CANVAS_WIDTH / 2;
  private useMouseControl = false;

  private combo = 0;
  private magnetBall: Ball | null = null;

  // Active powerup timers
  private activeTimers: Array<{ type: PowerupType; endTime: number }> = [];
  private gameTime = 0;

  // Custom level for editor
  customLevel: LevelData | null = null;

  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d')!;
    this.renderer = new ShinkaRenderer(this.ctx);
    this.audio = new AudioManager();
    this.paddle = new Paddle();
    this.particles = new ParticleSystem();
    this.bgParticles = new ParticleSystem();
  }

  init(): void {
    this.bindInput();
    this.loadLevel(get(level));
    this.resetBall();
  }

  private bindInput(): void {
    window.addEventListener('keydown', (e) => {
      this.keys.add(e.key);
      if (e.key === 'Escape') {
        const st = get(gameState);
        if (st === 'playing') gameState.set('paused');
        else if (st === 'paused') gameState.set('playing');
      }
      if (e.key === ' ' || e.key === 'Enter') {
        if (this.magnetBall) {
          this.launchMagnetBall();
        } else if (this.balls.length > 0 && this.balls[0].vx === 0 && this.balls[0].vy === 0) {
          this.balls[0].launch(-Math.PI / 2 + (Math.random() - 0.5) * 0.3, BALL_SPEED);
        }
      }
    });
    window.addEventListener('keyup', (e) => this.keys.delete(e.key));
    this.canvas.addEventListener('mousemove', (e) => {
      const rect = this.canvas.getBoundingClientRect();
      const scaleX = CANVAS_WIDTH / rect.width;
      this.mouseX = (e.clientX - rect.left) * scaleX;
      this.useMouseControl = true;
    });
    this.canvas.addEventListener('click', () => {
      if (this.magnetBall) {
        this.launchMagnetBall();
      } else if (this.balls.length > 0 && this.balls[0].vx === 0 && this.balls[0].vy === 0) {
        this.balls[0].launch(-Math.PI / 2 + (Math.random() - 0.5) * 0.3, BALL_SPEED);
      }
    });
  }

  private launchMagnetBall(): void {
    if (!this.magnetBall) return;
    this.magnetBall.launch(-Math.PI / 2 + (Math.random() - 0.5) * 0.3, BALL_SPEED);
    this.magnetBall = null;
    this.paddle.isMagnet = false;
  }

  loadLevel(levelNum: number): void {
    const data = this.customLevel || LEVELS[levelNum - 1];
    if (!data) return;
    this.bricks = data.bricks.map(b => new Brick(b.type, b.row, b.col));
    this.powerups = [];
    this.activeTimers = [];
    this.paddle.reset();
    this.combo = 0;
    comboMultiplier.set(1);
  }

  resetBall(): void {
    this.balls = [new Ball(CANVAS_WIDTH / 2, PADDLE_Y - BALL_RADIUS - 2)];
    this.magnetBall = null;
  }

  start(): void {
    gameState.set('playing');
    this.gameTime = performance.now();
    this.loop();
  }

  stop(): void {
    cancelAnimationFrame(this.animFrameId);
  }

  private loop = (): void => {
    const st = get(gameState);
    if (st === 'playing') {
      const now = performance.now();
      this.gameTime = now;
      this.update();
    }
    this.render();
    this.animFrameId = requestAnimationFrame(this.loop);
  };

  private update(): void {
    // Input
    if (this.useMouseControl) {
      this.paddle.moveTo(this.mouseX);
    }
    if (this.keys.has('ArrowLeft') || this.keys.has('a') || this.keys.has('A')) {
      this.paddle.moveLeft();
      this.useMouseControl = false;
    }
    if (this.keys.has('ArrowRight') || this.keys.has('d') || this.keys.has('D')) {
      this.paddle.moveRight();
      this.useMouseControl = false;
    }

    // Magnet ball follows paddle
    if (this.magnetBall) {
      this.magnetBall.x = this.paddle.x + this.paddle.width / 2;
      this.magnetBall.y = this.paddle.y - BALL_RADIUS - 2;
    }

    // Update balls
    for (const ball of this.balls) {
      if (ball === this.magnetBall) continue;
      ball.update();

      // Paddle collision
      if (ballRectCollision(ball, {
        x: this.paddle.x,
        y: this.paddle.y,
        width: this.paddle.width,
        height: this.paddle.height,
      })) {
        if (this.paddle.isMagnet && !this.magnetBall) {
          this.magnetBall = ball;
          ball.vx = 0;
          ball.vy = 0;
          ball.x = this.paddle.x + this.paddle.width / 2;
          ball.y = this.paddle.y - BALL_RADIUS - 2;
        } else {
          const hitX = ball.x;
          const angle = getBallPaddleReflectAngle(hitX, this.paddle.x, this.paddle.width);
          const speed = Math.sqrt(ball.vx * ball.vx + ball.vy * ball.vy) || BALL_SPEED;
          ball.vx = Math.cos(angle) * speed;
          ball.vy = Math.sin(angle) * speed;
          ball.y = this.paddle.y - BALL_RADIUS - 1;
          this.combo = 0;
          comboMultiplier.set(1);
          this.audio.playBounce();
        }
      }

      // Brick collisions
      for (const brick of this.bricks) {
        if (!brick.isAlive) continue;
        if (ballRectCollision(ball, {
          x: brick.x,
          y: brick.y,
          width: brick.width,
          height: brick.height,
        })) {
          if (!ball.isPiercing) {
            resolveBallBrickCollision(ball, {
              x: brick.x,
              y: brick.y,
              width: brick.width,
              height: brick.height,
            });
          }

          const destroyed = brick.hit();
          if (destroyed) {
            this.combo++;
            comboMultiplier.set(Math.min(this.combo, 5));
            addScore(brick.score);
            this.audio.playBrickBreak();

            // Particles
            this.particles.emit(
              brick.x + brick.width / 2,
              brick.y + brick.height / 2,
              '#F4A6C0',
              12,
            );

            // Powerup drop
            if (brick.dropsPowerup || Math.random() < 0.15) {
              this.powerups.push(new Powerup(
                randomPowerupType(),
                brick.x + brick.width / 2 - 12,
                brick.y,
              ));
            }
          } else if (brick.type !== 'unbreakable') {
            this.audio.playWallBounce();
          }

          if (!ball.isPiercing) break; // Only collide with one brick per frame (unless piercing)
        }
      }
    }

    // Remove fallen balls
    this.balls = this.balls.filter(b => !b.isOut());
    if (this.balls.length === 0) {
      this.loseLife();
    }

    // Update powerups
    for (const p of this.powerups) p.update();

    // Powerup collection
    this.powerups = this.powerups.filter(p => {
      if (p.isOut()) return false;
      if (ballRectCollision(
        { x: this.paddle.x + this.paddle.width / 2, y: this.paddle.y, radius: this.paddle.width / 2, vx: 0, vy: 0 } as any,
        { x: p.x, y: p.y, width: p.width, height: p.height }
      )) {
        this.applyPowerup(p.type);
        this.audio.playPowerup();
        return false;
      }
      // Simple rect overlap check for paddle
      if (
        p.x < this.paddle.x + this.paddle.width &&
        p.x + p.width > this.paddle.x &&
        p.y < this.paddle.y + this.paddle.height &&
        p.y + p.height > this.paddle.y
      ) {
        this.applyPowerup(p.type);
        this.audio.playPowerup();
        return false;
      }
      return true;
    });

    // Check powerup timers
    this.activeTimers = this.activeTimers.filter(t => {
      if (this.gameTime >= t.endTime) {
        this.expirePowerup(t.type);
        return false;
      }
      return true;
    });

    // Update particles
    this.particles.update();
    this.bgParticles.emitBackground(CANVAS_WIDTH, CANVAS_HEIGHT);
    this.bgParticles.update();

    // Check level clear
    const breakable = this.bricks.filter(b => b.type !== 'unbreakable' && b.isAlive);
    if (breakable.length === 0) {
      this.nextLevel();
    }
  }

  private applyPowerup(type: PowerupType): void {
    const duration = {
      widen: 15000,
      slow: 10000,
      pierce: 8000,
      magnet: Infinity,
      multiball: Infinity,
      life: Infinity,
    };

    switch (type) {
      case 'widen':
        this.paddle.widen();
        this.activeTimers.push({ type, endTime: this.gameTime + duration.widen });
        break;
      case 'multiball':
        if (this.balls.length > 0) {
          const b = this.balls[0];
          for (let i = 0; i < 2; i++) {
            const nb = new Ball(b.x, b.y);
            const angle = -Math.PI / 2 + (i === 0 ? -0.5 : 0.5);
            nb.launch(angle, BALL_SPEED);
            this.balls.push(nb);
          }
        }
        break;
      case 'slow':
        for (const b of this.balls) {
          b.vx *= 0.7;
          b.vy *= 0.7;
        }
        this.activeTimers.push({ type, endTime: this.gameTime + duration.slow });
        break;
      case 'pierce':
        for (const b of this.balls) b.isPiercing = true;
        this.activeTimers.push({ type, endTime: this.gameTime + duration.pierce });
        break;
      case 'life':
        lives.update(l => l + 1);
        break;
      case 'magnet':
        this.paddle.isMagnet = true;
        break;
    }
  }

  private expirePowerup(type: PowerupType): void {
    switch (type) {
      case 'widen':
        this.paddle.resetWidth();
        break;
      case 'slow':
        for (const b of this.balls) {
          const speed = Math.sqrt(b.vx * b.vx + b.vy * b.vy);
          if (speed > 0) {
            const factor = BALL_SPEED / speed;
            b.vx *= factor;
            b.vy *= factor;
          }
        }
        break;
      case 'pierce':
        for (const b of this.balls) b.isPiercing = false;
        break;
    }
  }

  private loseLife(): void {
    lives.update(l => l - 1);
    this.audio.playLoseLife();
    if (get(lives) <= 0) {
      gameState.set('gameover');
      this.audio.playGameOver();
    } else {
      this.resetBall();
    }
  }

  private nextLevel(): void {
    const currentLevel = get(level);
    const maxLevel = this.customLevel ? 1 : LEVELS.length;
    this.audio.playLevelClear();
    this.particles.emit(CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2, '#FFD700', 40);

    if (currentLevel >= maxLevel) {
      gameState.set('victory');
      this.audio.playVictory();
    } else {
      level.update(l => l + 1);
      this.loadLevel(get(level));
      this.resetBall();
    }
  }

  private render(): void {
    this.renderer.clear();
    this.renderer.drawParticles(this.bgParticles);

    for (const brick of this.bricks) {
      this.renderer.drawBrick(brick);
    }

    for (const p of this.powerups) {
      this.renderer.drawPowerup(p);
    }

    for (const ball of this.balls) {
      this.renderer.drawBall(ball);
    }

    this.renderer.drawPaddle(this.paddle);
    this.renderer.drawParticles(this.particles);

    this.renderer.drawHUD(get(score), get(lives), get(level), get(comboMultiplier));

    // Launch hint
    if (this.balls.length > 0 && this.balls[0].vx === 0 && this.balls[0].vy === 0 && !this.magnetBall) {
      this.ctx.save();
      this.ctx.fillStyle = 'rgba(255, 255, 255, 0.6)';
      this.ctx.font = '14px "Segoe UI", system-ui, sans-serif';
      this.ctx.textAlign = 'center';
      this.ctx.fillText('クリックまたはスペースで発射', CANVAS_WIDTH / 2, CANVAS_HEIGHT / 2);
      this.ctx.restore();
    }
  }

  destroy(): void {
    this.stop();
    window.removeEventListener('keydown', () => {});
    window.removeEventListener('keyup', () => {});
  }
}
```

**Step 2: Commit**

```bash
git add -A
git commit -m "feat: implement Game engine with main loop, powerups, and level progression"
```

---

## Task 14: Svelte Components — GameCanvas + App Shell

**Files:**
- Modify: `src/App.svelte`
- Create: `src/components/GameCanvas.svelte`
- Create: `src/components/MainMenu.svelte`
- Create: `src/components/PauseMenu.svelte`
- Create: `src/components/HUD.svelte`

**Step 1: Create GameCanvas component**

Create `src/components/GameCanvas.svelte`:
```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { Game } from '../lib/engine/Game';
  import { gameState } from '../lib/stores/gameStore';
  import { CANVAS_WIDTH, CANVAS_HEIGHT } from '../lib/engine/constants';

  let canvas: HTMLCanvasElement;
  let game: Game;

  export function getGame(): Game | undefined {
    return game;
  }

  onMount(() => {
    game = new Game(canvas);
    game.init();
    game.start();
    return () => game.destroy();
  });
</script>

<canvas
  bind:this={canvas}
  width={CANVAS_WIDTH}
  height={CANVAS_HEIGHT}
  class="game-canvas"
></canvas>

<style>
  .game-canvas {
    display: block;
    max-width: 100%;
    height: auto;
    border-radius: 8px;
    box-shadow: 0 0 40px rgba(135, 206, 235, 0.3), 0 0 80px rgba(255, 182, 163, 0.15);
    cursor: none;
  }
</style>
```

**Step 2: Create MainMenu component**

Create `src/components/MainMenu.svelte`:
```svelte
<script lang="ts">
  import { gameState } from '../lib/stores/gameStore';

  function startGame() { gameState.set('playing'); }
  function openEditor() { gameState.set('editor'); }
  function openLeaderboard() { gameState.set('leaderboard'); }
</script>

<div class="menu-overlay">
  <div class="menu">
    <h1 class="title">ブレイクアウト</h1>
    <p class="subtitle">— 光と影の物語 —</p>
    <div class="buttons">
      <button on:click={startGame}>ゲーム開始</button>
      <button on:click={openEditor}>レベルエディタ</button>
      <button on:click={openLeaderboard}>ランキング</button>
    </div>
  </div>
</div>

<style>
  .menu-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(135deg, rgba(135, 206, 235, 0.9), rgba(255, 182, 163, 0.9));
    backdrop-filter: blur(10px);
    z-index: 10;
  }
  .menu {
    text-align: center;
    color: white;
  }
  .title {
    font-size: 3rem;
    font-weight: 300;
    letter-spacing: 0.3em;
    margin-bottom: 0.25em;
    text-shadow: 0 2px 20px rgba(0, 0, 0, 0.2);
  }
  .subtitle {
    font-size: 1rem;
    opacity: 0.8;
    margin-bottom: 2em;
    letter-spacing: 0.2em;
  }
  .buttons {
    display: flex;
    flex-direction: column;
    gap: 0.75em;
    align-items: center;
  }
  button {
    background: rgba(255, 255, 255, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.4);
    color: white;
    padding: 0.75em 2.5em;
    font-size: 1.1rem;
    border-radius: 8px;
    cursor: pointer;
    backdrop-filter: blur(5px);
    transition: all 0.2s;
    letter-spacing: 0.15em;
    width: 220px;
  }
  button:hover {
    background: rgba(255, 255, 255, 0.35);
    transform: translateY(-2px);
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  }
</style>
```

**Step 3: Create PauseMenu component**

Create `src/components/PauseMenu.svelte`:
```svelte
<script lang="ts">
  import { gameState, resetGameState } from '../lib/stores/gameStore';

  function resume() { gameState.set('playing'); }
  function quit() { resetGameState(); }
</script>

<div class="pause-overlay">
  <div class="pause-menu">
    <h2>一時停止</h2>
    <div class="buttons">
      <button on:click={resume}>再開</button>
      <button on:click={quit}>メニューに戻る</button>
    </div>
  </div>
</div>

<style>
  .pause-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(0, 0, 0, 0.5);
    backdrop-filter: blur(5px);
    z-index: 20;
  }
  .pause-menu {
    text-align: center;
    color: white;
  }
  h2 {
    font-size: 2rem;
    font-weight: 300;
    letter-spacing: 0.3em;
    margin-bottom: 1em;
  }
  .buttons {
    display: flex;
    flex-direction: column;
    gap: 0.75em;
    align-items: center;
  }
  button {
    background: rgba(255, 255, 255, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.4);
    color: white;
    padding: 0.75em 2em;
    font-size: 1rem;
    border-radius: 8px;
    cursor: pointer;
    letter-spacing: 0.1em;
    width: 200px;
    transition: all 0.2s;
  }
  button:hover {
    background: rgba(255, 255, 255, 0.35);
    transform: translateY(-2px);
  }
</style>
```

**Step 4: Update App.svelte**

Replace `src/App.svelte`:
```svelte
<script lang="ts">
  import { gameState, resetGameState, score, lives, level } from './lib/stores/gameStore';
  import GameCanvas from './components/GameCanvas.svelte';
  import MainMenu from './components/MainMenu.svelte';
  import PauseMenu from './components/PauseMenu.svelte';
  import GameOver from './components/GameOver.svelte';
  import Leaderboard from './components/Leaderboard.svelte';
  import LevelEditor from './components/LevelEditor.svelte';
</script>

<main>
  <div class="game-container">
    {#if $gameState === 'menu'}
      <MainMenu />
    {:else if $gameState === 'leaderboard'}
      <Leaderboard />
    {:else if $gameState === 'editor'}
      <LevelEditor />
    {:else}
      <GameCanvas />
      {#if $gameState === 'paused'}
        <PauseMenu />
      {/if}
      {#if $gameState === 'gameover' || $gameState === 'victory'}
        <GameOver isVictory={$gameState === 'victory'} finalScore={$score} finalLevel={$level} />
      {/if}
    {/if}
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    background: #1a1a2e;
    font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    overflow: hidden;
  }
  main {
    display: flex;
    justify-content: center;
    align-items: center;
  }
  .game-container {
    position: relative;
    width: 800px;
    max-width: 100vw;
  }
</style>
```

**Step 5: Commit**

```bash
git add -A
git commit -m "feat: add GameCanvas, MainMenu, PauseMenu, and App shell components"
```

---

## Task 15: GameOver Component

**Files:**
- Create: `src/components/GameOver.svelte`

**Step 1: Create GameOver component**

Create `src/components/GameOver.svelte`:
```svelte
<script lang="ts">
  import { resetGameState } from '../lib/stores/gameStore';
  import { isHighScore, addLeaderboardEntry } from '../lib/stores/gameStore';

  export let isVictory: boolean;
  export let finalScore: number;
  export let finalLevel: number;

  let playerName = '';
  let submitted = false;
  const showNameInput = isHighScore(finalScore);

  function submitScore() {
    if (!playerName.trim()) return;
    addLeaderboardEntry({
      name: playerName.trim(),
      score: finalScore,
      level: finalLevel,
      date: new Date().toISOString().split('T')[0],
    });
    submitted = true;
  }

  function goToMenu() {
    resetGameState();
  }
</script>

<div class="overlay">
  <div class="panel">
    <h2>{isVictory ? '🌟 全クリア！' : 'ゲームオーバー'}</h2>
    <p class="score">スコア: {finalScore}</p>
    <p class="level">レベル {finalLevel}</p>

    {#if showNameInput && !submitted}
      <div class="name-input">
        <p>ハイスコア！名前を入力:</p>
        <input
          type="text"
          bind:value={playerName}
          maxlength="12"
          placeholder="名前"
          on:keydown={(e) => e.key === 'Enter' && submitScore()}
        />
        <button on:click={submitScore}>登録</button>
      </div>
    {/if}

    <button class="menu-btn" on:click={goToMenu}>メニューに戻る</button>
  </div>
</div>

<style>
  .overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(0, 0, 0, 0.6);
    backdrop-filter: blur(8px);
    z-index: 20;
  }
  .panel {
    text-align: center;
    color: white;
    padding: 2em;
  }
  h2 {
    font-size: 2.5rem;
    font-weight: 300;
    letter-spacing: 0.2em;
    margin-bottom: 0.5em;
  }
  .score {
    font-size: 1.5rem;
    color: #FFD700;
  }
  .level {
    opacity: 0.7;
    margin-bottom: 1.5em;
  }
  .name-input {
    margin: 1em 0;
  }
  .name-input p {
    margin-bottom: 0.5em;
    color: #F4A6C0;
  }
  input {
    background: rgba(255, 255, 255, 0.15);
    border: 1px solid rgba(255, 255, 255, 0.3);
    color: white;
    padding: 0.5em 1em;
    border-radius: 6px;
    font-size: 1rem;
    text-align: center;
    outline: none;
    margin-right: 0.5em;
  }
  input:focus {
    border-color: rgba(255, 255, 255, 0.6);
  }
  button {
    background: rgba(255, 255, 255, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.4);
    color: white;
    padding: 0.5em 1.5em;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.2s;
  }
  button:hover {
    background: rgba(255, 255, 255, 0.35);
  }
  .menu-btn {
    margin-top: 1.5em;
    padding: 0.75em 2em;
    font-size: 1rem;
    letter-spacing: 0.1em;
  }
</style>
```

**Step 2: Commit**

```bash
git add -A
git commit -m "feat: add GameOver component with high score entry"
```

---

## Task 16: Leaderboard Component

**Files:**
- Create: `src/components/Leaderboard.svelte`

**Step 1: Create Leaderboard component**

Create `src/components/Leaderboard.svelte`:
```svelte
<script lang="ts">
  import { gameState } from '../lib/stores/gameStore';
  import { getLeaderboard } from '../lib/stores/gameStore';

  const entries = getLeaderboard();
  function goBack() { gameState.set('menu'); }
</script>

<div class="overlay">
  <div class="panel">
    <h2>ランキング</h2>
    {#if entries.length === 0}
      <p class="empty">まだ記録がありません</p>
    {:else}
      <table>
        <thead>
          <tr><th>#</th><th>名前</th><th>スコア</th><th>Lv</th><th>日付</th></tr>
        </thead>
        <tbody>
          {#each entries as entry, i}
            <tr>
              <td class="rank">{i + 1}</td>
              <td>{entry.name}</td>
              <td class="score">{entry.score}</td>
              <td>{entry.level}</td>
              <td class="date">{entry.date}</td>
            </tr>
          {/each}
        </tbody>
      </table>
    {/if}
    <button on:click={goBack}>戻る</button>
  </div>
</div>

<style>
  .overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(135deg, rgba(135, 206, 235, 0.9), rgba(255, 182, 163, 0.9));
    backdrop-filter: blur(10px);
    z-index: 10;
  }
  .panel {
    text-align: center;
    color: white;
    min-width: 400px;
  }
  h2 {
    font-size: 2rem;
    font-weight: 300;
    letter-spacing: 0.3em;
    margin-bottom: 1em;
  }
  .empty {
    opacity: 0.7;
    margin: 2em 0;
  }
  table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 1.5em;
  }
  th, td {
    padding: 0.5em 0.75em;
    text-align: center;
  }
  th {
    opacity: 0.7;
    font-weight: normal;
    font-size: 0.85rem;
    border-bottom: 1px solid rgba(255,255,255,0.3);
  }
  .rank { font-weight: bold; }
  .score { color: #FFD700; }
  .date { opacity: 0.6; font-size: 0.85rem; }
  button {
    background: rgba(255, 255, 255, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.4);
    color: white;
    padding: 0.75em 2em;
    font-size: 1rem;
    border-radius: 8px;
    cursor: pointer;
    letter-spacing: 0.1em;
    transition: all 0.2s;
  }
  button:hover {
    background: rgba(255, 255, 255, 0.35);
    transform: translateY(-2px);
  }
</style>
```

**Step 2: Commit**

```bash
git add -A
git commit -m "feat: add Leaderboard component with top 10 display"
```

---

## Task 17: Level Editor Component

**Files:**
- Create: `src/components/LevelEditor.svelte`

**Step 1: Create LevelEditor component**

Create `src/components/LevelEditor.svelte`:
```svelte
<script lang="ts">
  import { gameState } from '../lib/stores/gameStore';
  import { BRICK_ROWS, BRICK_COLS } from '../lib/engine/constants';
  import type { BrickType, BrickData, LevelData } from '../lib/engine/types';

  type CellType = BrickType | 'empty';

  let grid: CellType[][] = Array.from({ length: BRICK_ROWS }, () =>
    Array(BRICK_COLS).fill('empty')
  );

  let selectedTool: CellType = 'normal';

  const tools: { type: CellType; label: string; color: string }[] = [
    { type: 'empty', label: '消去', color: '#333' },
    { type: 'normal', label: '普通', color: '#F4A6C0' },
    { type: 'hard', label: '硬い', color: '#8B6F9E' },
    { type: 'gold', label: '金', color: '#FFD700' },
    { type: 'unbreakable', label: '壊せない', color: '#4A5568' },
  ];

  function paint(row: number, col: number) {
    grid[row][col] = selectedTool;
    grid = grid; // trigger reactivity
  }

  function exportLevel(): LevelData {
    const bricks: BrickData[] = [];
    for (let r = 0; r < BRICK_ROWS; r++) {
      for (let c = 0; c < BRICK_COLS; c++) {
        if (grid[r][c] !== 'empty') {
          bricks.push({ type: grid[r][c] as BrickType, row: r, col: c });
        }
      }
    }
    return { name: 'Custom Level', bricks };
  }

  function saveToStorage() {
    const data = exportLevel();
    localStorage.setItem('breakout-custom-level', JSON.stringify(data));
    alert('保存しました！');
  }

  function loadFromStorage() {
    try {
      const raw = localStorage.getItem('breakout-custom-level');
      if (!raw) return;
      const data: LevelData = JSON.parse(raw);
      // Reset grid
      grid = Array.from({ length: BRICK_ROWS }, () => Array(BRICK_COLS).fill('empty'));
      for (const b of data.bricks) {
        if (b.row < BRICK_ROWS && b.col < BRICK_COLS) {
          grid[b.row][b.col] = b.type;
        }
      }
      grid = grid;
    } catch { /* ignore */ }
  }

  function clearGrid() {
    grid = Array.from({ length: BRICK_ROWS }, () => Array(BRICK_COLS).fill('empty'));
  }

  function playTest() {
    const data = exportLevel();
    if (data.bricks.length === 0) {
      alert('ブリックを配置してください');
      return;
    }
    localStorage.setItem('breakout-test-level', JSON.stringify(data));
    gameState.set('playing');
    // Game component will check for test level
  }

  function goBack() { gameState.set('menu'); }

  function getCellColor(type: CellType): string {
    return tools.find(t => t.type === type)?.color || '#333';
  }

  // Load on mount
  loadFromStorage();
</script>

<div class="editor-overlay">
  <div class="editor">
    <h2>レベルエディタ</h2>

    <div class="toolbar">
      {#each tools as tool}
        <button
          class="tool-btn"
          class:active={selectedTool === tool.type}
          style="background: {tool.color}"
          on:click={() => selectedTool = tool.type}
        >
          {tool.label}
        </button>
      {/each}
    </div>

    <div class="grid">
      {#each grid as row, r}
        <div class="grid-row">
          {#each row as cell, c}
            <button
              class="cell"
              style="background: {getCellColor(cell)}"
              on:click={() => paint(r, c)}
              on:mouseenter={(e) => e.buttons === 1 && paint(r, c)}
            ></button>
          {/each}
        </div>
      {/each}
    </div>

    <div class="actions">
      <button on:click={saveToStorage}>保存</button>
      <button on:click={clearGrid}>クリア</button>
      <button on:click={playTest}>テストプレイ</button>
      <button on:click={goBack}>戻る</button>
    </div>
  </div>
</div>

<style>
  .editor-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(135deg, rgba(135, 206, 235, 0.95), rgba(255, 182, 163, 0.95));
    z-index: 10;
    overflow-y: auto;
  }
  .editor {
    text-align: center;
    color: white;
    padding: 1em;
  }
  h2 {
    font-size: 1.5rem;
    font-weight: 300;
    letter-spacing: 0.3em;
    margin-bottom: 0.75em;
  }
  .toolbar {
    display: flex;
    gap: 0.5em;
    justify-content: center;
    margin-bottom: 0.75em;
    flex-wrap: wrap;
  }
  .tool-btn {
    border: 2px solid transparent;
    color: white;
    padding: 0.3em 0.8em;
    border-radius: 6px;
    cursor: pointer;
    font-size: 0.85rem;
    transition: all 0.15s;
  }
  .tool-btn.active {
    border-color: white;
    transform: scale(1.1);
  }
  .grid {
    display: inline-block;
    border: 1px solid rgba(255,255,255,0.2);
    border-radius: 4px;
    padding: 4px;
    background: rgba(0,0,0,0.15);
  }
  .grid-row {
    display: flex;
    gap: 2px;
  }
  .grid-row + .grid-row {
    margin-top: 2px;
  }
  .cell {
    width: 48px;
    height: 18px;
    border: 1px solid rgba(255,255,255,0.15);
    border-radius: 2px;
    cursor: pointer;
    padding: 0;
    transition: opacity 0.1s;
  }
  .cell:hover {
    opacity: 0.8;
  }
  .actions {
    display: flex;
    gap: 0.5em;
    justify-content: center;
    margin-top: 0.75em;
    flex-wrap: wrap;
  }
  .actions button {
    background: rgba(255, 255, 255, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.4);
    color: white;
    padding: 0.5em 1.5em;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.2s;
  }
  .actions button:hover {
    background: rgba(255, 255, 255, 0.35);
    transform: translateY(-1px);
  }
</style>
```

**Step 2: Commit**

```bash
git add -A
git commit -m "feat: add level editor with grid painting and test play"
```

---

## Task 18: Integration — Wire Editor to Game + Clean Up Imports

**Files:**
- Modify: `src/App.svelte` — ensure all imports resolve
- Modify: `src/components/GameCanvas.svelte` — load custom level if from editor
- Modify: `src/lib/engine/Game.ts` — check for test level in localStorage

**Step 1: Update GameCanvas to support custom levels**

In `src/components/GameCanvas.svelte`, update `onMount`:
```ts
  onMount(() => {
    game = new Game(canvas);

    // Check for test level from editor
    const testLevel = localStorage.getItem('breakout-test-level');
    if (testLevel) {
      try {
        game.customLevel = JSON.parse(testLevel);
        localStorage.removeItem('breakout-test-level');
      } catch { /* ignore */ }
    }

    game.init();
    game.start();
    return () => game.destroy();
  });
```

**Step 2: Clean up `src/app.css` and `src/main.ts`**

Replace `src/app.css` with minimal reset:
```css
* { box-sizing: border-box; margin: 0; padding: 0; }
```

Ensure `src/main.ts` imports `App.svelte` correctly.

**Step 3: Run dev server and verify visually**

```bash
npm run dev
```

Open browser. Verify:
- Main menu displays with Shinkai-style gradient
- "ゲーム開始" starts the game
- Ball launches on click/space
- Bricks render with glow effects
- Paddle follows mouse
- Esc pauses
- Level editor works

**Step 4: Commit**

```bash
git add -A
git commit -m "feat: wire editor to game, clean up imports, integration complete"
```

---

## Task 19: Run All Tests + Final Polish

**Step 1: Run full test suite**

```bash
npx vitest run
```

Fix any failures.

**Step 2: Build for production**

```bash
npm run build
```

Ensure no build errors.

**Step 3: Final commit**

```bash
git add -A
git commit -m "chore: verify all tests pass and production build succeeds"
```

---

## Summary

| Task | Description | Key Files |
|------|-------------|-----------|
| 1 | Project scaffolding | Vite + Svelte + Vitest |
| 2 | Types & constants | `types.ts`, `constants.ts` |
| 3 | Ball class | `Ball.ts` + tests |
| 4 | Paddle class | `Paddle.ts` + tests |
| 5 | Brick class | `Brick.ts` + tests |
| 6 | Collision detection | `Collision.ts` + tests |
| 7 | Powerup class | `Powerup.ts` + tests |
| 8 | Particle system | `Particle.ts` + tests |
| 9 | Level data | `levels.ts` + tests |
| 10 | Audio manager | `AudioManager.ts` + tests |
| 11 | Game store | `gameStore.ts` + tests |
| 12 | Shinkai renderer | `ShinkaRenderer.ts` |
| 13 | Game engine | `Game.ts` |
| 14 | Svelte components | Canvas, Menu, Pause |
| 15 | GameOver component | `GameOver.svelte` |
| 16 | Leaderboard | `Leaderboard.svelte` |
| 17 | Level editor | `LevelEditor.svelte` |
| 18 | Integration | Wire everything together |
| 19 | Tests + build | Final verification |
