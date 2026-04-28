<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import RandomDraw from './RandomDraw.vue';
import Scoreboard from './Scoreboard.vue';
import ClassTimer from './components/ClassTimer.vue';

// --- 基礎變數與分頁 ---
const tabs = [
    { id: 'exam', icon: '✎', name: '數位測驗' },
    { id: 'qa', icon: '★', name: '即時互動' },
    { id: 'lottery', icon: '⋆𐙚', name: '隨機抽籤' },
    { id: 'score', icon: '⋆⁺₊', name: '競賽計分' },
    { id: 'timer', icon: '❅', name: '報告計時' }
];
const currentTab = ref('exam');
const examMode = ref('take'); 
const examRecords = ref([]);
const timeString = ref('');
const bgDecorations = ref([]);

// --- 數位測驗資料 ---
const questions = ref([
    {
        id: 101, type: '是非', text: '本測驗系統支援即時閱卷與自動評分功能。',
        options: [{val:'T', text:'正確 (T)'}, {val:'F', text:'錯誤 (F)'}],
        answer: 'T', userAns: null, explanation: '是的，提交後系統會立即計算分數並顯示解析。'
    },
    {
        id: 102, type: '單選', text: '下列哪一種顏色是本系統的主視覺色調？',
        options: [{val: 'A', text: '科技綠'}, {val: 'B', text: '仙系藍 (Fairy Blue)'}, {val: 'C', text: '熱血紅'}, {val: 'D', text: '曜石黑'}],
        answer: 'B', userAns: null, explanation: '本系統採用 "Cyber Fairy" 風格，主要以仙系藍為主色。'
    },
    {
        id: 103, type: '複選', text: '「數位測驗」功能模組目前支援哪些題型？ (複選)',
        options: [
            {val: 'A', text: '是非題'}, 
            {val: 'B', text: '單選題'}, 
            {val: 'C', text: '複選題'}, 
            {val: 'D', text: '問答題'}
        ],
        answer: ['A', 'B', 'C'], userAns: [], explanation: '目前系統支援是非、單選與複選，問答題尚在開發中。'
    }
]);

const addNewQuestion = () => {
    const id = Date.now();
    questions.value.push({
        id, type: '是非', text: '',
        options: [{val:'T', text:'正確 (T)'}, {val:'F', text:'錯誤 (F)'}],
        answer: 'T', userAns: null, explanation: ''
    });
};
const removeQuestion = (idx) => questions.value.splice(idx, 1);
const syncAnswerFormat = (q) => {
    if (q.type === '複選') {
        q.answer = [];
        q.userAns = [];
        q.options = [{val: 'A', text: ''}, {val: 'B', text: ''}];
    } else if (q.type === '是非') {
        q.answer = 'T';
        q.userAns = null;
        q.options = [{val:'T', text:'正確 (T)'}, {val:'F', text:'錯誤 (F)'}];
    } else {
        q.answer = 'A';
        q.userAns = null;
        q.options = [{val: 'A', text: ''}, {val: 'B', text: ''}];
    }
};

const examSubmitted = ref(false);
const score = ref(0);
const correctCount = ref(0);

const isSelected = (q, val) => q.type === '複選' ? q.userAns.includes(val) : q.userAns === val;
const toggleOption = (q, val) => {
    if (examSubmitted.value) return;
    if (q.type === '複選') {
        const idx = q.userAns.indexOf(val);
        if (idx > -1) q.userAns.splice(idx, 1);
        else q.userAns.push(val);
    } else q.userAns = val;
};
const submitExam = () => {
    let correct = 0;
    questions.value.forEach(q => {
        let isQCorrect = false;
        if (q.type === '複選') {
            isQCorrect = Array.isArray(q.userAns) && q.userAns.length === q.answer.length && q.userAns.every(v => q.answer.includes(v));
        } else isQCorrect = q.userAns === q.answer;
        if (isQCorrect) correct++;
    });
    correctCount.value = correct;
    score.value = questions.value.length ? Math.round((correct / questions.value.length) * 100) : 0;

    // 儲存紀錄至 localStorage
    const newRecord = {
        id: Date.now(),
        time: new Date().toLocaleString(),
        score: score.value,
        correctCount: correctCount.value,
        total: questions.value.length
    };
    examRecords.value.unshift(newRecord);
    localStorage.setItem('exam_records', JSON.stringify(examRecords.value));

    examSubmitted.value = true;
};
const resetExam = () => {
    questions.value.forEach(q => q.userAns = q.type === '複選' ? [] : null);
    examSubmitted.value = false;
};
const clearRecords = () => {
    if (confirm('確定要清空所有紀錄嗎？')) {
        examRecords.value = [];
        localStorage.setItem('exam_records', JSON.stringify([]));
    }
};
const isCorrectAnswer = (q, val) => Array.isArray(q.answer) ? q.answer.includes(val) : q.answer === val;
const getFeedbackClass = (q, val) => {
    if (!examSubmitted.value) return '';
    const isCorrect = isCorrectAnswer(q, val);
    const isUserSelected = Array.isArray(q.userAns) ? q.userAns.includes(val) : q.userAns === val;
    if (isCorrect) return 'correct';
    if (isUserSelected && !isCorrect) return 'wrong';
    return '';
};

// --- 即時互動投票邏輯 ---
const pollMode = ref('vote'); // 'vote' or 'edit'
const pollTitle = ref('你最喜歡本系統的哪個功能？');
const pollOptions = ref([
    { id: 1, text: '數位測驗系統', color: '#a2cffe' },
    { id: 2, text: '隨機抽籤機', color: '#74a9d8' },
    { id: 3, text: '小組競賽計分', color: '#ffaaa5' },
    { id: 4, text: '報告計時器', color: '#a8e6cf' }
]);
const pollResults = ref({ 1: 0, 2: 0, 3: 0, 4: 0 });

const vote = (id) => pollResults.value[id]++;
const resetPoll = () => Object.keys(pollResults.value).forEach(id => pollResults.value[id] = 0);

const addPollOption = () => {
    const nextId = Date.now();
    const colors = ['#a2cffe', '#74a9d8', '#ffaaa5', '#a8e6cf', '#ffd8a8'];
    pollOptions.value.push({ id: nextId, text: '新選項', color: colors[pollOptions.value.length % colors.length] });
    pollResults.value[nextId] = 0;
};

const removePollOption = (index, id) => {
    pollOptions.value.splice(index, 1);
    delete pollResults.value[id];
};

// 計算總票數
const totalVotes = computed(() => Object.values(pollResults.value).reduce((a, b) => a + b, 0));

// 計算百分比工具函式
const getPercentage = (id) => {
    if (totalVotes.value === 0) return '0.0';
    return ((pollResults.value[id] / totalVotes.value) * 100).toFixed(1);
};

// 計算圓餅圖漸層數值
const pieGradient = computed(() => {
    if (totalVotes.value === 0) return 'var(--fairy-blue-light)';
    let currentPercent = 0;
    const stops = pollOptions.value.map(opt => {
        const start = currentPercent;
        const end = start + (pollResults.value[opt.id] / totalVotes.value * 100);
        currentPercent = end;
        return `${opt.color} ${start}% ${end}%`;
    });
    return `conic-gradient(${stops.join(', ')})`;
});

let clockInterval;
onMounted(() => {
    // 初始化背景裝飾插圖
    bgDecorations.value = [
        { icon: '✧˚ ༘ ⋆｡˚', left: '8%', top: '15%', size: '45px', duration: '15s', delay: '0s' },
        { icon: '✮⋆˙', left: '85%', top: '10%', size: '55px', duration: '18s', delay: '-5s' },
        { icon: '✩ ₊˚', left: '15%', top: '75%', size: '28px', duration: '12s', delay: '-2s' },
        { icon: '.𐙚 ̊', left: '90%', top: '65%', size: '35px', duration: '10s', delay: '-7s' },
        { icon: '⊹♡', left: '12%', top: '55%', size: '40px', duration: '20s', delay: '-3s' },
        { icon: '⋆｡‧˚ʚɞ˚‧｡⋆', left: '82%', top: '80%', size: '42px', duration: '14s', delay: '-1s' },
        { icon: '✩', left: '5%', top: '40%', size: '30px', duration: '16s', delay: '-4s' },
        { icon: '₊˚♡', left: '75%', top: '45%', size: '32px', duration: '19s', delay: '-8s' }
    ];

    // 初始化讀取紀錄
    const savedRecords = localStorage.getItem('exam_records');
    if (savedRecords) examRecords.value = JSON.parse(savedRecords);

    clockInterval = setInterval(() => { timeString.value = new Date().toLocaleTimeString(); }, 1000);
});
onUnmounted(() => {
    clearInterval(clockInterval);
});
</script>

<template>
    <div class="lace-trim lace-top"></div>
    <div class="lace-trim lace-bottom"></div>

    <!-- 背景裝飾層 -->
    <div class="bg-illustration-layer">
        <div v-for="(item, idx) in bgDecorations" :key="idx" 
             class="bg-illustration-item" 
             :style="{ left: item.left, top: item.top, fontSize: item.size, animationDuration: item.duration, animationDelay: item.delay }">
            {{ item.icon }}
        </div>
    </div>

    <div class="app-root">
        <header class="main-header">
            <p style="font-size: 10px; letter-spacing: 5px; opacity: 0.5;">CLASS_SYSTEM_HUD</p>
            <h1>Class System</h1>
            <p style="font-size: 12px; opacity: 0.8;">⋆⁺₊❅. 𝗹𝗲𝗮𝗿𝗻𝗶𝗻𝗴_𝘀𝘆𝘀𝘁𝗲𝗺_𝘃𝟭𝟬 .❅₊⁺⋆</p>
        </header>

        <nav class="nav-tabs">
            <div v-for="tab in tabs" :key="tab.id" class="tab-item" :class="{ active: currentTab === tab.id }" @click="currentTab = tab.id">
                {{ tab.icon }} {{ tab.name }}
            </div>
        </nav>

        <main class="cyber-window">
            <!-- 視窗右上角裝飾 GIF -->
            <img src="./bCwDZd.gif" class="window-mascot" alt="mascot">

            <div class="window-header">
                <span v-if="currentTab !== 'qa'">{{ currentTab.toUpperCase() }}.EXE</span>
                <span v-else>{{ pollMode === 'vote' ? 'POLL_VIEW.EXE' : 'POLL_CONFIG.EXE' }}</span>
                
                <div v-if="currentTab === 'exam'" class="exam-nav">
                    <span :class="{ active: examMode === 'take' }" @click="examMode = 'take'">[測驗]</span>
                    <span :class="{ active: examMode === 'edit' }" @click="examMode = 'edit'">[編輯]</span>
                    <span :class="{ active: examMode === 'records' }" @click="examMode = 'records'">[紀錄]</span>
                </div>
                <span v-if="currentTab === 'qa'" style="cursor:pointer;" @click="pollMode = pollMode === 'vote' ? 'edit' : 'vote'">
                    {{ pollMode === 'vote' ? '[編輯投票內容]' : '[返回投票介面]' }}
                </span>
            </div>
            
            <div class="window-content">
                
                <!-- 數位測驗系統 -->
                <div v-if="currentTab === 'exam'">
                    <!-- 編輯模式 -->
                    <div v-if="examMode === 'edit'">
                        <div style="display:flex; justify-content: space-between; margin-bottom: 15px; align-items: center;">
                            <h3>題目編輯器</h3>
                        </div>
                        <div v-for="(q, index) in questions" :key="q.id" class="card-item">
                            <div style="display:flex; justify-content:space-between; margin-bottom:10px;">
                                <span style="font-size:12px; font-weight:bold;">第 {{ index + 1 }} 題</span>
                                <button @click="removeQuestion(index)" class="pixel-btn mini danger">刪除</button>
                            </div>
                            <select v-model="q.type" class="edit-input" @change="syncAnswerFormat(q)">
                                <option value="是非">是非</option>
                                <option value="單選">單選</option>
                                <option value="複選">複選</option>
                            </select>
                            <input v-model="q.text" class="edit-input" placeholder="請輸入題目內容...">
                            
                            <!-- 選項編輯 -->
                            <div v-if="q.type !== '是非'">
                                <p style="font-size:11px; margin-bottom:5px;">選項設定 (勾選代表正確答案)：</p>
                                <div v-for="(opt, oIdx) in q.options" :key="oIdx" class="edit-option-row">
                                    <input v-if="q.type === '單選'" type="radio" :name="'correct-'+q.id" :value="opt.val" v-model="q.answer">
                                    <input v-else type="checkbox" :value="opt.val" v-model="q.answer">
                                    <input v-model="opt.text" class="edit-input" style="margin-bottom:0; flex:1;" placeholder="選項內容">
                                    <button @click="q.options.splice(oIdx, 1)" class="pixel-btn mini" v-if="q.options.length > 2">✕</button>
                                </div>
                                <button @click="q.options.push({val: String.fromCharCode(65 + q.options.length), text: ''})" class="pixel-btn mini" style="margin-top:5px;">＋ 新增選項</button>
                            </div>
                            <div v-else>
                                <p style="font-size:11px; margin-bottom:5px;">正確答案：</p>
                                <label><input type="radio" v-model="q.answer" value="T"> 正確 (T)</label>
                                <label style="margin-left:15px;"><input type="radio" v-model="q.answer" value="F"> 錯誤 (F)</label>
                            </div>

                            <textarea v-model="q.explanation" class="edit-input" style="margin-top:10px; height:60px;" placeholder="輸入題目解析..."></textarea>
                        </div>
                        <button @click="addNewQuestion" class="pixel-btn primary" style="width: 100%; justify-content: center; margin-top: 10px;">＋ 新增題目</button>
                    </div>

                    <!-- 測驗模式 (take) -->
                    <div v-else-if="examMode === 'take'">
                        <div v-if="!examSubmitted" class="exam-container">
                            <div v-for="(q, qIdx) in questions" :key="q.id" class="question-block">
                                <div class="question-text">{{ qIdx + 1 }}. [{{ q.type }}] {{ q.text }}</div>
                                <div class="options-list">
                                    <div v-for="opt in q.options" :key="opt.val" 
                                         class="option-item" 
                                         :class="{ 'selected': isSelected(q, opt.val) }"
                                         @click="toggleOption(q, opt.val)">
                                        <span v-if="q.type === '複選'">{{ isSelected(q, opt.val) ? '☑' : '☐' }}</span>
                                        <span v-else>{{ isSelected(q, opt.val) ? '●' : '○' }}</span>
                                        {{ opt.text }}
                                    </div>
                                </div>
                            </div>
                            <button @click="submitExam" class="pixel-btn primary" style="width:100%; justify-content:center;" :disabled="questions.length === 0">提交測驗卷</button>
                        </div>

                        <div v-else class="exam-result">
                            <div class="card-item" style="text-align:center; margin-bottom:20px;">
                                <h2 style="color:var(--fairy-blue-dark);">測驗結果</h2>
                                <div style="font-size:48px; font-weight:bold; margin:10px 0;">{{ score }} / 100</div>
                                <p>答對題數：{{ correctCount }} / {{ questions.length }}</p>
                                <button @click="resetExam" class="pixel-btn mini" style="margin-top:10px;">重新挑戰</button>
                            </div>

                            <div v-for="(q, qIdx) in questions" :key="q.id" class="question-block">
                                <div class="question-text">{{ qIdx + 1 }}. {{ q.text }}</div>
                                <div class="options-list">
                                    <div v-for="opt in q.options" :key="opt.val" 
                                         class="option-item" 
                                         :class="getFeedbackClass(q, opt.val)">
                                        {{ opt.text }} 
                                        <span v-if="isCorrectAnswer(q, opt.val)" style="margin-left:auto;">(正確答案)</span>
                                    </div>
                                </div>
                                <div class="explanation">
                                    <strong>解析：</strong>{{ q.explanation }}
                                </div>
                            </div>
                        </div>
                    </div>

                    <!-- 紀錄模式 (records) -->
                    <div v-else-if="examMode === 'records'">
                        <div style="display:flex; justify-content: space-between; margin-bottom: 15px; align-items: center;">
                            <h3>歷史答題紀錄 (共 {{ examRecords.length }} 筆)</h3>
                            <button @click="clearRecords" class="pixel-btn danger mini">清空紀錄</button>
                        </div>
                        <div v-if="examRecords.length === 0" style="text-align:center; padding: 40px; opacity:0.5;">目前尚無紀錄 ⋆⁺₊</div>
                        <div v-for="record in examRecords" :key="record.id" class="card-item record-item">
                            <div class="record-info">
                                <span class="record-time">{{ record.time }}</span>
                                <span class="record-score">{{ record.score }} 分 ({{ record.correctCount }}/{{ record.total }})</span>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- 即時問答 -->
                <div v-if="currentTab === 'qa'">
                    <div v-if="pollMode === 'vote'">
                        <div class="card-item qa-poll-card">
                            <h2 style="margin-bottom: 20px; color: var(--fairy-blue-dark);">{{ pollTitle }}</h2>
                            
                            <div class="poll-layout">
                                <div class="chart-container">
                                    <div class="pie-chart" :style="{ background: pieGradient }"></div>
                                    <div class="chart-legend">
                                        <div v-for="opt in pollOptions" :key="opt.id" class="legend-item">
                                            <span class="dot" :style="{ background: opt.color }"></span>
                                            <span class="label">{{ opt.text }}: {{ pollResults[opt.id] }} 票 ({{ getPercentage(opt.id) }}%)</span>
                                        </div>
                                    </div>
                                </div>

                                <div class="poll-options-grid">
                                    <button v-for="opt in pollOptions" :key="opt.id" @click="vote(opt.id)" class="pixel-btn poll-btn" :style="{ borderLeft: '6px solid ' + opt.color }">
                                        <div>{{ opt.text }}</div>
                                        <div style="font-size:14px; opacity:0.7;">{{ pollResults[opt.id] }} 票 ({{ getPercentage(opt.id) }}%)</div>
                                    </button>
                                </div>
                            </div>
                            <div style="margin-top:30px;"><button @click="resetPoll" class="pixel-btn mini danger">清空所有票數</button></div>
                        </div>
                    </div>

                    <div v-else class="poll-edit-mode">
                        <div class="card-item">
                            <h3 style="margin-bottom: 15px;">投票題目設定</h3>
                            <input v-model="pollTitle" class="edit-input" placeholder="請輸入投票問題...">
                            
                            <h3 style="margin: 20px 0 10px;">選項設定</h3>
                            <div v-for="(opt, index) in pollOptions" :key="opt.id" class="edit-option-row">
                                <input type="color" v-model="opt.color" style="width: 40px; height: 35px; border: none; background: none;">
                                <input v-model="opt.text" class="edit-input" style="margin-bottom:0; flex:1;" placeholder="選項名稱">
                                <button @click="removePollOption(index, opt.id)" class="pixel-btn mini danger">✕</button>
                            </div>
                            <button @click="addPollOption" class="pixel-btn primary mini" style="margin-top:15px;">＋ 新增投票選項</button>
                        </div>
                    </div>
                </div>

                <!-- 隨機抽籤 -->
                <RandomDraw v-if="currentTab === 'lottery'" />

                <!-- 競賽計分 -->
                <Scoreboard v-if="currentTab === 'score'" />

                <!-- 報告計時器 -->
                <ClassTimer v-if="currentTab === 'timer'" />

            </div>
        </main>

        <div class="taskbar">
            <span style="background:var(--fairy-blue-dark); color:white; padding:2px 10px; border-radius:10px; margin-right:10px;">SYSTEM</span>
            <span>{{ timeString }}</span>
        </div>
    </div>
</template>

<style>
@font-face {
    font-family: 'HYPixel';
    src: url('./HYPixel11pxU-2.ttf') format('truetype');
}

:root {
    --fairy-blue: #a2cffe;
    --fairy-blue-light: #d6eaff;
    --fairy-blue-dark: #74a9d8;
    --pixel-blue: #5a8fb2;
    --glass-white: rgba(255, 255, 255, 0.65);
    --divine-white: rgba(255, 255, 255, 0.85);
    --lace-color: rgba(255, 255, 255, 0.95);
    --accent-glow: rgba(162, 207, 254, 0.4);
    --correct-green: #a8e6cf;
    --wrong-red: #ffaaa5;
    --warning-orange: #ffd8a8;
}

* { 
    box-sizing: border-box; 
    margin: 0; 
    padding: 0; 
    cursor: url('https://cur.cursors-4u.net/nature/nat-10/nat985.cur'), auto !important; 
}

body {
    background: #f0f7ff;
    background-image: 
        radial-gradient(circle at 50% 50%, transparent 20%, rgba(162, 207, 254, 0.1) 40%, transparent 60%),
        radial-gradient(circle at 50% 50%, transparent 30%, rgba(162, 207, 254, 0.05) 50%, transparent 70%),
        linear-gradient(rgba(162, 207, 254, 0.05) 1px, transparent 1px),
        linear-gradient(90deg, rgba(162, 207, 254, 0.05) 1px, transparent 1px);
    background-size: 800px 800px, 600px 600px, 30px 30px, 30px 30px;
    background-position: center center;
    font-family: 'DotGothic16', 'Space Mono', sans-serif;
    color: var(--pixel-blue);
    min-height: 100vh;
    overflow-x: hidden;
    position: relative;
}
</style>

<style scoped>
.app-root { display: flex; flex-direction: column; align-items: center; padding: 40px 15px 100px; position: relative; min-height: 100vh; z-index: 5; }

/* 背景插圖樣式 */
.bg-illustration-layer {
    position: fixed;
    inset: 0;
    pointer-events: none;
    z-index: 0;
    opacity: 0.35;
    overflow: hidden;
}
.bg-illustration-item {
    position: absolute;
    filter: drop-shadow(0 5px 15px rgba(255, 172, 197, 0.2));
    animation: float-soft infinite ease-in-out;
}
@keyframes float-soft {
    0%, 100% { transform: translateY(0) rotate(0deg); }
    50% { transform: translateY(-30px) rotate(8deg); }
}

.lace-trim { position: fixed; left: 0; width: 100%; height: 16px; z-index: 1001; pointer-events: none; background-image: radial-gradient(circle at 8px 0px, var(--lace-color) 6px, transparent 7px); background-size: 16px 16px; }
.lace-top { top: 0; }
.lace-bottom { bottom: 40px; transform: rotate(180deg); }

header.main-header { text-align: center; margin-bottom: 25px; padding: 0 10px; position: relative; z-index: 10; }
.main-header h1 { font-family: 'HYPixel', sans-serif; font-size: 48px; letter-spacing: 4px; color: var(--fairy-blue-dark); text-shadow: 2px 2px 0 white; }
@media (max-width: 480px) { .main-header h1 { font-size: 28px; letter-spacing: 2px; } }

.nav-tabs { display: flex; gap: 8px; justify-content: center; margin-bottom: 20px; flex-wrap: wrap; }
.tab-item { 
    padding: 10px 16px; background: var(--glass-white); border-radius: 20px; font-size: 13px; border: 1.5px solid white; 
    cursor: pointer; transition: 0.3s; min-width: 80px; text-align: center;
}
.tab-item.active { background: var(--fairy-blue-dark); color: white; transform: scale(1.05); }
@media (max-width: 480px) { .tab-item { flex: 1 1 40%; font-size: 12px; padding: 8px 10px; } }

.cyber-window { position: relative; width: 95%; max-width: 750px; background: var(--glass-white); backdrop-filter: blur(10px); border: 2px solid white; border-radius: 24px; box-shadow: 0 20px 50px rgba(162, 207, 254, 0.15); margin: 0 auto 20px; }
.window-header { background: linear-gradient(90deg, var(--fairy-blue), var(--fairy-blue-light)); padding: 10px 20px; color: white; font-size: 11px; display: flex; justify-content: space-between; border-radius: 22px 22px 0 0; }
.window-content { padding: 25px; max-height: 75vh; overflow-y: auto; }
@media (max-width: 480px) { .window-content { padding: 15px 10px; } }

.pixel-btn { background: white; border: 1.5px solid var(--fairy-blue-dark); padding: 8px 15px; border-radius: 50px; cursor: pointer; transition: 0.2s; font-family: inherit; color: var(--pixel-blue); font-size: 12px; display: inline-flex; align-items: center; gap: 5px; }
.pixel-btn.primary { background: var(--fairy-blue-dark); color: white; }
.pixel-btn.mini { padding: 4px 10px; font-size: 10px; border-radius: 8px; }
.pixel-btn.danger { border-color: #ffaaa5; color: #ffaaa5; }
.pixel-btn:hover:not(:disabled) { transform: translateY(-2px); box-shadow: 0 5px 15px var(--accent-glow); }
.pixel-btn:disabled { opacity: 0.5; cursor: not-allowed !important; }

.card-item { background: white; padding: 20px; border-radius: 18px; border: 1px solid var(--fairy-blue-light); transition: 0.3s; margin-bottom: 15px; }

/* 測驗系統樣式 */
.question-block { margin-bottom: 20px; padding-bottom: 20px; border-bottom: 1px dashed var(--fairy-blue-light); }
.question-text { font-size: 16px; font-weight: bold; margin-bottom: 12px; color: var(--pixel-blue); }
.option-item { display: flex; align-items: center; gap: 10px; padding: 10px 15px; border-radius: 12px; border: 1.5px solid #f0f7ff; margin-bottom: 8px; cursor: pointer; transition: 0.2s; }
.option-item:hover { background: #f9fcff; border-color: var(--fairy-blue-light); }
.option-item.selected { background: var(--fairy-blue-light); border-color: var(--fairy-blue); }
.option-item.correct { background: var(--correct-green); border-color: #87d4b8; }
.option-item.wrong { background: var(--wrong-red); border-color: #f78e88; }
.explanation { font-size: 13px; color: var(--fairy-blue-dark); background: #f0f7ff; padding: 10px; border-radius: 8px; margin-top: 10px; }

/* 編輯模式樣式 */
.edit-input { width: 100%; padding: 8px 12px; border: 1.5px solid var(--fairy-blue-light); border-radius: 8px; font-family: inherit; margin-bottom: 10px; outline: none; }
.edit-input:focus { border-color: var(--fairy-blue); }
.edit-option-row { display: flex; gap: 8px; align-items: center; margin-bottom: 5px; }

/* 即時問答樣式 */
.qa-poll-card { text-align: center; padding: 25px; }
.poll-layout { display: flex; flex-direction: column; gap: 30px; align-items: center; }
@media (min-width: 600px) { .poll-layout { flex-direction: row; justify-content: space-around; } }

.chart-container { display: flex; flex-direction: column; align-items: center; gap: 15px; }
.pie-chart { width: 180px; height: 180px; border-radius: 50%; border: 3px solid white; box-shadow: 0 5px 15px rgba(0,0,0,0.05); transition: 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275); }
.chart-legend { text-align: left; font-size: 12px; }
.legend-item { display: flex; align-items: center; gap: 8px; margin-bottom: 4px; }
.legend-item .dot { width: 10px; height: 10px; border-radius: 2px; }

.poll-options-grid { display: grid; grid-template-columns: 1fr; gap: 12px; width: 100%; max-width: 300px; }
.poll-btn { 
    height: 70px; font-size: 16px; font-weight: bold; justify-content: space-between; 
    padding: 0 20px; background: #fff; border: 1.5px solid var(--fairy-blue-light);
    box-shadow: 4px 4px 0px var(--fairy-blue-light);
}
.poll-btn:active { transform: translate(2px, 2px); box-shadow: 0px 0px 0px; }

.exam-nav { display: flex; gap: 10px; }
.exam-nav span { cursor: pointer; opacity: 0.6; transition: 0.2s; font-size: 11px; }
.exam-nav span.active { opacity: 1; font-weight: bold; color: var(--fairy-blue-dark); text-decoration: underline; }
.record-item { display: flex; justify-content: space-between; padding: 12px 20px; margin-bottom: 8px; }
.record-info { width: 100%; display: flex; justify-content: space-between; align-items: center; }
.record-time { font-size: 11px; opacity: 0.6; }
.record-score { font-weight: bold; color: var(--fairy-blue-dark); }

.window-mascot { position: absolute; top: -65px; right: -10px; height: 90px; z-index: 1002; pointer-events: none; filter: drop-shadow(0 5px 15px rgba(0,0,0,0.1)); }
@media (max-width: 480px) {
    .window-mascot { height: 60px; top: -40px; right: -5px; }
}

.taskbar { position: fixed; bottom: 0; left: 0; width: 100%; height: 40px; background: rgba(255,255,255,0.8); border-top: 1px solid white; display: flex; align-items: center; padding: 0 20px; font-size: 11px; z-index: 2000; }
.team-name-input { border: none; border-bottom: 2px solid transparent; font-family: inherit; font-size: 16px; font-weight: bold; color: var(--pixel-blue); padding: 4px; width: 60%; }

.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>