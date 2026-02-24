<script lang="ts">
  import { onMount } from 'svelte';

  type GameStatus = 'ready' | 'running' | 'paused' | 'won' | 'lost';

  interface Paddle {
    x: number;
    y: number;
    width: number;
    height: number;
    speed: number;
  }

  interface Ball {
    x: number;
    y: number;
    vx: number;
    vy: number;
    radius: number;
    speed: number;
    trail: { x: number; y: number; alpha: number }[];
  }

  interface Brick {
    x: number;
    y: number;
    width: number;
    height: number;
    row: number;
    col: number;
    alive: boolean;
    hitAnim: number;
  }

  interface Particle {
    x: number;
    y: number;
    vx: number;
    vy: number;
    life: number;
    maxLife: number;
    color: string;
    size: number;
  }

  interface GameMetrics {
    paddleWidth: number;
    paddleHeight: number;
    paddleBottomGap: number;
    paddleSpeed: number;
    ballRadius: number;
    ballBaseSpeed: number;
    ballMaxSpeed: number;
    ballSpeedStep: number;
    brickWidth: number;
    brickHeight: number;
    brickGap: number;
    brickTop: number;
    brickOffsetX: number;
  }

  const HIGH_SCORE_KEY = 'breakout_high_score_v2';
  const BRICK_ROWS = 6;
  const BRICK_COLS = 10;
  const INITIAL_LIVES = 3;

  // Cyberpunk neon palette for bricks
  const BRICK_COLORS = [
    '#00fff7', // Cyan (top row)
    '#00ff88', // Neon green
    '#f0ff00', // Neon yellow
    '#ff6b00', // Neon orange
    '#ff00ff', // Neon pink
    '#ff0066', // Hot pink (bottom row)
  ];

  let canvasEl: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D | null = null;

  let arenaWidth = 900;
  let arenaHeight = 560;
  let status: GameStatus = 'ready';
  let score = 0;
  let lives = INITIAL_LIVES;
  let highScore = 0;
  let bricksRemaining = 0;
  let ballStuckToPaddle = true;

  let pointerActive = false;
  let leftPressed = false;
  let rightPressed = false;

  let animationFrame = 0;
  let lastFrameTime = 0;
  let gameTime = 0;

  let particles: Particle[] = [];

  let metrics: GameMetrics = {
    paddleWidth: 140,
    paddleHeight: 12,
    paddleBottomGap: 28,
    paddleSpeed: 860,
    ballRadius: 8,
    ballBaseSpeed: 380,
    ballMaxSpeed: 760,
    ballSpeedStep: 8,
    brickWidth: 80,
    brickHeight: 22,
    brickGap: 6,
    brickTop: 70,
    brickOffsetX: 60,
  };

  let paddle: Paddle = {
    x: 0,
    y: 0,
    width: metrics.paddleWidth,
    height: metrics.paddleHeight,
    speed: metrics.paddleSpeed,
  };

  let ball: Ball = {
    x: 0,
    y: 0,
    vx: 0,
    vy: 0,
    radius: metrics.ballRadius,
    speed: metrics.ballBaseSpeed,
    trail: [],
  };

  let bricks: Brick[] = [];

  $: statusTitle =
    status === 'ready'
      ? 'Break The Wall'
      : status === 'paused'
        ? 'Paused'
        : status === 'won'
          ? 'Stage Cleared'
          : status === 'lost'
            ? 'Game Over'
            : '';

  $: statusHint =
    status === 'ready'
      ? 'Press Space or tap Start.'
      : status === 'paused'
        ? 'Press Space or tap Resume.'
        : status === 'won'
          ? 'All bricks destroyed.'
          : status === 'lost'
            ? 'No lives left.'
            : '';

  $: primaryButtonLabel =
    status === 'paused' ? 'Resume' : status === 'won' || status === 'lost' ? 'Play Again' : 'Start';

  function clamp(value: number, min: number, max: number): number {
    return Math.min(max, Math.max(min, value));
  }

  function readHighScore(): number {
    try {
      const raw = localStorage.getItem(HIGH_SCORE_KEY);
      const parsed = Number.parseInt(raw ?? '0', 10);
      return Number.isFinite(parsed) && parsed > 0 ? parsed : 0;
    } catch {
      return 0;
    }
  }

  function persistHighScore(): void {
    try {
      localStorage.setItem(HIGH_SCORE_KEY, String(highScore));
    } catch {
      // Ignore storage errors and keep gameplay intact.
    }
  }

  function maybeUpdateHighScore(): void {
    if (score > highScore) {
      highScore = score;
      persistHighScore();
    }
  }

  function updateMetrics(): void {
    const brickGap = Math.max(4, Math.round(arenaWidth * 0.0065));
    const layoutWidth = arenaWidth * 0.88;
    const brickWidth = (layoutWidth - brickGap * (BRICK_COLS - 1)) / BRICK_COLS;
    const brickHeight = clamp(arenaHeight * 0.04, 16, 28);

    metrics = {
      paddleWidth: clamp(arenaWidth * 0.17, 84, 190),
      paddleHeight: clamp(arenaHeight * 0.023, 10, 18),
      paddleBottomGap: clamp(arenaHeight * 0.06, 16, 42),
      paddleSpeed: arenaWidth * 1.1,
      ballRadius: clamp(arenaWidth * 0.009, 6, 10),
      ballBaseSpeed: arenaHeight * 0.68,
      ballMaxSpeed: arenaHeight * 1.22,
      ballSpeedStep: arenaHeight * 0.013,
      brickWidth,
      brickHeight,
      brickGap,
      brickTop: clamp(arenaHeight * 0.12, 40, 92),
      brickOffsetX: (arenaWidth - (brickWidth * BRICK_COLS + brickGap * (BRICK_COLS - 1))) / 2,
    };
  }

  function setupCanvas(): void {
    if (!canvasEl) {
      return;
    }

    const dpr = window.devicePixelRatio || 1;
    canvasEl.width = Math.round(arenaWidth * dpr);
    canvasEl.height = Math.round(arenaHeight * dpr);
    canvasEl.style.width = `${arenaWidth}px`;
    canvasEl.style.height = `${arenaHeight}px`;

    ctx = canvasEl.getContext('2d');
    if (ctx) {
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }
  }

  function buildBricks(aliveMap?: boolean[]): void {
    const nextBricks: Brick[] = [];
    let index = 0;

    for (let row = 0; row < BRICK_ROWS; row += 1) {
      for (let col = 0; col < BRICK_COLS; col += 1) {
        const alive = aliveMap ? Boolean(aliveMap[index]) : true;
        nextBricks.push({
          x: metrics.brickOffsetX + col * (metrics.brickWidth + metrics.brickGap),
          y: metrics.brickTop + row * (metrics.brickHeight + metrics.brickGap),
          width: metrics.brickWidth,
          height: metrics.brickHeight,
          row,
          col,
          alive,
          hitAnim: 0,
        });
        index += 1;
      }
    }

    bricks = nextBricks;
    bricksRemaining = bricks.filter((brick) => brick.alive).length;
  }

  function stickBallToPaddle(): void {
    ball.x = paddle.x + paddle.width / 2;
    ball.y = paddle.y - ball.radius - 2;
    ball.trail = [];
  }

  function resetRound(centerPaddle: boolean): void {
    paddle.width = metrics.paddleWidth;
    paddle.height = metrics.paddleHeight;
    paddle.speed = metrics.paddleSpeed;
    paddle.y = arenaHeight - paddle.height - metrics.paddleBottomGap;
    paddle.x = centerPaddle ? (arenaWidth - paddle.width) / 2 : clamp(paddle.x, 0, arenaWidth - paddle.width);

    ball.radius = metrics.ballRadius;
    ball.speed = metrics.ballBaseSpeed;
    ball.vx = 0;
    ball.vy = 0;
    ball.trail = [];
    ballStuckToPaddle = true;
    stickBallToPaddle();
  }

  function resetGame(): void {
    score = 0;
    lives = INITIAL_LIVES;
    status = 'ready';
    leftPressed = false;
    rightPressed = false;
    particles = [];
    buildBricks();
    resetRound(true);
  }

  function launchBall(): void {
    ballStuckToPaddle = false;
    ball.speed = metrics.ballBaseSpeed;
    const spread = (Math.random() - 0.5) * 0.8;
    ball.vx = Math.sin(spread) * ball.speed;
    ball.vy = -Math.cos(spread) * ball.speed;
  }

  function beginPlay(): void {
    if (status === 'won' || status === 'lost') {
      resetGame();
    }
    if (ballStuckToPaddle) {
      launchBall();
    }
    status = 'running';
  }

  function togglePause(): void {
    if (status === 'running') {
      status = 'paused';
    } else if (status === 'paused') {
      status = 'running';
    }
  }

  function handleSpaceAction(): void {
    if (status === 'running') {
      status = 'paused';
      return;
    }
    beginPlay();
  }

  function loseLife(): void {
    lives -= 1;
    maybeUpdateHighScore();
    if (lives <= 0) {
      status = 'lost';
      return;
    }
    status = 'ready';
    resetRound(true);
  }

  function speedUpBall(): void {
    const nextSpeed = clamp(Math.hypot(ball.vx, ball.vy) + metrics.ballSpeedStep, metrics.ballBaseSpeed, metrics.ballMaxSpeed);
    const direction = Math.atan2(ball.vy, ball.vx);
    ball.speed = nextSpeed;
    ball.vx = Math.cos(direction) * nextSpeed;
    ball.vy = Math.sin(direction) * nextSpeed;
  }

  function spawnParticles(x: number, y: number, color: string, count: number): void {
    for (let i = 0; i < count; i++) {
      const angle = Math.random() * Math.PI * 2;
      const speed = Math.random() * 150 + 50;
      particles.push({
        x,
        y,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        life: 1,
        maxLife: Math.random() * 0.5 + 0.3,
        color,
        size: Math.random() * 3 + 1,
      });
    }
  }

  function movePaddleByInput(dt: number): void {
    if (leftPressed) {
      paddle.x -= paddle.speed * dt;
    }
    if (rightPressed) {
      paddle.x += paddle.speed * dt;
    }
    paddle.x = clamp(paddle.x, 0, arenaWidth - paddle.width);
  }

  function circleIntersectsRect(
    cx: number,
    cy: number,
    radius: number,
    rx: number,
    ry: number,
    rw: number,
    rh: number,
  ): boolean {
    const closestX = clamp(cx, rx, rx + rw);
    const closestY = clamp(cy, ry, ry + rh);
    const dx = cx - closestX;
    const dy = cy - closestY;
    return dx * dx + dy * dy <= radius * radius;
  }

  function reflectFromRect(brick: Brick, previousX: number, previousY: number): void {
    const fromLeft = previousX + ball.radius <= brick.x;
    const fromRight = previousX - ball.radius >= brick.x + brick.width;
    const fromTop = previousY + ball.radius <= brick.y;
    const fromBottom = previousY - ball.radius >= brick.y + brick.height;

    if (fromLeft || fromRight) {
      ball.vx *= -1;
      return;
    }
    if (fromTop || fromBottom) {
      ball.vy *= -1;
      return;
    }

    const overlapLeft = ball.x + ball.radius - brick.x;
    const overlapRight = brick.x + brick.width - (ball.x - ball.radius);
    const overlapTop = ball.y + ball.radius - brick.y;
    const overlapBottom = brick.y + brick.height - (ball.y - ball.radius);

    const minOverlap = Math.min(overlapLeft, overlapRight, overlapTop, overlapBottom);
    if (minOverlap === overlapLeft || minOverlap === overlapRight) {
      ball.vx *= -1;
    } else {
      ball.vy *= -1;
    }
  }

  function handleBrickCollision(previousX: number, previousY: number): void {
    for (const brick of bricks) {
      if (!brick.alive) {
        continue;
      }
      if (!circleIntersectsRect(ball.x, ball.y, ball.radius, brick.x, brick.y, brick.width, brick.height)) {
        continue;
      }

      brick.alive = false;
      brick.hitAnim = 1;
      bricksRemaining -= 1;
      score += 100;
      maybeUpdateHighScore();

      // Spawn particles at brick center
      const brickColor = BRICK_COLORS[brick.row % BRICK_COLORS.length];
      spawnParticles(
        brick.x + brick.width / 2,
        brick.y + brick.height / 2,
        brickColor,
        12
      );

      reflectFromRect(brick, previousX, previousY);
      speedUpBall();

      if (bricksRemaining <= 0) {
        status = 'won';
      }
      return;
    }
  }

  function handlePaddleCollision(): void {
    if (ball.vy <= 0) {
      return;
    }
    if (!circleIntersectsRect(ball.x, ball.y, ball.radius, paddle.x, paddle.y, paddle.width, paddle.height)) {
      return;
    }

    const paddleCenter = paddle.x + paddle.width / 2;
    const normalizedHit = clamp((ball.x - paddleCenter) / (paddle.width / 2), -1, 1);
    const maxBounceAngle = Math.PI / 3;
    const angle = normalizedHit * maxBounceAngle;
    const speed = clamp(Math.hypot(ball.vx, ball.vy) * 1.01, metrics.ballBaseSpeed, metrics.ballMaxSpeed);

    ball.speed = speed;
    ball.vx = Math.sin(angle) * speed;
    ball.vy = -Math.cos(angle) * speed;
    ball.y = paddle.y - ball.radius - 1;

    // Spawn particles on paddle hit
    spawnParticles(ball.x, paddle.y, '#00fff7', 5);
  }

  function updateParticles(dt: number): void {
    for (let i = particles.length - 1; i >= 0; i--) {
      const p = particles[i];
      p.x += p.vx * dt;
      p.y += p.vy * dt;
      p.vy += 200 * dt; // gravity
      p.life -= dt / p.maxLife;
      
      if (p.life <= 0) {
        particles.splice(i, 1);
      }
    }
  }

  function updateGame(dt: number): void {
    movePaddleByInput(dt);
    updateParticles(dt);

    // Update brick hit animations
    for (const brick of bricks) {
      if (brick.hitAnim > 0) {
        brick.hitAnim = Math.max(0, brick.hitAnim - dt * 5);
      }
    }

    if (ballStuckToPaddle) {
      stickBallToPaddle();
      return;
    }

    // Update ball trail
    ball.trail.unshift({ x: ball.x, y: ball.y, alpha: 1 });
    if (ball.trail.length > 12) {
      ball.trail.pop();
    }
    for (let i = 0; i < ball.trail.length; i++) {
      ball.trail[i].alpha = 1 - i / ball.trail.length;
    }

    const previousX = ball.x;
    const previousY = ball.y;

    ball.x += ball.vx * dt;
    ball.y += ball.vy * dt;

    if (ball.x - ball.radius <= 0) {
      ball.x = ball.radius;
      ball.vx = Math.abs(ball.vx);
    } else if (ball.x + ball.radius >= arenaWidth) {
      ball.x = arenaWidth - ball.radius;
      ball.vx = -Math.abs(ball.vx);
    }

    if (ball.y - ball.radius <= 0) {
      ball.y = ball.radius;
      ball.vy = Math.abs(ball.vy);
    }

    handlePaddleCollision();
    handleBrickCollision(previousX, previousY);

    if (ball.y - ball.radius > arenaHeight) {
      loseLife();
    }
  }

  function drawRoundedRect(x: number, y: number, width: number, height: number, radius: number): void {
    if (!ctx) {
      return;
    }
    const r = clamp(radius, 0, Math.min(width, height) / 2);
    ctx.beginPath();
    ctx.moveTo(x + r, y);
    ctx.lineTo(x + width - r, y);
    ctx.arcTo(x + width, y, x + width, y + r, r);
    ctx.lineTo(x + width, y + height - r);
    ctx.arcTo(x + width, y + height, x + width - r, y + height, r);
    ctx.lineTo(x + r, y + height);
    ctx.arcTo(x, y + height, x, y + height - r, r);
    ctx.lineTo(x, y + r);
    ctx.arcTo(x, y, x + r, y, r);
    ctx.closePath();
  }

  function drawScene(): void {
    if (!ctx) {
      return;
    }

    // Clear with dark background
    ctx.fillStyle = '#0a0a0f';
    ctx.fillRect(0, 0, arenaWidth, arenaHeight);

    // Draw grid pattern
    ctx.save();
    ctx.strokeStyle = 'rgba(0, 255, 247, 0.05)';
    ctx.lineWidth = 1;
    const gridSize = Math.max(28, Math.round(arenaWidth * 0.04));
    for (let x = 0; x <= arenaWidth; x += gridSize) {
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, arenaHeight);
      ctx.stroke();
    }
    for (let y = 0; y <= arenaHeight; y += gridSize) {
      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(arenaWidth, y);
      ctx.stroke();
    }
    ctx.restore();

    // Draw ambient glow at top
    const ambientGradient = ctx.createRadialGradient(
      arenaWidth / 2, 0, 0,
      arenaWidth / 2, 0, arenaHeight * 0.6
    );
    ambientGradient.addColorStop(0, 'rgba(255, 0, 255, 0.08)');
    ambientGradient.addColorStop(0.5, 'rgba(0, 255, 247, 0.04)');
    ambientGradient.addColorStop(1, 'transparent');
    ctx.fillStyle = ambientGradient;
    ctx.fillRect(0, 0, arenaWidth, arenaHeight);

    // Draw particles
    for (const p of particles) {
      ctx.save();
      ctx.globalAlpha = p.life;
      ctx.fillStyle = p.color;
      ctx.shadowColor = p.color;
      ctx.shadowBlur = 8;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.size * p.life, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    }

    // Draw bricks with glow effect
    ctx.save();
    for (const brick of bricks) {
      if (!brick.alive) {
        continue;
      }
      const rowColor = BRICK_COLORS[brick.row % BRICK_COLORS.length];
      
      // Hit animation scale
      const scale = 1 + brick.hitAnim * 0.1;
      const offsetX = brick.width * (1 - scale) / 2;
      const offsetY = brick.height * (1 - scale) / 2;
      
      ctx.fillStyle = rowColor;
      ctx.shadowColor = rowColor;
      ctx.shadowBlur = 15;
      
      // Main brick
      drawRoundedRect(
        brick.x + offsetX,
        brick.y + offsetY,
        brick.width * scale,
        brick.height * scale,
        4
      );
      ctx.fill();
      
      // Inner highlight
      ctx.fillStyle = 'rgba(255, 255, 255, 0.15)';
      ctx.shadowBlur = 0;
      drawRoundedRect(
        brick.x + offsetX + 2,
        brick.y + offsetY + 2,
        brick.width * scale - 4,
        brick.height * scale * 0.4,
        2
      );
      ctx.fill();
    }
    ctx.restore();

    // Draw paddle with neon glow
    ctx.save();
    
    // Paddle glow
    ctx.shadowColor = '#00fff7';
    ctx.shadowBlur = 25;
    
    // Paddle gradient
    const paddleGradient = ctx.createLinearGradient(
      paddle.x, paddle.y,
      paddle.x, paddle.y + paddle.height
    );
    paddleGradient.addColorStop(0, '#00fff7');
    paddleGradient.addColorStop(1, '#00b8b8');
    ctx.fillStyle = paddleGradient;
    
    drawRoundedRect(paddle.x, paddle.y, paddle.width, paddle.height, paddle.height / 2);
    ctx.fill();
    
    // Paddle center line
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.5)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(paddle.x + paddle.width / 2, paddle.y + 2);
    ctx.lineTo(paddle.x + paddle.width / 2, paddle.y + paddle.height - 2);
    ctx.stroke();
    
    ctx.restore();

    // Draw ball trail
    ctx.save();
    for (let i = ball.trail.length - 1; i >= 0; i--) {
      const t = ball.trail[i];
      const size = ball.radius * (1 - i / ball.trail.length) * 0.8;
      ctx.globalAlpha = t.alpha * 0.3;
      ctx.fillStyle = '#ff00ff';
      ctx.beginPath();
      ctx.arc(t.x, t.y, size, 0, Math.PI * 2);
      ctx.fill();
    }
    ctx.restore();

    // Draw ball with neon glow
    ctx.save();
    
    // Outer glow
    ctx.shadowColor = '#ff00ff';
    ctx.shadowBlur = 20;
    
    // Ball gradient
    const ballGradient = ctx.createRadialGradient(
      ball.x - ball.radius * 0.3,
      ball.y - ball.radius * 0.3,
      0,
      ball.x,
      ball.y,
      ball.radius
    );
    ballGradient.addColorStop(0, '#ffffff');
    ballGradient.addColorStop(0.3, '#ff88ff');
    ballGradient.addColorStop(1, '#ff00ff');
    ctx.fillStyle = ballGradient;
    
    ctx.beginPath();
    ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
    ctx.fill();
    
    ctx.restore();

    // Draw border frame
    ctx.save();
    ctx.strokeStyle = 'rgba(0, 255, 247, 0.3)';
    ctx.lineWidth = 2;
    ctx.strokeRect(1, 1, arenaWidth - 2, arenaHeight - 2);
    ctx.restore();
  }

  function frame(time: number): void {
    if (lastFrameTime === 0) {
      lastFrameTime = time;
    }

    const dt = Math.min((time - lastFrameTime) / 1000, 0.034);
    lastFrameTime = time;
    gameTime += dt;

    if (status === 'running') {
      updateGame(dt);
    } else if (ballStuckToPaddle) {
      stickBallToPaddle();
    }

    drawScene();
    animationFrame = window.requestAnimationFrame(frame);
  }

  function movePaddleToClientX(clientX: number): void {
    if (!canvasEl) {
      return;
    }
    const rect = canvasEl.getBoundingClientRect();
    const normalizedX = (clientX - rect.left) / rect.width;
    const targetX = normalizedX * arenaWidth;
    paddle.x = clamp(targetX - paddle.width / 2, 0, arenaWidth - paddle.width);
    if (ballStuckToPaddle) {
      stickBallToPaddle();
    }
  }

  function handlePointerDown(event: PointerEvent): void {
    pointerActive = true;
    movePaddleToClientX(event.clientX);
  }

  function handlePointerMove(event: PointerEvent): void {
    if (!pointerActive) {
      return;
    }
    movePaddleToClientX(event.clientX);
  }

  function handlePointerUp(): void {
    pointerActive = false;
  }

  function relayoutArena(preserveState: boolean): void {
    const previousWidth = arenaWidth;
    const previousHeight = arenaHeight;

    const horizontalPadding = window.innerWidth < 700 ? 24 : 80;
    const availableWidth = window.innerWidth - horizontalPadding;
    const availableHeight = window.innerHeight - 220;
    let nextWidth = clamp(availableWidth, 300, 960);
    let nextHeight = Math.round(nextWidth * 0.62);

    if (availableHeight > 260 && nextHeight > availableHeight) {
      nextHeight = Math.max(260, availableHeight);
      nextWidth = Math.round(nextHeight / 0.62);
    }

    arenaWidth = nextWidth;
    arenaHeight = nextHeight;

    updateMetrics();
    setupCanvas();

    if (!preserveState || previousWidth === 0 || previousHeight === 0) {
      resetRound(true);
      buildBricks();
      return;
    }

    const sx = arenaWidth / previousWidth;
    const sy = arenaHeight / previousHeight;
    const speedScale = (sx + sy) / 2;

    paddle.width = metrics.paddleWidth;
    paddle.height = metrics.paddleHeight;
    paddle.speed = metrics.paddleSpeed;
    paddle.y = arenaHeight - paddle.height - metrics.paddleBottomGap;
    paddle.x = clamp(paddle.x * sx, 0, arenaWidth - paddle.width);

    ball.radius = metrics.ballRadius;
    if (ballStuckToPaddle) {
      stickBallToPaddle();
    } else {
      ball.x = clamp(ball.x * sx, ball.radius, arenaWidth - ball.radius);
      ball.y = clamp(ball.y * sy, ball.radius, arenaHeight - ball.radius);
      ball.vx *= speedScale;
      ball.vy *= speedScale;
      ball.speed = clamp(Math.hypot(ball.vx, ball.vy), metrics.ballBaseSpeed, metrics.ballMaxSpeed);
    }

    buildBricks(bricks.map((brick) => brick.alive));
  }

  function handleKeyDown(event: KeyboardEvent): void {
    const key = event.key.toLowerCase();
    if (key === 'arrowleft' || key === 'a') {
      leftPressed = true;
    }
    if (key === 'arrowright' || key === 'd') {
      rightPressed = true;
    }
    if (event.code === 'Space') {
      event.preventDefault();
      handleSpaceAction();
    }
    if (key === 'r') {
      resetGame();
    }
  }

  function handleKeyUp(event: KeyboardEvent): void {
    const key = event.key.toLowerCase();
    if (key === 'arrowleft' || key === 'a') {
      leftPressed = false;
    }
    if (key === 'arrowright' || key === 'd') {
      rightPressed = false;
    }
  }

  onMount(() => {
    highScore = readHighScore();
    relayoutArena(false);
    resetGame();
    drawScene();

    const handleResize = () => relayoutArena(true);

    window.addEventListener('keydown', handleKeyDown);
    window.addEventListener('keyup', handleKeyUp);
    window.addEventListener('resize', handleResize);

    animationFrame = window.requestAnimationFrame(frame);

    return () => {
      window.cancelAnimationFrame(animationFrame);
      window.removeEventListener('keydown', handleKeyDown);
      window.removeEventListener('keyup', handleKeyUp);
      window.removeEventListener('resize', handleResize);
    };
  });
</script>

<main class="breakout-page">
  <header class="title-block">
    <h1>Neon Breakout</h1>
    <p>// crack every brick before your lives run out</p>
  </header>

  <section class="hud" aria-label="Game HUD">
    <div class="metric">
      <span>Score</span>
      <strong>{score}</strong>
    </div>
    <div class="metric">
      <span>Lives</span>
      <strong>{lives}</strong>
    </div>
    <div class="metric">
      <span>High Score</span>
      <strong>{highScore}</strong>
    </div>
    <div class="hud-actions">
      <button type="button" on:click={togglePause} disabled={status === 'ready' || status === 'won' || status === 'lost'}>
        {status === 'running' ? 'Pause' : 'Resume'}
      </button>
      <button type="button" on:click={resetGame}>Restart</button>
    </div>
  </section>

  <section class="arena-shell" style={`width:${arenaWidth}px;height:${arenaHeight}px;`}>
    <canvas
      bind:this={canvasEl}
      class="arena-canvas"
      on:pointerdown={handlePointerDown}
      on:pointermove={handlePointerMove}
      on:pointerup={handlePointerUp}
      on:pointerleave={handlePointerUp}
      on:pointercancel={handlePointerUp}
      aria-label="Breakout game canvas"
    ></canvas>

    {#if status !== 'running'}
      <div class="status-overlay">
        <h2>{statusTitle}</h2>
        <p>{statusHint}</p>
        <div class="overlay-actions">
          <button type="button" on:click={beginPlay}>{primaryButtonLabel}</button>
          <button type="button" on:click={resetGame}>Restart</button>
        </div>
      </div>
    {/if}
  </section>

  <footer class="controls-hint">
    <kbd>A</kbd>/<kbd>D</kbd> or <kbd>&#8592;</kbd>/<kbd>&#8594;</kbd>
    <span>|</span>
    <kbd>Space</kbd> start/pause
    <span>|</span>
    <kbd>R</kbd> restart
    <span>|</span>
    Touch: drag paddle
  </footer>
</main>