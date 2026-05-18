<template>
  <aside class="party-inventory-panel">
    <header class="panel-header">
      <h3>INVENTARIO DE ESCUADRA</h3>
      <small>{{ combatActive ? 'COMBATE ACTIVO' : 'EXPLORACIÓN' }}</small>
    </header>

    <div class="hero-tabs" v-if="heroes.length > 0">
      <button
        v-for="hero in heroes"
        :key="hero.id"
        class="hero-tab"
        :class="{ active: hero.id === selectedHeroId }"
        @click="selectHero(hero.id)"
      >
        <span>{{ hero.icon }}</span>
        <strong>{{ hero.name }}</strong>
      </button>
    </div>

    <div v-if="activeHero" class="panel-body">
      <section class="hero-overview">
        <div class="hero-id">
          <strong>{{ activeHero.name }}</strong>
          <small>{{ activeHero.role }}</small>
        </div>
        <div class="hero-level-row">
          <span class="level-badge">NV {{ activeHero.level || 1 }}</span>
          <small>XP {{ activeHero.experience || 0 }} / {{ activeHero.nextLevelXp || 1 }}</small>
        </div>
        <div class="xp-strip">
          <div class="xp-fill" :style="{ width: `${heroXpPercent(activeHero)}%` }"></div>
        </div>
        <div class="hero-hp">
          <div class="hp-strip">
            <div class="hp-fill" :style="{ width: `${Math.max(0, (activeHero.hp / activeHero.maxHp) * 100)}%` }"></div>
          </div>
          <small>HP {{ activeHero.hp }} / {{ activeHero.maxHp }}</small>
        </div>
        <div class="hero-stats-grid">
          <span>ATQ <b>{{ activeHero.attack || 0 }}</b></span>
          <span>DEF <b>{{ activeHero.defense || 0 }}</b></span>
          <span>MAG <b>{{ activeHero.magic || 0 }}</b></span>
          <span>AGI <b>{{ activeHero.agility || 0 }}</b></span>
        </div>
      </section>

      <div class="panel-tabs">
        <button class="panel-tab-btn" :class="{ active: activePanelTab === 'state' }" @click="activePanelTab = 'state'">ESTADO</button>
        <button class="panel-tab-btn" :class="{ active: activePanelTab === 'inventory' }" @click="activePanelTab = 'inventory'">INVENTARIO</button>
        <button class="panel-tab-btn" :class="{ active: activePanelTab === 'skills' }" @click="activePanelTab = 'skills'">HABILIDADES</button>
      </div>

      <transition name="panel-tab-fade" mode="out-in">
      <section class="slot-section" v-if="activePanelTab === 'state'" key="state">
        <h4>Equipado</h4>
        <div class="slot-row">
          <span>Arma</span>
          <button class="slot-btn" @click="unequip('weapon')">
            {{ activeHero.equipment?.weapon?.name || 'Vacío' }}
          </button>
        </div>
        <div class="slot-row">
          <span>Armadura</span>
          <button class="slot-btn" @click="unequip('armor')">
            {{ activeHero.equipment?.armor?.name || 'Vacío' }}
          </button>
        </div>
        <div class="slot-row">
          <span>Reliquia</span>
          <button class="slot-btn" @click="unequip('trinket')">
            {{ activeHero.equipment?.trinket?.name || 'Vacío' }}
          </button>
        </div>
      </section>

      <section class="consumables-section" v-else-if="activePanelTab === 'inventory'" key="inventory">
        <div class="section-head">
          <h4>Consumibles</h4>
          <select v-model="targetHeroId">
            <option v-for="hero in heroes" :key="`target_${hero.id}`" :value="hero.id">{{ hero.name }}</option>
          </select>
        </div>
        <div class="item-list">
          <article v-for="item in consumables" :key="item.id" class="item-card">
            <div>
              <strong>{{ item.name }}</strong>
              <small>x{{ item.quantity }} · {{ item.description }}</small>
            </div>
            <button class="item-btn" @click="useConsumable(item.id)">USAR</button>
          </article>
          <p v-if="consumables.length === 0" class="empty-note">No hay consumibles disponibles.</p>
        </div>
      </section>

      <section class="skills-section" v-else key="skills">
        <h4>Habilidades</h4>
        <div class="skill-list">
          <article v-for="skill in activeHero.skills || []" :key="skill.id" class="skill-card">
            <strong>{{ skill.name }}</strong>
            <small>{{ skill.description }}</small>
            <small class="skill-meta">{{ skill.type.toUpperCase() }} · {{ skill.dice }} · Poder {{ skill.power }}</small>
          </article>
          <p v-if="!activeHero.skills || activeHero.skills.length === 0" class="empty-note">Sin habilidades cargadas.</p>
        </div>
      </section>
      </transition>

      <section class="equipment-section" v-if="activePanelTab === 'inventory'">
        <h4>Mochila de Equipo</h4>
        <div class="item-list">
          <article v-for="item in equippables" :key="item.id" class="item-card">
            <div>
              <strong>{{ item.name }}</strong>
              <small>x{{ item.quantity }} · {{ item.description }}</small>
            </div>
            <button class="item-btn" @click="equipItem(item.id)">EQUIPAR</button>
          </article>
          <p v-if="equippables.length === 0" class="empty-note">No hay equipo en la mochila.</p>
        </div>
      </section>

    </div>

    <p v-else class="empty-note">No hay personajes para gestionar inventario.</p>
  </aside>
</template>

<script setup>
import { computed, ref, watch } from 'vue';
import {
  heroConsumables,
  heroEquippables,
  useHeroConsumable,
  equipItemOnHero,
  unequipHeroSlot
} from '../utils/partySystem';

const props = defineProps({
  party: {
    type: Array,
    default: () => []
  },
  combatActive: {
    type: Boolean,
    default: false
  }
});

const emit = defineEmits(['update-party', 'log']);

const selectedHeroId = ref('');
const targetHeroId = ref('');
const activePanelTab = ref('state');

const heroes = computed(() => (Array.isArray(props.party) ? props.party : []));
const activeHero = computed(() => heroes.value.find((hero) => hero.id === selectedHeroId.value) || null);
const consumables = computed(() => heroConsumables(activeHero.value));
const equippables = computed(() => heroEquippables(activeHero.value));

function toInt(value, fallback = 0) {
  const parsed = Number(value);
  return Number.isFinite(parsed) ? Math.floor(parsed) : fallback;
}

function heroXpPercent(hero) {
  if (!hero) return 0;
  const xp = Math.max(0, toInt(hero.experience, 0));
  const next = Math.max(1, toInt(hero.nextLevelXp, 1));
  return Math.max(0, Math.min(100, (xp / next) * 100));
}

function selectHero(heroId) {
  selectedHeroId.value = heroId;
  targetHeroId.value = heroId;
}

function emitResult(result) {
  if (result?.error) {
    emit('log', result.error);
    return;
  }
  if (Array.isArray(result?.party)) emit('update-party', result.party);
  if (result?.message) emit('log', result.message);
}

function useConsumable(itemId) {
  if (!activeHero.value || !targetHeroId.value) return;
  const result = useHeroConsumable(props.party, activeHero.value.id, itemId, targetHeroId.value);
  emitResult(result);
}

function equipItem(itemId) {
  if (!activeHero.value) return;
  const result = equipItemOnHero(props.party, activeHero.value.id, itemId);
  emitResult(result);
}

function unequip(slot) {
  if (!activeHero.value) return;
  const result = unequipHeroSlot(props.party, activeHero.value.id, slot);
  emitResult(result);
}

watch(
  heroes,
  (value) => {
    if (!Array.isArray(value) || value.length === 0) {
      selectedHeroId.value = '';
      targetHeroId.value = '';
      return;
    }
    const stillExists = value.some((hero) => hero.id === selectedHeroId.value);
    if (!stillExists) {
      selectedHeroId.value = value[0].id;
      activePanelTab.value = 'state';
    }
    const targetExists = value.some((hero) => hero.id === targetHeroId.value);
    if (!targetExists) {
      targetHeroId.value = selectedHeroId.value || value[0].id;
    }
  },
  { immediate: true, deep: true }
);
</script>

<style scoped lang="scss">
$gold: #c5a059;

.party-inventory-panel {
  --font-display: 'Cinzel', serif;
  --font-ui: 'Inter', 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  --color-text: #efe2c4;
  --color-soft: #c5b897;
  --color-dim: #94886f;
  --color-accent: #f0d7a8;
  border: 1px solid rgba($gold, 0.32);
  border-radius: 24px;
  background: rgba(4, 4, 4, 0.84);
  backdrop-filter: blur(8px);
  padding: 14px;
  overflow-y: auto;
  min-height: 0;
  box-shadow: inset 0 1px 0 rgba(255,255,255,.04), 0 18px 32px rgba(0,0,0,.35);
  font-family: var(--font-ui);
  color: var(--color-text);
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
.panel-header {
  margin-bottom: 12px;
  h3 {
    margin: 0;
    color: var(--color-accent);
    font-size: 0.82rem;
    letter-spacing: 1.5px;
    font-family: var(--font-display);
  }
  small {
    color: var(--color-dim);
    font-size: 0.64rem;
    letter-spacing: 1px;
    font-family: var(--font-display);
  }
}
.hero-tabs {
  display: grid;
  grid-template-columns: 1fr;
  gap: 8px;
  margin-bottom: 12px;
}
.hero-tab {
  border: 1px solid rgba($gold, 0.22);
  border-radius: 12px;
  background: rgba(15, 15, 15, 0.75);
  color: var(--color-accent);
  padding: 10px;
  display: flex;
  align-items: center;
  gap: 6px;
  cursor: pointer;
  &.active {
    border-color: rgba($gold, 0.8);
    box-shadow: 0 0 12px rgba($gold, 0.18);
  }
  strong { font-size: 0.72rem; font-family: var(--font-display); }
}
.panel-body {
  display: grid;
  gap: 14px;
}
.hero-overview {
  border: 1px solid rgba($gold, 0.25);
  border-radius: 14px;
  background: rgba(10, 10, 10, 0.65);
  padding: 11px;
}
.hero-id {
  display: flex;
  flex-direction: column;
  strong { color: var(--color-accent); font-size: 0.82rem; font-family: var(--font-display); }
  small { color: var(--color-dim); font-size: 0.64rem; text-transform: uppercase; font-family: var(--font-display); }
}
.hero-level-row {
  margin-top: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  small {
    color: #a8c1ee;
    font-size: 0.66rem;
    font-weight: 600;
  }
}
.level-badge {
  border: 1px solid rgba($gold, 0.48);
  border-radius: 999px;
  background: rgba($gold, 0.12);
  color: #f8e7c0;
  font-weight: 700;
  font-size: 0.7rem;
  padding: 4px 10px;
  letter-spacing: 0.9px;
  font-family: var(--font-display);
}
.xp-strip {
  margin-top: 8px;
  height: 7px;
  border: 1px solid rgba(90, 139, 228, 0.42);
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.7);
  overflow: hidden;
}
.xp-fill {
  height: 100%;
  background: linear-gradient(90deg, #2b5db8, #73b2ff);
}
.hp-strip {
  margin-top: 8px;
  height: 6px;
  border: 1px solid rgba($gold, 0.25);
  border-radius: 999px;
  background: rgba(0, 0, 0, 0.7);
  overflow: hidden;
}
.hp-fill {
  height: 100%;
  background: linear-gradient(90deg, #a72f2f, #ec5b5b);
}
.hero-hp small {
  color: var(--color-soft);
  font-size: 0.66rem;
  font-weight: 600;
}
.hero-stats-grid {
  margin-top: 10px;
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 8px;
  span {
    border: 1px solid rgba($gold, 0.16);
    border-radius: 9px;
    background: rgba($gold, 0.04);
    padding: 5px 7px;
    color: #e0cca2;
    font-size: 0.66rem;
    display: flex;
    justify-content: space-between;
  }
  b { color: #fff0cf; }
}
.panel-tabs {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 8px;
}
.panel-tab-btn {
  border: 1px solid rgba($gold, 0.2);
  border-radius: 11px;
  background: rgba(0, 0, 0, 0.4);
  color: #dbc08a;
  font-size: 0.66rem;
  font-weight: 700;
  letter-spacing: 0.8px;
  padding: 7px;
  cursor: pointer;
  &.active {
    border-color: rgba($gold, 0.68);
    color: #f0d7a6;
    box-shadow: 0 0 12px rgba($gold, 0.16);
  }
  &:not(.active):hover {
    border-color: rgba($gold, 0.38);
    color: #ddc28f;
  }
}
.slot-section, .consumables-section, .equipment-section, .skills-section {
  border: 1px solid rgba($gold, 0.2);
  border-radius: 14px;
  background: rgba(8, 8, 8, 0.6);
  padding: 11px;
}
  h4 {
  margin: 0 0 8px;
  color: var(--color-accent);
  font-size: 0.76rem;
  letter-spacing: 1px;
  font-family: var(--font-display);
}
.slot-row {
  display: grid;
  grid-template-columns: 56px 1fr;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  span { color: #8c8c8c; font-size: 0.62rem; text-transform: uppercase; }
}
.slot-btn {
  border: 1px solid rgba($gold, 0.3);
  border-radius: 10px;
  background: rgba($gold, 0.07);
  color: #edd4a1;
  font-size: 0.66rem;
  padding: 8px;
  text-align: left;
  cursor: pointer;
}
.section-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  select {
    background: rgba(0, 0, 0, 0.65);
    border: 1px solid rgba($gold, 0.28);
    border-radius: 10px;
    color: #efd8aa;
    font-size: 0.66rem;
    padding: 4px 6px;
  }
}
.item-list {
  display: grid;
  gap: 8px;
}
.item-card {
  border: 1px solid rgba($gold, 0.14);
  border-radius: 12px;
  background: rgba(0, 0, 0, 0.35);
  padding: 10px;
  display: grid;
  gap: 8px;
  strong { color: #f2dfbb; font-size: 0.72rem; font-family: var(--font-display); }
  small { color: #b9ac8f; font-size: 0.64rem; line-height: 1.35; }
}
.item-btn {
  border: 1px solid rgba($gold, 0.45);
  border-radius: 10px;
  background: rgba($gold, 0.08);
  color: #f1d8a7;
  font-size: 0.64rem;
  padding: 5px 8px;
  cursor: pointer;
  justify-self: end;
}
.skill-list {
  display: grid;
  gap: 8px;
}
.skill-card {
  border: 1px solid rgba($gold, 0.14);
  border-radius: 12px;
  background: rgba(0, 0, 0, 0.35);
  padding: 10px;
  display: grid;
  gap: 4px;
  strong { color: #f2ddaf; font-size: 0.72rem; font-family: var(--font-display); }
  small { color: #b9ac8f; font-size: 0.64rem; line-height: 1.35; }
  .skill-meta { color: #d2b57d; }
}
.panel-tab-fade-enter-active,
.panel-tab-fade-leave-active {
  transition: opacity 0.2s ease, transform 0.2s ease;
}
.panel-tab-fade-enter-from,
.panel-tab-fade-leave-to {
  opacity: 0;
  transform: translateY(4px);
}
.empty-note {
  color: #a59a83;
  font-size: 0.66rem;
  line-height: 1.4;
}
</style>
