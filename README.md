# index.ht
小恐龍遊戲
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dino Pro: Parallax Edition</title>
    <style>
        body { margin: 0; background: #222; display: flex; justify-content: center; align-items: center; height: 100vh; color: white; font-family: sans-serif; overflow: hidden; }
        #game-container { position: relative; box-shadow: 0 10px 40px rgba(0,0,0,0.8); overflow: hidden; border-radius: 4px; }
        canvas { background: #87CEEB; /* 預設天空藍 */ display: block; cursor: pointer; }
        .hud { position: absolute; top: 15px; left: 20px; font-weight: bold; text-shadow: 1px 1px 2px rgba(0,0,0,0.5); pointer-events: none; }
        .score { font-size: 28px; font-family: 'Courier New', monospace; }
        .loading-overlay { position: absolute; top: 0; left: 0; width: 800px; height: 250px; background: #222; display: flex; justify-content: center; align-items: center; z-index: 100; transition: opacity 0.5s; }
    </style>
</head>
<body>

<div id="game-container">
    <div id="loading" class="loading-overlay">LOADING ASSETS (FALLBACK IN 3s)...</div>
    <div class="hud">
        <div id="score" class="score">00000</div>
        <div id="status" style="font-size: 10px; color: #ddd;">Status: Normal</div>
    </div>
    <canvas id="gameCanvas" width="800" height="250"></canvas>
</div>

<script>
/**
 * 配置與專業調教參數
 */
const CONFIG = {
    CANVAS_WIDTH: 800,
    CANVAS_HEIGHT: 250,
    GROUND_Y: 200,          // 地平線高度
    GRAVITY: 2600,          // 像素/秒^2
    INITIAL_SPEED: 400,     // 初始滾動速度 (像素/秒)
    JUMP_FORCE: -850,       // 跳躍初速度
    HITBOX_PADDING: 6,       // 碰撞箱內縮
    FALLBACK_TIMEOUT: 3000   // 如果 3 秒載入失敗則啟用 Fallback
};

/**
 * 資源管理器 (Asset Manager)
 */
const Assets = {
    images: {},
    manifest: {
        dino: 'assets/dino_sheet.png',     // 裁剪座標 100x100
        cactus: 'assets/cactus_sheet.png', // 裁剪座標 50x100
        ground: 'assets/ground.png',       // 捲動快
        clouds: 'assets/clouds.png',       // 捲動慢 (視差)
        distant: 'assets/distant.png'      // 捲動中 (視差遠山)
    },
    async loadAll() {
        const timeout = new Promise(resolve => setTimeout(() => resolve(false), CONFIG.FALLBACK_TIMEOUT));
        const loads = Object.entries(this.manifest).map(([key, src]) => {
            return new Promise(resolve => {
                const img = new Image();
                img.src = src;
                img.onload = () => { this.images[key] = img; resolve(true); };
                img.onerror = () => resolve(false); // 失敗不崩潰
            });
        });
        const results = await Promise.race([Promise.all(loads), timeout]);
        return results && results.every(Boolean); // 全部成功才算完成
    }
};

/**
 * 專業視差背景管理器
 */
class ParallaxManager {
    constructor(ctx) {
        this.ctx = ctx;
        this.layers = [];
        this.currentSpeed = CONFIG.INITIAL_SPEED;
        
        // 初始化視差層 (從遠到近)
        if (Assets.images.clouds) this.addLayer(Assets.images.clouds, 0.1, 50);   // 速度比 0.1, Y=50
        if (Assets.images.distant) this.addLayer(Assets.images.distant, 0.4, 120); // 速度比 0.4, Y=120
        if (Assets.images.ground) this.addLayer(Assets.images.ground, 1.0, CONFIG.GROUND_Y - 20);  // 速度比 1.0, 地面
    }

    addLayer(image, speedRatio, yOffset) {
        this.layers.push({
            image: image,
            pattern: this.ctx.createPattern(image, 'repeat'), // 使用 Pattern 實現無限捲動
            speedRatio: speedRatio,
            yOffset: yOffset,
            offsetX: 0
        });
    }

    update(dt, gameSpeed) {
        this.layers.forEach(layer => {
            // offsetX = 速度 * 視差比 * 時間 (s = v * dt)
            // 因為是向左捲動，偏移量需累加
            layer.offsetX += gameSpeed * layer.speedRatio * dt;
        });
    }

    draw() {
        this.layers.forEach(layer => {
            this.ctx.save();
            this.ctx.translate(-Math.floor(layer.offsetX), layer.yOffset); // 移動坐標系
            this.ctx.fillStyle = layer.pattern; // 使用 Pattern 填滿
            // 繪製一個超大的矩形，確保捲動時不會出現邊界
            this.ctx.fillRect(Math.floor(layer.offsetX), 0, CONFIG.CANVAS_WIDTH, layer.image.height);
            this.ctx.restore();
        });
    }
}

/**
 * 恐龍實體 - 含動畫狀態機
 */
class Dino {
    constructor() {
        this.reset();
        this.animTimer = 0;
        this.frame = 0;
        this.width = 64, this.height = 64; // 視覺大小
        this.y = CONFIG.GROUND_Y - this.height;
    }

    reset() {
        this.x = 50;
        this.width = 64, this.height = 64;
        this.y = CONFIG.GROUND_Y - this.height;
        this.dy = 0;
        this.isGrounded = true;
        this.state = 'RUN'; // RUN, JUMP, EVOLVE
    }

    jump() {
        if (this.isGrounded) {
            this.dy = CONFIG.JUMP_FORCE;
            this.isGrounded = false;
        }
    }

    update(dt, score) {
        // 進化系統
        if (score >= 500) { this.state = 'EVOLVE'; this.width = 72; }
        else if (!this.isGrounded) this.state = 'JUMP';
        else this.state = 'RUN';

        // 物理
        this.dy += CONFIG.GRAVITY * dt;
        this.y += this.dy * dt;

        if (this.y + this.height > CONFIG.GROUND_Y) {
            this.y = CONFIG.GROUND_Y - this.height;
            this.dy = 0;
            this.isGrounded = true;
        }

        // 動畫幀 (0.1s 切換)
        this.animTimer += dt;
        if (this.animTimer > 0.1) {
            this.frame = (this.frame + 1) % 2;
            this.animTimer = 0;
        }
    }

    draw(ctx) {
        const img = Assets.images.dino;
        if (img) {
            let sx = this.frame * 100; // 裁剪 X (假設 100x100 一幀)
            let sy = (this.state === 'EVOLVE') ? 100 : 0; // 裁剪 Y
            ctx.drawImage(img, sx, sy, 100, 100, this.x, this.y, this.width, this.height);
        } else {
            // Fallback
            ctx.fillStyle = (this.state === 'EVOLVE') ? '#2a9d8f' : '#535353';
            ctx.fillRect(this.x, this.y, this.width, this.height);
        }
    }

    get hitbox() {
        const pad = CONFIG.HITBOX_PADDING;
        return { x: this.x + pad, y: this.y + pad, w: this.width - pad*2, h: this.height - pad*2 };
    }
}

/**
 * 障礙物實體
 */
class Obstacle {
    constructor(gameSpeed) {
        this.width = 30 + Math.random() * 20;
        this.height = 40 + Math.random() * 40;
        this.x = CONFIG.CANVAS_WIDTH;
        this.y = CONFIG.GROUND_Y - this.height;
        // 隨機攝動速度 0.8x ~ 1.2x
        this.speed = gameSpeed * (0.8 + Math.random() * 0.4);
        this.type = Math.floor(Math.random() * 3); // 隨機樣式
    }

    update(dt) { this.x -= this.speed * dt; }

    draw(ctx) {
        const img = Assets.images.cactus;
        if (img) {
            // 裁剪 Cactus Spritesheet (假設每格 50x100)
            ctx.drawImage(img, this.type * 50, 0, 50, 100, this.x, this.y, this.width, this.height);
        } else {
            // Fallback
            ctx.fillStyle = '#ff4d4d';
            ctx.fillRect(this.x, this.y, this.width, this.height);
        }
    }

    get hitbox() {
        const pad = CONFIG.HITBOX_PADDING;
        return { x: this.x + pad, y: this.y + pad, w: this.width - pad*2, h: this.height - pad*2 };
    }
}

/**
 * 遊戲引擎 (OOP 架構)
 */
class Engine {
    constructor() {
        this.canvas = document.getElementById('gameCanvas');
        this.ctx = this.canvas.getContext('2d');
        this.score = 0;
        this.gameSpeed = CONFIG.INITIAL_SPEED;
        this.isGameOver = false;
        this.lastTime = performance.now();
        this.spawnTimer = 0;

        this.parallax = new ParallaxManager(this.ctx);
        this.dino = new Dino();
        this.obstacles = [];

        this.initInput();
        requestAnimationFrame(t => this.loop(t));
    }

    initInput() {
        const handleAction = () => { if (this.isGameOver) location.reload(); else this.dino.jump(); };
        window.addEventListener('keydown', e => { if (e.code === 'Space' || e.code === 'ArrowUp') handleAction(); });
        this.canvas.addEventListener('touchstart', e => { e.preventDefault(); handleAction(); });
    }

    update(dt) {
        // 非線性難度曲線 (基於分數)
        this.score += dt * 10;
        this.gameSpeed = CONFIG.INITIAL_SPEED + Math.pow(this.score, 0.6) * 5;

        // 更新各系統
        this.parallax.update(dt, this.gameSpeed);
        this.dino.update(dt, this.score);

        // 生成障礙 (間隔隨速度縮短)
        this.spawnTimer += dt * 1000;
        const interval = 1600 * (CONFIG.INITIAL_SPEED / this.gameSpeed);
        if (this.spawnTimer > interval) {
            this.obstacles.push(new Obstacle(this.gameSpeed));
            this.spawnTimer = 0;
        }

        // 障礙物與碰撞偵測
        for (let i = this.obstacles.length - 1; i >= 0; i--) {
            const obs = this.obstacles[i];
            obs.update(dt);
            if (this.checkCollision(this.dino, obs)) this.gameOver();
            if (obs.x < -100) this.obstacles.splice(i, 1); // 移除出界物件
        }

        // 更新 HUD
        document.getElementById('score').innerText = Math.floor(this.score).toString().padStart(5, '0');
        document.getElementById('status').innerText = `Status: ${this.dino.state} | Speed: ${Math.floor(this.gameSpeed)}`;
    }

    checkCollision(a, b) {
        const r1 = a.hitbox; const r2 = b.hitbox;
        return r1.x < r2.x + r2.w && r1.x + r1.w > r2.x && r1.y < r2.y + r2.h && r1.y + r1.h > r2.y;
    }

    draw() {
        this.ctx.clearRect(0, 0, CONFIG.CANVAS_WIDTH, CONFIG.CANVAS_HEIGHT);
        
        // 天空背景 (Fallback)
        if (!Assets.images.clouds) { this.ctx.fillStyle = '#87CEEB'; this.ctx.fillRect(0,0,800,250); }

        this.parallax.draw(); // 繪製視差背景

        // Fallback 地面線
        if (!Assets.images.ground) {
            this.ctx.strokeStyle = '#535353'; this.ctx.lineWidth = 2;
            this.ctx.beginPath(); this.ctx.moveTo(0, CONFIG.GROUND_Y); this.ctx.lineTo(800, CONFIG.GROUND_Y); this.ctx.stroke();
        }

        this.dino.draw(this.ctx);
        this.obstacles.forEach(o => o.draw(this.ctx));

        if (this.isGameOver) {
            this.ctx.fillStyle = 'rgba(0,0,0,0.7)';
            this.ctx.fillRect(0,0,CONFIG.CANVAS_WIDTH,CONFIG.CANVAS_HEIGHT);
            this.ctx.fillStyle = 'white';
            this.ctx.font = 'bold 24px sans-serif'; this.ctx.textAlign = 'center';
            this.ctx.fillText("GAME OVER", CONFIG.CANVAS_WIDTH/2, CONFIG.CANVAS_HEIGHT/2);
            this.ctx.font = '14px sans-serif';
            this.ctx.fillText("PRESS SPACE TO RESTART", CONFIG.CANVAS_WIDTH/2, CONFIG.CANVAS_HEIGHT/2 + 30);
        }
    }

    gameOver() { this.isGameOver = true; }

    loop(timestamp) {
        const dt = Math.min((timestamp - this.lastTime) / 1000, 0.1); // 限制 dt 防止跳窗崩潰
        this.lastTime = timestamp;

        if (!this.isGameOver) {
            this.update(dt);
            this.draw();
            requestAnimationFrame(t => this.loop(t));
        }
    }
}

/**
 * 引擎啟動流程 (專業非同步載入)
 */
async function bootstrap() {
    const success = await Assets.loadAll();
    
    // 隱藏載入畫面
    const loading = document.getElementById('loading');
    loading.style.opacity = '0';
    setTimeout(() => loading.style.display = 'none', 500);

    console.log(success ? "Assets loaded successfully." : "Using color fallback mode.");
    new Engine(); // 啟動遊戲
}

bootstrap();
</script>
</body>
</html>
