<script setup>
import { ref, reactive, onUnmounted } from 'vue';

const lottery = reactive({
  rawInput: "學生 A\n學生 B\n學生 C\n第 1 組\n第 2 組\n第 3 組",
  pool: [],
  history: [],
  winners: [],
  display: "⋆⁺₊❅.⋆𐙚",
  isRolling: false,
  isWinner: false,
  showEdit: false,
  excludeWinners: true,
  drawCount: 1
});

let rollInterval = null;

// 音效初始化
const rollSound = new Audio('https://assets.mixkit.co/active_storage/sfx/2571/2571-preview.mp3');
const winSound = new Audio('https://assets.mixkit.co/active_storage/sfx/2019/2019-preview.mp3');
rollSound.volume = 0.3;

const resetLottery = () => {
  lottery.pool = lottery.rawInput.split('\n').map(s => s.trim()).filter(n => n);
  lottery.history = [];
  lottery.winners = [];
  lottery.display = "名單已重載 (" + lottery.pool.length + ")";
  lottery.isWinner = false;
  lottery.isRolling = false;
  if (rollInterval) clearInterval(rollInterval);
};

const startLottery = () => {
  const countToDraw = Math.min(lottery.drawCount, lottery.pool.length);
  if (countToDraw <= 0) return;

  lottery.isRolling = true;
  lottery.isWinner = false;
  lottery.winners = [];
  let count = 0;
  const totalRolls = 30;
  rollInterval = setInterval(() => {
    // 播放滾動音效（重置進度以實現快速連續播放）
    rollSound.currentTime = 0;
    rollSound.play().catch(() => {});
    lottery.display = lottery.pool[Math.floor(Math.random() * lottery.pool.length)];
    if (++count > totalRolls) {
      clearInterval(rollInterval);
      
      // 隨機選出指定人數
      const shuffled = [...lottery.pool].sort(() => 0.5 - Math.random());
      const selected = shuffled.slice(0, countToDraw);
      
      lottery.winners = selected;
      if (countToDraw === 1) lottery.display = selected[0];
      else lottery.display = `已抽出 ${countToDraw} 位`;

      if (lottery.excludeWinners) {
        lottery.history.push(...selected);
        lottery.pool = lottery.pool.filter(n => !selected.includes(n));
      }
      lottery.isRolling = false;
      lottery.isWinner = true;
      // 播放中獎音效
      winSound.play().catch(() => {});
    }
  }, 60);
};

onUnmounted(() => { if (rollInterval) clearInterval(rollInterval); });

resetLottery();
</script>

<template>
  <div class="cyber-window">
    <div class="window-header">
      <span>❅.⋆𐙚 RANDOM_SELECTOR.EXE</span>
      <div class="window-controls"><div class="win-dot"></div><div class="win-dot"></div></div>
    </div>
    <div class="window-content">
      <div class="tool-panel">
        <div class="lottery-display" :class="{ 
          rolling: lottery.isRolling, 
          winner: lottery.isWinner,
          'winner-flash': lottery.isWinner 
        }">
          <template v-if="lottery.isWinner && lottery.winners.length > 1">
            <div class="winners-grid">
              <span v-for="name in lottery.winners" :key="name" class="winner-item">{{ name }}</span>
            </div>
          </template>
          <template v-else>
            {{ lottery.display }}
          </template>
          
          <!-- 抽中瞬間的噴發星芒特效 -->
          <div v-if="lottery.isWinner" class="celebration-overlay">
            <div v-for="i in 12" :key="i" class="burst-sparkle" :style="`--i: ${i}`">⋆</div>
          </div>
        </div>
        <div class="tool-status">
          <span v-if="lottery.isRolling">⋆ 正在檢索數據流... ⋆</span>
          <span v-else-if="lottery.isWinner">⋆ CONGRATULATIONS! ⋆</span>
          <span v-else>⋆ 等待指令中 ⋆</span>
        </div>
      </div>
      
      <div class="action-flex">
        <div class="toggle-container">
          <span>人數：</span>
          <input type="number" v-model.number="lottery.drawCount" min="1" :max="lottery.pool.length || 1" class="draw-count-input">
          <span>排除重複中獎</span>
          <div class="toggle-switch" :class="{ on: lottery.excludeWinners }" @click="lottery.excludeWinners = !lottery.excludeWinners"></div>
          <span class="pool-text">(剩餘: {{ lottery.pool.length }} 人)</span>
        </div>

        <div class="btn-group">
          <button @click="startLottery" class="pixel-btn primary" :disabled="lottery.isRolling || lottery.pool.length === 0">
            <span>⋆𐙚</span> 開始抽籤
          </button>
          <button @click="resetLottery" class="pixel-btn" :disabled="lottery.isRolling"><span>❅</span> 重置名單</button>
          <button @click="lottery.showEdit = !lottery.showEdit" class="pixel-btn" :class="{ 'edit-mode-on': lottery.showEdit }">
            <span>{{ lottery.showEdit ? '✕' : '✎' }}</span> 
            {{ lottery.showEdit ? '取消匯入' : '匯入名單' }}
          </button>
        </div>
      </div>

      <transition name="fade">
        <div v-if="lottery.showEdit" class="edit-area">
          <textarea v-model="lottery.rawInput" class="raw-textarea"></textarea>
          <button @click="lottery.showEdit = false; resetLottery();" class="pixel-btn sm w-full">確認更新</button>
        </div>
      </transition>
    </div>
  </div>
</template>

<style scoped>
.cyber-window { background: var(--glass-white); backdrop-filter: blur(15px); border: 2px solid white; border-radius: 20px; box-shadow: 0 15px 45px rgba(162, 207, 254, 0.2); overflow: hidden; }
.window-header { background: linear-gradient(90deg, var(--fairy-blue-dark), var(--fairy-blue)); padding: 12px 20px; color: white; font-size: 11px; display: flex; justify-content: space-between; align-items: center; }
.win-dot { width: 8px; height: 8px; border-radius: 50%; background: white; opacity: 0.6; }
.window-content { padding: 30px; }
.tool-panel { background: var(--divine-white); border-radius: 15px; padding: 25px; margin-bottom: 20px; }
.lottery-display { font-size: 3rem; font-weight: bold; text-align: center; color: var(--fairy-blue-dark); min-height: 120px; display: flex; align-items: center; justify-content: center; transition: all 0.1s; }
@media (max-width: 480px) { .lottery-display { font-size: 2.2rem; min-height: 90px; } }
.lottery-display.rolling { transform: scale(1.1); filter: blur(1px); }
.lottery-display.winner { animation: winPulse 0.5s ease infinite alternate; }
@keyframes winPulse { 
  from { transform: scale(1); text-shadow: 0 0 10px var(--accent-glow); } 
  to { transform: scale(1.1); text-shadow: 0 0 30px var(--fairy-blue-dark); } 
}

/* 抽中瞬間的閃光特效 */
.winner-flash::after {
  content: '';
  position: absolute;
  inset: 0;
  background: white;
  opacity: 0;
  z-index: 10;
  pointer-events: none;
  animation: flash-anim 0.6s ease-out;
}
@keyframes flash-anim {
  0% { opacity: 0.7; }
  100% { opacity: 0; }
}

.tool-status { text-align: center; font-size: 11px; color: var(--fairy-blue-dark); }

.winners-grid { display: flex; flex-wrap: wrap; gap: 10px; justify-content: center; width: 100%; }
.winner-item { background: var(--fairy-blue-light); padding: 5px 15px; border-radius: 12px; font-size: 1.5rem; border: 1px solid white; animation: popIn 0.3s ease; color: var(--fairy-blue-dark); }
@keyframes popIn { from { transform: scale(0.8); opacity: 0; } to { transform: scale(1); opacity: 1; } }

/* 星芒噴發特效 */
.celebration-overlay {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: none;
  z-index: 5;
}
.burst-sparkle {
  position: absolute;
  color: var(--fairy-blue);
  font-size: 24px;
  animation: burst-anim 0.8s ease-out forwards;
  opacity: 0;
}
@keyframes burst-anim {
  0% { transform: rotate(calc(var(--i) * 30deg)) translateY(0) scale(0); opacity: 1; }
  100% { transform: rotate(calc(var(--i) * 30deg)) translateY(-120px) scale(1.5); opacity: 0; }
}

.action-flex { display: flex; flex-direction: column; gap: 15px; align-items: center; }
.toggle-container { display: flex; align-items: center; gap: 10px; font-size: 12px; }
@media (max-width: 480px) { .toggle-container { flex-direction: column; gap: 8px; text-align: center; } }
.toggle-switch { width: 40px; height: 20px; background: #ccc; border-radius: 10px; position: relative; cursor: pointer; }
.toggle-switch.on { background: var(--fairy-blue-dark); }
.toggle-switch::after { content: ""; position: absolute; top: 2px; left: 2px; width: 16px; height: 16px; background: white; border-radius: 50%; transition: 0.3s; }
.toggle-switch.on::after { left: 22px; }
.draw-count-input { width: 50px; padding: 2px 5px; border-radius: 5px; border: 1px solid var(--fairy-blue-light); font-family: inherit; color: var(--pixel-blue); outline: none; }
.btn-group { display: flex; gap: 12px; flex-wrap: wrap; justify-content: center; }
.pixel-btn { background: white; border: 2px solid var(--fairy-blue-dark); padding: 10px 24px; font-family: inherit; font-size: 13px; color: var(--pixel-blue); border-radius: 25px; box-shadow: 0 4px 0 var(--fairy-blue-light); cursor: pointer; }
.pixel-btn.primary { background: var(--fairy-blue-dark); color: white; border-color: white; }
.edit-area { margin-top: 20px; width: 100%; }
.raw-textarea { width: 100%; height: 120px; padding: 10px; border-radius: 15px; border: 1px solid var(--fairy-blue-light); resize: none; font-family: 'Space Mono'; }
.w-full { width: 100%; justify-content: center; }
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>