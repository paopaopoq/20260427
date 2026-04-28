<script setup>
import { ref, reactive, computed, onUnmounted } from 'vue';

// 導入音效檔案，確保 Vite 在打包時能正確處理資源路徑
import alarmMp3 from './501880__greenworm__cellphone-alarm-clock.mp3';
import tickMp3 from './704134__entershift__beep-sound.mp3';

const customMin = ref(5);
const customSec = ref(0);

const timer = reactive({ totalSeconds: 300, remainingSeconds: 300, isRunning: false, interval: null });

// 音效初始化
const alarmSound = new Audio(alarmMp3);
const tickSound = new Audio(tickMp3); // 倒數每秒音效
tickSound.volume = 0.5;

const stopTickSound = () => {
  tickSound.pause();
  tickSound.currentTime = 0;
};

const formattedTime = computed(() => {
  const m = Math.floor(timer.remainingSeconds / 60);
  const s = timer.remainingSeconds % 60;
  return `${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
});
const setTimer = (sec) => { timer.totalSeconds = sec; timer.remainingSeconds = sec; };
const toggleTimer = () => {
  if (timer.isRunning) { 
    clearInterval(timer.interval); 
    timer.isRunning = false; 
    stopTickSound(); // 暫停時停止音效
  }
  else {
    timer.isRunning = true;
    
    // 在開始計時的瞬間立即檢查，確保 10 秒整時立即響一聲
    if (timer.remainingSeconds <= 10 && timer.remainingSeconds > 0) {
      tickSound.currentTime = 0;
      tickSound.play().catch(() => {});
    }

    timer.interval = setInterval(() => {
      if (timer.remainingSeconds > 0) {
        timer.remainingSeconds--;

        if (timer.remainingSeconds === 0) {
          // 剛好歸零，立即執行結束邏輯
          clearInterval(timer.interval);
          timer.isRunning = false;
          stopTickSound();
          alarmSound.currentTime = 0;
          alarmSound.play().catch(() => {});
        } else if (timer.remainingSeconds <= 10) {
          // 還沒歸零但在 10 秒內，播放倒數滴答聲
          tickSound.currentTime = 0;
          tickSound.play().catch(() => {});
        }
      }
    }, 1000);
  }
};

const applyCustomTimer = () => {
  const total = (customMin.value * 60) + (customSec.value || 0);
  if (total > 0) {
    stopTickSound();
    setTimer(total);
  }
};

onUnmounted(() => {
  clearInterval(timer.interval);
  stopTickSound();
});
</script>
<template>
  <div class="cyber-window">
    <div class="window-header"><span>❅.⋆𐙚 TIMER</span></div>
    <div class="window-content">
      <div class="tool-panel">
        <div class="timer-display" :class="{ 
          'warning-flash': timer.remainingSeconds <= 10 && timer.remainingSeconds > 0 && timer.isRunning,
          'timer-finished': timer.remainingSeconds === 0
        }">
          {{ formattedTime }}
        </div>
        <div class="custom-setting">
          <input type="number" v-model.number="customMin" min="0" class="time-input"> 分
          <input type="number" v-model.number="customSec" min="0" max="59" class="time-input"> 秒
          <button @click="applyCustomTimer" class="pixel-btn sm primary">套用</button>
        </div>
        <div class="preset-group">
          <button v-for="t in [180, 300, 600]" :key="t" @click="setTimer(t)" class="pixel-btn sm">{{ Math.floor(t/60) }}m</button>
        </div>
        <div class="timer-actions">
          <button @click="toggleTimer" class="pixel-btn primary">{{ timer.isRunning ? '⏸ 暫停' : '▶ 開始' }}</button>
          <button @click="timer.remainingSeconds = timer.totalSeconds; stopTickSound();" class="pixel-btn">⏹ 重置</button>
        </div>
      </div>
    </div>
  </div>
</template>
<style scoped>
.tool-panel { background: var(--divine-white); border-radius: 15px; padding: 25px; text-align: center; }
.timer-display { font-size: 4.5rem; font-family: 'DotGothic16', sans-serif; color: var(--fairy-blue-dark); margin-bottom: 20px; }
@media (max-width: 480px) { .timer-display { font-size: 3.2rem; } }
.warning-flash { animation: flash-intense 0.5s infinite alternate; color: #ff922b; }
@keyframes flash-intense { from { opacity: 1; transform: scale(1); } to { opacity: 0.4; transform: scale(1.05); } }
.timer-finished { animation: finish-pulse 0.4s infinite alternate; color: #fa5252; text-shadow: 0 0 15px rgba(250, 82, 82, 0.4); }
@keyframes finish-pulse { from { transform: scale(1); } to { transform: scale(1.2); } }

.preset-group { display: flex; justify-content: center; gap: 10px; margin-bottom: 20px; }
.custom-setting { display: flex; justify-content: center; align-items: center; gap: 8px; margin-bottom: 15px; font-size: 14px; }
.time-input { width: 60px; height: 40px; border: 1.5px solid var(--fairy-blue-light); border-radius: 8px; padding: 4px; text-align: center; font-family: 'DotGothic16', sans-serif; outline: none; font-size: 18px; }
.time-input:focus { border-color: var(--fairy-blue); }
.timer-actions { display: flex; justify-content: center; gap: 15px; }
.pixel-btn { background: white; border: 2px solid var(--fairy-blue-dark); padding: 8px 20px; border-radius: 20px; cursor: pointer; }
.pixel-btn.primary { background: var(--fairy-blue-dark); color: white; }
.pixel-btn.sm { padding: 4px 12px; font-size: 12px; }
</style>