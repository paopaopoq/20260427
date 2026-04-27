<script setup>
import { ref, computed, onMounted, watch } from 'vue';

const teams = ref([
  { id: 1, name: 'Cyber_Alpha', score: 0 },
  { id: 2, name: 'Snow_Beta', score: 0 },
  { id: 3, name: 'Fairy_Gamma', score: 0 }
]);
const activityName = ref('競賽記分板'); // 新增：活動名稱
const isEditingActivityName = ref(false); // 新增：控制活動名稱編輯狀態

const sortedTeams = computed(() => [...teams.value].sort((a, b) => b.score - a.score));

const isResetting = ref(false);

const saveTeamsToLocal = () => {
  localStorage.setItem('cyber_data_teams', JSON.stringify(teams.value));
};

const saveActivityNameToLocal = () => {
  localStorage.setItem('cyber_data_activity_name', activityName.value);
};

// 監聽 teams 變化，自動儲存
watch(teams, saveTeamsToLocal, { deep: true });

const addScore = (team, val) => { team.score += val; saveToLocal(); };
const addNewTeam = () => { teams.value.push({ id: Date.now(), name: '新組別', score: 0 }); }; // watch 會自動儲存
const removeTeam = (id) => { teams.value = teams.value.filter(t => t.id !== id); }; // watch 會自動儲存
const resetScores = () => { 
  teams.value.forEach(t => t.score = 0); 
  isResetting.value = true;
  setTimeout(() => { isResetting.value = false; }, 500);
};

onMounted(() => {
  // 載入隊伍資料
  const savedTeams = localStorage.getItem('cyber_data_teams');
  if (savedTeams) teams.value = JSON.parse(savedTeams);

  // 載入活動名稱
  const savedActivityName = localStorage.getItem('cyber_data_activity_name');
  if (savedActivityName) activityName.value = savedActivityName;
});
</script>

<template>
  <div class="cyber-window">
    <div class="window-header">
      <span v-if="!isEditingActivityName" @click="isEditingActivityName = true" class="activity-title-display">
        ❅.⋆𐙚 {{ activityName }}
      </span>
      <input v-else
             v-model="activityName"
             @blur="isEditingActivityName = false; saveActivityNameToLocal()"
             @keyup.enter="isEditingActivityName = false; saveActivityNameToLocal()"
             class="activity-name-input"
             maxlength="25">
      <button @click="resetScores" class="pixel-btn sm danger" style="padding: 2px 10px;">全部歸零</button>
    </div>
    <div class="window-content" :class="{ 'reset-flash': isResetting }">
      <transition-group name="flip-list">
        <div v-for="(team, index) in sortedTeams" 
             :key="team.id" 
             class="score-row"
             :class="{ 'leader-row': index === 0 && team.score > 0 }">
          <div class="team-meta">
            <div class="rank-badge">{{ (index === 0 && team.score > 0) ? '👑' : index + 1 }}</div>
            <input v-model="team.name" class="team-input">
          </div>
          <div class="score-controls">
            <input type="number" v-model.number="team.score" class="score-val-input">
            <button @click="addScore(team, 1)" class="pixel-btn sm">+1</button>
            <button @click="addScore(team, 5)" class="pixel-btn sm">+5</button>
            <button @click="addScore(team, -1)" class="pixel-btn sm minus">-1</button>
            <button @click="removeTeam(team.id)" class="pixel-btn sm danger" title="刪除組別">✕</button>
          </div>
        </div>
      </transition-group>
      <button @click="addNewTeam" class="pixel-btn w-full mt-15">⋆⁺₊ 新增組別</button>
    </div>
  </div>
</template>

<style scoped>
.window-content { padding: 30px; transition: background-color 0.3s ease; }

/* 歸零時的閃爍反應 */
.reset-flash {
  animation: reset-blink 0.5s ease;
}
@keyframes reset-blink {
  0% { background-color: rgba(162, 207, 254, 0.4); }
  100% { background-color: transparent; }
}

.score-row { display: flex; justify-content: space-between; align-items: center; padding: 15px 20px; background: white; margin-bottom: 10px; border-radius: 12px; border: 1px solid var(--fairy-blue-light); transition: all 0.5s; }
@media (max-width: 600px) { 
  .score-row { padding: 15px 12px; flex-direction: column; gap: 12px; align-items: flex-start; } 
  .score-controls { width: 100%; justify-content: space-between; }
}
.team-meta { display: flex; align-items: center; gap: 15px; }
@media (max-width: 480px) { 
  .team-meta { gap: 8px; width: 100%; } 
  .team-input { flex: 1; width: auto; font-size: 16px; }
}
.rank-badge { background: var(--fairy-blue-light); color: white; width: 28px; height: 28px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: bold; }

/* 第一名特殊樣式與動畫 */
.leader-row { 
  border: 4px solid #ffacc5;
  background: #fff0f5 !important;
  animation: leader-pulse 2s infinite ease-in-out, chrome-shine 3s infinite linear;
  z-index: 2;
}
@keyframes leader-pulse {
  0%, 100% { transform: scale(1); box-shadow: 0 5px 15px rgba(162, 207, 254, 0.2); }
  50% { transform: scale(1.02); box-shadow: 0 10px 25px rgba(162, 207, 254, 0.4); border-color: var(--fairy-blue-dark); }
}
@keyframes chrome-shine {
  0% { filter: brightness(1); }
  50% { filter: brightness(1.1) contrast(1.1); }
  100% { filter: brightness(1); }
}

.team-input { border: 1px solid transparent; background: transparent; font-family: inherit; font-size: 14px; color: var(--pixel-blue); font-weight: bold; width: 120px; outline: none; padding: 2px 5px; border-radius: 4px; transition: 0.2s; }
.team-input:hover, .team-input:focus { border-color: var(--fairy-blue-light); background: #fdfdff; }
.score-val-input { width: 60px; border: 1.5px solid var(--fairy-blue-light); border-radius: 8px; background: #f8fbff; font-family: 'Space Mono', monospace; font-size: 18px; font-weight: bold; text-align: center; color: var(--fairy-blue-dark); outline: none; padding: 2px; transition: 0.2s; }
.score-val-input:focus { border-color: var(--fairy-blue); background: white; }
.score-controls { display: flex; align-items: center; gap: 10px; }
.score-val { font-size: 24px; font-weight: bold; width: 60px; text-align: center; color: var(--fairy-blue-dark); }
.pixel-btn { background: white; border: 2px solid var(--fairy-blue-dark); padding: 5px 12px; color: var(--pixel-blue); border-radius: 20px; cursor: pointer; }
.pixel-btn.sm { font-size: 11px; }
.minus { opacity: 0.6; }
.pixel-btn.danger { border-color: #ffaaa5; color: #ffaaa5; }
.w-full { width: 100%; justify-content: center; display: flex; margin-top: 15px; }

/* 排序動畫核心 */
.flip-list-move { transition: transform 0.5s; }
.flip-list-enter-active, .flip-list-leave-active { transition: all 0.5s; }
.flip-list-enter-from, .flip-list-leave-to { opacity: 0; transform: translateX(30px); }
</style>