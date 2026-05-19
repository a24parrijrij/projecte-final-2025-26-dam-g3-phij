<template>
  <div class="codex-page">
    <div class="fx-bg"></div>

    <nav class="top-nav glass">
      <router-link to="/userpage" class="back-btn">← BASTIÓ</router-link>
      <h1>CÒDEX DE VALKRYPT</h1>
      <div class="campaign-pill">{{ campaignTitle }}</div>
    </nav>

    <section class="codex-shell glass">
      <aside class="tabs">
        <button
          v-for="item in tabs"
          :key="item.id"
          class="tab-btn"
          :class="{ active: activeTab === item.id }"
          @click="activeTab = item.id"
        >
          {{ item.label }}
        </button>
      </aside>

      <div class="content">
        <article v-if="activeTab === 'lore'" class="entry-list">
          <h2>Cròniques del Regne</h2>
          <div class="entry" v-for="entry in loreEntries" :key="entry.title">
            <h3>{{ entry.title }}</h3>
            <p>{{ entry.text }}</p>
          </div>
        </article>

        <article v-else-if="activeTab === 'factions'" class="entry-list">
          <h2>Faccions</h2>
          <div class="entry" v-for="entry in factionEntries" :key="entry.title">
            <h3>{{ entry.title }}</h3>
            <p>{{ entry.text }}</p>
          </div>
        </article>

        <article v-else-if="activeTab === 'bestiary'" class="entry-list">
          <h2>Bestiari</h2>
          <div class="entry" v-for="entry in bestiaryEntries" :key="entry.title">
            <h3>{{ entry.title }}</h3>
            <p>{{ entry.text }}</p>
            <small>Debilitats: {{ entry.weakness }}</small>
          </div>
        </article>

        <article v-else-if="activeTab === 'commands'" class="entry-list">
          <h2>Comandes Narratives</h2>
          <div class="entry" v-for="entry in commandEntries" :key="entry.cmd">
            <h3>{{ entry.cmd }}</h3>
            <p>{{ entry.text }}</p>
          </div>
        </article>

        <article v-else class="entry-list">
          <h2>Consells Tàctics</h2>
          <ul class="tips-list">
            <li v-for="tip in tipEntries" :key="tip">{{ tip }}</li>
          </ul>
        </article>
      </div>
    </section>
  </div>
</template>

<script setup>
import { computed, ref } from 'vue'

const activeTab = ref('lore')

const tabs = [
  { id: 'lore', label: 'Lore' },
  { id: 'factions', label: 'Faccions' },
  { id: 'bestiary', label: 'Bestiari' },
  { id: 'commands', label: 'Comandes' },
  { id: 'tips', label: 'Tàctica' }
]

const currentGame = (() => {
  try {
    return JSON.parse(localStorage.getItem('valkrypt_current_game') || '{}')
  } catch {
    return {}
  }
})()

const campaignTitle = computed(() => currentGame?.campaignTitle || 'Sense campanya activa')
const campaignLocation = computed(() => currentGame?.location || 'Regió desconeguda')

const loreEntries = computed(() => [
  {
    title: `Sombras sobre ${campaignLocation.value}`,
    text: 'El vel entre el món mortal i l’Abisme és més prim aquí. Les decisions violentes acceleren la corrupció de la zona.'
  },
  {
    title: 'El Pacte de Cendra',
    text: 'Antics arcanistes van segellar relíquies amb juraments de sang. Trencar aquests segells pot donar poder, però també invoca represàlies.'
  },
  {
    title: 'Fragments de memòria',
    text: 'Cada combat deixa una empremta. Els companys acumulen trauma, glòria i vincles que afecten la narrativa i el rendiment.'
  }
])

const factionEntries = [
  {
    title: 'Orde del Ferro Buit',
    text: 'Mercenaris disciplinats. Valoren la força i la logística. Poden oferir equipament però exigeixen deutes.'
  },
  {
    title: 'Conclave de Vidre Negre',
    text: 'Mags ritualistes experts en malediccions i previsió tàctica. Intercanvien informació per artefactes.'
  },
  {
    title: 'Gremi de Cendra i Sal',
    text: 'Contrabandistes i exploradors de ruïnes. Coneixen rutes segures, però només treballen amb qui té reputació.'
  }
]

const bestiaryEntries = [
  {
    title: 'Taberner Corrupto',
    text: 'Humano mutado por energía abisal. Ataca rápido y escala su agresividad por ronda.',
    weakness: 'Control de iniciativa y daño explosivo temprano'
  },
  {
    title: 'Mineros Deformados',
    text: 'Escuadras numerosas con presión constante. Peligrosos cuando te rodean.',
    weakness: 'Daño en área, ralentización y foco por objetivos'
  },
  {
    title: 'Custodio Umbrío',
    text: 'Entidad blindada con contraataques. Castiga ataques débiles y turnos mal ordenados.',
    weakness: 'Ruptura de defensa, magia sostenida y defensa activa'
  }
]

const commandEntries = [
  { cmd: '/talk [objetivo], [mensaje]', text: 'Habla con NPCs o enemigos para negociar, provocar, engañar o conseguir pistas.' },
  { cmd: '/attack [objetivo]', text: 'Fuerza una acción agresiva fuera de las opciones sugeridas por el narrador.' },
  { cmd: '/stealth [acción]', text: 'Intenta aproximarte sin ser detectado para ventaja de apertura.' },
  { cmd: '/inspect [objetivo]', text: 'Analiza entorno, objetos o enemigos para descubrir debilidades ocultas.' },
  { cmd: '/help', text: 'Muestra guía rápida de comandos disponibles en el flujo narrativo.' }
]

const tipEntries = [
  'Planifica el orden de turno antes de confirmar: el primer aliado define el ritmo del round.',
  'Gasta consumibles cuando el combate ya está inclinado en tu contra, no al inicio.',
  'Sube agilidad en personajes de control para asegurar iniciativa y utilidad.',
  'Si un enemigo escala por turnos, prioriza daño explosivo sobre desgaste.',
  'Alterna minijuegos para repartir progreso de atributos y evitar builds desbalanceadas.'
]
</script>

<style scoped>
.codex-page {
  min-height: 100vh;
  position: relative;
  background: radial-gradient(circle at 15% 20%, rgba(84, 44, 17, 0.2), transparent 40%),
    radial-gradient(circle at 85% 80%, rgba(25, 51, 96, 0.2), transparent 42%),
    #05070d;
  color: #efe6d3;
  padding: 20px;
  font-family: 'Cinzel', serif;
}

.fx-bg {
  position: fixed;
  inset: 0;
  background: linear-gradient(120deg, rgba(255, 201, 107, 0.03), rgba(126, 168, 255, 0.03));
  pointer-events: none;
}

.glass {
  border: 1px solid rgba(201, 165, 95, 0.28);
  background: linear-gradient(180deg, rgba(8, 11, 20, 0.86), rgba(7, 9, 15, 0.92));
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.35), inset 0 1px 0 rgba(255, 230, 180, 0.06);
  border-radius: 16px;
}

.top-nav {
  max-width: 1200px;
  margin: 0 auto 18px;
  padding: 12px 16px;
  display: grid;
  grid-template-columns: 160px 1fr 220px;
  align-items: center;
  gap: 12px;
}

.top-nav h1 {
  margin: 0;
  text-align: center;
  color: #e9cb8d;
  letter-spacing: 1.5px;
  font-size: 1.2rem;
}

.back-btn,
.campaign-pill {
  border: 1px solid rgba(214, 176, 104, 0.34);
  background: rgba(6, 9, 14, 0.8);
  border-radius: 10px;
  padding: 8px 12px;
  color: #dfbe80;
  text-decoration: none;
  font-size: 0.82rem;
  letter-spacing: 0.6px;
  text-align: center;
}

.codex-shell {
  max-width: 1200px;
  margin: 0 auto;
  min-height: calc(100vh - 120px);
  display: grid;
  grid-template-columns: 220px 1fr;
  overflow: hidden;
}

.tabs {
  border-right: 1px solid rgba(185, 149, 81, 0.23);
  padding: 14px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.tab-btn {
  border: 1px solid rgba(188, 153, 88, 0.32);
  background: rgba(10, 13, 20, 0.72);
  color: #ceb07a;
  border-radius: 10px;
  padding: 10px 12px;
  text-align: left;
  font-family: inherit;
  cursor: pointer;
  transition: all .2s ease;
}

.tab-btn:hover { transform: translateX(2px); }

.tab-btn.active {
  background: linear-gradient(90deg, rgba(122, 83, 31, 0.28), rgba(29, 41, 74, 0.2));
  color: #f0d7a4;
  box-shadow: inset 0 0 0 1px rgba(232, 196, 129, 0.18);
}

.content {
  padding: 18px;
}

.entry-list h2 {
  margin: 0 0 14px;
  color: #e9ca8d;
  font-size: 1.05rem;
  letter-spacing: 1px;
}

.entry {
  border: 1px solid rgba(196, 157, 86, 0.25);
  border-radius: 12px;
  padding: 12px 14px;
  margin-bottom: 10px;
  background: rgba(8, 11, 18, 0.7);
}

.entry h3 {
  margin: 0 0 6px;
  color: #edd9ad;
  font-size: 0.96rem;
}

.entry p,
.entry small {
  color: #c8bea8;
  margin: 0;
  font-size: 0.88rem;
  line-height: 1.45;
}

.entry small {
  display: inline-block;
  margin-top: 8px;
  color: #bea26f;
}

.tips-list {
  margin: 0;
  padding-left: 20px;
  display: grid;
  gap: 10px;
  color: #cfbea2;
  font-size: 0.9rem;
}

@media (max-width: 900px) {
  .top-nav {
    grid-template-columns: 1fr;
  }

  .top-nav h1 {
    text-align: left;
  }

  .codex-shell {
    grid-template-columns: 1fr;
  }

  .tabs {
    border-right: 0;
    border-bottom: 1px solid rgba(185, 149, 81, 0.23);
    flex-direction: row;
    flex-wrap: wrap;
  }
}
</style>
