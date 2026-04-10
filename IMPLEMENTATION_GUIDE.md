# SLINGSHOT GAME - IMPLEMENTATION GUIDE

**Complete guide to understanding, extending, and deploying the gesture-based physics game.**

---

## 🎯 QUICK START

### **Installation**
```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview
```

### **Camera Permission Flow**
1. User opens app
2. Browser requests camera access
3. User grants/denies permission
4. Game initializes with vision (or fallback to touch/mouse)

### **First Game**
1. Show hand to camera (or click/touch screen)
2. Pinch thumb and index finger around the ball (or click)
3. Drag backward to aim and charge power
4. Release finger/mouse to launch projectile

---

## 🧠 CORE CONCEPTS

### **Why This Architecture?**

**Problem**: Real-time gesture-based games require low latency (< 50ms) and high frame rate (60 FPS).

**Solution**: Decouple concerns into 4 independent engines:

```
┌─────────────────────────────────────────────┐
│ Engine         │ Responsibility              │
├────────────────┼─────────────────────────────┤
│ Vision         │ Extract hand landmarks      │
│ Gesture        │ Interpret hand movements    │
│ Physics        │ Simulate realistic motion   │
│ Rendering      │ Visual feedback + effects   │
└─────────────────────────────────────────────┘
```

Each engine:
- ✅ Has clear input/output contracts
- ✅ Can be tested independently
- ✅ Can be optimized separately
- ✅ Can be replaced with alternatives

---

## 📚 ENGINE DEEP DIVES

### **1. Computer Vision Engine**

**Location**: `src/engines/ComputerVisionEngine.ts`

**What It Does**:
- Captures video from camera
- Runs MediaPipe hand detection
- Outputs normalized hand landmarks (0-1)
- Applies exponential smoothing filter

**Key Functions**:

```typescript
// Initialize camera
await cvEngine.initialize(videoElement, canvasElement);

// Start detection
await cvEngine.start();

// Get current hand data
const handData = cvEngine.getCurrentHandData();

// Get specific landmarks
const thumbTip = cvEngine.getThumbTip();    // Landmark 4
const indexTip = cvEngine.getIndexTip();    // Landmark 8

// Calculate distance
const distance = cvEngine.getLandmarkDistance(4, 8);

// Get hand size
const size = cvEngine.getHandSize();

// Register callback
cvEngine.onResults((handData) => {
  console.log("Hand detected:", handData.landmarks.length);
});
```

**Smoothing Details**:

MediaPipe outputs "jittery" raw landmarks. The engine applies exponential smoothing:

```typescript
// Before smoothing: raw data from MediaPipe
// {x: 0.524, y: 0.381, z: -0.123}  // Frame 1
// {x: 0.528, y: 0.379, z: -0.121}  // Frame 2 (slight jitter)

// After smoothing (α = 0.8):
// smoothed = prev * 0.8 + current * 0.2
// {x: 0.5245, y: 0.3803, z: -0.1223}  // Much smoother
```

**Why 0.8?**
- Too low (0.5): Follows hand too slowly, laggy feel
- Too high (0.95): Too much smoothing, unresponsive
- 0.8 (Goldilocks): Removes jitter while maintaining responsiveness

---

### **2. Gesture Engine**

**Location**: `src/engines/GestureEngine.ts`

**What It Does**:
- Converts raw hand data into gestures
- Implements state machine (IDLE → GRAB → DRAG → RELEASE)
- Calculates drag metrics (distance, angle, velocity)
- Provides temporal stability (3-frame hysteresis)

**State Machine Logic**:

```
IDLE
  └─ if (pinchStrength > 0.6 for 3+ frames)
     └─→ GRAB

GRAB
  ├─ if (movement > 10px)
  │  └─→ DRAG
  └─ if (pinch released)
     └─→ RELEASE

DRAG
  ├─ if (hand moved)
  │  └─ Update drag position/angle
  └─ if (pinch released)
     └─→ RELEASE

RELEASE
  └─ (Auto-reset)
     └─→ IDLE
```

**Key Functions**:

```typescript
// Update gesture each frame
const gesture = gestureEngine.update(handData, canvasWidth, canvasHeight);

// Check current state
if (gesture.state === GestureState.DRAG) {
  console.log("Dragging at:", gesture.dragX, gesture.dragY);
  console.log("Power:", gesture.dragDistance / 120);  // 0-1
  console.log("Angle:", gesture.dragAngle * 180 / Math.PI);  // Degrees
}

// Register callbacks
gestureEngine.onStateChange((newState) => {
  console.log("State changed to:", newState);
});

gestureEngine.onGesture((gesture) => {
  if (gesture.state === GestureState.RELEASE) {
    // Launch projectile
    launchProjectile(gesture);
  }
});
```

**Dynamic Threshold Example**:

```typescript
// Hand 1: Small hand
handSize1 = 0.05
threshold1 = 0.05 * (1 + 0.05 * 0.5) = 0.0525

// Hand 2: Large hand
handSize2 = 0.15
threshold2 = 0.05 * (1 + 0.15 * 0.5) = 0.0575

// Large hand has larger pinch threshold → more pinching required
// Adapts to individual hand anatomy!
```

---

### **3. Physics Engine**

**Location**: `src/engines/PhysicsEngine.ts`

**What It Does**:
- Simulates projectile motion with gravity
- Handles collisions with static bodies
- Calculates object-to-object interactions
- Updates rotation and angular velocity

**Creating Objects**:

```typescript
// Create a projectile at slingshot position
const projectile = physicsEngine.createObject(
  x = 60,          // Initial X position
  y = 100,         // Initial Y position
  vx = -10,        // Velocity X (moving left)
  vy = -15,        // Velocity Y (moving up)
  radius = 8,      // Collision radius
  mass = 1         // Mass (1 = standard)
);

// Add static obstacles
physicsEngine.addStaticBody(
  x = 300,         // Position
  y = 200,
  width = 150,     // Dimensions
  height = 20,
  type = "rect"    // or "circle"
);
```

**Physics Update (Each Frame)**:

```
For each object:
  1. Apply gravity:
     vy += 0.6

  2. Apply air drag:
     vx *= 0.998
     vy *= 0.998

  3. Update position:
     x += vx
     y += vy

  4. Check ground collision:
     if (y + radius >= groundLevel)
       vy *= -0.4  // Bounce (40% restitution)
       vx *= 0.8   // Friction

  5. Check static collisions:
     For each static body:
       Check circle-rect collision
       If collision:
         Separate object
         Reflect velocity
         Trigger callback

  6. Check object-to-object:
     For all pairs:
       Check circle-circle
       If collision:
         Exchange momentum
         Update velocities
```

**Tuning Physics**:

```typescript
// To make game "floaty" (low gravity):
gravity: 0.3,          // 0.6 → 0.3 (2x less gravity)
restitution: 0.6,      // 0.4 → 0.6 (bouncier)
groundFriction: 0.9,   // 0.8 → 0.9 (less friction)

// To make game "heavy" (high gravity):
gravity: 1.0,          // 0.6 → 1.0 (1.67x more gravity)
restitution: 0.2,      // 0.4 → 0.2 (less bounce)
groundFriction: 0.6,   // 0.8 → 0.6 (more friction)
```

**Collision Detection Performance**:

Current: O(n*m) where n = projectiles, m = static bodies

For 20 projectiles + 10 static bodies = 200 checks per frame = ~0.3ms

Future: Use spatial hashing for O(n) collision detection

---

### **4. Rendering Engine**

**Location**: `src/engines/RenderingEngine.ts`

**What It Does**:
- Clears canvas with fade effect
- Draws slingshot with elastic bands
- Renders projectiles with rotation/trails
- Manages particle system
- Applies screen shake effect

**Drawing Pipeline**:

```typescript
// Per frame:
renderingEngine.render(
  projectiles,        // Physics objects to draw
  isDragging,         // Show slingshot or not
  dragX, dragY,       // Where user is dragging to
  dragDistance,       // Total drag power
  score               // Display score
);
```

**Behind the scenes**:

```
1. Clear canvas (fade to dark)
   ctx.fillStyle = "rgba(20, 20, 30, 0.1)";  // Slight fade
   ctx.fillRect(0, 0, width, height);

2. Draw background grid
   For visual reference

3. For each projectile:
   - Rotate sprite based on rotation
   - Draw with brightness based on speed
   - Draw trail if moving (motion blur)

4. Draw slingshot:
   - Base rectangle
   - Two elastic bands (Bezier curves)
   - Color indicates power

5. Update particles:
   - Apply gravity
   - Fade out
   - Draw with current alpha

6. Screen shake:
   - Apply small random offsets
   - Decay intensity
```

**Custom Effects**:

```typescript
// Add particle burst on impact
renderingEngine.addParticles(
  x, y,                     // Position
  count = 20,               // Particle count
  {
    speed: 5,               // Initial velocity
    color: "rgba(255, 180, 80, 0.8)",
    size: 4                 // Particle size
  }
);

// Trigger screen shake
renderingEngine.triggerScreenShake(intensity = 5);
```

---

## 🔌 EXTENDING THE GAME

### **Add New Obstacle Type**

**Step 1**: Define obstacle in `GameController.setupPhysics()`

```typescript
private addObstacles(): void {
  const groundY = this.config.canvasHeight - 30;

  // Add your custom obstacle
  this.physicsEngine.addStaticBody(
    x = 400,
    y = 300,
    width = 50,
    height = 50,
    type = "rect"
  );
}
```

**Step 2**: Handle collision effects

```typescript
private onCollision(result: any): void {
  // Trigger effect based on collision position
  this.renderingEngine.triggerScreenShake(3);
  this.renderingEngine.addParticles(
    result.impactX,
    result.impactY,
    15,
    { speed: 4, color: "rgba(200, 100, 100, 0.8)" }
  );

  // Award points
  this.score += 10;
}
```

### **Add New Gesture**

**Step 1**: Add gesture state in `GestureEngine.ts`

```typescript
enum GestureState {
  IDLE = "IDLE",
  GRAB = "GRAB",
  DRAG = "DRAG",
  RELEASE = "RELEASE",
  SWIPE = "SWIPE",  // New gesture
}
```

**Step 2**: Implement detection logic

```typescript
private updateGestureState(): void {
  // Existing logic...

  // Add swipe detection
  const speedX = Math.abs(this.velocity.x);
  if (speedX > 20 && !this.pinchStrength) {
    this.state = GestureState.SWIPE;
  }
}
```

**Step 3**: Handle in `GameController`

```typescript
private processGesture(gesture: GestureData): void {
  switch (gesture.state) {
    case GestureState.SWIPE:
      this.handleSwipe(gesture);
      break;
    // ...
  }
}
```

### **Add New Visual Effect**

**Step 1**: Create particle effect in `RenderingEngine`

```typescript
public addExplosion(x: number, y: number): void {
  this.addParticles(x, y, 50, {
    speed: 8,
    color: "rgba(255, 100, 50, 0.9)",
    size: 6
  });

  this.triggerScreenShake(15);
}
```

**Step 2**: Trigger from game logic

```typescript
if (projectile.speed > 20) {
  this.renderingEngine.addExplosion(collision.impactX, collision.impactY);
}
```

---

## 🧪 TESTING & DEBUGGING

### **Test Vision Mode**

```javascript
// In browser console:
const cvEngine = window.gameController.cvEngine;

// Check hand detection
console.log(cvEngine.getCurrentHandData());
// {
//   landmarks: [21 landmarks],
//   handedness: "Right",
//   confidence: 0.95,
//   detected: true,
//   timestamp: 1234567890
// }

// Check specific landmarks
console.log(cvEngine.getThumbTip());
// {x: 0.524, y: 0.381, z: -0.123}

console.log(cvEngine.getIndexTip());
// {x: 0.482, y: 0.312, z: -0.098}

// Check hand size
console.log(cvEngine.getHandSize());
// 0.085 (roughly 8.5% of screen)
```

### **Test Gesture Engine**

```javascript
// Monitor gesture state changes
gestureEngine.onStateChange((state) => {
  console.log("State:", state);
});

// Monitor gesture updates
gestureEngine.onGesture((gesture) => {
  if (gesture.state === "DRAG") {
    console.log("Drag distance:", gesture.dragDistance);
    console.log("Drag angle:", gesture.dragAngle * 180 / Math.PI);
    console.log("Pinch strength:", gesture.pinchStrength);
  }
});
```

### **Test Physics**

```javascript
// Create test projectile
const projectile = physicsEngine.createObject(
  100, 100,  // Position
  10, -15,   // Velocity
  8, 1       // Radius, mass
);

// Monitor physics updates
physics.onCollision((result) => {
  console.log("Collision at:", result.impactX, result.impactY);
  console.log("Normal:", result.normal);
});
```

### **Performance Profiling**

```javascript
// In browser DevTools:
1. Open Performance tab
2. Click "Record"
3. Play game for 5 seconds
4. Stop recording

// Look for:
- Frame rate: 60 FPS (solid line)
- Long tasks: < 16ms each
- Memory: stable, no growth
- GPU usage: < 50%
```

---

## 🚀 DEPLOYMENT

### **Build for Production**

```bash
npm run build
```

Generates: `dist/index.html` (~274 KB gzipped)

### **Deploy to Vercel**

```bash
npm install -g vercel
vercel
```

### **Deploy to Netlify**

```bash
npm install -g netlify-cli
netlify deploy --prod --dir=dist
```

### **Deploy to GitHub Pages**

```bash
# Update vite.config.ts base path
# Push to GitHub
# Enable Pages in settings
```

### **HTTPS Requirement**

Camera access requires HTTPS:
- ✅ localhost (development)
- ✅ https://yourdomain.com (production)
- ❌ http://yourdomain.com (will fail)

---

## 📊 METRICS & MONITORING

### **Key Metrics to Track**

```typescript
// Frame rate
const fps = Math.round(1000 / deltaTime);

// Hand detection lag
const visionLag = performance.now() - handData.timestamp;

// Physics object count
const activeObjects = physicsEngine.getObjects().length;

// Particle count
const particleCount = renderingEngine.particles.length;

// Memory usage
const memory = performance.memory?.usedJSHeapSize / 1e6;  // MB
```

### **Health Checks**

```typescript
const healthCheck = {
  avgFps: 0,              // Should be ~60
  maxLatency: 0,          // Should be < 50ms
  avgMemory: 0,           // Should be < 50MB
  crashCount: 0,          // Should be 0
  permissionDenied: false // Check camera access
};
```

---

## 🎨 CUSTOMIZATION

### **Change Slingshot Position**

```typescript
// In GameController.setupCanvas()
const slinghotX = 60;      // Change this
const slinghotY = this.config.canvasHeight - 100;  // Or this

this.renderingEngine.setSlinghotAnchor(slinghotX, slinghotY);
```

### **Change Projectile Color**

```typescript
// In RenderingEngine.drawProjectile()
const brightness = Math.min(50 + speed * 2, 100);
ctx.fillStyle = `hsl(40, 100%, ${brightness}%)`;  // Change hue (40) to 0-360
```

### **Change Gravity**

```typescript
// In GameController constructor
const physicsEngine = new PhysicsEngine({
  gravity: 0.6,  // Change this (try 0.3-1.5)
  // ...
});
```

### **Change Slingshot Max Power**

```typescript
// In GameController.launchProjectile()
const power = Math.min(gesture.dragDistance / 120, 1.0);  // Change divisor
const velocity = power * 20;  // Change multiplier
```

---

## 📝 CODE STYLE GUIDE

### **Naming Conventions**

```typescript
// Classes: PascalCase
class ComputerVisionEngine {}
class PhysicsEngine {}

// Functions: camelCase
function updatePhysics() {}
function drawProjectile() {}

// Constants: UPPER_SNAKE_CASE
const MAX_DRAG_DISTANCE = 300;
const GRAVITY = 0.6;

// Enums: PascalCase members
enum GestureState {
  IDLE = "IDLE",
  GRAB = "GRAB",
}
```

### **Type Safety**

Always use TypeScript interfaces:

```typescript
// ✅ Good
interface PhysicsObject {
  x: number;
  y: number;
  vx: number;
  vy: number;
}

// ❌ Bad
const object = {
  x: 100,
  y: 100,
  vx: 10,
  vy: -15,
};
```

### **Error Handling**

```typescript
// ✅ Good
try {
  await cvEngine.initialize(videoElement, canvasElement);
} catch (error) {
  console.error("Vision initialization failed:", error);
  this.gameState = GameState.ERROR;
}

// ❌ Bad
await cvEngine.initialize(videoElement, canvasElement);
```

---

## 🎓 LEARNING RESOURCES

### **Topics to Study**

1. **Computer Vision**
   - MediaPipe documentation
   - Hand pose estimation
   - Real-time video processing

2. **Gesture Recognition**
   - State machines
   - Temporal filtering
   - Stability detection

3. **Physics Simulation**
   - Newtonian mechanics
   - Collision detection
   - Impulse-based dynamics

4. **Real-Time Rendering**
   - Canvas 2D API
   - Animation loops
   - Performance optimization

### **Recommended Reading**

- Game Physics Engine Development (Ian Millington)
- Computer Vision with OpenCV (Gary Bradski)
- WebGL Programming Guide (Kouichi Matsuda)

---

**Document Version**: 1.0  
**Status**: ✅ COMPLETE  
**Last Updated**: 2026
