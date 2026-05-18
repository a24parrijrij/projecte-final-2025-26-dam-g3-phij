<template>
  <section class="tactical-combat-panel">
    <header class="combat-head">
      <div class="combat-title">
        <small class="eyebrow">COMBATE TACTICO</small>
        <h3>{{ enemy.name }}</h3>
      </div>
      <div class="combat-head-status">
        <span class="head-chip danger">{{ enemy.typeLabel }}</span>
        <span class="head-chip">RONDA {{ round }}</span>
        <span class="head-chip accent" v-if="isPlanningOrder">ORDEN</span>
        <span class="head-chip accent" v-else-if="activeHero">TURNO · {{ activeHero.name }}</span>
      </div>
    </header>
    <section class="combat-atmosphere">
      <div class="atmosphere-left">
        <small class="atmo-label">RITMO DEL ENCUENTRO</small>
        <strong>{{ momentumTitle }}</strong>
        <div class="momentum-bar">
          <div class="momentum-fill" :style="{ width: `${momentumPercent}%` }"></div>
        </div>
      </div>
      <div class="atmosphere-right">
        <small class="atmo-label">INTENCIÓN ENEMIGA</small>
        <strong>{{ enemyIntentLabel }}</strong>
      </div>
    </section>
    <transition-group name="hud-toast" tag="div" class="hud-toast-stack">
      <article
        v-for="toast in hudToasts"
        :key="toast.id"
        class="hud-toast"
        :class="toast.type"
      >
        <strong>{{ toast.title }}</strong>
        <small>{{ toast.detail }}</small>
      </article>
    </transition-group>

    <section class="order-planner" v-if="isPlanningOrder">
      <div class="planner-topline">
        <strong>Elige el orden</strong>
        <div class="planner-actions">
          <button class="planner-btn minor" @click="setAutoOrder">AUTO</button>
          <button class="planner-btn" :disabled="planningOrder.length !== aliveHeroes.length" @click="confirmOrder">
            LISTO
          </button>
        </div>
      </div>
      <div class="planner-grid">
        <button
          v-for="hero in aliveHeroes"
          :key="`planner_${hero.id}`"
          class="planner-hero"
          :class="{ selected: planningOrder.includes(hero.id) }"
          @click="toggleHeroInOrder(hero.id)"
        >
          <span class="planner-icon">
            <img :src="heroPortrait(hero)" :alt="hero.name" />
            <span class="portrait-glyph">{{ hero.icon }}</span>
          </span>
          <span class="planner-name">{{ hero.name }}</span>
          <span v-if="plannerOrderNumber(hero.id)" class="planner-rank">{{ plannerOrderNumber(hero.id) }}</span>
        </button>
      </div>
      <div v-if="planningOrder.length > 0" class="planner-sequence">
        <span v-for="(heroId, idx) in planningOrder" :key="`seq_${heroId}_${idx}`">
          <b>{{ idx + 1 }}</b>{{ heroName(heroId) }}
        </span>
      </div>
    </section>

    <section class="duel-stage">
      <article class="enemy-stage" :class="[enemy.type, enemyFx.tone]">
        <div class="enemy-aura"></div>
        <span
          v-if="enemyFx.pop"
          :key="`enemy_pop_${enemyFx.token}`"
          class="combat-pop enemy-pop"
          :class="enemyFx.tone"
        >
          {{ enemyFx.pop }}
        </span>
        <div class="enemy-stage-top">
          <span class="enemy-icon">{{ enemy.icon }}</span>
          <div class="enemy-identity">
            <small>{{ enemy.typeLabel }}</small>
            <strong>{{ enemy.name }}</strong>
          </div>
        </div>

        <div class="enemy-health">
          <div class="enemy-health-bar">
            <div class="enemy-health-fill" :style="{ width: `${enemyHealthPercent}%` }"></div>
          </div>
        <div class="enemy-health-meta">
            <span class="enemy-hp-numeric">HP {{ enemy.hp }} / {{ enemy.maxHp }}</span>
            <span v-if="enemy.accuracyPenalty > 0">PREC -{{ enemy.accuracyPenalty }}</span>
          </div>
        </div>

        <div v-if="enemy.statusEffects && enemy.statusEffects.length > 0" class="status-row enemy-status-row">
          <span
            v-for="effect in visibleStatuses(enemy.statusEffects)"
            :key="`enemy_${effect.id}`"
            class="status-pill"
            :class="`is-${effect.type}`"
          >
            {{ shortStatus(effect) }}
          </span>
          <span v-if="extraStatusCount(enemy.statusEffects) > 0" class="status-pill extra">
            +{{ extraStatusCount(enemy.statusEffects) }}
          </span>
        </div>
      </article>

      <div class="ally-rack">
        <article
          v-for="hero in localHeroes"
          :key="hero.id"
          class="ally-card"
          :class="[heroFxTone(hero.id), {
            active: activeHero && hero.id === activeHero.id,
            dead: hero.hp <= 0,
            ordered: roundOrder.includes(hero.id)
          }]"
        >
          <span
            v-if="heroFxPop(hero.id)"
            :key="`hero_pop_${hero.id}_${heroFxToken(hero.id)}`"
            class="combat-pop ally-pop"
            :class="heroFxTone(hero.id)"
          >
            {{ heroFxPop(hero.id) }}
          </span>
          <div class="ally-top">
            <span class="ally-icon">
              <img :src="heroPortrait(hero)" :alt="hero.name" />
              <span class="portrait-glyph">{{ hero.icon }}</span>
            </span>
            <small class="ally-order" v-if="roundOrder.includes(hero.id)">
              #{{ roundOrder.indexOf(hero.id) + 1 }}
            </small>
          </div>
          <strong>{{ hero.name }}</strong>
          <small class="ally-level">NV {{ hero.level || 1 }}</small>
          <div class="ally-hp-bar">
            <div class="ally-hp-fill" :style="{ width: `${Math.max(0, (hero.hp / hero.maxHp) * 100)}%` }"></div>
          </div>
          <small class="ally-hp-text">{{ hero.hp }} / {{ hero.maxHp }}</small>
          <div class="ally-xp-bar">
            <div class="ally-xp-fill" :style="{ width: `${heroXpPercent(hero)}%` }"></div>
          </div>
          <small class="ally-xp-text">XP {{ hero.experience || 0 }} / {{ hero.nextLevelXp || 1 }}</small>
          <div class="ally-attr-row">
            <span>ATQ {{ hero.attack || 0 }}</span>
            <span>DEF {{ hero.defense || 0 }}</span>
            <span>MAG {{ hero.magic || 0 }}</span>
            <span>AGI {{ hero.agility || 0 }}</span>
          </div>
          <div class="ally-flags">
            <span class="mini-flag guard" v-if="hero.guarding">GUARDIA</span>
            <span
              v-for="effect in visibleStatuses(hero.statusEffects)"
              :key="`${hero.id}_${effect.id}`"
              class="mini-flag status"
            >
              {{ shortStatus(effect) }}
            </span>
            <span v-if="extraStatusCount(hero.statusEffects) > 0" class="mini-flag status">
              +{{ extraStatusCount(hero.statusEffects) }}
            </span>
          </div>
        </article>
      </div>
    </section>

    <section class="combat-controls" v-if="!isPlanningOrder && activeHero">
      <div class="control-tabs">
        <button class="control-tab-btn" :class="{ active: activeControlTab === 'state' }" @click="activeControlTab = 'state'">ESTADO</button>
        <button class="control-tab-btn" :class="{ active: activeControlTab === 'skills' }" @click="activeControlTab = 'skills'">HABILIDADES</button>
        <button class="control-tab-btn" :class="{ active: activeControlTab === 'items' }" @click="activeControlTab = 'items'">INVENTARIO</button>
      </div>

      <div class="actor-row">
        <div class="actor-badge">
          <span class="actor-icon">
            <img :src="heroPortrait(activeHero)" :alt="activeHero.name" />
            <span class="portrait-glyph">{{ activeHero.icon }}</span>
          </span>
          <div>
            <strong>{{ activeHero.name }}</strong>
            <small>HP {{ activeHero.hp }} / {{ activeHero.maxHp }}</small>
          </div>
        </div>
        <div class="actor-stats">
          <span>NV {{ activeHero.level || 1 }}</span>
          <span>XP {{ activeHero.experience || 0 }}/{{ activeHero.nextLevelXp || 1 }}</span>
          <span>ATQ {{ activeStats.attack }}</span>
          <span>DEF {{ activeStats.defense }}</span>
          <span>MAG {{ activeStats.magic }}</span>
          <span>AGI {{ activeStats.agility }}</span>
        </div>
      </div>

      <div class="active-hero-xp">
        <div class="active-hero-xp-bar">
          <div class="active-hero-xp-fill" :style="{ width: `${heroXpPercent(activeHero)}%` }"></div>
        </div>
      </div>

      <transition name="tab-fade" mode="out-in">
        <div class="state-block" v-if="activeControlTab === 'state'" key="tab_state">
          <div class="main-actions">
            <button class="combat-btn primary" :disabled="controlsLocked" @click="performBasicAttack">ATACAR</button>
            <button class="combat-btn primary" :disabled="controlsLocked" @click="performDefend">DEFENDER</button>
          </div>
          <div class="state-stats-grid">
            <span><b>ATAQUE</b>{{ activeStats.attack }}</span>
            <span><b>DEFENSA</b>{{ activeStats.defense }}</span>
            <span><b>MAGIA</b>{{ activeStats.magic }}</span>
            <span><b>AGILIDAD</b>{{ activeStats.agility }}</span>
          </div>
        </div>

        <div class="action-grid" v-else key="tab_actions">
          <div class="skill-block" v-if="activeControlTab === 'skills'">
          <div class="block-head">
            <h4>HABILIDADES</h4>
            <small>{{ (activeHero.skills || []).length }}</small>
          </div>
          <div class="action-list">
            <article v-for="skill in activeHero.skills || []" :key="skill.id" class="action-card">
              <div class="action-copy">
                <strong>{{ skill.name }}</strong>
                <small>{{ compactSkillMeta(skill) }}</small>
              </div>
              <button class="combat-btn minor" :disabled="controlsLocked" @click="performSkill(skill.id)">USAR</button>
            </article>
            <p v-if="!activeHero.skills || activeHero.skills.length === 0" class="empty-note">Sin habilidades.</p>
          </div>
        </div>

          <div class="item-block" v-if="activeControlTab === 'items'">
            <div class="item-head">
              <div class="block-head">
                <h4>CONSUMIBLES</h4>
                <small>{{ activeConsumables.length }}</small>
              </div>
              <label class="target-select">
                <span>Objetivo</span>
                <select v-model="itemTargetHeroId">
                  <option v-for="hero in aliveHeroes" :key="`target_${hero.id}`" :value="hero.id">{{ hero.name }}</option>
                </select>
              </label>
            </div>
            <div class="action-list">
              <article v-for="item in activeConsumables" :key="item.id" class="action-card">
                <div class="action-copy">
                  <strong>{{ item.name }}</strong>
                  <small>{{ compactConsumableMeta(item) }}</small>
                </div>
                <button class="combat-btn minor" :disabled="controlsLocked" @click="performConsumable(item.id)">USAR</button>
              </article>
              <p v-if="activeConsumables.length === 0" class="empty-note">Sin consumibles.</p>
            </div>
          </div>
        </div>
      </transition>
    </section>

    <div v-if="recentLines.length > 0" class="combat-feed">
      <span v-for="(line, idx) in recentLines" :key="`${idx}_${line}`" class="feed-line">{{ line }}</span>
    </div>
  </section>
</template>

<script setup>
import { computed, onBeforeUnmount, ref, watch } from 'vue';
import archerPortrait from '../../images/archer.png';
import warriorPortrait from '../../images/warrior.jpg';
import magePortrait from '../../images/mage.jpg';
import dwarfPortrait from '../../images/dwarf.jpg';
import {
  normalizePartyState,
  computeHeroCombatStats,
  useHeroConsumable,
  heroConsumables,
  normalizeStatusEffects,
  grantExperienceToParty
} from '../utils/partySystem';

const props = defineProps({
  encounter: {
    type: Object,
    default: null
  },
  party: {
    type: Array,
    default: () => []
  }
});

const emit = defineEmits(['sync-party', 'end']);

const localHeroes = ref([]);
const enemy = ref({
  name: 'Hostil desconocido',
  type: 'escaramuza',
  typeLabel: 'ESCARAMUZA',
  icon: '👹',
  hp: 60,
  maxHp: 60,
  attack: 8,
  defense: 2,
  accuracyPenalty: 0,
  statusEffects: [],
  statusOnHit: null
});

const round = ref(1);
const battleLog = ref([]);
const locked = ref(false);
const finished = ref(false);

const isPlanningOrder = ref(true);
const planningOrder = ref([]);
const roundOrder = ref([]);
const turnPointer = ref(0);
const itemTargetHeroId = ref('');
const activeControlTab = ref('state');
const ownsLocalPartyState = ref(false);
const enemyFx = ref({ tone: '', pop: '', token: 0 });
const heroFx = ref({});
const hudToasts = ref([]);
const comboStreak = ref(0);
const enemyIntentBias = ref(0);
const fxTimers = new Set();

const ENEMY_PRESETS = {
  escaramuza: {
    hp: 56,
    attack: 7,
    defense: 2,
    icon: '👹',
    label: 'ESCARAMUZA',
    statusOnHit: { type: 'bleed', chance: 24, duration: 2, potency: 2 }
  },
  elite: {
    hp: 92,
    attack: 9,
    defense: 3,
    icon: '🧟',
    label: 'ELITE',
    statusOnHit: { type: 'poison', chance: 28, duration: 2, potency: 3 }
  },
  jefe: {
    hp: 150,
    attack: 12,
    defense: 5,
    icon: '💀',
    label: 'JEFE',
    statusOnHit: { type: 'stun', chance: 20, duration: 1, potency: 0 }
  }
};

const aliveHeroes = computed(() => localHeroes.value.filter((hero) => hero.hp > 0));
const activeHeroId = computed(() => roundOrder.value[turnPointer.value] || '');
const activeHero = computed(() => localHeroes.value.find((hero) => hero.id === activeHeroId.value) || null);
const activeStats = computed(() => computeHeroCombatStats(activeHero.value));
const activeConsumables = computed(() => heroConsumables(activeHero.value));
const controlsLocked = computed(() => locked.value || finished.value || !activeHero.value || activeHero.value.hp <= 0);
const enemyHealthPercent = computed(() => {
  const maxHp = Number(enemy.value?.maxHp || 0);
  const hp = Number(enemy.value?.hp || 0);
  if (!maxHp) return 0;
  return Math.max(0, Math.min(100, (hp / maxHp) * 100));
});
const recentLines = computed(() => battleLog.value.slice(-4));
const momentumPercent = computed(() => Math.min(100, comboStreak.value * 20));
const momentumTitle = computed(() => {
  if (comboStreak.value >= 4) return 'DOMINIO TÁCTICO';
  if (comboStreak.value >= 2) return 'PRESIÓN CONSTANTE';
  if (comboStreak.value >= 1) return 'INICIATIVA';
  return 'NEUTRAL';
});
const enemyIntentLabel = computed(() => {
  const hpRatio = enemy.value.maxHp > 0 ? enemy.value.hp / enemy.value.maxHp : 1;
  const intentScore = enemy.value.attack + enemyIntentBias.value;
  if (hpRatio < 0.3) return 'RÁFAGA DESESPERADA';
  if (enemy.value.accuracyPenalty > 1) return 'REORGANIZANDO ATAQUE';
  if (intentScore >= 11) return 'GOLPE BRUTAL';
  if (intentScore >= 8) return 'ATAQUE DIRECTO';
  return 'BUSCA APERTURA';
});
const EMPTY_FX_STATE = Object.freeze({ tone: '', pop: '', token: 0 });
const HERO_PORTRAITS = Object.freeze({
  archer: archerPortrait,
  warrior: warriorPortrait,
  mage: magePortrait,
  dwarf: dwarfPortrait
});
const HERO_PORTRAIT_BY_NAME = Object.freeze({
  kaelen: 'warrior',
  'elara vane': 'mage',
  'vax "dedos de hollin"': 'archer',
  'vax "soot fingers"': 'archer',
  sorin: 'dwarf',
  'lyra de l alba': 'mage',
  'lyra de l’alba': 'mage',
  'thoren ferrorscala': 'dwarf'
});

function toInt(value, fallback = 0) {
  const parsed = Number(value);
  return Number.isFinite(parsed) ? Math.floor(parsed) : fallback;
}

function clamp(value, min, max) {
  return Math.max(min, Math.min(max, value));
}

function parseDiceFaces(diceLabel = 'd8') {
  const parsed = String(diceLabel).toLowerCase().match(/d(\d+)/);
  return parsed ? Math.max(2, toInt(parsed[1], 8)) : 8;
}
function pickLine(list, fallback = '') {
  if (!Array.isArray(list) || list.length === 0) return fallback;
  return list[Math.floor(Math.random() * list.length)] || fallback;
}

function rollDice(faces) {
  return Math.floor(Math.random() * faces) + 1;
}

function shortStatus(effect) {
  const type = String(effect?.type || '').toLowerCase();
  if (type === 'bleed') return 'SANGRADO';
  if (type === 'poison') return 'VENENO';
  if (type === 'stun') return 'ATURDIDO';
  return String(effect?.name || 'ESTADO').toUpperCase();
}

function visibleStatuses(statusList, limit = 2) {
  return Array.isArray(statusList) ? statusList.slice(0, limit) : [];
}

function extraStatusCount(statusList, limit = 2) {
  return Math.max(0, (Array.isArray(statusList) ? statusList.length : 0) - limit);
}

function normalizePortraitText(value) {
  return String(value || '')
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .replace(/[^a-zA-Z0-9]+/g, ' ')
    .trim()
    .toLowerCase();
}

function heroPortraitKey(hero) {
  const normalizedName = normalizePortraitText(hero?.name);
  if (normalizedName && HERO_PORTRAIT_BY_NAME[normalizedName]) {
    return HERO_PORTRAIT_BY_NAME[normalizedName];
  }

  const rawIcon = String(hero?.icon || '');
  const haystack = [
    hero?.portraitKey,
    hero?.role,
    hero?.archetype,
    hero?.weapon,
  ]
    .filter(Boolean)
    .map(normalizePortraitText)
    .join(' ');

  if (
    /\b(archer|arquera|arquero|ranger|hunter|scout|rogue|picaro|ladron|thief|dagger|bow)\b/.test(haystack)
    || rawIcon.includes('🗡')
  ) return 'archer';

  if (
    /\b(mage|mago|wizard|healer|sanadora|cleric|oracle|mystic|arcan|scholar|sage)\b/.test(haystack)
    || rawIcon.includes('✨')
    || rawIcon.includes('⚖')
  ) return 'mage';

  if (
    /\b(dwarf|enano|guardian|guardia|runic|runico|tank|defender|shield|smith)\b/.test(haystack)
    || rawIcon.includes('🛡')
  ) return 'dwarf';

  if (
    /\b(warrior|guerrero|fighter|captain|capitan|soldier|knight|paladin|blade)\b/.test(haystack)
    || rawIcon.includes('⚔')
  ) return 'warrior';

  return 'warrior';
}

function heroPortrait(hero) {
  return HERO_PORTRAITS[heroPortraitKey(hero)] || HERO_PORTRAITS.warrior;
}

function plannerOrderNumber(heroId) {
  const idx = planningOrder.value.indexOf(heroId);
  return idx === -1 ? '' : idx + 1;
}

function compactSkillMeta(skill) {
  const type = String(skill?.type || 'attack').toLowerCase();
  const dice = String(skill?.dice || 'd8').toUpperCase();
  if (type === 'heal') return `CURA · ${dice} · ${Math.max(0, toInt(skill?.power, 0))}`;
  if (type === 'defense') return `DEFENSA · +${Math.max(3, toInt(skill?.guardBonus, 0))}`;
  if (type === 'debuff') return `DEBUFF · PREC -${Math.max(1, toInt(skill?.enemyAccuracyPenalty, 1))}`;
  return `${type.toUpperCase()} · ${dice} · PODER ${Math.max(0, toInt(skill?.power, 0))}`;
}

function heroXpPercent(hero) {
  if (!hero) return 0;
  const xp = Math.max(0, toInt(hero.experience, 0));
  const next = Math.max(1, toInt(hero.nextLevelXp, 1));
  return Math.max(0, Math.min(100, (xp / next) * 100));
}

function compactConsumableMeta(item) {
  const quantity = Math.max(1, toInt(item?.quantity, 1));
  const healMax = Math.max(0, toInt(item?.effects?.healMax, 0));
  const cleanse = Boolean(item?.effects?.cleanse);
  if (healMax > 0 && cleanse) return `x${quantity} · CURA Y LIMPIA`;
  if (healMax > 0) return `x${quantity} · CURA`;
  if (cleanse) return `x${quantity} · LIMPIA`;
  return `x${quantity}`;
}

function compactConsumableResult(message, fallbackActor, fallbackTarget) {
  const source = String(fallbackActor || 'Aliado');
  const target = String(fallbackTarget || 'objetivo');
  const match = String(message || '').match(
    /^(.+?) usa (.+?) sobre (.+?) y restaura (\d+) HP\.(?: Elimina (\d+) estado\(s\) negativo\(s\)\.)?$/i
  );
  if (!match) {
    return {
      line: `${source} usa consumible`,
      heal: 0,
      cleanse: 0,
      targetName: target
    };
  }
  const [, actorName, itemName, targetName, healAmount, cleanseCount] = match;
  const cleanseText = cleanseCount ? ` · limpia ${cleanseCount}` : '';
  return {
    line: `${actorName} usa ${itemName} · +${healAmount} a ${targetName}${cleanseText}`,
    heal: Math.max(0, toInt(healAmount, 0)),
    cleanse: Math.max(0, toInt(cleanseCount, 0)),
    targetName
  };
}

function scheduleFxReset(applyReset, duration = 760) {
  const timer = setTimeout(() => {
    fxTimers.delete(timer);
    applyReset();
  }, duration);
  fxTimers.add(timer);
}

function resetFeedbackStates() {
  enemyFx.value = { tone: '', pop: '', token: 0 };
  heroFx.value = {};
}

function triggerEnemyFx(tone, pop = '', duration = 760) {
  const token = (enemyFx.value.token || 0) + 1;
  enemyFx.value = { tone, pop, token };
  scheduleFxReset(() => {
    if (enemyFx.value.token !== token) return;
    enemyFx.value = { tone: '', pop: '', token };
  }, duration);
}

function triggerHeroFx(heroId, tone, pop = '', duration = 760) {
  if (!heroId) return;
  const currentToken = heroFx.value[heroId]?.token || 0;
  const token = currentToken + 1;
  heroFx.value = {
    ...heroFx.value,
    [heroId]: { tone, pop, token }
  };
  scheduleFxReset(() => {
    if (heroFx.value[heroId]?.token !== token) return;
    const next = { ...heroFx.value };
    delete next[heroId];
    heroFx.value = next;
  }, duration);
}

function pushHudToast(title, detail = '', type = 'info', duration = 2200) {
  const id = `toast_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`;
  hudToasts.value = [...hudToasts.value, { id, title, detail, type }];
  scheduleFxReset(() => {
    hudToasts.value = hudToasts.value.filter((toast) => toast.id !== id);
  }, duration);
}

function heroFxState(heroId) {
  return heroFx.value[heroId] || EMPTY_FX_STATE;
}

function heroFxTone(heroId) {
  return heroFxState(heroId).tone;
}

function heroFxPop(heroId) {
  return heroFxState(heroId).pop;
}

function heroFxToken(heroId) {
  return heroFxState(heroId).token;
}

function mergeStatusEffects(currentStatus, rawStatus, fallbackPrefix = 'status') {
  const current = normalizeStatusEffects(currentStatus || [], `${fallbackPrefix}_current`);
  const incoming = normalizeStatusEffects([rawStatus], `${fallbackPrefix}_incoming`);
  if (incoming.length === 0) return { list: current, changed: false, applied: null };
  const effect = incoming[0];
  const index = current.findIndex((entry) => entry.type === effect.type);
  if (index === -1) {
    return { list: [...current, effect], changed: true, applied: effect };
  }
  const existing = current[index];
  const merged = {
    ...existing,
    duration: Math.max(toInt(existing.duration, 1), toInt(effect.duration, 1)),
    potency: Math.max(toInt(existing.potency, 0), toInt(effect.potency, 0)),
    source: effect.source || existing.source
  };
  const changed = merged.duration !== existing.duration || merged.potency !== existing.potency;
  const next = [...current];
  next[index] = merged;
  return { list: next, changed, applied: changed ? merged : null };
}

function consumeStunStatus(statusList, fallbackPrefix = 'status') {
  const normalized = normalizeStatusEffects(statusList || [], `${fallbackPrefix}_consume`);
  let stunned = false;
  const next = normalized.flatMap((effect) => {
    if (effect.type !== 'stun') return [effect];
    stunned = true;
    const nextDuration = toInt(effect.duration, 1) - 1;
    if (nextDuration > 0) {
      return [{ ...effect, duration: nextDuration }];
    }
    return [];
  });
  return {
    stunned,
    nextStatus: normalizeStatusEffects(next, `${fallbackPrefix}_next`)
  };
}

function tickDamageStatuses(statusList, unitLabel, fallbackPrefix = 'status') {
  const normalized = normalizeStatusEffects(statusList || [], `${fallbackPrefix}_tick`);
  let totalDamage = 0;
  const lines = [];

  const nextStatus = normalized.flatMap((effect) => {
    if (effect.type === 'stun') return [effect];
    const currentDuration = toInt(effect.duration, 1);
    if (effect.type === 'bleed' || effect.type === 'poison') {
      const variance = effect.type === 'bleed' ? rollDice(3) - 1 : rollDice(4) - 1;
      const damage = Math.max(1, toInt(effect.potency, 2) + variance);
      totalDamage += damage;
      lines.push(
        effect.type === 'bleed'
          ? `${unitLabel} sufre ${damage} de daño por sangrado.`
          : `${unitLabel} sufre ${damage} de daño por veneno.`
      );
    }
    const nextDuration = currentDuration - 1;
    if (nextDuration > 0) return [{ ...effect, duration: nextDuration }];
    return [];
  });

  return {
    damage: totalDamage,
    lines,
    nextStatus: normalizeStatusEffects(nextStatus, `${fallbackPrefix}_next`)
  };
}

function tryApplyStatusToHero(heroId, statusConfig, sourceLabel = '') {
  if (!statusConfig || typeof statusConfig !== 'object') return;
  const chance = clamp(toInt(statusConfig.chance, 0), 0, 100);
  if (chance <= 0) return;
  const roll = rollDice(100);
  if (roll > chance) return;
  const statusType = String(statusConfig.type || '').toLowerCase();
  if (!statusType) return;

  let applied = null;
  updateHeroById(heroId, (hero) => {
    const merged = mergeStatusEffects(
      hero.statusEffects,
      {
        type: statusType,
        duration: Math.max(1, toInt(statusConfig.duration, 1)),
        potency: Math.max(0, toInt(statusConfig.potency, 0)),
        source: sourceLabel
      },
      `${hero.id}_status`
    );
    applied = merged.applied;
    return {
      ...hero,
      statusEffects: merged.list
    };
  });

  if (applied) {
    triggerHeroFx(heroId, 'fx-hit', shortStatus(applied), 900);
    pushBattleLine(`${heroName(heroId)} sufre ${shortStatus(applied)}.`);
  }
}

function tryApplyStatusToEnemy(statusConfig, sourceLabel = '') {
  if (!statusConfig || typeof statusConfig !== 'object') return;
  const chance = clamp(toInt(statusConfig.chance, 0), 0, 100);
  if (chance <= 0) return;
  const roll = rollDice(100);
  if (roll > chance) return;
  const statusType = String(statusConfig.type || '').toLowerCase();
  if (!statusType) return;

  const merged = mergeStatusEffects(
    enemy.value.statusEffects,
    {
      type: statusType,
      duration: Math.max(1, toInt(statusConfig.duration, 1)),
      potency: Math.max(0, toInt(statusConfig.potency, 0)),
      source: sourceLabel
    },
    'enemy_status'
  );
  enemy.value = {
    ...enemy.value,
    statusEffects: merged.list
  };
  if (merged.applied) {
    triggerEnemyFx('fx-debuff', shortStatus(merged.applied), 900);
    pushBattleLine(`${enemy.value.name} sufre ${shortStatus(merged.applied)}.`);
  }
}

function pushBattleLine(line) {
  if (!line) return;
  battleLog.value.push(line);
  if (battleLog.value.length > 60) battleLog.value.shift();
}

function heroName(heroId) {
  return localHeroes.value.find((hero) => hero.id === heroId)?.name || 'Aventurero';
}

function mergeTacticalState(nextList) {
  const previousById = new Map(localHeroes.value.map((hero) => [hero.id, hero]));
  return nextList.map((hero) => {
    const prev = previousById.get(hero.id);
    if (!prev) {
      return {
        ...hero,
        guarding: false,
        guardBonus: 0
      };
    }
    return {
      ...hero,
      guarding: Boolean(prev.guarding),
      guardBonus: toInt(prev.guardBonus, 0)
    };
  });
}

function normalizePartyForCombat(rawParty) {
  const normalized = normalizePartyState(rawParty);
  return mergeTacticalState(normalized);
}

function heroSnapshot() {
  return localHeroes.value.map((hero) => ({
    id: hero.id,
    name: hero.name,
    role: hero.role,
    weapon: hero.weapon,
    icon: hero.icon,
    hp: hero.hp,
    maxHp: hero.maxHp,
    level: hero.level,
    experience: hero.experience,
    nextLevelXp: hero.nextLevelXp,
    attack: hero.attack,
    defense: hero.defense,
    magic: hero.magic,
    agility: hero.agility,
    inventory: hero.inventory || [],
    equipment: hero.equipment || { weapon: null, armor: null, trinket: null },
    skills: hero.skills || [],
    statusEffects: hero.statusEffects || []
  }));
}

function emitPartySync() {
  emit('sync-party', heroSnapshot());
}

function prepareEncounter() {
  if (!props.encounter) return;
  resetFeedbackStates();
  ownsLocalPartyState.value = false;

  const normalizedParty = normalizePartyForCombat(props.party);
  localHeroes.value = normalizedParty.length > 0
    ? normalizedParty
    : normalizePartyForCombat([
      { id: 'fallback_hero', name: 'Kaelen', role: 'Guerrero', icon: '⚔️', hp: 52, maxHp: 52 }
    ]);

  const enemyType = String(props.encounter?.enemyType || 'escaramuza').toLowerCase();
  const preset = ENEMY_PRESETS[enemyType] || ENEMY_PRESETS.escaramuza;
  enemy.value = {
    name: props.encounter?.enemyName || 'Amenaza hostil',
    type: Object.prototype.hasOwnProperty.call(ENEMY_PRESETS, enemyType) ? enemyType : 'escaramuza',
    typeLabel: preset.label,
    icon: preset.icon,
    hp: preset.hp,
    maxHp: preset.hp,
    attack: preset.attack,
    defense: preset.defense,
    accuracyPenalty: 0,
    statusEffects: [],
    statusOnHit: preset.statusOnHit || null
  };

  round.value = 1;
  battleLog.value = [`${enemy.value.name} aparece.`];
  hudToasts.value = [];
  locked.value = false;
  finished.value = false;
  planningOrder.value = [];
  roundOrder.value = [];
  turnPointer.value = 0;
  isPlanningOrder.value = true;
  activeControlTab.value = 'state';
  comboStreak.value = 0;
  enemyIntentBias.value = 0;
  itemTargetHeroId.value = aliveHeroes.value[0]?.id || localHeroes.value[0]?.id || '';
  ownsLocalPartyState.value = true;

  emitPartySync();
}

function ensureValidTargetHero() {
  const alive = aliveHeroes.value;
  if (alive.length === 0) {
    itemTargetHeroId.value = '';
    return;
  }
  const exists = alive.some((hero) => hero.id === itemTargetHeroId.value);
  if (!exists) itemTargetHeroId.value = alive[0].id;
}

function setAutoOrder() {
  const ordered = [...aliveHeroes.value]
    .map((hero) => {
      const stats = computeHeroCombatStats(hero);
      return {
        id: hero.id,
        score: stats.attack + stats.defense + toInt(hero.hp / 10, 0)
      };
    })
    .sort((a, b) => b.score - a.score)
    .map((entry) => entry.id);
  planningOrder.value = ordered;
}

function toggleHeroInOrder(heroId) {
  if (planningOrder.value.includes(heroId)) {
    planningOrder.value = planningOrder.value.filter((id) => id !== heroId);
    return;
  }
  if (!aliveHeroes.value.some((hero) => hero.id === heroId)) return;
  planningOrder.value = [...planningOrder.value, heroId];
}

function confirmOrder() {
  if (planningOrder.value.length !== aliveHeroes.value.length) return;
  roundOrder.value = [...planningOrder.value];
  turnPointer.value = 0;
  isPlanningOrder.value = false;
  ensureValidTargetHero();
  pushBattleLine('Orden listo.');
  enemyIntentBias.value = rollDice(4) - 2;
}

function allHeroesDown() {
  return localHeroes.value.every((hero) => hero.hp <= 0);
}

function clearDefensiveStates() {
  localHeroes.value = localHeroes.value.map((hero) => ({
    ...hero,
    guarding: false,
    guardBonus: 0
  }));
}

function finishCombat(result) {
  if (finished.value) return;
  finished.value = true;
  ownsLocalPartyState.value = false;

  if (result === 'derrota') {
    localHeroes.value = localHeroes.value.map((hero) => ({
      ...hero,
      hp: hero.hp > 0 ? hero.hp : Math.max(1, Math.floor(hero.maxHp * 0.25)),
      guarding: false,
      guardBonus: 0
    }));
    pushBattleLine('Retirada forzada.');
  } else {
    const reward = grantExperienceToParty(localHeroes.value, enemy.value.type, round.value);
    localHeroes.value = mergeTacticalState(reward.party || localHeroes.value);
    if (reward.xpPerHero > 0) {
      pushHudToast('EXPEDICIÓN', `+${reward.xpPerHero} XP por superviviente`, 'xp', 2800);
    }
    reward.logs.forEach((entry) => {
      if (entry.xpGained > 0) {
        pushBattleLine(`${entry.heroName} +${entry.xpGained} XP`);
      }
      (entry.levelUps || []).forEach((levelEvent) => {
        pushHudToast(
          `${entry.heroName} SUBE A NV ${levelEvent.to}`,
          `ATQ +${levelEvent.gained.attack} · DEF +${levelEvent.gained.defense} · MAG +${levelEvent.gained.magic} · AGI +${levelEvent.gained.agility} · VIDA +${levelEvent.gained.maxHp}`,
          'level',
          4200
        );
        pushBattleLine(
          `${entry.heroName} sube a NV ${levelEvent.to} · ATQ +${levelEvent.gained.attack} DEF +${levelEvent.gained.defense} MAG +${levelEvent.gained.magic} AGI +${levelEvent.gained.agility} VIDA +${levelEvent.gained.maxHp}`
        );
      });
    });
    pushBattleLine('Enemigo derrotado.');
  }

  emitPartySync();
  const summaryState = heroSnapshot().map((hero) => `${hero.name} ${hero.hp}/${hero.maxHp}`).join(', ');
  const summary = result === 'victoria'
    ? `Resolución de combate: victoria contra ${enemy.value.name} (${enemy.value.typeLabel}). Rondas: ${round.value}. Estado del grupo: ${summaryState}.`
    : `Resolución de combate: retirada tras derrota contra ${enemy.value.name} (${enemy.value.typeLabel}). Rondas: ${round.value}. Estado del grupo: ${summaryState}.`;

  emit('end', {
    result,
    summary,
    partySnapshot: heroSnapshot(),
    transcript: battleLog.value.slice(-12)
  });
}

function resolveHit(attackValue, targetDefense, bonus = 0) {
  const die = rollDice(20);
  const total = die + attackValue + bonus;
  const hit = die === 20 || total >= targetDefense;
  return { die, total, hit };
}

function damageRoll(diceLabel, power, attackValue) {
  const faces = parseDiceFaces(diceLabel);
  return rollDice(faces) + Math.max(0, toInt(power, 0)) + Math.floor(Math.max(0, attackValue) / 2);
}

function updateHeroById(heroId, updater) {
  const idx = localHeroes.value.findIndex((hero) => hero.id === heroId);
  if (idx === -1) return;
  const updated = updater({ ...localHeroes.value[idx] });
  localHeroes.value[idx] = updated;
}

function applyDamageToEnemy(amount) {
  enemy.value = {
    ...enemy.value,
    hp: clamp(enemy.value.hp - Math.max(0, amount), 0, enemy.value.maxHp)
  };
}

function applyHealToHero(heroId, amount) {
  updateHeroById(heroId, (hero) => ({
    ...hero,
    hp: clamp(hero.hp + Math.max(0, amount), 0, hero.maxHp)
  }));
}

function applyEnemyTurn() {
  const targets = aliveHeroes.value;
  if (targets.length === 0) {
    finishCombat('derrota');
    return;
  }

  const target = targets[Math.floor(Math.random() * targets.length)];
  const targetStats = computeHeroCombatStats(target);
  const effectiveDefense = 11 + targetStats.defense + toInt(target.guardBonus, 0);
  const enemyAccuracyPenalty = Math.max(0, toInt(enemy.value.accuracyPenalty, 0));

  const hitCheck = resolveHit(enemy.value.attack, effectiveDefense, -enemyAccuracyPenalty);

  if (hitCheck.hit) {
    const base = damageRoll('d8', 0, enemy.value.attack);
    const mitigated = target.guarding ? Math.floor(base * 0.58) : base;
    const damage = Math.max(1, mitigated);
    updateHeroById(target.id, (hero) => ({ ...hero, hp: Math.max(0, hero.hp - damage) }));
    triggerEnemyFx('fx-strike', '', 480);
    triggerHeroFx(target.id, target.guarding ? 'fx-guard' : 'fx-hit', `-${damage}`, 880);
    pushBattleLine(pickLine([
      `${enemy.value.name} golpea a ${target.name} · ${damage}`,
      `${enemy.value.name} rompe la guardia de ${target.name} · ${damage}`,
      `${target.name} recibe un impacto feroz · ${damage}`
    ], `${enemy.value.name} golpea a ${target.name} · ${damage}`));
    comboStreak.value = 0;
  } else {
    triggerEnemyFx('fx-miss', 'FALLO', 680);
    triggerHeroFx(target.id, 'fx-miss', 'ESQUIVA', 760);
    pushBattleLine(pickLine([
      `${enemy.value.name} falla.`,
      `${target.name} esquiva por poco.`,
      `El ataque enemigo se pierde en la oscuridad.`
    ], `${enemy.value.name} falla.`));
  }

  enemy.value = {
    ...enemy.value,
    accuracyPenalty: Math.max(0, enemyAccuracyPenalty - 1)
  };

  clearDefensiveStates();
  emitPartySync();

  if (allHeroesDown()) {
    finishCombat('derrota');
    return;
  }

  round.value += 1;
  planningOrder.value = [];
  roundOrder.value = [];
  turnPointer.value = 0;
  isPlanningOrder.value = true;
  enemyIntentBias.value = rollDice(6) - 3;
  ensureValidTargetHero();
  pushBattleLine('Nueva ronda.');
}

function finalizeHeroAction() {
  emitPartySync();

  if (enemy.value.hp <= 0) {
    finishCombat('victoria');
    locked.value = false;
    return;
  }

  const nextPointer = turnPointer.value + 1;
  if (nextPointer >= roundOrder.value.length) {
    applyEnemyTurn();
    locked.value = false;
    return;
  }

  turnPointer.value = nextPointer;
  ensureValidTargetHero();
  locked.value = false;
}

function performBasicAttack() {
  if (controlsLocked.value) return;
  const actor = activeHero.value;
  if (!actor) return;

  locked.value = true;
  const actorStats = computeHeroCombatStats(actor);
  const targetDefense = 11 + enemy.value.defense;
  const hitCheck = resolveHit(actorStats.attack, targetDefense, 0);

  if (hitCheck.hit) {
    const damage = damageRoll('d8', 0, actorStats.attack);
    const finisherBonus = comboStreak.value >= 3 ? rollDice(6) : 0;
    const totalDamage = damage + finisherBonus;
    applyDamageToEnemy(totalDamage);
    triggerHeroFx(actor.id, 'fx-cast', 'ATAQUE', 420);
    triggerEnemyFx('fx-hit', `-${totalDamage}`, 880);
    if (finisherBonus > 0) {
      pushHudToast('RÁFAGA', `Daño extra +${finisherBonus}`, 'level', 1800);
    }
    comboStreak.value += 1;
    pushBattleLine(pickLine([
      `${actor.name} golpea por ${totalDamage}.`,
      `${actor.name} conecta un corte limpio · ${totalDamage}.`,
      `${actor.name} mantiene la presión · ${totalDamage}.`
    ], `${actor.name} golpea por ${totalDamage}.`));
  } else {
    triggerHeroFx(actor.id, 'fx-cast', 'ATAQUE', 420);
    triggerEnemyFx('fx-miss', 'FALLO', 680);
    comboStreak.value = 0;
    pushBattleLine(pickLine([
      `${actor.name} falla.`,
      `${actor.name} pierde el tempo del asalto.`,
      `El ataque de ${actor.name} no encuentra hueco.`
    ], `${actor.name} falla.`));
  }

  finalizeHeroAction();
}

function performDefend() {
  if (controlsLocked.value) return;
  const actor = activeHero.value;
  if (!actor) return;

  locked.value = true;
  updateHeroById(actor.id, (hero) => ({
    ...hero,
    guarding: true,
    guardBonus: Math.max(toInt(hero.guardBonus, 0), 3)
  }));
  triggerHeroFx(actor.id, 'fx-guard', 'GUARDIA', 900);
  comboStreak.value = Math.max(0, comboStreak.value - 1);
  pushBattleLine(`${actor.name} entra en guardia.`);
  finalizeHeroAction();
}

function performSkill(skillId) {
  if (controlsLocked.value) return;
  const actor = activeHero.value;
  if (!actor) return;

  const skill = (actor.skills || []).find((entry) => entry.id === skillId);
  if (!skill) {
    pushBattleLine('Habilidad no disponible.');
    return;
  }

  locked.value = true;
  const actorStats = computeHeroCombatStats(actor);

  if (skill.type === 'heal') {
    const targetId = itemTargetHeroId.value || actor.id;
    const healValue = damageRoll(skill.dice, skill.power + actorStats.healPower, 0);
    applyHealToHero(targetId, healValue);
    triggerHeroFx(targetId, 'fx-heal', `+${healValue}`, 900);
    comboStreak.value = Math.max(0, comboStreak.value - 1);
    pushBattleLine(`${actor.name} cura ${healValue} a ${heroName(targetId)}.`);
    finalizeHeroAction();
    return;
  }

  if (skill.type === 'defense') {
    const bonus = Math.max(3, toInt(skill.guardBonus, 0));
    updateHeroById(actor.id, (hero) => ({
      ...hero,
      guarding: true,
      guardBonus: Math.max(toInt(hero.guardBonus, 0), bonus)
    }));
    triggerHeroFx(actor.id, 'fx-guard', '+DEF', 900);
    comboStreak.value = Math.max(0, comboStreak.value - 1);
    pushBattleLine(`${actor.name} refuerza defensa.`);
    finalizeHeroAction();
    return;
  }

  if (skill.type === 'debuff') {
    const penalty = Math.max(1, toInt(skill.enemyAccuracyPenalty, 1));
    enemy.value = {
      ...enemy.value,
      accuracyPenalty: Math.max(enemy.value.accuracyPenalty, penalty)
    };
    triggerHeroFx(actor.id, 'fx-cast', 'RITUAL', 520);
    triggerEnemyFx('fx-debuff', `PREC-${penalty}`, 900);
    comboStreak.value += 1;
    pushBattleLine(`${actor.name} baja la precision enemiga.`);
    finalizeHeroAction();
    return;
  }

  const targetDefense = 11 + enemy.value.defense;
  const hitBonus = toInt(skill.accuracy, 0) + actorStats.skillAccuracy;
  const hitCheck = resolveHit(actorStats.attack, targetDefense, hitBonus);

  if (hitCheck.hit) {
    let damage = damageRoll(skill.dice, skill.power, actorStats.attack);
    if (hitCheck.die === 20 || (toInt(skill.critBoost, 0) > 0 && hitCheck.die >= 20 - toInt(skill.critBoost, 0))) {
      damage += rollDice(8);
    }
    const chainBonus = comboStreak.value >= 2 ? rollDice(4) : 0;
    const totalDamage = damage + chainBonus;
    applyDamageToEnemy(totalDamage);
    triggerHeroFx(actor.id, 'fx-cast', 'HABIL', 520);
    triggerEnemyFx('fx-hit', `-${totalDamage}`, 900);
    comboStreak.value += 1;
    pushBattleLine(`${actor.name} usa ${skill.name} · ${totalDamage}.`);
  } else {
    triggerHeroFx(actor.id, 'fx-cast', 'HABIL', 520);
    triggerEnemyFx('fx-miss', 'FALLO', 680);
    comboStreak.value = 0;
    pushBattleLine(`${actor.name} falla ${skill.name}.`);
  }

  finalizeHeroAction();
}

function performConsumable(itemId) {
  if (controlsLocked.value) return;
  const actor = activeHero.value;
  if (!actor) return;

  const targetId = itemTargetHeroId.value || actor.id;
  const result = useHeroConsumable(localHeroes.value, actor.id, itemId, targetId);
  if (result?.error) {
    pushBattleLine(result.error);
    return;
  }

  locked.value = true;
  localHeroes.value = mergeTacticalState(result.party || localHeroes.value);
  if (result?.message) {
    const feedback = compactConsumableResult(result.message, actor.name, heroName(targetId));
    pushBattleLine(feedback.line);
    if (feedback.heal > 0) {
      triggerHeroFx(targetId, 'fx-heal', `+${feedback.heal}`, 900);
      comboStreak.value = Math.max(0, comboStreak.value - 1);
    } else if (feedback.cleanse > 0) {
      triggerHeroFx(targetId, 'fx-heal', 'LIMPIA', 900);
      comboStreak.value = Math.max(0, comboStreak.value - 1);
    } else {
      triggerHeroFx(targetId, 'fx-cast', 'OBJ', 760);
    }
  }
  finalizeHeroAction();
}

onBeforeUnmount(() => {
  fxTimers.forEach((timer) => clearTimeout(timer));
  fxTimers.clear();
});

watch(
  () => props.encounter,
  (encounter) => {
    if (encounter) {
      prepareEncounter();
      return;
    }
    ownsLocalPartyState.value = false;
  },
  { immediate: true, deep: true }
);

watch(
  () => props.party,
  (partyValue) => {
    if (!props.encounter || finished.value || ownsLocalPartyState.value) return;
    const incoming = normalizePartyForCombat(partyValue);
    if (incoming.length === 0) return;
    localHeroes.value = incoming;
    ensureValidTargetHero();
  },
  { deep: true }
);

watch(
  aliveHeroes,
  (alive) => {
    if (!alive || alive.length === 0) return;
    if (planningOrder.value.length > alive.length) {
      planningOrder.value = planningOrder.value.filter((id) => alive.some((hero) => hero.id === id));
    }
    ensureValidTargetHero();
  },
  { deep: true }
);
</script>

<style scoped lang="scss">
$gold: #c5a059;
$crimson: #8a1c1c;

.tactical-combat-panel {
  --font-display: 'Cinzel', serif;
  --font-ui: 'Inter', 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --color-text: #f1e6c7;
  --color-text-soft: #c7bc9e;
  --color-text-dim: #998f76;
  --color-accent: #d8b66d;
  --color-accent-strong: #f1d8a5;
  --combat-text-micro: 0.58rem;
  --combat-text-xs: 0.7rem;
  --combat-text-sm: 0.82rem;
  --combat-text-md: 0.96rem;
  --combat-text-lg: 1.16rem;
  --combat-text-xl: 1.7rem;
  --combat-icon-sm: 1.28rem;
  --combat-icon-md: 1.55rem;
  --combat-icon-lg: clamp(3.7rem, 7.6vw, 5.1rem);
  display: grid;
  gap: 14px;
  position: relative;
  overflow: auto;
  border: 1px solid rgba($gold, 0.35);
  border-radius: 12px;
  background:
    radial-gradient(circle at top, rgba(93, 55, 33, 0.18), transparent 34%),
    linear-gradient(180deg, rgba(7, 7, 7, 0.98), rgba(7, 7, 7, 0.88));
  padding: 16px;
  margin: 0;
  min-height: 0;
  height: 100%;
  box-shadow: 0 26px 44px rgba(0, 0, 0, 0.42), inset 0 1px 0 rgba(255, 255, 255, 0.04);
  font-family: var(--font-ui);
  color: var(--color-text);
  line-height: 1.35;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  scrollbar-width: thin;
  scrollbar-color: rgba($gold, 0.38) rgba(0, 0, 0, 0.16);
  &::-webkit-scrollbar { width: 8px; }
  &::-webkit-scrollbar-track { background: rgba(0, 0, 0, 0.18); }
  &::-webkit-scrollbar-thumb { background: rgba($gold, 0.36); border-radius: 999px; }
}
.combat-head {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 14px;
  flex-wrap: wrap;
}
.combat-title {
  display: grid;
  gap: 4px;
}
.eyebrow {
  color: var(--color-accent);
  font-size: var(--combat-text-xs);
  letter-spacing: 2.2px;
  text-transform: uppercase;
  font-family: var(--font-display);
}
.combat-title h3 {
  margin: 0;
  color: var(--color-accent-strong);
  letter-spacing: 0.6px;
  font-size: clamp(1.35rem, 2.8vw, 1.9rem);
  line-height: 1.04;
  font-family: var(--font-display);
  text-shadow: 0 1px 10px rgba(0, 0, 0, 0.45);
}
.combat-head-status {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}
.combat-atmosphere {
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 12px;
  border: 1px solid rgba($gold, 0.2);
  border-radius: 10px;
  padding: 12px 14px;
  background:
    linear-gradient(120deg, rgba(40, 26, 10, 0.22), transparent 56%),
    rgba(8, 8, 10, 0.5);
}
.atmosphere-left,
.atmosphere-right {
  display: grid;
  gap: 4px;
}
.atmo-label {
  font-size: 0.56rem;
  letter-spacing: 1px;
  color: var(--color-text-dim);
  font-family: var(--font-display);
}
.atmosphere-left strong,
.atmosphere-right strong {
  color: var(--color-accent-strong);
  font-size: 0.72rem;
  letter-spacing: 0.8px;
  font-family: var(--font-display);
}
.momentum-bar {
  width: min(320px, 100%);
  height: 7px;
  border-radius: 999px;
  border: 1px solid rgba($gold, 0.26);
  background: rgba(0, 0, 0, 0.7);
  overflow: hidden;
}
.momentum-fill {
  height: 100%;
  background: linear-gradient(90deg, #8f4f14, #cd8f40, #f0d39e);
  transition: width 0.28s ease;
}
.hud-toast-stack {
  display: grid;
  gap: 8px;
  pointer-events: none;
}
.hud-toast {
  border-radius: 11px;
  border: 1px solid rgba($gold, 0.32);
  background: linear-gradient(140deg, rgba(16, 16, 24, 0.94), rgba(10, 10, 14, 0.9));
  color: #e6c98f;
  padding: 8px 11px;
  display: grid;
  gap: 2px;
  box-shadow: 0 12px 24px rgba(0, 0, 0, 0.32);
}
.hud-toast strong {
  font-size: 0.68rem;
  letter-spacing: 0.9px;
}
.hud-toast small {
  font-size: 0.6rem;
  color: var(--color-text-soft);
}
.hud-toast.xp {
  border-color: rgba(88, 137, 224, 0.52);
  color: #b8d2ff;
}
.hud-toast.level {
  border-color: rgba($gold, 0.75);
  color: #f4deae;
  box-shadow: 0 14px 28px rgba(0, 0, 0, 0.35), 0 0 22px rgba($gold, 0.16);
}
.hud-toast-enter-active,
.hud-toast-leave-active {
  transition: all 0.32s ease;
}
.hud-toast-enter-from,
.hud-toast-leave-to {
  opacity: 0;
  transform: translateY(-8px) scale(0.98);
}
.hud-toast-move {
  transition: transform 0.32s ease;
}
.head-chip {
  border: 1px solid rgba($gold, 0.25);
  border-radius: 999px;
  background: rgba(197, 160, 89, 0.08);
  color: var(--color-accent-strong);
  padding: 5px 10px;
  font-size: 0.62rem;
  letter-spacing: 0.95px;
  text-transform: uppercase;
  font-family: var(--font-display);
}
.head-chip.danger {
  border-color: rgba($crimson, 0.4);
  background: rgba($crimson, 0.12);
  color: #f2b2b2;
}
.head-chip.accent {
  border-color: rgba($gold, 0.38);
  color: #f4ddac;
}
.order-planner {
  display: grid;
  gap: 12px;
  border: 1px solid rgba($gold, 0.2);
  background:
    linear-gradient(180deg, rgba(28, 24, 17, 0.34), rgba(0, 0, 0, 0.32)),
    rgba(0, 0, 0, 0.28);
  border-radius: 10px;
  padding: 12px;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.04);
}
.planner-topline {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
}
.planner-topline strong {
  color: #efd7a4;
  font-size: 0.92rem;
  letter-spacing: 1px;
  text-transform: uppercase;
}
.planner-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(132px, 1fr));
  gap: 8px;
}

%hud-action-button {
  appearance: none;
  border-radius: 12px;
  border: 1px solid rgba($gold, 0.28);
  background:
    linear-gradient(180deg, rgba(255, 255, 255, 0.04), transparent 34%),
    linear-gradient(170deg, rgba(18, 18, 26, 0.96), rgba(9, 9, 15, 0.92));
  color: #dcc086;
  cursor: pointer;
  font-family: 'Cinzel', serif;
  font-weight: 700;
  letter-spacing: 1.1px;
  text-transform: uppercase;
  position: relative;
  overflow: hidden;
  box-shadow:
    0 12px 22px rgba(0, 0, 0, 0.34),
    inset 0 1px 0 rgba(255, 255, 255, 0.06),
    inset 0 -10px 18px rgba(0, 0, 0, 0.18);
  transition:
    transform 0.26s cubic-bezier(.2, .8, .2, 1),
    border-color 0.26s ease,
    color 0.26s ease,
    box-shadow 0.26s ease,
    background 0.26s ease,
    opacity 0.2s ease;

  &::before {
    content: '';
    position: absolute;
    inset: 0;
    background: linear-gradient(120deg, transparent 0%, rgba(255, 255, 255, 0.14) 45%, transparent 74%);
    transform: translateX(-135%);
    transition: transform 0.65s ease;
    pointer-events: none;
  }

  &:not(:disabled):hover {
    transform: translateY(-2px);
    border-color: rgba($gold, 0.76);
    color: #f1d8a5;
    background:
      linear-gradient(180deg, rgba(255, 255, 255, 0.06), transparent 34%),
      linear-gradient(170deg, rgba(32, 25, 12, 0.82), rgba(14, 14, 20, 0.94));
    box-shadow:
      0 18px 30px rgba(0, 0, 0, 0.42),
      0 0 22px rgba($gold, 0.18),
      inset 0 1px 0 rgba(255, 255, 255, 0.07);
  }

  &:not(:disabled):hover::before {
    transform: translateX(145%);
  }

  &:not(:disabled):active {
    transform: translateY(0) scale(0.99);
    box-shadow:
      0 8px 18px rgba(0, 0, 0, 0.32),
      inset 0 2px 8px rgba(0, 0, 0, 0.3);
  }

  &:disabled {
    cursor: not-allowed;
    opacity: 0.46;
    color: rgba(220, 192, 134, 0.78);
    border-color: rgba($gold, 0.18);
    background:
      linear-gradient(180deg, rgba(255, 255, 255, 0.02), transparent 30%),
      linear-gradient(170deg, rgba(16, 16, 20, 0.88), rgba(9, 9, 12, 0.84));
    box-shadow:
      0 8px 16px rgba(0, 0, 0, 0.24),
      inset 0 1px 0 rgba(255, 255, 255, 0.03);
  }
}

.planner-hero {
  @extend %hud-action-button;
  min-height: 72px;
  padding: 12px 14px;
  display: flex;
  align-items: center;
  gap: 12px;
  justify-content: flex-start;
  text-align: left;
  font-size: 0.78rem;
  &.selected {
    border-color: rgba($gold, 0.9);
    color: #f4ddac;
    background:
      linear-gradient(180deg, rgba(255, 255, 255, 0.07), transparent 34%),
      linear-gradient(170deg, rgba(48, 36, 16, 0.86), rgba(14, 14, 20, 0.94));
    box-shadow:
      0 18px 30px rgba(0, 0, 0, 0.42),
      0 0 18px rgba($gold, 0.2),
      inset 0 1px 0 rgba(255, 255, 255, 0.08);
  }
}
.planner-icon,
.ally-icon,
.actor-icon {
  position: relative;
  overflow: hidden;
  display: block;
  flex-shrink: 0;
  border: 1px solid rgba($gold, 0.3);
  background:
    linear-gradient(180deg, rgba(255, 255, 255, 0.06), transparent 34%),
    linear-gradient(170deg, rgba(18, 18, 24, 0.96), rgba(10, 10, 14, 0.9));
  box-shadow:
    0 12px 22px rgba(0, 0, 0, 0.28),
    inset 0 1px 0 rgba(255, 255, 255, 0.05);
  isolation: isolate;

  img {
    width: 100%;
    height: 100%;
    display: block;
    object-fit: cover;
    object-position: center top;
    filter: saturate(1.04) contrast(1.02);
  }

  &::after {
    content: '';
    position: absolute;
    inset: 0;
    background:
      linear-gradient(180deg, rgba(255, 255, 255, 0.1), transparent 34%),
      linear-gradient(0deg, rgba(0, 0, 0, 0.32), transparent 54%);
    pointer-events: none;
  }
}
.portrait-glyph {
  position: absolute;
  right: -3px;
  bottom: -3px;
  width: 18px;
  height: 18px;
  display: grid;
  place-items: center;
  border-radius: 999px;
  border: 1px solid rgba($gold, 0.26);
  background: rgba(8, 8, 12, 0.92);
  color: #f0d8a4;
  font-size: 0.62rem;
  line-height: 1;
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.28);
  z-index: 1;
}
.planner-icon {
  width: 54px;
  height: 66px;
  border-radius: 14px;
}
.planner-name {
  flex: 1;
  min-width: 0;
  font-size: 0.82rem;
  line-height: 1.15;
  color: inherit;
}
.planner-rank {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  display: grid;
  place-items: center;
  background: rgba($gold, 0.16);
  border: 1px solid rgba($gold, 0.28);
  color: #f0d8a4;
  font-size: 0.78rem;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.06);
}
.planner-sequence {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  span {
    border: 1px solid rgba($gold, 0.2);
    border-radius: 999px;
    background: rgba($gold, 0.06);
    color: #d5b477;
    font-size: 0.62rem;
    padding: 5px 9px;
  }
  b { color: #f1d8a5; margin-right: 4px; }
}
.planner-actions {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}
.planner-btn {
  @extend %hud-action-button;
  min-height: 42px;
  padding: 10px 14px;
  font-size: 0.7rem;

  &.minor {
    padding-inline: 12px;
    font-size: 0.64rem;
    letter-spacing: 0.9px;
    border-color: rgba($gold, 0.24);
    color: #d2b47b;
    background:
      linear-gradient(180deg, rgba(255, 255, 255, 0.03), transparent 34%),
      linear-gradient(170deg, rgba(15, 15, 22, 0.94), rgba(8, 8, 13, 0.9));
  }
}
.duel-stage {
  display: grid;
  gap: 12px;
  grid-template-columns: 1fr;
  align-items: stretch;
}
.enemy-stage {
  position: relative;
  overflow: hidden;
  border: 1px solid rgba($crimson, 0.44);
  border-radius: 10px;
  padding: 16px;
  background: linear-gradient(180deg, rgba(27, 7, 7, 0.94), rgba(12, 7, 10, 0.92));
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.03), 0 22px 36px rgba(0, 0, 0, 0.35);
  min-height: 0;
  transition: transform 0.24s ease, border-color 0.24s ease, box-shadow 0.24s ease, filter 0.24s ease;
}
.enemy-stage::before {
  content: '';
  position: absolute;
  inset: 0;
  background:
    radial-gradient(circle at 50% 18%, rgba(255, 255, 255, 0.06), transparent 24%),
    linear-gradient(180deg, rgba(122, 20, 20, 0.14), transparent 42%);
  pointer-events: none;
}
.enemy-stage::after,
.ally-card::after {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  opacity: 0;
  pointer-events: none;
}
.combat-pop {
  position: absolute;
  z-index: 2;
  pointer-events: none;
  border-radius: 999px;
  border: 1px solid rgba($gold, 0.22);
  background: rgba(8, 8, 12, 0.82);
  color: #f2dfb3;
  padding: 5px 8px;
  font-family: 'Cinzel', serif;
  font-size: 0.6rem;
  letter-spacing: 0.95px;
  text-transform: uppercase;
  animation: popRise 0.76s ease forwards;
}
.enemy-pop {
  top: 10px;
  right: 10px;
}
.ally-pop {
  top: 6px;
  right: 6px;
  font-size: 0.55rem;
  padding: 4px 7px;
}
.enemy-stage.elite {
  border-color: rgba(176, 114, 65, 0.48);
}
.enemy-stage.jefe {
  border-color: rgba(162, 68, 68, 0.62);
}
.enemy-aura {
  position: absolute;
  inset: auto 8% -18% 8%;
  height: 72%;
  background: radial-gradient(circle, rgba(171, 42, 42, 0.34), transparent 70%);
  filter: blur(28px);
  opacity: 0.88;
  pointer-events: none;
}
.enemy-stage.elite .enemy-aura {
  background: radial-gradient(circle, rgba(208, 127, 63, 0.28), transparent 70%);
}
.enemy-stage.jefe .enemy-aura {
  background: radial-gradient(circle, rgba(138, 28, 28, 0.4), transparent 72%);
}
.enemy-stage-top,
.enemy-health,
.enemy-status-row {
  position: relative;
  z-index: 1;
}
.enemy-stage-top {
  display: grid;
  justify-items: center;
  gap: 8px;
  text-align: center;
}
.enemy-icon {
  font-size: var(--combat-icon-lg);
  line-height: 1;
  text-shadow: 0 10px 26px rgba(0, 0, 0, 0.38);
}
.enemy-stage.fx-hit {
  border-color: rgba(255, 129, 129, 0.72);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.04), 0 24px 42px rgba(0, 0, 0, 0.42), 0 0 24px rgba(183, 53, 53, 0.18);
  animation: enemyRecoil 0.46s ease;
}
.enemy-stage.fx-hit::after {
  background: radial-gradient(circle at center, rgba(255, 104, 104, 0.34), transparent 64%);
  animation: flashBurst 0.48s ease;
}
.enemy-stage.fx-miss::after {
  background: linear-gradient(135deg, rgba(229, 203, 138, 0.18), transparent 60%);
  animation: flashBurst 0.52s ease;
}
.enemy-stage.fx-debuff {
  border-color: rgba(150, 121, 230, 0.56);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.04), 0 22px 36px rgba(0, 0, 0, 0.35), 0 0 20px rgba(120, 97, 212, 0.14);
}
.enemy-stage.fx-debuff::after {
  background: radial-gradient(circle at center, rgba(118, 90, 212, 0.28), transparent 64%);
  animation: flashBurst 0.6s ease;
}
.enemy-stage.fx-strike {
  transform: scale(1.01);
}
.enemy-stage.fx-strike .enemy-icon {
  animation: enemyLunge 0.42s ease;
}
.enemy-stage.fx-miss .enemy-icon,
.enemy-stage.fx-debuff .enemy-icon {
  animation: iconJolt 0.48s ease;
}
.enemy-pop.fx-hit,
.ally-pop.fx-hit {
  color: #ffb6b6;
  border-color: rgba(255, 120, 120, 0.34);
}
.enemy-pop.fx-heal,
.ally-pop.fx-heal {
  color: #c9f1d0;
  border-color: rgba(113, 201, 134, 0.32);
}
.enemy-pop.fx-guard,
.ally-pop.fx-guard {
  color: #cae0ff;
  border-color: rgba(120, 170, 235, 0.34);
}
.enemy-pop.fx-cast,
.ally-pop.fx-cast,
.enemy-pop.fx-debuff,
.ally-pop.fx-debuff,
.enemy-pop.fx-miss,
.ally-pop.fx-miss {
  color: #ead7a3;
}
.enemy-identity {
  display: grid;
  gap: 4px;
}
.enemy-identity small {
  color: #d8a5a5;
  font-size: 0.78rem;
  letter-spacing: 1.8px;
  text-transform: uppercase;
}
.enemy-identity strong {
  color: #f7cccc;
  font-size: clamp(1.55rem, 3.2vw, 2rem);
  letter-spacing: 0.2px;
  line-height: 1.02;
}
.enemy-health {
  display: grid;
  gap: 6px;
  margin-top: 12px;
}
.enemy-health-bar {
  width: 100%;
  height: 13px;
  background: rgba(0, 0, 0, 0.7);
  border: 1px solid rgba($crimson, 0.34);
  border-radius: 999px;
  overflow: hidden;
  box-shadow: inset 0 2px 10px rgba(0, 0, 0, 0.42);
}
.enemy-health-fill {
  height: 100%;
  background: linear-gradient(90deg, #7f1818, #df4747 58%, #ff8c8c);
}
.enemy-health-meta {
  display: flex;
  justify-content: space-between;
  gap: 10px;
  flex-wrap: wrap;
  color: #e8c8c8;
  font-size: 0.76rem;
  font-family: var(--font-ui);
}
.enemy-hp-numeric {
  color: #ffe2e2;
  font-weight: 700;
  letter-spacing: 0.35px;
}
.status-row {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}
.enemy-status-row {
  justify-content: center;
}
.status-pill {
  border-radius: 999px;
  padding: 5px 8px;
  border: 1px solid rgba($gold, 0.18);
  background: rgba(255, 255, 255, 0.04);
  color: #f5e0b6;
  font-size: 0.64rem;
  letter-spacing: 0.85px;
  text-transform: uppercase;
  font-family: var(--font-display);
}
.status-pill.is-bleed {
  border-color: rgba(208, 72, 72, 0.42);
  color: #ffb0b0;
  background: rgba(138, 28, 28, 0.16);
}
.status-pill.is-poison {
  border-color: rgba(91, 168, 111, 0.35);
  color: #b8e4c4;
  background: rgba(50, 96, 56, 0.16);
}
.status-pill.is-stun {
  border-color: rgba(130, 117, 204, 0.38);
  color: #d9d0ff;
  background: rgba(65, 55, 122, 0.16);
}
.status-pill.extra {
  border-color: rgba($gold, 0.22);
  color: #ddc289;
}
.ally-rack {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(110px, 1fr));
  gap: 12px;
  min-height: 0;
}
.ally-card {
  position: relative;
  border: 1px solid rgba($gold, 0.18);
  border-radius: 8px;
  background:
    linear-gradient(180deg, rgba(34, 31, 23, 0.18), transparent 34%),
    rgba(10, 10, 10, 0.9);
  padding: 12px;
  display: grid;
  grid-template-columns: 58px minmax(0, 1fr);
  column-gap: 10px;
  row-gap: 5px;
  min-height: 0;
  transition: transform 0.24s ease, border-color 0.24s ease, box-shadow 0.24s ease, opacity 0.24s ease;
  &.active {
    border-color: rgba($gold, 0.84);
    transform: translateY(-2px);
    background:
      linear-gradient(180deg, rgba(78, 56, 19, 0.26), transparent 38%),
      rgba(13, 13, 13, 0.94);
    box-shadow: 0 16px 28px rgba(0, 0, 0, 0.32), 0 0 16px rgba($gold, 0.16);
  }
  &.dead {
    opacity: 0.42;
    filter: grayscale(1);
  }
  &.ordered {
    box-shadow: inset 0 0 0 1px rgba($gold, 0.14);
  }
  &.fx-hit {
    border-color: rgba(225, 101, 101, 0.6);
    box-shadow: 0 14px 24px rgba(0, 0, 0, 0.34), 0 0 18px rgba(176, 58, 58, 0.14);
    animation: heroImpact 0.48s ease;
  }
  &.fx-hit::after {
    background: linear-gradient(180deg, rgba(182, 62, 62, 0.28), transparent 70%);
    animation: flashBurst 0.48s ease;
  }
  &.fx-heal {
    border-color: rgba(92, 184, 120, 0.54);
    box-shadow: 0 14px 24px rgba(0, 0, 0, 0.32), 0 0 18px rgba(84, 176, 117, 0.14);
  }
  &.fx-heal::after {
    background: radial-gradient(circle at center, rgba(98, 203, 128, 0.26), transparent 68%);
    animation: flashBurst 0.6s ease;
  }
  &.fx-guard {
    border-color: rgba(109, 163, 226, 0.56);
    box-shadow: 0 14px 24px rgba(0, 0, 0, 0.32), 0 0 18px rgba(90, 144, 220, 0.14);
  }
  &.fx-guard::after {
    background: radial-gradient(circle at center, rgba(109, 163, 226, 0.24), transparent 68%);
    animation: flashBurst 0.56s ease;
  }
  &.fx-miss {
    border-color: rgba(214, 186, 118, 0.52);
  }
  &.fx-miss::after,
  &.fx-cast::after {
    background: linear-gradient(135deg, rgba(233, 208, 147, 0.2), transparent 66%);
    animation: flashBurst 0.48s ease;
  }
  &.fx-cast {
    border-color: rgba(211, 177, 104, 0.48);
  }
}
.ally-top {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  align-items: center;
  grid-row: 1 / span 4;
  min-height: 100%;
}
.ally-icon {
  width: 58px;
  height: 72px;
  border-radius: 12px;
}
.ally-card.fx-hit .ally-icon,
.ally-card.fx-miss .ally-icon {
  animation: iconJolt 0.46s ease;
}
.ally-card.fx-heal .ally-icon,
.ally-card.fx-guard .ally-icon,
.ally-card.fx-cast .ally-icon {
  animation: iconLift 0.52s ease;
}
.ally-order {
  border-radius: 999px;
  border: 1px solid rgba($gold, 0.22);
  background: rgba($gold, 0.08);
  padding: 3px 7px;
  color: #e4c98e;
  font-size: 0.6rem;
}
.ally-card strong {
  color: #f0dcba;
  font-size: 0.9rem;
  letter-spacing: 0.15px;
  line-height: 1.1;
  align-self: end;
  font-family: var(--font-display);
}
.ally-hp-bar {
  width: 100%;
  height: 7px;
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.68);
  border: 1px solid rgba($gold, 0.18);
  overflow: hidden;
}
.ally-hp-fill {
  height: 100%;
  background: linear-gradient(90deg, #a03a3a, #ff7979);
}
.ally-hp-text {
  color: #c0b393;
  font-size: 0.66rem;
  font-weight: 600;
}
.ally-level {
  color: var(--color-accent-strong);
  font-size: 0.67rem;
  letter-spacing: 0.8px;
  font-weight: 700;
  font-family: var(--font-display);
}
.ally-xp-bar {
  width: 100%;
  height: 6px;
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.65);
  border: 1px solid rgba(84, 132, 224, 0.35);
  overflow: hidden;
}
.ally-xp-fill {
  height: 100%;
  background: linear-gradient(90deg, #2b5eb9, #6eb1ff);
}
.ally-xp-text {
  color: #adc5f1;
  font-size: 0.64rem;
  font-weight: 600;
}
.ally-attr-row {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 4px;
}
.ally-attr-row span {
  border-radius: 7px;
  border: 1px solid rgba($gold, 0.14);
  background: rgba($gold, 0.04);
  color: #e2cea3;
  font-size: 0.6rem;
  padding: 3px 5px;
  letter-spacing: 0.4px;
  font-weight: 600;
}
.ally-flags {
  display: flex;
  flex-wrap: wrap;
  gap: 5px;
}
.mini-flag {
  border-radius: 999px;
  padding: 3px 6px;
  font-size: 0.6rem;
  letter-spacing: 0.75px;
  text-transform: uppercase;
  border: 1px solid rgba($gold, 0.16);
  background: rgba(255, 255, 255, 0.04);
  color: #ebd8b0;
  font-family: var(--font-display);
}
.mini-flag.guard {
  border-color: rgba(101, 156, 221, 0.32);
  background: rgba(65, 98, 135, 0.18);
  color: #b9d8ff;
}
.mini-flag.status {
  color: #dfc998;
}
.combat-controls {
  border: 1px solid rgba($gold, 0.2);
  background:
    linear-gradient(180deg, rgba(30, 23, 12, 0.18), transparent 24%),
    rgba(0, 0, 0, 0.32);
  border-radius: 10px;
  padding: 14px;
  display: grid;
  gap: 12px;
}
.control-tabs {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 8px;
}
.control-tab-btn {
  @extend %hud-action-button;
  min-height: 42px;
  padding: 8px;
  font-size: 0.68rem;
  &.active {
    border-color: rgba($gold, 0.72);
    color: #f2dbab;
    box-shadow: 0 0 16px rgba($gold, 0.2);
  }
  &:not(.active):hover {
    border-color: rgba($gold, 0.4);
    color: #e2c68f;
  }
}
.actor-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 12px;
}
.actor-badge {
  display: flex;
  align-items: center;
  gap: 14px;
  min-width: min(100%, 360px);
  padding: 12px 14px;
  border: 1px solid rgba($gold, 0.2);
  border-radius: 12px;
  background:
    linear-gradient(135deg, rgba(61, 48, 23, 0.22), transparent 55%),
    rgba(8, 8, 10, 0.78);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.04);
}
.actor-icon {
  width: 72px;
  height: 86px;
  border-radius: 14px;
}
.actor-icon .portrait-glyph {
  width: 20px;
  height: 20px;
  font-size: 0.7rem;
}
.actor-badge strong {
  display: block;
  color: var(--color-accent-strong);
  font-size: 1.03rem;
  line-height: 1.05;
  font-family: var(--font-display);
}
.actor-badge small {
  color: #cbb793;
  font-size: 0.72rem;
  letter-spacing: 0.5px;
}
.actor-stats {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}
.actor-stats span {
  border-radius: 999px;
  border: 1px solid rgba($gold, 0.2);
  background: rgba($gold, 0.06);
  color: #f0ddb8;
  padding: 5px 8px;
  font-size: 0.66rem;
  letter-spacing: 0.75px;
  font-size: 0.74rem;
  font-weight: 700;
  font-family: var(--font-ui);
}
.active-hero-xp {
  display: grid;
  gap: 6px;
}
.active-hero-xp-bar {
  width: 100%;
  height: 9px;
  border-radius: 999px;
  border: 1px solid rgba(84, 132, 224, 0.35);
  background: rgba(0, 0, 0, 0.68);
  overflow: hidden;
}
.active-hero-xp-fill {
  height: 100%;
  background: linear-gradient(90deg, #2d58a8, #6ca9ff);
}
.state-block {
  display: grid;
  gap: 12px;
  animation: panelRise 0.24s ease;
}
.state-stats-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
}
.state-stats-grid span {
  border: 1px solid rgba($gold, 0.16);
  border-radius: 10px;
  background: rgba($gold, 0.05);
  padding: 10px 12px;
  color: #f0ddb9;
  font-size: 0.72rem;
  display: flex;
  justify-content: space-between;
}
.state-stats-grid b {
  color: #fff3d4;
}
.main-actions {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
}
.combat-btn {
  @extend %hud-action-button;
  min-height: 52px;
  padding: 13px 16px;
  font-size: 0.76rem;
  width: 100%;

  &.primary {
    border-color: rgba($gold, 0.42);
    color: #f0d7a4;
    background:
      linear-gradient(180deg, rgba(255, 255, 255, 0.06), transparent 34%),
      linear-gradient(170deg, rgba(34, 26, 12, 0.9), rgba(13, 13, 20, 0.94));
    box-shadow:
      0 16px 28px rgba(0, 0, 0, 0.4),
      inset 0 1px 0 rgba(255, 255, 255, 0.08),
      inset 0 -10px 18px rgba(0, 0, 0, 0.22);
  }

  &.minor {
    min-width: 102px;
    width: auto;
    justify-self: end;
    padding: 10px 13px;
    font-size: 0.64rem;
    letter-spacing: 0.92px;
  }
}
.action-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 12px;
  animation: panelRise 0.24s ease;
}
.tab-fade-enter-active,
.tab-fade-leave-active {
  transition: opacity 0.22s ease, transform 0.22s ease;
}
.tab-fade-enter-from,
.tab-fade-leave-to {
  opacity: 0;
  transform: translateY(4px);
}
.skill-block, .item-block {
  border: 1px solid rgba($gold, 0.16);
  background: rgba(0, 0, 0, 0.26);
  border-radius: 10px;
  padding: 12px;
  display: grid;
  gap: 10px;
}
.block-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}
.block-head h4 {
  margin: 0;
  color: var(--color-accent-strong);
  font-size: 0.86rem;
  letter-spacing: 1.4px;
  font-family: var(--font-display);
}
.block-head small {
  border-radius: 999px;
  border: 1px solid rgba($gold, 0.18);
  background: rgba($gold, 0.08);
  color: #d5bc84;
  padding: 4px 7px;
  font-size: 0.59rem;
}
.item-head {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 10px;
  flex-wrap: wrap;
}
.target-select {
  display: grid;
  gap: 4px;
}
.target-select span {
  color: #c5b089;
  font-size: 0.64rem;
  letter-spacing: 0.85px;
  text-transform: uppercase;
}
.target-select select {
  background: rgba(0, 0, 0, 0.62);
  border: 1px solid rgba($gold, 0.22);
  border-radius: 8px;
  color: #f0dbb0;
  font-size: 0.7rem;
  padding: 6px 8px;
}
.action-list {
  display: grid;
  gap: 10px;
}
.action-card {
  border: 1px solid rgba($gold, 0.12);
  background: rgba(12, 12, 12, 0.72);
  border-radius: 8px;
  padding: 11px;
  display: grid;
  grid-template-columns: 1fr auto;
  align-items: center;
  gap: 10px;
}
.action-copy {
  display: grid;
  gap: 4px;
  min-width: 0;
}
.action-copy strong {
  color: #f3e2c0;
  font-size: 0.85rem;
  line-height: 1.08;
  font-family: var(--font-display);
}
.action-copy small {
  color: #c4b694;
  font-size: 0.67rem;
  line-height: 1.2;
}
.empty-note {
  color: #a89d87;
  font-size: 0.67rem;
  margin: 0;
  padding: 2px 0;
}
.combat-feed {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}
.feed-line {
  border-radius: 8px;
  border: 1px solid rgba($gold, 0.18);
  background: rgba(255, 255, 255, 0.03);
  color: #ead5ac;
  padding: 6px 10px;
  font-size: 0.67rem;
  letter-spacing: 0.68px;
  font-family: var(--font-display);
}
@keyframes panelRise {
  from {
    opacity: 0;
    transform: translateY(4px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
@keyframes popRise {
  0% { opacity: 0; transform: translateY(10px) scale(0.92); }
  18% { opacity: 1; }
  100% { opacity: 0; transform: translateY(-18px) scale(1.02); }
}
@keyframes flashBurst {
  0% { opacity: 0; }
  20% { opacity: 1; }
  100% { opacity: 0; }
}
@keyframes enemyRecoil {
  0% { transform: translateX(0) scale(1); }
  28% { transform: translateX(-8px) scale(1.01); }
  62% { transform: translateX(4px) scale(0.995); }
  100% { transform: translateX(0) scale(1); }
}
@keyframes enemyLunge {
  0% { transform: translateY(0) scale(1); }
  40% { transform: translateY(4px) scale(1.08); }
  100% { transform: translateY(0) scale(1); }
}
@keyframes heroImpact {
  0% { transform: translateX(0); }
  25% { transform: translateX(4px); }
  55% { transform: translateX(-3px); }
  100% { transform: translateX(0); }
}
@keyframes iconJolt {
  0% { transform: scale(1) rotate(0deg); }
  30% { transform: scale(1.12) rotate(-6deg); }
  60% { transform: scale(0.98) rotate(4deg); }
  100% { transform: scale(1) rotate(0deg); }
}
@keyframes iconLift {
  0% { transform: translateY(0) scale(1); }
  40% { transform: translateY(-4px) scale(1.08); }
  100% { transform: translateY(0) scale(1); }
}
@media (max-width: 900px) {
  .combat-atmosphere {
    grid-template-columns: 1fr;
  }
  .combat-title h3 {
    font-size: clamp(1.2rem, 4.2vw, 1.6rem);
  }
  .enemy-icon {
    font-size: clamp(3rem, 8vw, 4.2rem);
  }
  .enemy-identity strong {
    font-size: clamp(1.32rem, 4vw, 1.72rem);
  }
  .planner-hero {
    min-height: 64px;
    padding: 10px 12px;
  }
  .planner-icon {
    width: 48px;
    height: 58px;
  }
  .ally-icon {
    width: 50px;
    height: 62px;
  }
  .actor-icon {
    width: 64px;
    height: 78px;
  }
  .ally-rack {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .action-grid {
    grid-template-columns: 1fr;
  }
  .main-actions {
    grid-template-columns: 1fr;
  }
  .action-card {
    grid-template-columns: 1fr;
  }
  .combat-btn.minor {
    width: 100%;
    justify-self: stretch;
  }
}
@media (max-width: 680px) {
  .tactical-combat-panel {
    padding: 12px;
    border-radius: 10px;
  }
  .eyebrow {
    font-size: 0.64rem;
    letter-spacing: 1.8px;
  }
  .combat-title h3 {
    font-size: 1.22rem;
  }
  .combat-head,
  .planner-topline,
  .item-head,
  .actor-row {
    align-items: stretch;
  }
  .combat-head-status,
  .planner-actions,
  .actor-stats {
    width: 100%;
  }
  .target-select {
    width: 100%;
  }
  .target-select select {
    width: 100%;
  }
  .ally-rack {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .planner-topline strong,
  .block-head h4 {
    font-size: 0.74rem;
  }
  .planner-hero {
    min-height: 54px;
    font-size: 0.74rem;
  }
  .ally-card {
    grid-template-columns: 48px minmax(0, 1fr);
    column-gap: 8px;
  }
  .planner-name,
  .ally-card strong,
  .action-copy strong {
    font-size: 0.74rem;
  }
  .planner-icon {
    width: 42px;
    height: 50px;
  }
  .ally-icon {
    width: 44px;
    height: 54px;
  }
  .portrait-glyph {
    width: 16px;
    height: 16px;
    font-size: 0.56rem;
  }
  .enemy-icon {
    font-size: clamp(2.8rem, 13vw, 3.6rem);
  }
  .enemy-identity small {
    font-size: 0.7rem;
    letter-spacing: 1.4px;
  }
  .enemy-identity strong {
    font-size: 1.34rem;
  }
  .actor-icon {
    width: 52px;
    height: 62px;
  }
  .actor-icon .portrait-glyph {
    width: 18px;
    height: 18px;
    font-size: 0.62rem;
  }
  .actor-badge strong {
    font-size: 0.84rem;
  }
  .combat-btn {
    min-height: 46px;
    font-size: 0.7rem;
  }
  .enemy-health-meta {
    font-size: 0.62rem;
  }
}
</style>
