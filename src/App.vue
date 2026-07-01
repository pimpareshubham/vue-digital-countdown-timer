<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue';

// Configuration constants
const STORAGE_KEY = 'chronos_countdown_target';

// Target date ref
const targetDate = ref(null);
const isPaused = ref(false);
const pausedTimeRemaining = ref(null);
const currentDate = ref(new Date());

// State for time breakdown
const timeRemaining = ref({
  total: 0,
  years: 0,
  months: 0,
  days: 0,
  hours: 0,
  minutes: 0,
  seconds: 0,
  milliseconds: 0
});

// Canvas particle settings
const canvasRef = ref(null);
let animationFrameId = null;
let countdownIntervalId = null;

// Initialize target date: checks localStorage, otherwise sets to exactly 1 year from now
const initTargetDate = () => {
  const stored = localStorage.getItem(STORAGE_KEY);
  if (stored) {
    targetDate.value = new Date(stored);
  } else {
    resetToDefault();
  }
};

// Reset target date to exactly 1 year from now
const resetToDefault = () => {
  const future = new Date();
  future.setFullYear(future.getFullYear() + 1);
  targetDate.value = future;
  localStorage.setItem(STORAGE_KEY, future.toISOString());
  isPaused.value = false;
  pausedTimeRemaining.value = null;
};

// Precise countdown calculation
const updateCountdown = () => {
  if (isPaused.value) {
    if (pausedTimeRemaining.value !== null) {
      currentDate.value = new Date(targetDate.value.getTime() - pausedTimeRemaining.value);
    }
  } else {
    currentDate.value = new Date();
  }

  const target = targetDate.value;
  const current = currentDate.value;
  
  let diff = target.getTime() - current.getTime();
  
  if (diff <= 0) {
    timeRemaining.value = {
      total: 0, years: 0, months: 0, days: 0, hours: 0, minutes: 0, seconds: 0, milliseconds: 0
    };
    return;
  }

  let years = target.getFullYear() - current.getFullYear();
  let months = target.getMonth() - current.getMonth();
  let days = target.getDate() - current.getDate();
  let hours = target.getHours() - current.getHours();
  let minutes = target.getMinutes() - current.getMinutes();
  let seconds = target.getSeconds() - current.getSeconds();
  let milliseconds = target.getMilliseconds() - current.getMilliseconds();

  // Adjustments for calendar rollover
  if (milliseconds < 0) {
    milliseconds += 1000;
    seconds--;
  }
  if (seconds < 0) {
    seconds += 60;
    minutes--;
  }
  if (minutes < 0) {
    minutes += 60;
    hours--;
  }
  if (hours < 0) {
    hours += 24;
    days--;
  }
  if (days < 0) {
    const tempDate = new Date(current.getFullYear(), current.getMonth() + 1, 0);
    days += tempDate.getDate();
    months--;
  }
  if (months < 0) {
    months += 12;
    years--;
  }

  timeRemaining.value = {
    total: diff,
    years: Math.max(0, years),
    months: Math.max(0, months),
    days: Math.max(0, days),
    hours: Math.max(0, hours),
    minutes: Math.max(0, minutes),
    seconds: Math.max(0, seconds),
    milliseconds: Math.max(0, milliseconds)
  };
};

// Formatted utility functions
const formatNumber = (num) => String(num).padStart(2, '0');
const formatMilliseconds = (num) => String(num).padStart(3, '0');

// Percentage of year completed (for progress tracking)
const progressPercentage = computed(() => {
  const oneYearMs = 365 * 24 * 60 * 60 * 1000;
  const remaining = timeRemaining.value.total;
  const elapsed = Math.max(0, oneYearMs - remaining);
  return Math.min(100, (elapsed / oneYearMs) * 100);
});

// Controls
const togglePause = () => {
  isPaused.value = !isPaused.value;
  if (isPaused.value) {
    // Save remaining ms
    pausedTimeRemaining.value = targetDate.value.getTime() - new Date().getTime();
  } else {
    // Update target date to be current time + pausedTimeRemaining
    if (pausedTimeRemaining.value !== null) {
      const newTarget = new Date(new Date().getTime() + pausedTimeRemaining.value);
      targetDate.value = newTarget;
      localStorage.setItem(STORAGE_KEY, newTarget.toISOString());
    }
  }
};

const setCustomDate = (event) => {
  const val = event.target.value;
  if (val) {
    const custom = new Date(val);
    if (custom > new Date()) {
      targetDate.value = custom;
      localStorage.setItem(STORAGE_KEY, custom.toISOString());
      isPaused.value = false;
      pausedTimeRemaining.value = null;
    } else {
      alert("Please select a future date & time!");
    }
  }
};

// Canvas Background Particle Logic
class Particle {
  constructor(canvas) {
    this.canvas = canvas;
    this.x = Math.random() * canvas.width;
    this.y = Math.random() * canvas.height;
    this.vx = (Math.random() - 0.5) * 0.4;
    this.vy = (Math.random() - 0.5) * 0.4;
    this.radius = Math.random() * 2 + 0.5;
    this.alpha = Math.random() * 0.5 + 0.2;
    this.color = Math.random() > 0.5 ? '168, 85, 247' : '6, 182, 212'; // Purple vs Cyan
  }

  update() {
    this.x += this.vx;
    this.y += this.vy;

    if (this.x < 0 || this.x > this.canvas.width) this.vx = -this.vx;
    if (this.y < 0 || this.y > this.canvas.height) this.vy = -this.vy;
  }

  draw(ctx) {
    ctx.beginPath();
    ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
    ctx.fillStyle = `rgba(${this.color}, ${this.alpha})`;
    ctx.shadowBlur = this.radius * 2;
    ctx.shadowColor = `rgba(${this.color}, ${this.alpha})`;
    ctx.fill();
  }
}

const setupCanvas = () => {
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  
  const resize = () => {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
  };
  window.addEventListener('resize', resize);
  resize();

  const particles = Array.from({ length: 80 }, () => new Particle(canvas));

  const animate = () => {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    
    // Draw background grid lines faintly
    ctx.strokeStyle = 'rgba(255, 255, 255, 0.02)';
    ctx.lineWidth = 1;
    const gridSize = 60;
    for (let x = 0; x < canvas.width; x += gridSize) {
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, canvas.height);
      ctx.stroke();
    }
    for (let y = 0; y < canvas.height; y += gridSize) {
      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(canvas.width, y);
      ctx.stroke();
    }

    particles.forEach(p => {
      p.update();
      p.draw(ctx);
    });

    animationFrameId = requestAnimationFrame(animate);
  };
  animate();

  onUnmounted(() => {
    window.removeEventListener('resize', resize);
    cancelAnimationFrame(animationFrameId);
  });
};

const isFullscreen = ref(false);

const toggleFullscreen = () => {
  if (!document.fullscreenElement) {
    document.documentElement.requestFullscreen().then(() => {
      isFullscreen.value = true;
    }).catch(err => {
      console.error(`Error attempting to enable fullscreen: ${err.message}`);
    });
  } else {
    document.exitFullscreen().then(() => {
      isFullscreen.value = false;
    });
  }
};

const onFullscreenChange = () => {
  isFullscreen.value = !!document.fullscreenElement;
};

onMounted(() => {
  initTargetDate();
  updateCountdown();
  countdownIntervalId = setInterval(updateCountdown, 10); // high precision
  setupCanvas();
  document.addEventListener('fullscreenchange', onFullscreenChange);
});

onUnmounted(() => {
  clearInterval(countdownIntervalId);
  document.removeEventListener('fullscreenchange', onFullscreenChange);
});
</script>

<template>
  <div class="chronos-container">
    <!-- Particle canvas background -->
    <canvas ref="canvasRef" class="bg-particles"></canvas>
    
    <!-- Ambient blurred background elements -->
    <div class="ambient-glow glow-1"></div>
    <div class="ambient-glow glow-2"></div>
    
    <header class="header">
      <div class="logo">
        <span class="logo-dot"></span> CHRONOS
      </div>
      <div class="header-actions">
        <button class="fullscreen-btn" @click="toggleFullscreen" :title="isFullscreen ? 'Exit Fullscreen' : 'Enter Fullscreen'">
          <span v-if="isFullscreen">🗗</span>
          <span v-else>🗖</span>
        </button>
        <div class="badge" :class="{ 'paused': isPaused }">
          {{ isPaused ? 'PAUSED' : 'ACTIVE' }}
        </div>
      </div>
    </header>

    <main class="main-content">
      <div class="title-section">
        <h1>Time Horizon</h1>
        <p class="subtitle">A full-year cosmic countdown sequence</p>
      </div>

      <!-- Glassmorphic Main Card -->
      <div class="countdown-card">
        
        <!-- Interactive Progress bar -->
        <div class="progress-container">
          <div class="progress-bar" :style="{ width: progressPercentage + '%' }"></div>
          <span class="progress-label">{{ progressPercentage.toFixed(4) }}% Elapsed</span>
        </div>

        <div class="grid-countdown">
          <!-- Years -->
          <div class="time-block">
            <div class="digits-wrapper">
              <span class="digits">{{ formatNumber(timeRemaining.years) }}</span>
            </div>
            <span class="label">Years</span>
          </div>

          <!-- Divider -->
          <div class="colon">:</div>

          <!-- Months -->
          <div class="time-block">
            <div class="digits-wrapper">
              <span class="digits">{{ formatNumber(timeRemaining.months) }}</span>
            </div>
            <span class="label">Months</span>
          </div>

          <!-- Divider -->
          <div class="colon">:</div>

          <!-- Days -->
          <div class="time-block">
            <div class="digits-wrapper">
              <span class="digits">{{ formatNumber(timeRemaining.days) }}</span>
            </div>
            <span class="label">Days</span>
          </div>

          <!-- Divider -->
          <div class="colon d-hide-mobile">:</div>

          <!-- Hours -->
          <div class="time-block">
            <div class="digits-wrapper">
              <span class="digits">{{ formatNumber(timeRemaining.hours) }}</span>
            </div>
            <span class="label">Hours</span>
          </div>

          <!-- Divider -->
          <div class="colon">:</div>

          <!-- Minutes -->
          <div class="time-block">
            <div class="digits-wrapper">
              <span class="digits">{{ formatNumber(timeRemaining.minutes) }}</span>
            </div>
            <span class="label">Minutes</span>
          </div>

          <!-- Divider -->
          <div class="colon">:</div>

          <!-- Seconds -->
          <div class="time-block">
            <div class="digits-wrapper">
              <span class="digits">{{ formatNumber(timeRemaining.seconds) }}</span>
              <span class="ms-digits">{{ formatMilliseconds(timeRemaining.milliseconds) }}</span>
            </div>
            <span class="label">Seconds</span>
          </div>
        </div>
      </div>

      <!-- Interactive Controls Drawer -->
      <div class="controls-panel">
        <button class="control-btn play-pause-btn" @click="togglePause">
          <span v-if="isPaused" class="btn-icon">▶</span>
          <span v-else class="btn-icon">⏸</span>
          {{ isPaused ? 'Resume' : 'Pause' }}
        </button>

        <button class="control-btn reset-btn" @click="resetToDefault">
          <span class="btn-icon">🔄</span> Reset 1 Year
        </button>

        <div class="custom-date-picker">
          <label for="target-picker">Custom Target</label>
          <input 
            type="datetime-local" 
            id="target-picker" 
            @change="setCustomDate"
            :min="new Date().toISOString().slice(0, 16)"
          />
        </div>
      </div>
    </main>

    <footer class="footer">
      <p>Precision Digital Chronometer &bull; Built in Vue.js</p>
    </footer>
  </div>
</template>

<style scoped>
.chronos-container {
  min-height: 100vh;
  width: 100vw;
  background-color: #030303;
  color: #ffffff;
  font-family: 'Inter', sans-serif;
  position: relative;
  overflow-x: hidden;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  box-sizing: border-box;
}

/* Background Canvas Particles */
.bg-particles {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 1;
}

/* Glowing background ambient elements */
.ambient-glow {
  position: absolute;
  width: 50vw;
  height: 50vw;
  border-radius: 50%;
  filter: blur(120px);
  opacity: 0.15;
  pointer-events: none;
  z-index: 0;
}

.glow-1 {
  background: radial-gradient(circle, #a855f7 0%, rgba(0,0,0,0) 70%);
  top: -20%;
  left: -10%;
  animation: float 20s ease-in-out infinite alternate;
}

.glow-2 {
  background: radial-gradient(circle, #06b6d4 0%, rgba(0,0,0,0) 70%);
  bottom: -20%;
  right: -10%;
  animation: float 20s ease-in-out infinite alternate-reverse;
}

@keyframes float {
  0% { transform: translateY(0) scale(1); }
  100% { transform: translateY(40px) scale(1.1); }
}

/* Layout Elements */
.header {
  position: relative;
  z-index: 10;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.5rem clamp(1rem, 5vw, 4rem);
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.fullscreen-btn {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: #ffffff;
  padding: 0.4rem 0.6rem;
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  font-size: 1rem;
}

.fullscreen-btn:hover {
  background: rgba(255, 255, 255, 0.12);
  border-color: rgba(255, 255, 255, 0.25);
  transform: scale(1.05);
}

.logo {
  font-family: 'Outfit', sans-serif;
  font-weight: 800;
  font-size: clamp(1rem, 2vw, 1.25rem);
  letter-spacing: 0.25em;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.logo-dot {
  width: 8px;
  height: 8px;
  background-color: #a855f7;
  border-radius: 50%;
  box-shadow: 0 0 10px #a855f7;
  display: inline-block;
}

.badge {
  background: rgba(168, 85, 247, 0.1);
  border: 1px solid rgba(168, 85, 247, 0.3);
  color: #c084fc;
  padding: 0.25rem 0.75rem;
  border-radius: 20px;
  font-size: clamp(0.65rem, 1.5vw, 0.75rem);
  font-weight: 600;
  letter-spacing: 0.1em;
  transition: all 0.3s ease;
}

.badge.paused {
  background: rgba(239, 68, 68, 0.1);
  border: 1px solid rgba(239, 68, 68, 0.3);
  color: #ef4444;
}

.main-content {
  position: relative;
  z-index: 10;
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
  padding: 0 clamp(1rem, 4vw, 2rem);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  flex-grow: 1;
  box-sizing: border-box;
}

.title-section {
  text-align: center;
  margin-bottom: clamp(1.5rem, 5vw, 3rem);
}

.title-section h1 {
  font-family: 'Outfit', sans-serif;
  font-size: clamp(2rem, 6vw, 3.5rem);
  font-weight: 800;
  letter-spacing: -0.02em;
  margin-bottom: 0.5rem;
  background: linear-gradient(135deg, #ffffff 40%, #a855f7 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.subtitle {
  color: #8e8e93;
  font-size: clamp(0.85rem, 2.5vw, 1.1rem);
  font-weight: 300;
  letter-spacing: 0.05em;
}

/* Glassmorphic Card Styles */
.countdown-card {
  background: rgba(15, 15, 20, 0.5);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 24px;
  padding: clamp(1rem, 4vw, 3rem);
  width: 100%;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.4);
  position: relative;
  overflow: hidden;
  margin-bottom: 2.5rem;
  box-sizing: border-box;
}

/* Progress bar inside card */
.progress-container {
  width: 100%;
  background: rgba(255, 255, 255, 0.03);
  height: clamp(20px, 3vw, 28px);
  border-radius: 14px;
  position: relative;
  overflow: hidden;
  margin-bottom: clamp(1.5rem, 4vw, 3rem);
  border: 1px solid rgba(255, 255, 255, 0.05);
}

.progress-bar {
  height: 100%;
  background: linear-gradient(90deg, #a855f7 0%, #06b6d4 100%);
  width: 0%;
  transition: width 0.1s linear;
  box-shadow: 0 0 15px rgba(168, 85, 247, 0.5);
}

.progress-label {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  font-size: clamp(0.6rem, 1.5vw, 0.75rem);
  font-weight: 600;
  letter-spacing: 0.05em;
  color: #ffffff;
  mix-blend-mode: difference;
}

/* Grid display */
.grid-countdown {
  display: grid;
  grid-template-columns: 1fr auto 1fr auto 1fr auto 1fr auto 1fr auto 1.3fr;
  align-items: center;
  justify-content: center;
  gap: clamp(0.25rem, 1vw, 0.75rem);
}

.time-block {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

.digits-wrapper {
  background: rgba(255, 255, 255, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.04);
  border-radius: 16px;
  padding: clamp(0.75rem, 2vw, 1.5rem) clamp(0.25rem, 1vw, 1rem);
  width: 100%;
  display: flex;
  align-items: baseline;
  justify-content: center;
  box-shadow: inset 0 2px 4px rgba(255, 255, 255, 0.05);
  box-sizing: border-box;
}

.digits {
  font-family: 'JetBrains Mono', monospace;
  font-size: clamp(1.8rem, 5.5vw, 4rem);
  font-weight: 700;
  color: #ffffff;
  text-shadow: 0 0 20px rgba(255, 255, 255, 0.1);
  letter-spacing: -0.05em;
}

.ms-digits {
  font-family: 'JetBrains Mono', monospace;
  font-size: clamp(0.75rem, 2vw, 1.5rem);
  color: #a855f7;
  font-weight: 400;
  margin-left: 0.15rem;
  opacity: 0.8;
  display: inline-block;
  width: clamp(2rem, 4.5vw, 3.5rem);
}

.colon {
  font-family: 'JetBrains Mono', monospace;
  font-size: clamp(1.5rem, 4vw, 3rem);
  color: rgba(255, 255, 255, 0.2);
  animation: pulse 1s infinite;
  padding-bottom: clamp(0.75rem, 2vw, 1.5rem);
}

@keyframes pulse {
  0%, 100% { opacity: 0.2; }
  50% { opacity: 0.8; }
}

.label {
  font-size: clamp(0.55rem, 1.5vw, 0.75rem);
  text-transform: uppercase;
  letter-spacing: 0.2em;
  color: #8e8e93;
  margin-top: clamp(0.5rem, 1.5vw, 1rem);
  font-weight: 500;
  text-align: center;
}

/* Controls panel drawer */
.controls-panel {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  background: rgba(255, 255, 255, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.05);
  padding: 1.25rem clamp(1rem, 3vw, 2.5rem);
  border-radius: 50px;
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  width: auto;
  box-sizing: border-box;
}

.control-btn {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: #ffffff;
  padding: 0.65rem 1.25rem;
  border-radius: 30px;
  font-size: clamp(0.75rem, 1.8vw, 0.875rem);
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.2s ease;
}

.control-btn:hover {
  background: rgba(255, 255, 255, 0.1);
  transform: translateY(-2px);
}

.play-pause-btn {
  background: linear-gradient(135deg, #a855f7 0%, #7c3aed 100%);
  border: none;
  box-shadow: 0 4px 15px rgba(168, 85, 247, 0.3);
}

.play-pause-btn:hover {
  background: linear-gradient(135deg, #b55fe6 0%, #8b5cf6 100%);
  box-shadow: 0 6px 20px rgba(168, 85, 247, 0.5);
}

.custom-date-picker {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: clamp(0.75rem, 1.8vw, 0.875rem);
}

.custom-date-picker label {
  color: #8e8e93;
  font-weight: 500;
  white-space: nowrap;
}

.custom-date-picker input {
  background: rgba(0, 0, 0, 0.4);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: #ffffff;
  padding: 0.4rem 0.8rem;
  border-radius: 20px;
  outline: none;
  cursor: pointer;
  font-family: inherit;
  font-size: clamp(0.75rem, 1.8vw, 0.875rem);
  transition: border-color 0.2s;
}

.custom-date-picker input:focus {
  border-color: #a855f7;
}

/* Footer */
.footer {
  position: relative;
  z-index: 10;
  text-align: center;
  padding: 1.5rem;
  color: #48484a;
  font-size: clamp(0.65rem, 1.5vw, 0.75rem);
  letter-spacing: 0.05em;
}

/* Responsive constraints */
@media (max-width: 900px) {
  .grid-countdown {
    grid-template-columns: repeat(3, 1fr);
    gap: 1.25rem 1rem;
  }
  
  .colon {
    display: none;
  }
  
  .countdown-card {
    border-radius: 20px;
  }
  
  .controls-panel {
    border-radius: 24px;
    padding: 1.25rem;
    width: 100%;
    justify-content: space-around;
  }
}

@media (max-width: 600px) {
  .grid-countdown {
    grid-template-columns: repeat(2, 1fr);
    gap: 1rem 0.75rem;
  }
  
  .header {
    padding: 1rem 1.5rem;
  }

  .controls-panel {
    flex-direction: column;
    gap: 1rem;
    align-items: stretch;
  }

  .control-btn, .custom-date-picker {
    justify-content: center;
    width: 100%;
  }

  .custom-date-picker {
    flex-direction: column;
    align-items: stretch;
    gap: 0.25rem;
  }

  .custom-date-picker input {
    width: 100%;
    box-sizing: border-box;
    text-align: center;
  }
}
</style>

