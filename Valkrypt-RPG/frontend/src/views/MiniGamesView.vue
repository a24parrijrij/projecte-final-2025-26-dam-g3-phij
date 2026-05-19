<template>
  <div class="mini-page">
    <header class="top">
      <button class="btn ghost" @click="router.push('/userpage')">← Tornar</button>
      <h1>Entrenament Arcà</h1>
      <button class="btn" @click="loadProgress" :disabled="loading">{{ loading ? 'Carregant...' : 'Recarregar' }}</button>
    </header>

    <section class="progress-card">
      <div>
        <small>NIVELL DE COMPTE</small>
        <h2>NV {{ progression.level }}</h2>
      </div>
      <div class="xp">
        <div class="xp-bar"><div class="xp-fill" :style="{ width: `${xpPercent}%` }"></div></div>
        <small>XP {{ progression.xp }} / {{ progression.nextLevelXp }} · Partides: {{ progression.gamesPlayed }}</small>
      </div>
      <div class="hero-picker">
        <label>Heroi a entrenar</label>
        <select v-model="selectedHeroId">
          <option v-for="hero in party" :key="hero.id || hero.name" :value="hero.id">{{ hero.name }} · NV {{ hero.level || 1 }}</option>
        </select>
      </div>
    </section>

    <section class="tabs">
      <button class="tab" :class="{ active: tab === 'reflex' }" @click="tab = 'reflex'">Reflexos</button>
      <button class="tab" :class="{ active: tab === 'memory' }" @click="tab = 'memory'">Memòria</button>
      <button class="tab" :class="{ active: tab === 'coop' }" @click="tab = 'coop'">Coop Online</button>
      <button class="tab" :class="{ active: tab === 'timing' }" @click="tab = 'timing'">Timing</button>
      <button class="tab" :class="{ active: tab === 'sequence' }" @click="tab = 'sequence'">Seqüència</button>
      <button class="tab" :class="{ active: tab === 'dodge' }" @click="tab = 'dodge'">Esquiva</button>
      <button class="tab" :class="{ active: tab === 'aim' }" @click="tab = 'aim'">Punteria</button>
      <button class="tab" :class="{ active: tab === 'arithmetic' }" @click="tab = 'arithmetic'">Càlcul</button>
    </section>

    <section class="panel">
      <h3>Rànquing setmanal</h3>
      <p class="muted">Top XP minijocs (últims 7 dies)</p>
      <div v-if="leaderboardLoading" class="muted">Carregant rànquing...</div>
      <div v-else-if="leaderboard.length === 0" class="muted">Encara no hi ha puntuacions.</div>
      <table v-else class="rank-table">
        <thead>
          <tr><th>#</th><th>Jugador</th><th>XP</th><th>Partides</th><th>Millor</th></tr>
        </thead>
        <tbody>
          <tr v-for="entry in leaderboard" :key="entry.userId">
            <td>{{ entry.rank }}</td>
            <td>{{ entry.displayName || entry.username }}</td>
            <td>{{ entry.weeklyXp }}</td>
            <td>{{ entry.gamesPlayed }}</td>
            <td>{{ entry.bestScore }}</td>
          </tr>
        </tbody>
      </table>
    </section>

    <section v-if="tab === 'reflex'" class="panel">
      <h3>Runa de Reflexos</h3>
      <p>Fes clic quan la runa aparegui. 20 segons.</p>
      <div class="arena">
        <button
          class="rune"
          :class="{ active: reflex.active }"
          :style="{ left: `${reflex.x}%`, top: `${reflex.y}%` }"
          @click="hitReflex"
        >✦</button>
      </div>
      <div class="row">
        <span>Punts: {{ reflex.score }}</span>
        <span>Temps: {{ reflex.timeLeft }}s</span>
      </div>
      <button class="btn" @click="startReflex" :disabled="reflex.running">{{ reflex.running ? 'En curs...' : 'Començar' }}</button>
    </section>

    <section v-if="tab === 'memory'" class="panel">
      <h3>Cercle de Memòria</h3>
      <p>Recorda la seqüència i repeteix-la.</p>
      <div class="memory-grid">
        <button
          v-for="(color, idx) in memory.colors"
          :key="color"
          class="memory-cell"
          :class="[color, { glow: memory.flashIndex === idx }]"
          @click="pressMemory(idx)"
          :disabled="!memory.acceptInput"
        ></button>
      </div>
      <div class="row">
        <span>Nivell: {{ memory.level }}</span>
        <span>Errors: {{ memory.errors }}/3</span>
      </div>
      <button class="btn" @click="startMemory" :disabled="memory.running">{{ memory.running ? 'En curs...' : 'Començar' }}</button>
    </section>

    <section v-if="tab === 'coop'" class="panel">
      <h3>Duel Cooperatiu de Reflexos</h3>
      <div class="row wrap">
        <button class="btn" @click="createCoopRoom">Crear sala</button>
        <input v-model="coop.joinCode" placeholder="Codi sala" />
        <button class="btn" @click="joinCoopRoom">Entrar</button>
        <button class="btn" :disabled="matchmaking.searching" @click="joinMatchmaking">
          {{ matchmaking.searching ? `Buscant... (${matchmaking.queueSize})` : 'Matchmaking auto' }}
        </button>
        <button class="btn ghost" :disabled="!matchmaking.searching" @click="leaveMatchmaking">Cancel·lar cua</button>
      </div>
      <div v-if="coop.room" class="coop-box">
        <p><strong>Sala:</strong> {{ coop.room.roomCode }} · <strong>Estat:</strong> {{ coop.room.status }}</p>
        <p>Membres: {{ coop.room.members?.length || 0 }}</p>
        <ul>
          <li v-for="m in coop.room.members || []" :key="m.userId">
            {{ memberName(m.userId) }} · ready: {{ m.ready ? 'sí' : 'no' }} · score: {{ m.score ?? '-' }}
          </li>
        </ul>
        <div class="row wrap">
          <button class="btn" @click="setReady(true)">Ready</button>
          <button class="btn ghost" @click="setReady(false)">Not ready</button>
          <button class="btn" @click="startCoopRun" :disabled="coop.running || coop.room.status !== 'running'">Jugar run</button>
        </div>
        <div class="row">
          <span>Punts run: {{ coop.localScore }}</span>
          <span>Temps: {{ coop.timeLeft }}s</span>
        </div>
        <button class="btn big-hit" :disabled="!coop.running" @click="coop.localScore += 1">CLIC!</button>
      </div>
    </section>

    <section v-if="tab === 'timing'" class="panel">
      <h3>Pols Arcà (Timing)</h3>
      <p>Atura la barra el més a prop possible del centre.</p>
      <div class="timing-track">
        <div class="timing-center"></div>
        <div class="timing-cursor" :style="{ left: `${timing.pos}%` }"></div>
      </div>
      <div class="row">
        <span>Ronda: {{ timing.round }}/5</span>
        <span>Punts: {{ timing.score }}</span>
      </div>
      <div class="row wrap">
        <button class="btn" @click="startTiming" :disabled="timing.running">Començar</button>
        <button class="btn" @click="stopTimingHit" :disabled="!timing.running">Aturar</button>
      </div>
    </section>

    <section v-if="tab === 'sequence'" class="panel">
      <h3>Runes en Seqüència</h3>
      <p>Prem números en ordre creixent abans que acabi el temps.</p>
      <div class="row"><span>Temps: {{ sequence.timeLeft }}s</span><span>Punts: {{ sequence.score }}</span></div>
      <div class="sequence-grid">
        <button v-for="n in sequence.numbers" :key="n.id" class="seq-btn" @click="hitSequence(n.value)" :disabled="!sequence.running">{{ n.value }}</button>
      </div>
      <button class="btn" @click="startSequence" :disabled="sequence.running">{{ sequence.running ? 'En curs...' : 'Començar' }}</button>
    </section>

    <section v-if="tab === 'dodge'" class="panel">
      <h3>Pluja de Fletxes (Esquiva)</h3>
      <p>Mou-te esquerra/dreta i evita els projectils. Tecles ← i →.</p>
      <div class="dodge-arena">
        <div class="dodge-player" :style="{ left: `${dodge.playerX}%` }"></div>
        <div v-for="a in dodge.arrows" :key="a.id" class="dodge-arrow" :style="{ left: `${a.x}%`, top: `${a.y}%` }">↓</div>
      </div>
      <div class="row"><span>Vides: {{ dodge.lives }}</span><span>Supervivència: {{ dodge.score }}</span></div>
      <button class="btn" @click="startDodge" :disabled="dodge.running">{{ dodge.running ? 'En curs...' : 'Començar' }}</button>
    </section>

    <section v-if="tab === 'aim'" class="panel">
      <h3>Diana Mística (Punteria)</h3>
      <p>Fes clic a les dianes que apareixen.</p>
      <div class="arena">
        <button
          v-for="t in aim.targets"
          :key="t.id"
          class="target"
          :style="{ left: `${t.x}%`, top: `${t.y}%` }"
          @click="hitAim(t.id)"
        >◎</button>
      </div>
      <div class="row"><span>Punts: {{ aim.score }}</span><span>Temps: {{ aim.timeLeft }}s</span></div>
      <button class="btn" @click="startAim" :disabled="aim.running">{{ aim.running ? 'En curs...' : 'Començar' }}</button>
    </section>

    <section v-if="tab === 'arithmetic'" class="panel">
      <h3>Ritual de Càlcul</h3>
      <p>Resol operacions ràpidament.</p>
      <div class="math-box">
        <strong>{{ arithmetic.question }}</strong>
        <input v-model="arithmetic.answer" @keydown.enter.prevent="submitArithmetic" placeholder="Resposta" />
      </div>
      <div class="row"><span>Punts: {{ arithmetic.score }}</span><span>Temps: {{ arithmetic.timeLeft }}s</span></div>
      <div class="row wrap">
        <button class="btn" @click="startArithmetic" :disabled="arithmetic.running">{{ arithmetic.running ? 'En curs...' : 'Començar' }}</button>
        <button class="btn" @click="submitArithmetic" :disabled="!arithmetic.running">Enviar</button>
      </div>
    </section>

    <p v-if="feedback" class="feedback">{{ feedback }}</p>
  </div>
</template>

<script setup>
import { computed, onBeforeUnmount, onMounted, reactive, ref } from 'vue';
import { useRouter } from 'vue-router';
import { getApiErrorMessage } from '../services/apiClient';
import { wsService } from '../services/WebSocketService';

const router = useRouter();
const loading = ref(false);
const feedback = ref('');
const tab = ref('reflex');
const party = ref([]);
const selectedHeroId = ref('');
const progression = reactive({ level: 1, xp: 0, nextLevelXp: 120, gamesPlayed: 0, byType: { memory: 0, reflex: 0, coop_reflex: 0, timing: 0, sequence: 0, dodge: 0, aim: 0, arithmetic: 0 } });
const leaderboard = ref([]);
const leaderboardLoading = ref(false);
const user = JSON.parse(localStorage.getItem('user') || '{}');
const userId = String(user?.id || user?._id || '');

const reflex = reactive({ running: false, score: 0, timeLeft: 20, active: false, x: 50, y: 50 });
let reflexTick = null;
let reflexSpawn = null;

const memory = reactive({ running: false, level: 1, errors: 0, colors: ['c1', 'c2', 'c3', 'c4'], sequence: [], input: [], flashIndex: -1, acceptInput: false });
const timing = reactive({ running: false, score: 0, round: 0, pos: 0, dir: 1, timer: null });
const sequence = reactive({ running: false, score: 0, timeLeft: 25, expected: 1, numbers: [], timer: null });
const dodge = reactive({ running: false, lives: 3, score: 0, playerX: 50, arrows: [], tick: null, spawn: null, arrowId: 0 });
const aim = reactive({ running: false, score: 0, timeLeft: 20, targets: [], tick: null, spawn: null, targetId: 0 });
const arithmetic = reactive({ running: false, score: 0, timeLeft: 25, question: '', result: 0, answer: '', timer: null });

const coop = reactive({ joinCode: '', room: null, poll: null, running: false, localScore: 0, timeLeft: 15, timer: null });
const matchmaking = reactive({ searching: false, queueSize: 0, poll: null });

const xpPercent = computed(() => {
  const next = Math.max(1, Number(progression.nextLevelXp || 1));
  const current = Math.max(0, Number(progression.xp || 0));
  return Math.max(0, Math.min(100, (current / next) * 100));
});

function memberName(memberId) {
  if (String(memberId) === userId) return `${user.username || 'Tu'} (tu)`;
  return String(memberId).slice(0, 8);
}

async function loadProgress() {
  if (!userId) return;
  loading.value = true;
  feedback.value = '';
  try {
    const response = await fetch(`/api/game/minigames/${encodeURIComponent(userId)}`);
    const data = await response.json();
    if (!response.ok || !data.success) throw new Error(data.error || 'No s’ha pogut carregar el progrés.');
    party.value = Array.isArray(data.party) ? data.party : [];
    if (!selectedHeroId.value && party.value.length > 0) {
      selectedHeroId.value = party.value[0].id || '';
    }
    Object.assign(progression, data.progression || {});
  } catch (error) {
    feedback.value = getApiErrorMessage(error, 'Error carregant minijocs.');
  } finally {
    loading.value = false;
  }
}

async function loadWeeklyLeaderboard() {
  leaderboardLoading.value = true;
  try {
    const response = await fetch('/api/game/minigames/leaderboard/weekly');
    const data = await response.json();
    if (!response.ok || !data.success) throw new Error(data.error || 'No s’ha pogut carregar el rànquing.');
    leaderboard.value = Array.isArray(data.leaderboard) ? data.leaderboard : [];
  } catch (error) {
    feedback.value = getApiErrorMessage(error, 'Error carregant el rànquing.');
    leaderboard.value = [];
  } finally {
    leaderboardLoading.value = false;
  }
}

async function claimReward(gameType, score, durationMs) {
  try {
    const response = await fetch('/api/game/minigames/complete', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId, gameType, score, durationMs, heroId: selectedHeroId.value })
    });
    const data = await response.json();
    if (!response.ok || !data.success) throw new Error(data.error || 'No s’ha pogut aplicar recompensa.');
    party.value = Array.isArray(data.party) ? data.party : party.value;
    Object.assign(progression, data.progression || progression);
    feedback.value = `+${data.reward?.xp || 0} XP · ${gameType}`;
  } catch (error) {
    feedback.value = getApiErrorMessage(error, 'Error aplicant recompensa.');
  }
}

function spawnReflexRune() {
  reflex.active = true;
  reflex.x = 10 + Math.random() * 80;
  reflex.y = 12 + Math.random() * 74;
}

function hitReflex() {
  if (!reflex.running || !reflex.active) return;
  reflex.score += 1;
  reflex.active = false;
}

function stopReflex() {
  reflex.running = false;
  reflex.active = false;
  if (reflexTick) clearInterval(reflexTick);
  if (reflexSpawn) clearInterval(reflexSpawn);
  reflexTick = null;
  reflexSpawn = null;
}

function startReflex() {
  stopReflex();
  reflex.running = true;
  reflex.score = 0;
  reflex.timeLeft = 20;
  spawnReflexRune();
  reflexSpawn = setInterval(spawnReflexRune, 850);
  reflexTick = setInterval(async () => {
    reflex.timeLeft -= 1;
    if (reflex.timeLeft <= 0) {
      stopReflex();
      await claimReward('reflex', reflex.score * 10, 20000);
    }
  }, 1000);
}

function stopTiming() {
  timing.running = false;
  if (timing.timer) clearInterval(timing.timer);
  timing.timer = null;
}

function startTiming() {
  stopTiming();
  timing.running = true;
  timing.score = 0;
  timing.round = 1;
  timing.pos = 0;
  timing.dir = 1;
  timing.timer = setInterval(() => {
    timing.pos += timing.dir * 2.5;
    if (timing.pos >= 100) { timing.pos = 100; timing.dir = -1; }
    if (timing.pos <= 0) { timing.pos = 0; timing.dir = 1; }
  }, 40);
}

async function stopTimingHit() {
  if (!timing.running) return;
  const diff = Math.abs(50 - timing.pos);
  const points = Math.max(0, 20 - Math.floor(diff / 2.5));
  timing.score += points;
  timing.round += 1;
  if (timing.round > 5) {
    stopTiming();
    await claimReward('timing', timing.score * 5, 12000);
  }
}

function makeSequenceNumbers() {
  const nums = [];
  for (let i = 1; i <= 9; i += 1) nums.push(i);
  nums.sort(() => Math.random() - 0.5);
  sequence.numbers = nums.map((value, idx) => ({ id: `${idx}_${value}_${Date.now()}`, value }));
  sequence.expected = 1;
}

function stopSequence() {
  sequence.running = false;
  if (sequence.timer) clearInterval(sequence.timer);
  sequence.timer = null;
}

function startSequence() {
  stopSequence();
  sequence.running = true;
  sequence.score = 0;
  sequence.timeLeft = 25;
  makeSequenceNumbers();
  sequence.timer = setInterval(async () => {
    sequence.timeLeft -= 1;
    if (sequence.timeLeft <= 0) {
      stopSequence();
      await claimReward('sequence', sequence.score * 8, 25000);
    }
  }, 1000);
}

function hitSequence(value) {
  if (!sequence.running) return;
  if (value === sequence.expected) {
    sequence.score += 1;
    sequence.expected += 1;
    if (sequence.expected > 9) {
      sequence.score += 3;
      makeSequenceNumbers();
    }
  } else {
    sequence.score = Math.max(0, sequence.score - 1);
  }
}

function moveDodgePlayer(delta) {
  if (!dodge.running) return;
  dodge.playerX = Math.max(5, Math.min(95, dodge.playerX + delta));
}

function onKeyDown(event) {
  if (event.key === 'ArrowLeft') moveDodgePlayer(-6);
  if (event.key === 'ArrowRight') moveDodgePlayer(6);
}

function stopDodge() {
  dodge.running = false;
  if (dodge.tick) clearInterval(dodge.tick);
  if (dodge.spawn) clearInterval(dodge.spawn);
  dodge.tick = null;
  dodge.spawn = null;
}

function startDodge() {
  stopDodge();
  dodge.running = true;
  dodge.lives = 3;
  dodge.score = 0;
  dodge.playerX = 50;
  dodge.arrows = [];
  dodge.spawn = setInterval(() => {
    dodge.arrowId += 1;
    dodge.arrows.push({ id: dodge.arrowId, x: 8 + Math.random() * 84, y: 0 });
  }, 550);
  dodge.tick = setInterval(async () => {
    dodge.score += 1;
    dodge.arrows = dodge.arrows.map((a) => ({ ...a, y: a.y + 5 })).filter((a) => {
      const hit = a.y >= 90 && Math.abs(a.x - dodge.playerX) < 8;
      if (hit) dodge.lives -= 1;
      return a.y <= 102 && !hit;
    });
    if (dodge.lives <= 0) {
      const finalScore = dodge.score;
      stopDodge();
      await claimReward('dodge', finalScore, 18000);
    }
  }, 120);
}

function stopAim() {
  aim.running = false;
  if (aim.tick) clearInterval(aim.tick);
  if (aim.spawn) clearInterval(aim.spawn);
  aim.tick = null;
  aim.spawn = null;
}

function startAim() {
  stopAim();
  aim.running = true;
  aim.score = 0;
  aim.timeLeft = 20;
  aim.targets = [];
  aim.spawn = setInterval(() => {
    aim.targetId += 1;
    aim.targets.push({ id: aim.targetId, x: 8 + Math.random() * 84, y: 10 + Math.random() * 78 });
    if (aim.targets.length > 5) aim.targets.shift();
  }, 500);
  aim.tick = setInterval(async () => {
    aim.timeLeft -= 1;
    if (aim.timeLeft <= 0) {
      const finalScore = aim.score;
      stopAim();
      await claimReward('aim', finalScore * 6, 20000);
    }
  }, 1000);
}

function hitAim(targetId) {
  if (!aim.running) return;
  const before = aim.targets.length;
  aim.targets = aim.targets.filter((target) => target.id !== targetId);
  if (aim.targets.length !== before) aim.score += 1;
}

function newArithmeticQuestion() {
  const a = 1 + Math.floor(Math.random() * 20);
  const b = 1 + Math.floor(Math.random() * 20);
  const ops = ['+', '-', '*'];
  const op = ops[Math.floor(Math.random() * ops.length)];
  arithmetic.question = `${a} ${op} ${b} = ?`;
  arithmetic.result = op === '+' ? a + b : op === '-' ? a - b : a * b;
  arithmetic.answer = '';
}

function stopArithmetic() {
  arithmetic.running = false;
  if (arithmetic.timer) clearInterval(arithmetic.timer);
  arithmetic.timer = null;
}

function startArithmetic() {
  stopArithmetic();
  arithmetic.running = true;
  arithmetic.score = 0;
  arithmetic.timeLeft = 25;
  newArithmeticQuestion();
  arithmetic.timer = setInterval(async () => {
    arithmetic.timeLeft -= 1;
    if (arithmetic.timeLeft <= 0) {
      const finalScore = arithmetic.score;
      stopArithmetic();
      await claimReward('arithmetic', finalScore * 7, 25000);
    }
  }, 1000);
}

function submitArithmetic() {
  if (!arithmetic.running) return;
  const value = Number(arithmetic.answer);
  if (Number.isFinite(value) && value === arithmetic.result) arithmetic.score += 1;
  else arithmetic.score = Math.max(0, arithmetic.score - 1);
  newArithmeticQuestion();
}

async function flashSequence() {
  memory.acceptInput = false;
  for (const idx of memory.sequence) {
    memory.flashIndex = idx;
    await new Promise((r) => setTimeout(r, 420));
    memory.flashIndex = -1;
    await new Promise((r) => setTimeout(r, 180));
  }
  memory.input = [];
  memory.acceptInput = true;
}

async function startMemory() {
  memory.running = true;
  memory.level = 1;
  memory.errors = 0;
  memory.sequence = [Math.floor(Math.random() * 4)];
  await flashSequence();
}

async function pressMemory(idx) {
  if (!memory.running || !memory.acceptInput) return;
  memory.input.push(idx);
  const pos = memory.input.length - 1;
  if (memory.sequence[pos] !== idx) {
    memory.errors += 1;
    memory.input = [];
    if (memory.errors >= 3) {
      memory.running = false;
      memory.acceptInput = false;
      await claimReward('memory', memory.level * 12, memory.level * 3000);
      return;
    }
    await flashSequence();
    return;
  }

  if (memory.input.length === memory.sequence.length) {
    memory.level += 1;
    memory.sequence.push(Math.floor(Math.random() * 4));
    await flashSequence();
  }
}

async function createCoopRoom() {
  try {
    const response = await fetch('/api/game/minigames/coop/create', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId, gameType: 'coop_reflex' })
    });
    const data = await response.json();
    if (!response.ok || !data.success) throw new Error(data.error || 'No s’ha pogut crear la sala.');
    await ensureWsConnected();
    coop.room = data.room;
    coop.joinCode = data.room.roomCode;
    wsService.joinMiniCoop(coop.room.roomCode, userId, user.username || 'Jugador');
    startCoopPolling();
  } catch (error) {
    feedback.value = getApiErrorMessage(error, 'Error creant sala coop.');
  }
}

async function joinCoopRoom() {
  if (!coop.joinCode.trim()) return;
  try {
    const response = await fetch('/api/game/minigames/coop/join', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId, roomCode: coop.joinCode.trim().toUpperCase() })
    });
    const data = await response.json();
    if (!response.ok || !data.success) throw new Error(data.error || 'No s’ha pogut entrar a la sala.');
    await ensureWsConnected();
    coop.room = data.room;
    wsService.joinMiniCoop(coop.room.roomCode, userId, user.username || 'Jugador');
    startCoopPolling();
  } catch (error) {
    feedback.value = getApiErrorMessage(error, 'Error entrant a sala coop.');
  }
}

async function joinMatchmaking() {
  try {
    await ensureWsConnected();
    const response = await fetch('/api/game/minigames/coop/matchmaking/join', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId, gameType: 'coop_reflex' })
    });
    const data = await response.json();
    if (!response.ok || !data.success) throw new Error(data.error || 'No s’ha pogut entrar a matchmaking.');
    if (data.matched && data.roomCode) {
      matchmaking.searching = false;
      coop.joinCode = data.roomCode;
      await joinCoopRoom();
      return;
    }
    matchmaking.searching = true;
    matchmaking.queueSize = Number(data.queueSize || 1);
    startMatchmakingPolling();
  } catch (error) {
    feedback.value = getApiErrorMessage(error, 'Error iniciant matchmaking.');
  }
}

async function leaveMatchmaking() {
  try {
    await fetch('/api/game/minigames/coop/matchmaking/leave', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId, gameType: 'coop_reflex' })
    });
  } catch {}
  matchmaking.searching = false;
  matchmaking.queueSize = 0;
  if (matchmaking.poll) clearInterval(matchmaking.poll);
  matchmaking.poll = null;
}

async function pollMatchmakingStatus() {
  if (!matchmaking.searching) return;
  const response = await fetch(`/api/game/minigames/coop/matchmaking/status/${encodeURIComponent(userId)}`);
  const data = await response.json();
  if (!response.ok || !data.success) return;
  if (data.matched && data.roomCode) {
    matchmaking.searching = false;
    coop.joinCode = data.roomCode;
    await joinCoopRoom();
    return;
  }
  matchmaking.queueSize = Number(data.queueSize || matchmaking.queueSize || 1);
}

function startMatchmakingPolling() {
  if (matchmaking.poll) clearInterval(matchmaking.poll);
  matchmaking.poll = setInterval(() => pollMatchmakingStatus().catch(() => {}), 3000);
}

async function pollCoopRoom() {
  if (!coop.room?.roomCode) return;
  const response = await fetch(`/api/game/minigames/coop/${encodeURIComponent(coop.room.roomCode)}`);
  const data = await response.json();
  if (response.ok && data.success) coop.room = data.room;
}

function startCoopPolling() {
  if (coop.poll) clearInterval(coop.poll);
  coop.poll = setInterval(() => pollCoopRoom().catch(() => {}), 10000);
}

async function setReady(ready) {
  if (!coop.room?.roomCode) return;
  const response = await fetch('/api/game/minigames/coop/submit', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId, roomCode: coop.room.roomCode, ready })
  });
  const data = await response.json();
  if (response.ok && data.success) {
    coop.room = data.room;
    wsService.miniCoopReady(coop.room.roomCode, userId, user.username || 'Jugador', ready);
  }
}

function stopCoopRun() {
  coop.running = false;
  if (coop.timer) clearInterval(coop.timer);
  coop.timer = null;
}

function startCoopRun() {
  stopCoopRun();
  coop.running = true;
  coop.localScore = 0;
  coop.timeLeft = 15;
  coop.timer = setInterval(async () => {
    coop.timeLeft -= 1;
    if (coop.timeLeft <= 0) {
      stopCoopRun();
      await fetch('/api/game/minigames/coop/submit', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ userId, roomCode: coop.room.roomCode, score: coop.localScore, ready: true })
      });
      wsService.miniCoopScore(coop.room.roomCode, userId, user.username || 'Jugador', coop.localScore);
      await claimReward('coop_reflex', coop.localScore * 8, 15000);
      await pollCoopRoom();
      await loadWeeklyLeaderboard();
    }
  }, 1000);
}

async function ensureWsConnected() {
  try {
    await wsService.connect();
  } catch {}
}

function setupMiniCoopRealtime() {
  wsService.on('miniCoopPresence', async (msg) => {
    if (!coop.room?.roomCode || msg.roomCode !== coop.room.roomCode) return;
    await pollCoopRoom();
  });
  wsService.on('miniCoopSignal', async (msg) => {
    if (!coop.room?.roomCode || msg.roomCode !== coop.room.roomCode) return;
    await pollCoopRoom();
  });
  wsService.on('miniMatchFound', async (msg) => {
    if (!matchmaking.searching) return;
    if (!msg?.roomCode) return;
    matchmaking.searching = false;
    coop.joinCode = String(msg.roomCode);
    await joinCoopRoom();
  });
}

onMounted(() => {
  if (!userId) {
    router.push('/login');
    return;
  }
  setupMiniCoopRealtime();
  window.addEventListener('keydown', onKeyDown);
  loadProgress();
  loadWeeklyLeaderboard();
});

onBeforeUnmount(() => {
  stopReflex();
  stopTiming();
  stopSequence();
  stopDodge();
  stopAim();
  stopArithmetic();
  stopCoopRun();
  window.removeEventListener('keydown', onKeyDown);
  if (coop.poll) clearInterval(coop.poll);
  if (matchmaking.poll) clearInterval(matchmaking.poll);
  leaveMatchmaking();
  if (coop.room?.roomCode) wsService.leaveMiniCoop(coop.room.roomCode);
});
</script>

<style scoped>
.mini-page { min-height: 100vh; padding: 20px; color: #efe8d8; background: radial-gradient(circle at 20% 10%, #1a1321 0%, #060709 60%, #020304 100%); }
.top { display: flex; justify-content: space-between; align-items: center; gap: 10px; margin-bottom: 14px; }
.top h1 { margin: 0; color: #d4b06f; }
.btn { border: 1px solid #85693b; color: #f5e7c6; background: #141015; padding: 8px 12px; border-radius: 10px; cursor: pointer; }
.btn.ghost { opacity: .9; }
.progress-card,.panel { border: 1px solid rgba(212,176,111,.35); background: rgba(8,10,14,.86); border-radius: 14px; padding: 14px; margin-bottom: 12px; }
.progress-card { display: grid; grid-template-columns: 180px 1fr 280px; gap: 14px; align-items: center; }
.xp-bar { height: 10px; background: #1a1d24; border-radius: 99px; overflow: hidden; border: 1px solid #3a2f1d; }
.xp-fill { height: 100%; background: linear-gradient(90deg, #8d6b2e, #e1c27d); }
.hero-picker select,.panel input { width: 100%; background: #0f131a; color: #f1ead8; border: 1px solid #42311a; border-radius: 8px; padding: 8px; }
.tabs { display: flex; gap: 10px; margin-bottom: 10px; }
.tab { border: 1px solid #5b4623; background: #0f1116; color: #d7c49b; border-radius: 999px; padding: 8px 12px; cursor: pointer; }
.tab.active { background: #2b1f11; color: #ffe0a8; }
.arena { position: relative; height: 300px; border: 1px dashed #5a4320; border-radius: 12px; background: rgba(18,13,15,.7); overflow: hidden; }
.rune { position: absolute; transform: translate(-50%, -50%); width: 54px; height: 54px; border-radius: 50%; border: 1px solid #5c4a2d; background: #1b1d23; color: #f6e0ad; font-size: 24px; opacity: .25; }
.rune.active { opacity: 1; box-shadow: 0 0 30px rgba(244,199,120,.55); }
.row { display: flex; justify-content: space-between; gap: 10px; margin: 10px 0; }
.row.wrap { flex-wrap: wrap; justify-content: flex-start; }
.memory-grid { display: grid; grid-template-columns: repeat(2, 120px); gap: 10px; }
.memory-cell { width: 120px; height: 120px; border-radius: 16px; border: 1px solid #53452d; opacity: .65; }
.memory-cell.glow { opacity: 1; box-shadow: 0 0 22px rgba(255,255,255,.35); }
.memory-cell.c1 { background: #8a2d2d; } .memory-cell.c2 { background: #2c6a8e; } .memory-cell.c3 { background: #3a7b4d; } .memory-cell.c4 { background: #8b6b2f; }
.coop-box { border: 1px solid #473318; border-radius: 12px; padding: 12px; }
.big-hit { font-size: 1.2rem; padding: 14px 20px; }
.timing-track { position: relative; height: 28px; border: 1px solid #5e4820; border-radius: 999px; background: #12151d; overflow: hidden; }
.timing-center { position: absolute; left: 49%; width: 2%; top: 0; bottom: 0; background: rgba(225, 194, 125, 0.55); }
.timing-cursor { position: absolute; top: 2px; bottom: 2px; width: 10px; border-radius: 6px; background: #f1d28f; transform: translateX(-50%); }
.sequence-grid { display: grid; grid-template-columns: repeat(3, 90px); gap: 8px; margin: 8px 0; }
.seq-btn { height: 56px; border-radius: 10px; border: 1px solid #5f4a27; background: #121722; color: #efd8a9; font-weight: 700; }
.dodge-arena { position: relative; height: 280px; border-radius: 12px; border: 1px solid #614923; background: rgba(9, 11, 16, 0.8); overflow: hidden; }
.dodge-player { position: absolute; bottom: 8px; width: 28px; height: 28px; border-radius: 50%; background: #8cc4ff; box-shadow: 0 0 14px rgba(140, 196, 255, 0.6); transform: translateX(-50%); }
.dodge-arrow { position: absolute; font-size: 18px; color: #f8b07f; transform: translateX(-50%); }
.target { position: absolute; transform: translate(-50%, -50%); width: 52px; height: 52px; border-radius: 50%; border: 1px solid #5f4e30; background: rgba(33, 26, 31, 0.88); color: #f4d69a; font-size: 24px; }
.math-box { display: grid; gap: 8px; margin-bottom: 10px; }
.math-box strong { font-size: 1.4rem; color: #f0d7a2; }
.math-box input { max-width: 240px; }
.feedback { margin-top: 8px; color: #f3d39c; }
.muted { color: #b8ab8f; font-size: .92rem; margin: 4px 0 10px; }
.rank-table { width: 100%; border-collapse: collapse; font-size: .92rem; }
.rank-table th,.rank-table td { border-bottom: 1px solid rgba(212,176,111,.16); padding: 8px; text-align: left; }
@media (max-width: 980px) { .progress-card { grid-template-columns: 1fr; } }
</style>
