## Game Dev Learning Path — CS Background, No Game Experience

---

## Mental Model Shift First

Coming from data engineering, you think in:
```
pipelines → batch jobs → schemas → queries → dashboards
```

Game dev thinks in:
```
game loop → state → events → render → repeat (60x/second)
```

The **game loop** is the single most important concept:

```javascript
// This runs 60 times per second. Everything lives here.
function gameLoop(timestamp) {
    processInput()    // what is the player doing RIGHT NOW
    update(delta)     // move things, check collisions, apply physics
    render()          // draw the current state to screen
    requestAnimationFrame(gameLoop)  // schedule next frame
}
```

Your Kafka/Flink background: think of it as a **micro-batch every 16ms**. Input = source, update = transform, render = sink.

---

## Stage 1 — Core Concepts (Week 1, ~10 hours)

### 1. The Coordinate System

Games use a **canvas coordinate system** — different from math:

```
(0,0) ─────────────────► X increases RIGHT
  │
  │
  │
  ▼
  Y increases DOWN        (800, 500) bottom-right corner
```

```javascript
// Top-left corner of screen
sprite.x = 0; sprite.y = 0;

// Center of 800x500 canvas
sprite.x = 400; sprite.y = 250;

// Move right: increase X
sprite.x += 5;

// Move down: increase Y (counter-intuitive from math class)
sprite.y += 5;
```

---

### 2. Phaser 3 Scene Structure

Every Phaser game has scenes. A scene = a game state (menu, gameplay, game-over):

```javascript
class GameScene extends Phaser.Scene {

    constructor() {
        super({ key: 'GameScene' });
    }

    // Called ONCE — load assets (images, audio, tilemaps)
    preload() {
        this.load.image('player', 'assets/player.png');
        this.load.image('enemy',  'assets/enemy.png');
        this.load.image('bullet', 'assets/bullet.png');
        this.load.audio('jump',   'assets/jump.mp3');
        this.load.tilemapTiledJSON('map', 'assets/level1.json');
    }

    // Called ONCE after preload — create game objects
    create() {
        // Background
        this.add.image(400, 300, 'background');

        // Player sprite with physics
        this.player = this.physics.add.sprite(100, 400, 'player');
        this.player.setCollideWorldBounds(true);  // can't leave screen

        // Input
        this.cursors = this.input.keyboard.createCursorKeys();
        this.wasd = this.input.keyboard.addKeys('W,A,S,D');

        // Score (text object)
        this.scoreText = this.add.text(16, 16, 'Score: 0', {
            fontSize: '24px',
            fill: '#fff'
        });

        // Groups (manage multiple similar objects)
        this.enemies = this.physics.add.group();
        this.bullets = this.physics.add.group();

        // Collision detection
        this.physics.add.overlap(
            this.bullets,
            this.enemies,
            this.bulletHitsEnemy,  // callback
            null,
            this
        );

        this.physics.add.collider(
            this.player,
            this.enemies,
            this.playerDies,
            null,
            this
        );
    }

    // Called 60x/second — your game logic lives here
    update(time, delta) {
        // delta = milliseconds since last frame (~16.67ms at 60fps)
        // Always use delta for movement so speed is frame-rate independent

        this.handleInput();
        this.moveEnemies();
        this.checkWinCondition();
    }

    handleInput() {
        this.player.setVelocityX(0);  // stop horizontal by default

        if (this.cursors.left.isDown || this.wasd.A.isDown) {
            this.player.setVelocityX(-300);
            this.player.setFlipX(true);  // face left
        }
        if (this.cursors.right.isDown || this.wasd.D.isDown) {
            this.player.setVelocityX(300);
            this.player.setFlipX(false);
        }

        // Jump — only when touching ground
        if ((this.cursors.up.isDown || this.wasd.W.isDown)
            && this.player.body.blocked.down) {
            this.player.setVelocityY(-600);
        }
    }

    bulletHitsEnemy(bullet, enemy) {
        bullet.destroy();
        enemy.destroy();
        this.score += 10;
        this.scoreText.setText('Score: ' + this.score);
    }

    playerDies(player, enemy) {
        this.scene.start('GameOverScene');  // switch scenes
    }
}

// Boot the game
const config = {
    type: Phaser.AUTO,           // WebGL if available, Canvas fallback
    width: 800,
    height: 500,
    backgroundColor: '#1a1a2e',
    physics: {
        default: 'arcade',
        arcade: { gravity: { y: 800 }, debug: false }
    },
    scene: [MenuScene, GameScene, GameOverScene]
};

new Phaser.Game(config);
```

---

### 3. Physics Systems — Which to Pick

**Arcade Physics** — use this for 90% of games:
```javascript
// Simple AABB (axis-aligned bounding box) collisions
// No rotation physics, but very fast
this.physics.add.sprite(x, y, 'key')

// Velocity-based movement
sprite.setVelocityX(300)   // pixels per second
sprite.setVelocityY(-600)  // negative = up (remember Y axis)

// Gravity (set per object or globally)
sprite.setGravityY(800)

// Bounce
sprite.setBounce(0.5)       // 0 = no bounce, 1 = full bounce
```

**Matter.js** — for complex physics (rotation, joints, constraints):
```javascript
// Enable in config
physics: { default: 'matter', matter: { gravity: { y: 1 } } }

// Objects rotate, have real mass and friction
const box = this.matter.add.image(x, y, 'box');
box.setMass(5);
box.setFriction(0.5);
```

**Rapier (WASM)** — most realistic, use for production games:
```javascript
// Import Rapier
import RAPIER from '@dimforge/rapier2d';
// Better simulation, handles edge cases Matter.js misses
// Worth using when physics needs to feel "correct"
```

---

### 4. Sprites and Animations

```javascript
// In preload() — load a spritesheet (single image with frames)
this.load.spritesheet('player', 'assets/player.png', {
    frameWidth: 48,   // each frame is 48px wide
    frameHeight: 48
});

// In create() — define animations
this.anims.create({
    key: 'walk',
    frames: this.anims.generateFrameNumbers('player', { start: 0, end: 7 }),
    frameRate: 12,      // 12 frames per second
    repeat: -1          // -1 = loop forever
});

this.anims.create({
    key: 'jump',
    frames: this.anims.generateFrameNumbers('player', { start: 8, end: 11 }),
    frameRate: 8,
    repeat: 0           // play once
});

this.anims.create({
    key: 'idle',
    frames: this.anims.generateFrameNumbers('player', { start: 0, end: 0 }),
    frameRate: 1
});

// In update() — play based on state
if (this.player.body.blocked.down) {
    if (Math.abs(this.player.body.velocity.x) > 10) {
        this.player.anims.play('walk', true);  // true = don't restart if already playing
    } else {
        this.player.anims.play('idle', true);
    }
} else {
    this.player.anims.play('jump', true);
}
```

---

### 5. Tilemaps (Level Design)

Use **Tiled** (free tool) to design levels, export as JSON, load in Phaser:

```javascript
// preload()
this.load.tilemapTiledJSON('level1', 'assets/level1.json');
this.load.image('tiles', 'assets/tileset.png');

// create()
const map    = this.make.tilemap({ key: 'level1' });
const tiles  = map.addTilesetImage('tileset', 'tiles');

// Background layer (no collision)
const bgLayer  = map.createLayer('Background', tiles, 0, 0);

// Ground layer (collision)
const ground   = map.createLayer('Ground', tiles, 0, 0);
ground.setCollisionByProperty({ collides: true });  // set in Tiled

// Player collides with ground
this.physics.add.collider(this.player, ground);

// Objects from Tiled (spawn points, coins, enemies)
const spawnPoint = map.findObject('Objects', obj => obj.name === 'Spawn');
this.player.setPosition(spawnPoint.x, spawnPoint.y);
```

---

## Stage 2 — Multiplayer Layer (Colyseus, Week 2–3)

### Server State → All Clients Automatically

This is Colyseus's killer feature — define state on server, it syncs to all clients:

```typescript
// SERVER — server.ts
import { Schema, type, MapSchema } from "@colyseus/schema";

class Player extends Schema {
    @type("number") x: number = 0;
    @type("number") y: number = 0;
    @type("number") score: number = 0;
    @type("string") animation: string = "idle";
}

class GameState extends Schema {
    @type({ map: Player }) players = new MapSchema<Player>();
    @type("number") timeLeft: number = 60;
    @type("string") phase: string = "waiting"; // waiting → playing → ended
}

class GameRoom extends Room<GameState> {

    maxClients = 4;  // max 4 players per room

    onCreate(options: any) {
        this.setState(new GameState());

        // Handle movement from any client
        this.onMessage("move", (client, data: { x: number, y: number }) => {
            const player = this.state.players.get(client.sessionId);
            if (!player) return;
            // Validate server-side (never trust client)
            player.x = Math.max(0, Math.min(800, data.x));
            player.y = Math.max(0, Math.min(500, data.y));
            // ↑ This change auto-broadcasts to ALL clients
        });

        this.onMessage("shoot", (client, data) => {
            // Broadcast bullet creation to all clients
            this.broadcast("bullet_fired", {
                fromPlayer: client.sessionId,
                x: data.x, y: data.y,
                angle: data.angle
            }, { except: client }); // send to everyone EXCEPT shooter
        });

        // Countdown timer
        this.clock.setInterval(() => {
            if (this.state.phase !== "playing") return;
            this.state.timeLeft--;
            if (this.state.timeLeft <= 0) this.endGame();
        }, 1000);
    }

    onJoin(client: Client, options: any) {
        const player = new Player();
        player.x = Phaser.Math.Between(100, 700);  // random spawn
        player.y = 400;
        this.state.players.set(client.sessionId, player);

        if (this.state.players.size >= 2) {
            this.state.phase = "playing";
        }
    }

    onLeave(client: Client) {
        this.state.players.delete(client.sessionId);
    }

    endGame() {
        this.state.phase = "ended";
        // Find winner
        let winner = { id: "", score: -1 };
        this.state.players.forEach((player, id) => {
            if (player.score > winner.score) winner = { id, score: player.score };
        });
        this.broadcast("game_over", { winner: winner.id });
    }
}
```

```javascript
// CLIENT — Phaser scene with Colyseus
import { Client } from "colyseus.js";

class MultiplayerScene extends Phaser.Scene {

    create() {
        this.otherPlayers = {};  // track other players' sprites

        const client = new Client("ws://localhost:3000");

        client.joinOrCreate("game_room").then(room => {
            this.room = room;

            // When a player joins (including yourself)
            room.state.players.onAdd((player, sessionId) => {
                const isMe = sessionId === room.sessionId;
                const sprite = this.physics.add.sprite(player.x, player.y, 'player');

                if (isMe) {
                    this.mySprite = sprite;
                } else {
                    this.otherPlayers[sessionId] = sprite;
                }

                // React to server state changes for this player
                player.onChange(() => {
                    sprite.setPosition(player.x, player.y);
                    sprite.anims.play(player.animation, true);
                });
            });

            // When a player leaves
            room.state.players.onRemove((_, sessionId) => {
                if (this.otherPlayers[sessionId]) {
                    this.otherPlayers[sessionId].destroy();
                    delete this.otherPlayers[sessionId];
                }
            });

            // Listen for server events
            room.onMessage("bullet_fired", (data) => {
                this.spawnBullet(data.x, data.y, data.angle);
            });

            room.onMessage("game_over", (data) => {
                this.scene.start('ResultsScene', { winner: data.winner });
            });

            // Timer sync
            room.state.listen("timeLeft", (value) => {
                this.timerText.setText(value);
            });
        });
    }

    update() {
        if (!this.room || !this.mySprite) return;

        // Only send input when something changes
        const moved = this.handleInput();
        if (moved) {
            this.room.send("move", {
                x: this.mySprite.x,
                y: this.mySprite.y
            });
        }
    }
}
```

---

## Stage 3 — Game Feel (What Separates Good From Great)

These are the details that make games feel polished:

```javascript
// 1. SCREEN SHAKE — on hit, explosion
this.cameras.main.shake(200, 0.01);

// 2. ZOOM — for dramatic moments
this.cameras.main.zoomTo(2, 500);  // zoom to 2x over 500ms

// 3. PARTICLE EFFECTS — explosions, collectibles
const particles = this.add.particles(x, y, 'spark', {
    speed: { min: 50, max: 200 },
    angle: { min: 0, max: 360 },
    scale: { start: 1, end: 0 },
    lifespan: 500,
    quantity: 20
});

// 4. TWEENS — smooth animations
this.tweens.add({
    targets: sprite,
    y: sprite.y - 50,      // float up 50px
    alpha: 0,              // fade out
    duration: 800,
    ease: 'Power2',
    onComplete: () => sprite.destroy()
});

// 5. FLASH ON HIT
this.tweens.add({
    targets: player,
    alpha: 0,
    duration: 100,
    yoyo: true,        // go back to alpha 1
    repeat: 3          // flash 3 times
});

// 6. TIME SCALE — slow motion
this.physics.world.timeScale = 0.2;   // slow everything to 20%
this.time.delayedCall(1000, () => {
    this.physics.world.timeScale = 1; // back to normal
});
```

---

## Best Free Assets to Start With

| Asset | Source | Use |
|-------|--------|-----|
| Sprites | **itch.io** (free section) | Characters, enemies, tiles |
| Tilemaps | **Kenney.nl** | Complete free game assets |
| Tiled | **mapeditor.org** | Level editor (free) |
| Audio | **freesound.org** | Sound effects |
| Fonts | **Google Fonts** | UI text |

**Kenney.nl** is the go-to for getting started — entire game asset packs, free, no attribution needed.

---

## 10 Phaser Examples to Run (In This Order)

```
1.  physics/arcade/simple body         → understand velocity
2.  physics/arcade/platformer          → player + jump + ground
3.  animation/create from spritesheet  → sprite animation
4.  input/keyboard/cursor keys         → input handling
5.  physics/arcade/overlap vs collider → collision types
6.  game objects/group/create multiple → managing groups of enemies
7.  tilemaps/collision/matter physics  → level design
8.  camera/follow                      → camera follows player
9.  tweens/basics/yoyo                 → visual polish
10. scenes/basic scene                 → scene management
```

All at: **phaser.io/examples** — click "Edit" on any example to see the full code.

---

## Your Learning Path in Hours

```
Hours 1-3:   Game loop concept + coordinate system + Phaser 3 setup
Hours 4-8:   Build a basic platformer (player, ground, jump, enemies)
Hours 9-12:  Add spritesheet animation + sound + score
Hours 13-16: Add tilemap level from Tiled editor
Hours 17-20: Add Colyseus server + basic multiplayer (2 players move together)
Hours 21-24: Wrap it all in an XBlock, push score events to Open edX

= 24 hours → functional multiplayer game XBlock
```

That's 3 focused days of work. With your CS fundamentals, the concepts will click fast — game dev is mostly applied data structures (spatial grids, event queues, state machines) with a rendering layer on top.

## Your Background vs. This Project

You're stronger than you think going in. Let me be precise about what you know, what transfers, and what's genuinely new.

---

## What You Already Have (No Study Needed)

| Your Skill | How It Applies Directly |
|-----------|------------------------|
| Python | XBlock backend — exact same language |
| MySQL | Open edX primary DB — you already know it |
| MongoDB Atlas | Open edX content store — same thing |
| Docker + Kubernetes | Tutor is Docker Compose → K8s, you're already ahead |
| React + JavaScript | XBlock frontends + MFE apps |
| Flask | Django will feel familiar — same request/response pattern |
| AWS (S3, EC2, EKS, RDS) | Production deployment — Phase 5 |
| Kafka/Airflow | Celery patterns will make sense immediately — same async job concept |
| Redis | You've used it — same Redis, now for sessions + leaderboards |
| Data pipelines | You understand data flow deeply — XP event pipeline is trivial for you |

**Honest assessment: ~60% of this project is in your wheelhouse.**

---

## What You Need to Study (Prioritized)

### 🔴 Critical — Study Before Week 2

**1. Django (3–5 days)**

Your Flask background helps but Django has conventions Flask doesn't:
```
Focus on:
- Django ORM (models, migrations, querysets) — you'll use this constantly
- Django views and URL routing
- Django settings system (how Open edX layers settings)
- Django management commands

Skip:
- Django templates (Open edX uses its own rendering)
- Django forms (XBlocks don't use them)
```
Best resource: **Django official tutorial** (djangoproject.com/start) — do parts 1–4 only, takes 1 day.

**2. XBlock API (2 days)**

Nothing else in your background prepares you for this — it's Open edX-specific:
```
Focus on:
- XBlock fields and Scopes
- student_view() / studio_view()
- @XBlock.json_handler
- runtime.publish() for events
- Fragment-based rendering
```
Best resource: **XBlock Tutorial** — docs.openedx.org/projects/xblock

---

### 🟡 Important — Study During Build Phases

**3. Django ORM in depth (1–2 days)**

You know SQL deeply from your data engineering work. The translation is:
```python
# Your instinct (raw SQL):
SELECT * FROM auth_user WHERE is_active = 1

# Django way (what you must use):
User.objects.filter(is_active=True)
```
Your SQL expertise actually makes ORM easier to learn — you understand what's happening underneath.

**4. WebSockets + Colyseus (2–3 days, before Multiplayer XBlock)**

You've done Kafka (pub/sub at scale). WebSockets are simpler:
```
Kafka    → broker → consumer  (async, durable, distributed)
WebSocket→ direct connection  (real-time, stateful, per-session)

Colyseus = room management on top of WebSockets
         = like Kafka consumer groups but for game sessions
```
Your Kafka mental model transfers — the concepts are similar, just real-time and lighter.

**5. Phaser 3 / Game Development (3–5 days, before game XBlocks)**

No game dev on your resume. Start here:
```
Focus on:
- Game loop (update/render cycle)
- Sprites and physics
- Input handling
- Scene management

Your advantage: you understand data structures + algorithms deeply,
which is 80% of game logic
```
Best resource: **Phaser 3 official examples** (phaser.io/examples) — run 10 examples, then build something small.

---

### 🟢 Nice to Have — Pick Up As Needed

**6. Mapbox GL JS (1 day)**
- Well-documented, JavaScript — you'll pick it up in an afternoon
- Geospatial concepts: you have PostGIS/data background, coordinates are trivial

**7. LTI 1.3 / OAuth2 concepts (1 day, Phase 4)**
- You've worked with GDPR/CCPA compliance — security model mindset already there
- LTI is just a structured OAuth2 handshake with an edu-specific payload

**8. SCSS/CSS theming (half a day)**
- You have enough JS/React — SCSS is just CSS with variables

---

## Your Unique Advantages on This Project

Things your background gives you that most web devs wouldn't have:

**1. Learning Analytics** — Your ML + data pipeline background means you can instrument the platform for research-grade data collection. This is directly publishable at LAK/CHI/AIED (the goal Dr. Gupta mentioned).

**2. Production data at scale** — You've handled 150TB+ pipelines. 500 students is trivial operationally. You'll spot scaling issues before they happen.

**3. LLM integration** — Your LangChain + HuggingFace + agentic AI background opens doors the project doesn't even have planned yet:
- AI-powered quest generation (QuestEngine)
- Adaptive difficulty in game XBlocks
- LLM-assisted peer matching in GlobalMap

**4. Monitoring** — You built Grafana/Superset dashboards professionally. The platform monitoring for the pilot (SLAs, uptime, user activity) is second nature.

---

## Recommended Study Order (Before Week 2 Build Starts)

```
Week 1 (now) — environment is set up ✅

Before Week 2:
Day 1-2:  Django tutorial (parts 1-4 only)
Day 3:    Django ORM — run queries in tutor dev run lms ./manage.py lms shell
Day 4-5:  XBlock API docs + run the architecture.md example from the guide
Day 6:    Build the simplest possible XBlock (just displays "Hello" + handler)
Day 7:    Get it working in Studio + LMS ← this is your proof of concept

Then Week 2: Start Gamification XBlock for real
```

---

## Bottom Line

You're a strong data engineer with ML depth going into a Django/XBlock web project. The gap is **not Python, not databases, not Docker** — it's specifically **Django conventions** and the **XBlock API**. Two focused days on those and you're unblocked. Everything else you'll pick up as you build.