<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import importedLevels from './levels.json'

const seedLevels = [
  {
    id: 'pale-machine',
    rank: 1,
    name: 'Pale Machine',
    creator: 'Milo',
    verifier: 'Zoink',
    nationality: 'Sweden',
    flag: '🇸🇪',
    difficulty: 'Extreme Demon',
    description: 'A patient, mechanical journey through cold light and tighter-than-tight sync.',
    tags: ['normal', 'pre 1.9', 'self imposed challenge'],
    video: 'https://www.youtube.com/embed/dQw4w9WgXcQ',
  },
  {
    id: 'glass-temple',
    rank: 2,
    name: 'Glass Temple',
    creator: 'Aster',
    verifier: 'Cersia',
    nationality: 'United States',
    flag: '🇺🇸',
    difficulty: 'Extreme Demon',
    description: 'A bright, brittle level that rewards nerve, memory, and a little bit of luck.',
    tags: ['normal', 'long', 'modern'],
    video: 'https://www.youtube.com/embed/dQw4w9WgXcQ',
  },
  {
    id: 'afterimage',
    rank: 3,
    name: 'Afterimage',
    creator: 'Nox',
    verifier: 'Kuro',
    nationality: 'Japan',
    flag: '🇯🇵',
    difficulty: 'Extreme Demon',
    description: 'What stays on your screen after the level ends: a study in motion and restraint.',
    tags: ['normal', 'wave heavy', 'self imposed challenge'],
    video: 'https://www.youtube.com/embed/dQw4w9WgXcQ',
  },
  {
    id: 'blue-hour',
    rank: 4,
    name: 'Blue Hour',
    creator: 'Luma',
    verifier: 'Riot',
    nationality: 'Canada',
    flag: '🇨🇦',
    difficulty: 'Extreme Demon',
    description: 'An old-school pulse with new-school edges, made for the last attempt of the night.',
    tags: ['normal', 'classic', 'pre 1.9'],
    video: 'https://www.youtube.com/embed/dQw4w9WgXcQ',
  },
  {
    id: 'soft-reset',
    rank: 5,
    name: 'Soft Reset',
    creator: 'Moss',
    verifier: 'Nisha',
    nationality: 'United Kingdom',
    flag: '🇬🇧',
    difficulty: 'Extreme Demon',
    description: 'A deceptively soft opening that gives way to one of the list’s meanest endings.',
    tags: ['normal', 'memory', 'modern'],
    video: 'https://www.youtube.com/embed/dQw4w9WgXcQ',
  },
]

const storageKey = 'everything-list-levels-v4'
let storedLevels = null
try {
  const stored = localStorage.getItem(storageKey)
  storedLevels = stored ? JSON.parse(stored) : null
} catch {
  localStorage.removeItem(storageKey)
}
const importedTopTen = [...importedLevels]
  .sort((a, b) => Number(a.rank) - Number(b.rank))
  .slice(0, 10)
const levels = ref(Array.isArray(storedLevels) ? storedLevels : importedTopTen)
const page = ref('home')
const selectedLevel = ref(null)
const search = ref('')
const editing = ref(null)
const toast = ref('')
const draggedLevelId = ref(null)
const dragOverLevelId = ref(null)
const pointerDragging = ref(false)
const placementLevel = ref(null)
const placementRank = ref(1)
let dragFrame = 0
let pendingPointer = null
let placementPointerStart = null
let placementRankAtPointer = 1
let placementWheelDelta = 0
let placementWheelFrame = 0

const visibleLevels = computed(() => {
  const query = search.value.toLowerCase().trim()
  if (!query) return levels.value
  return levels.value.filter((level) =>
    `${level.name} ${level.creator} ${level.tags.join(' ')}`.toLowerCase().includes(query),
  )
})

function save() {
  localStorage.setItem(storageKey, JSON.stringify(levels.value))
  toast.value = 'List saved locally'
  window.setTimeout(() => (toast.value = ''), 2400)
}

function openLevel(level) {
  selectedLevel.value = level
}

function closeLevel() {
  selectedLevel.value = null
}

function embedUrl(url) {
  if (!url) return ''
  const match = url.match(/(?:youtu\.be\/|youtube\.com\/(?:watch\?v=|live\/))([^?&/]+)/)
  return match ? `https://www.youtube.com/embed/${match[1]}` : url
}

function thumbnailUrl(url) {
  const match = url?.match(/(?:youtu\.be\/|youtube\.com\/(?:watch\?v=|live\/))([^?&/]+)/)
  return match ? `https://i.ytimg.com/vi/${match[1]}/hqdefault.jpg` : ''
}

function startEdit(level) {
  const tags = Array.isArray(level.tags) ? level.tags : []
  editing.value = { ...level, tags, tagsText: tags.join(', ') }
}

function newLevel() {
  editing.value = {
    id: `level-${Date.now()}`,
    rank: levels.value.length + 1,
    name: '',
    creator: '',
    verifier: '',
    nationality: '',
    flag: '🌐',
    difficulty: 'Extreme Demon',
    date: '',
    topOne: false,
    description: '',
    tags: ['normal'],
    tagsText: 'normal',
    video: '',
  }
}

function saveLevel() {
  const clean = {
    ...editing.value,
    tags: String(editing.value.tagsText || '')
      .split(',')
      .map((tag) => tag.trim())
      .filter(Boolean),
  }
  delete clean.tagsText
  const index = levels.value.findIndex((level) => level.id === clean.id)
  if (index === -1) levels.value.push(clean)
  else levels.value[index] = clean
  levels.value.sort((a, b) => Number(a.rank) - Number(b.rank))
  levels.value.forEach((level, index) => (level.rank = index + 1))
  editing.value = null
}

function removeLevel(level) {
  levels.value = levels.value.filter((item) => item.id !== level.id)
  levels.value.forEach((item, index) => (item.rank = index + 1))
}

function move(level, direction) {
  const index = levels.value.findIndex((item) => item.id === level.id)
  const next = index + direction
  if (next < 0 || next >= levels.value.length) return
  const list = [...levels.value]
  ;[list[index], list[next]] = [list[next], list[index]]
  list.forEach((item, itemIndex) => (item.rank = itemIndex + 1))
  levels.value = list
}

const placementLevels = computed(() => {
  if (!placementLevel.value) return []
  const remaining = levels.value.filter((item) => item.id !== placementLevel.value.id)
  const insertionIndex = Math.max(0, Math.min(remaining.length, placementRank.value - 1))
  const above = remaining.slice(Math.max(0, insertionIndex - 2), insertionIndex)
  const below = remaining.slice(insertionIndex, insertionIndex + 2)
  return [
    ...Array.from({ length: 2 - above.length }, () => null),
    ...above,
    placementLevel.value,
    ...below,
    ...Array.from({ length: 2 - below.length }, () => null),
  ].map((item, index) => ({
    item,
    rank: placementRank.value + index - 2,
  }))
})

function openPlacement(level) {
  placementLevel.value = level
  placementRank.value = level.rank
}

function confirmPlacement() {
  if (!placementLevel.value) return
  const list = levels.value.filter((item) => item.id !== placementLevel.value.id)
  list.splice(placementRank.value - 1, 0, placementLevel.value)
  list.forEach((item, index) => (item.rank = index + 1))
  levels.value = list
  closePlacement()
}

function closePlacement() {
  placementLevel.value = null
}

function placementMiddleDown(event) {
  if (event.button !== 1) return
  event.preventDefault()
  placementPointerStart = event.clientY
  placementRankAtPointer = placementRank.value
  document.addEventListener('pointermove', placementMiddleMove)
  document.addEventListener('pointerup', placementMiddleUp)
}

function placementMiddleMove(event) {
  if (placementPointerStart === null) return
  const rowHeight = 42
  const movement = placementPointerStart - event.clientY
  const next = placementRankAtPointer + Math.round(movement / rowHeight)
  placementRank.value = Math.max(1, Math.min(levels.value.length, next))
}

function placementMiddleUp() {
  placementPointerStart = null
  document.removeEventListener('pointermove', placementMiddleMove)
  document.removeEventListener('pointerup', placementMiddleUp)
}

function placementWheel(event) {
  event.preventDefault()
  placementWheelDelta += event.deltaY
  if (placementWheelFrame) return
  placementWheelFrame = window.requestAnimationFrame(() => {
    placementWheelFrame = 0
    const step = placementWheelDelta > 0 ? 1 : -1
    placementWheelDelta = 0
    placementRank.value = Math.max(1, Math.min(levels.value.length, placementRank.value + step))
  })
}

function dragStart(level) {
  draggedLevelId.value = level.id
}

function dragOver(level) {
  if (draggedLevelId.value === level.id) return
  const currentIndex = levels.value.findIndex((item) => item.id === draggedLevelId.value)
  const targetIndex = levels.value.findIndex((item) => item.id === level.id)
  if (currentIndex === -1 || targetIndex === -1) return
  const list = [...levels.value]
  const [dragged] = list.splice(currentIndex, 1)
  list.splice(targetIndex, 0, dragged)
  list.forEach((item, index) => (item.rank = index + 1))
  levels.value = list
  dragOverLevelId.value = level.id
}

function dropLevel(target) {
  draggedLevelId.value = null
  dragOverLevelId.value = null
}

function dragEnd() {
  draggedLevelId.value = null
  dragOverLevelId.value = null
}

function reorderDraggedOver(level) {
  if (!pointerDragging.value || draggedLevelId.value === level.id) return
  const currentIndex = levels.value.findIndex((item) => item.id === draggedLevelId.value)
  const targetIndex = levels.value.findIndex((item) => item.id === level.id)
  if (currentIndex === -1 || targetIndex === -1) return
  const list = [...levels.value]
  const [dragged] = list.splice(currentIndex, 1)
  list.splice(targetIndex, 0, dragged)
  list.forEach((item, index) => (item.rank = index + 1))
  levels.value = list
}

function pointerMove(event) {
  if (!pointerDragging.value) return
  pendingPointer = { x: event.clientX, y: event.clientY }
  if (dragFrame) return
  dragFrame = window.requestAnimationFrame(() => {
    dragFrame = 0
    if (!pointerDragging.value || !pendingPointer) return
    const row = document.elementFromPoint(pendingPointer.x, pendingPointer.y)?.closest('[data-level-id]')
    pendingPointer = null
    if (!row) return
    const level = levels.value.find((item) => item.id === row.dataset.levelId)
    if (level) reorderDraggedOver(level)
  })
}

function pointerUp() {
  if (!pointerDragging.value) return
  pointerDragging.value = false
  pendingPointer = null
  if (dragFrame) {
    window.cancelAnimationFrame(dragFrame)
    dragFrame = 0
  }
  dragEnd()
  document.removeEventListener('pointermove', pointerMove)
  document.removeEventListener('pointerup', pointerUp)
}

function startPointerDrag(level, event) {
  if (event.target.closest('button, input, a')) return
  event.preventDefault()
  pointerDragging.value = true
  draggedLevelId.value = level.id
  event.currentTarget.setPointerCapture?.(event.pointerId)
  document.addEventListener('pointermove', pointerMove)
  document.addEventListener('pointerup', pointerUp)
}

function handleDialogKeydown(event) {
  const dialogs = [...document.querySelectorAll('[role="dialog"]')]
  const dialog = dialogs[dialogs.length - 1]
  if (!dialog) return

  if (event.key === 'Escape') {
    if (editing.value) editing.value = null
    else if (placementLevel.value) closePlacement()
    else if (selectedLevel.value) closeLevel()
    event.preventDefault()
    return
  }

  if (event.key !== 'Tab') return
  const focusable = [...dialog.querySelectorAll('button:not([disabled]), input:not([disabled]), textarea:not([disabled]), select:not([disabled]), [tabindex]:not([tabindex="-1"])')]
    .filter((element) => element.offsetParent !== null)
  if (!focusable.length) {
    event.preventDefault()
    return
  }

  const first = focusable[0]
  const last = focusable[focusable.length - 1]
  if (!dialog.contains(document.activeElement)) {
    event.preventDefault()
    first.focus()
  } else if (event.shiftKey && document.activeElement === first) {
    event.preventDefault()
    last.focus()
  } else if (!event.shiftKey && document.activeElement === last) {
    event.preventDefault()
    first.focus()
  }
}

watch([placementLevel, selectedLevel, editing], async () => {
  await nextTick()
  const dialogs = [...document.querySelectorAll('[role="dialog"]')]
  const dialog = dialogs[dialogs.length - 1]
  dialog?.querySelector('button:not([disabled]), input:not([disabled]), textarea:not([disabled])')?.focus()
})

onMounted(() => {
  document.addEventListener('keydown', handleDialogKeydown)
})

onBeforeUnmount(() => {
  if (dragFrame) window.cancelAnimationFrame(dragFrame)
  placementMiddleUp()
  if (placementWheelFrame) window.cancelAnimationFrame(placementWheelFrame)
  document.removeEventListener('pointermove', pointerMove)
  document.removeEventListener('pointerup', pointerUp)
  document.removeEventListener('keydown', handleDialogKeydown)
})
</script>

<template>
  <div class="app-shell">
    <header class="site-header">
      <button class="brand" @click="page = 'home'; closeLevel()">
        <span>The Everything List</span>
      </button>
      <nav aria-label="Main navigation">
        <button :class="{ active: page === 'home' }" @click="page = 'home'">Home</button>
        <button :class="{ active: page === 'list' }" @click="page = 'list'">The list</button>
        <button :class="{ active: page === 'moderator' }" @click="page = 'moderator'">Moderator</button>
      </nav>
      <span class="header-note"></span>
    </header>

    <main>
      <section v-if="page === 'home'" class="home-page">
        <div class="home-intro">
          <h1>The Everything List</h1>
          <button class="button button-dark" @click="page = 'list'">Open the list <span>↗</span></button>
        </div>
      </section>

      <section v-else-if="page === 'list'" class="list-page">
        <label class="search-box list-search"><span>⌕</span><input v-model="search" type="search" placeholder="Search levels..." /></label>
        <div class="level-table">
          <button v-for="level in visibleLevels" :key="level.id" class="level-row" @click="openLevel(level)">
            <img v-if="thumbnailUrl(level.video)" class="level-thumb" :src="thumbnailUrl(level.video)" :alt="`${level.name} thumbnail`" />
            <span v-else class="level-thumb thumb-empty"></span>
            <span class="rank">{{ String(level.rank).padStart(2, '0') }}</span>
            <span class="level-name"><strong>{{ level.name }}</strong><small>{{ level.difficulty }} · {{ level.verifier }} verified</small></span>
            <span class="level-meta">{{ level.date || '—' }}<small>{{ level.nationality }}</small></span>
            <span class="arrow">↗</span>
          </button>
          <p v-if="!visibleLevels.length" class="empty-state">No levels found. Try a different search.</p>
        </div>
      </section>

      <section v-else class="moderator-page">
        <div class="moderator-intro"><div><h2>Manage levels</h2><p class="moderator-hint">Drag a row to change its place. Changes stay local until saved.</p></div><div class="moderator-actions-top"><button class="button button-coral" @click="newLevel">Add level</button><button class="button button-save" @click="save">Save changes</button></div></div>
        <transition-group name="admin-list" tag="div" class="admin-list">
          <div v-for="level in levels" :key="level.id" :data-level-id="level.id" :class="['admin-row', { 'is-dragging': draggedLevelId === level.id }]" @pointerdown="startPointerDrag(level, $event)" @dragover.prevent="dragOver(level)">
            <img v-if="thumbnailUrl(level.video)" class="admin-thumb" :src="thumbnailUrl(level.video)" :alt="`${level.name} thumbnail`" /><span v-else class="admin-thumb thumb-empty"></span>
            <div class="admin-rank">{{ String(level.rank).padStart(2, '0') }}</div>
            <div class="admin-title"><strong>{{ level.name || 'Untitled level' }}</strong><small>verified by {{ level.verifier || 'Unknown' }} · {{ level.difficulty }}</small></div>
            <div class="admin-meta">{{ level.date || '—' }}<small>{{ level.nationality || '—' }}</small></div>
            <div class="admin-actions"><button class="placement-action" @click.stop="openPlacement(level)">Change placement</button><button class="edit-action" @click.stop="startEdit(level)">Edit</button><button class="delete-action" @click.stop="removeLevel(level)">Remove</button></div>
          </div>
          </transition-group>
      </section>
    </main>
    <footer class="site-footer">
      <span class="footer-brand">GD bean news</span>
      <nav class="footer-links" aria-label="Social links">
        <a href="https://www.youtube.com/@GDBeanNews/videos" target="_blank" rel="noopener">YouTube</a>
        <a href="https://discord.gg/sj8YQKkjn" target="_blank" rel="noopener">GD bean news Discord</a>
        <a href="https://discord.gg/Xbm7DCQzj" target="_blank" rel="noopener">The Everything List Discord</a>
      </nav>
    </footer>

    <div v-if="placementLevel" class="placement-backdrop" @click.self="closePlacement">
      <section class="placement-modal" role="dialog" aria-modal="true" aria-label="Change level placement">
        <button class="close-button" aria-label="Close placement picker" @click="closePlacement">×</button>
        <p class="eyebrow">Change placement</p>
        <h2>{{ placementLevel.name }}</h2>
        <div class="placement-window" @wheel="placementWheel" @pointerdown="placementMiddleDown">
          <div v-for="(entry, index) in placementLevels" :key="entry.item?.id || `empty-${index}`" :class="['placement-row', { selected: entry.item?.id === placementLevel.id, 'placement-empty': !entry.item, 'placement-edge': index === 0 || index === placementLevels.length - 1 }]">
            <template v-if="entry.item">
              <strong>#{{ entry.item.id === placementLevel.id ? placementRank : entry.rank }}</strong><span>{{ entry.item.name }}</span><small v-if="entry.item.id === placementLevel.id">new position</small>
            </template>
          </div>
        </div>
        <p class="placement-position">New position: <strong>#{{ placementRank }}</strong></p>
        <div class="placement-actions"><button class="button button-quiet" type="button" @click="closePlacement">Cancel</button><button class="button button-coral" type="button" @click="confirmPlacement">OK</button></div>
      </section>
    </div>

    <div v-if="selectedLevel" class="drawer-backdrop" @click.self="closeLevel">
      <article class="level-drawer" role="dialog" aria-modal="true" aria-label="Level details">
        <button class="close-button" aria-label="Close level details" @click="closeLevel">×</button>
        <div class="detail-hero">
          <img v-if="thumbnailUrl(selectedLevel.video)" :src="thumbnailUrl(selectedLevel.video)" :alt="`${selectedLevel.name} thumbnail`" />
          <div class="detail-hero-shade"></div>
          <div class="drawer-top"><span class="drawer-rank">#{{ String(selectedLevel.rank).padStart(2, '0') }}</span><span class="drawer-badge">{{ selectedLevel.difficulty }}</span></div>
        </div>
        <h2>{{ selectedLevel.name }}</h2><p class="drawer-by">Verified by {{ selectedLevel.verifier }} <span v-if="selectedLevel.topOne">· Top 1</span></p>
        <div class="drawer-grid"><div><span class="label">Verifier nationality</span><strong>{{ selectedLevel.nationality }}</strong></div><div><span class="label">Verification date</span><strong>{{ selectedLevel.date || '—' }}</strong></div></div>
        <p class="drawer-description">{{ selectedLevel.description }}</p>
        <div v-if="selectedLevel.video" class="video-frame"><iframe :src="embedUrl(selectedLevel.video)" title="Level verification video" allowfullscreen></iframe></div>
      </article>
    </div>

    <div v-if="editing" class="modal-backdrop" @click.self="editing = null">
      <form class="edit-modal" role="dialog" aria-modal="true" aria-label="Edit level" @submit.prevent="saveLevel"><div class="modal-heading"><p class="eyebrow">List editor</p><button type="button" class="close-button" @click="editing = null">×</button><h2>{{ editing.name ? 'Edit level' : 'Add a level' }}</h2></div>
        <div class="form-grid">
          <label>Level name<input v-model="editing.name" required /></label>
          <label>Creator<input v-model="editing.creator" required /></label>
          <label>Verifier<input v-model="editing.verifier" required /></label>
          <label>Nationality<input v-model="editing.nationality" required /></label>
          <label>Difficulty<input v-model="editing.difficulty" required /></label>
          <label>Verification date<input v-model="editing.date" placeholder="e.g. May 24th 2025" /></label>
          <label class="wide">Tags <small>separate with commas</small><input v-model="editing.tagsText" placeholder="normal, pre 1.9, self imposed challenge" /></label>
          <label class="checkbox-field"><input v-model="editing.topOne" type="checkbox" /> Mark as Top 1</label>
          <label class="wide">Description<textarea v-model="editing.description" rows="3"></textarea></label>
          <label class="wide">Video embed URL<input v-model="editing.video" type="url" /></label>
        </div>
        <div class="modal-footer"><button type="button" class="button button-quiet" @click="editing = null">Cancel</button><button class="button button-dark" type="submit">Save level</button></div>
      </form>
    </div>
    <transition name="toast"><div v-if="toast" class="toast">{{ toast }} ✓</div></transition>
  </div>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Mono:wght@400;500&family=DM+Sans:wght@400;500;700&family=Space+Grotesk:wght@500;600;700&display=swap');
:root{--ink:#182125;--paper:#f4f0e8;--muted:#757b75;--line:#d9d5cc;--coral:#ef765f;--lime:#d9ed58;--blue:#b9d8db}*{box-sizing:border-box}body{margin:0;background:var(--paper);color:var(--ink);font-family:'DM Sans',sans-serif}button,input,textarea{font:inherit}button{cursor:pointer;color:inherit}.app-shell{min-height:100vh;overflow:hidden}.site-header{height:78px;border-bottom:1px solid var(--line);display:flex;align-items:center;justify-content:space-between;padding:0 5vw;position:relative;z-index:2}.brand{border:0;background:none;display:flex;align-items:center;gap:10px;font-family:'Space Grotesk';font-weight:700;font-size:15px}.brand-mark{display:grid;place-items:center;width:27px;height:27px;background:var(--coral);border-radius:50%;font-size:19px}.site-header nav{display:flex;gap:30px;align-items:center}.site-header nav button{border:0;background:none;padding:29px 0 25px;font-size:13px;color:var(--muted);position:relative}.site-header nav button.active,.site-header nav button:hover{color:var(--ink)}.site-header nav button.active:after{content:'';position:absolute;bottom:-1px;left:0;right:0;height:3px;background:var(--coral)}.key{font:10px 'DM Mono';background:#e7e3da;padding:3px 5px;margin-left:4px}.header-note{font:11px 'DM Mono';color:var(--muted);letter-spacing:.08em}.home-page{min-height:calc(100vh - 78px);padding:9vh 10vw 35px;display:grid;grid-template-columns:1fr 1fr;position:relative}.home-copy{align-self:center;z-index:1}.eyebrow{font:11px 'DM Mono';text-transform:uppercase;letter-spacing:.13em;color:var(--muted);margin:0 0 28px}.home-copy h1{font:clamp(65px,9vw,136px);font-family:'Space Grotesk';line-height:.82;letter-spacing:-.09em;margin:0 0 35px;font-weight:600}.home-copy h1 em,.page-heading h2 em,.moderator-intro h2 em{font-family:Georgia,serif;font-weight:400;letter-spacing:-.08em}.dot{color:var(--coral)}.intro{max-width:360px;line-height:1.6;color:#5d655e;font-size:15px}.home-actions{display:flex;align-items:center;gap:25px;margin-top:36px}.button{border:0;padding:14px 18px;font-weight:700;font-size:13px;display:inline-flex;gap:20px;align-items:center}.button-dark{background:var(--ink);color:var(--paper)}.button-dark:hover{background:#34434a}.button-coral{background:var(--coral);color:#fff}.button-quiet{background:transparent;border:1px solid var(--line)}.micro-copy{font:10px 'DM Mono';line-height:1.5;color:var(--muted)}.micro-copy strong{color:var(--ink)}.home-art{position:relative;min-height:540px}.art-sun{position:absolute;width:230px;height:230px;background:var(--lime);border-radius:50%;top:8%;right:13%}.art-ring{position:absolute;border:1px solid var(--ink);border-radius:50%}.ring-one{width:370px;height:370px;top:-5%;right:-6%}.ring-two{width:455px;height:455px;top:-14%;right:-17%;border-color:#abb1a6}.art-card{position:absolute;font-family:'Space Grotesk';box-shadow:9px 11px 0 var(--ink)}.card-back{background:var(--blue);color:#3b5357;width:205px;height:280px;right:26%;top:30%;padding:24px;font-size:38px;line-height:.82;transform:rotate(-8deg)}.card-front{background:var(--coral);width:215px;height:295px;right:4%;top:26%;padding:21px;display:flex;flex-direction:column;justify-content:space-between;transform:rotate(7deg)}.card-front span,.card-front small{font:11px 'DM Mono'}.card-front strong{font-size:38px;line-height:.82;letter-spacing:-.08em}.art-caption{position:absolute;bottom:12%;left:10%;font:10px 'DM Mono';line-height:1.45;transform:rotate(-7deg)}.home-footer{position:absolute;bottom:30px;left:10vw;right:10vw;border-top:1px solid var(--line);padding-top:13px;display:flex;justify-content:space-between;font:10px 'DM Mono';color:var(--muted)}.list-page,.moderator-page{max-width:1100px;margin:auto;padding:9vh 5vw 70px}.page-heading,.moderator-intro{display:flex;justify-content:space-between;align-items:flex-end;margin-bottom:70px}.page-heading h2,.moderator-intro h2{font:600 clamp(56px,7vw,94px)/.84 'Space Grotesk';letter-spacing:-.09em;margin:0}.list-meta{display:grid;gap:9px;text-align:right;font:10px 'DM Mono';color:var(--muted)}.list-meta strong{color:var(--ink);font-weight:400}.list-toolbar{display:flex;justify-content:space-between;align-items:center;border-bottom:1px solid var(--ink);padding-bottom:15px;margin-bottom:0}.search-box{display:flex;align-items:center;gap:10px}.search-box span{font-size:25px;line-height:1}.search-box input{border:0;background:transparent;outline:0;font-size:14px;width:260px;color:var(--ink)}.toolbar-count,.list-footnote{font:10px 'DM Mono';color:var(--muted)}.level-row{display:grid;grid-template-columns:75px 1fr 160px 90px 30px;align-items:center;gap:10px;width:100%;border:0;border-bottom:1px solid var(--line);background:transparent;text-align:left;padding:22px 0}.level-row:hover{background:#ebe7dd;margin:0 -20px;padding-left:20px;padding-right:20px;width:calc(100% + 40px)}.rank,.admin-rank,.drawer-rank{font:12px 'DM Mono';color:var(--coral)}.level-name{display:grid;gap:5px}.level-name strong{font:21px 'Space Grotesk';letter-spacing:-.04em}.level-name small,.admin-title small{font-size:11px;color:var(--muted)}.level-type{font:10px 'DM Mono';color:var(--muted);text-transform:uppercase}.level-rating{font:14px 'Space Grotesk';font-weight:700}.level-rating small{font:10px;color:var(--muted);font-weight:400}.arrow{font-size:22px}.list-footnote{margin-top:35px}.drawer-backdrop,.modal-backdrop{position:fixed;inset:0;background:rgba(24,33,37,.58);z-index:10;display:flex;justify-content:flex-end}.level-drawer{background:var(--paper);width:min(690px,100%);height:100%;padding:7vh 6vw 5vh;overflow:auto;position:relative;animation:slide-in .3s ease}.close-button{position:absolute;right:28px;top:23px;border:0;background:none;font-size:30px;font-weight:300}.drawer-top{display:flex;justify-content:space-between;align-items:center;margin-bottom:42px}.drawer-badge{font:10px 'DM Mono';border:1px solid var(--line);padding:7px 9px;text-transform:uppercase}.level-drawer h2{font:600 clamp(48px,7vw,82px)/.85 'Space Grotesk';letter-spacing:-.08em;margin:0}.drawer-by{color:var(--muted);margin:20px 0 45px}.drawer-by strong{color:var(--ink)}.drawer-grid{border-top:1px solid var(--line);border-bottom:1px solid var(--line);display:grid;grid-template-columns:1fr 1fr;padding:23px 0;margin-bottom:28px}.drawer-grid div{display:grid;gap:6px}.drawer-grid div+div{border-left:1px solid var(--line);padding-left:24px}.label{font:10px 'DM Mono';color:var(--muted);text-transform:uppercase}.drawer-grid strong{font:20px 'Space Grotesk'}.drawer-grid small{font-size:11px;color:var(--muted)}.flag{font-size:17px}.drawer-description{font-family:Georgia,serif;font-size:19px;line-height:1.5;max-width:510px}.video-frame{aspect-ratio:16/9;background:#d1cec4;margin:32px 0}.video-frame iframe{width:100%;height:100%;border:0}.tag-list{display:flex;gap:7px;flex-wrap:wrap}.tag-list span{font:10px 'DM Mono';background:var(--lime);padding:8px 10px;text-transform:uppercase}.moderator-page{max-width:1200px}.moderator-intro{margin-bottom:38px}.moderator-bar{background:var(--ink);color:var(--paper);padding:13px 18px;display:flex;justify-content:space-between;font:10px 'DM Mono';margin-bottom:20px}.status-dot{display:inline-block;width:7px;height:7px;border-radius:50%;background:var(--lime);margin-right:8px}.admin-list{border-top:1px solid var(--ink)}.admin-row{display:grid;grid-template-columns:75px 1fr auto;align-items:center;border-bottom:1px solid var(--line);padding:18px 0}.admin-title{display:grid;gap:5px}.admin-title strong{font:19px 'Space Grotesk'}.admin-actions{display:flex;gap:5px}.admin-actions button{border:1px solid var(--line);background:transparent;padding:8px 11px;font-size:11px}.admin-actions button:hover{background:#e8e4da}.admin-actions .edit-action{background:var(--blue);border-color:var(--blue);font-weight:700}.admin-actions .delete-action{color:#b64a3c}.modal-backdrop{align-items:center;justify-content:center;padding:20px}.edit-modal{background:var(--paper);width:min(690px,100%);padding:35px 40px;position:relative;box-shadow:10px 10px 0 var(--coral)}.modal-heading{border-bottom:1px solid var(--line);margin-bottom:25px}.modal-heading .eyebrow{margin-bottom:13px}.modal-heading h2{font:600 38px 'Space Grotesk';letter-spacing:-.06em;margin:0 0 25px}.modal-heading .close-button{top:0;right:0}.form-grid{display:grid;grid-template-columns:1fr 1fr;gap:16px}.form-grid label{display:grid;gap:7px;font:10px 'DM Mono';text-transform:uppercase}.form-grid input,.form-grid textarea{width:100%;border:1px solid var(--line);background:#faf8f2;padding:11px;outline-color:var(--coral);font:14px 'DM Sans';text-transform:none}.form-grid .wide{grid-column:1/-1}.modal-footer{display:flex;justify-content:flex-end;gap:10px;margin-top:27px}.toast{position:fixed;bottom:25px;left:50%;transform:translateX(-50%);background:var(--ink);color:var(--paper);padding:12px 18px;font:11px 'DM Mono';z-index:20}.toast-enter-active,.toast-leave-active{transition:.3s}.toast-enter-from,.toast-leave-to{opacity:0;transform:translate(-50%,10px)}@keyframes slide-in{from{transform:translateX(40px);opacity:0}to{transform:none;opacity:1}}
.home-page{min-height:calc(100vh - 78px);display:grid;grid-template-columns:minmax(0,1.35fr) minmax(260px,.65fr);grid-template-rows:1fr auto auto;padding:9vh 10vw 30px;gap:55px 9vw}
.home-intro{align-self:center;max-width:680px}.home-intro h1{font:600 clamp(45px,6vw,78px)/.95 'Space Grotesk';letter-spacing:-.07em;margin:0 0 25px}.home-intro h1 span{color:var(--coral)}.home-intro .intro{max-width:480px;margin-bottom:30px}.home-feature{align-self:center;border-left:2px solid var(--coral);padding:8px 0 8px 28px;max-width:320px}.feature-label{font:10px 'DM Mono';text-transform:uppercase;color:var(--muted);letter-spacing:.1em;margin-bottom:26px}.home-feature p{font:21px/1.35 Georgia,serif;margin:0 0 28px}.feature-rule{border-top:1px solid var(--line);margin-bottom:22px}.feature-stat{display:flex;align-items:center;gap:12px}.feature-stat strong{font:36px 'Space Grotesk';color:var(--lime)}.feature-stat span{font:10px/1.4 'DM Mono';color:var(--muted);text-transform:uppercase}.home-community{grid-column:1/-1;border-top:1px solid var(--line);padding-top:24px;display:flex;justify-content:space-between;align-items:flex-end}.home-community .eyebrow{margin-bottom:10px}.home-community strong{font:22px 'Space Grotesk'}.community-copy{color:var(--muted);font-size:13px;margin:7px 0 0}.social-links{display:flex;gap:25px}.social-links a{color:var(--ink);font:11px 'DM Mono';text-decoration:none;border-bottom:1px solid var(--muted);padding-bottom:4px}.social-links a:hover{color:var(--coral);border-color:var(--coral)}.home-footer{position:static;grid-column:1/-1;border-top:0;padding-top:0}.page-heading h2,.moderator-intro h2{font-size:48px;letter-spacing:-.06em}.page-heading,.moderator-intro{margin-bottom:45px}
@media(max-width:700px){.site-header{padding:0 20px;height:67px}.site-header nav{gap:13px}.site-header nav button{font-size:11px;padding:24px 0 21px}.key,.header-note{display:none}.home-page{display:block;padding:60px 25px 80px}.home-copy h1{font-size:76px}.home-art{min-height:430px;margin-top:25px}.art-sun{width:160px;height:160px;right:20%}.ring-one{width:290px;height:290px;right:-10%}.ring-two{width:350px;height:350px;right:-20%}.card-back{width:145px;height:200px;font-size:27px;right:31%;top:25%}.card-front{width:155px;height:220px;right:5%;top:20%}.card-front strong{font-size:28px}.home-footer{left:25px;right:25px;bottom:20px}.page-heading,.moderator-intro{display:block;margin-bottom:45px}.page-heading h2,.moderator-intro h2{font-size:61px}.list-meta{text-align:left;margin-top:25px}.list-toolbar{display:block}.toolbar-count{display:block;margin-top:15px}.search-box input{width:calc(100vw - 100px)}.level-row{grid-template-columns:40px 1fr 30px;padding:18px 0}.level-type,.level-rating{display:none}.level-name strong{font-size:18px}.list-page,.moderator-page{padding:55px 25px}.moderator-intro .button{margin-top:30px}.moderator-bar{display:block;line-height:2}.admin-row{grid-template-columns:40px 1fr}.admin-actions{grid-column:2;justify-content:flex-start;margin-top:14px;flex-wrap:wrap}.edit-modal{padding:28px 22px}.form-grid{grid-template-columns:1fr}.form-grid .wide{grid-column:auto}.level-drawer{padding:60px 25px 35px}.drawer-grid{gap:10px}.drawer-grid div+div{padding-left:12px}.drawer-grid strong{font-size:17px}}
.home-community{flex-direction:row}.home-footer{display:flex}
@media(max-width:700px){.home-page{display:flex;flex-direction:column;gap:42px;padding:55px 25px 30px}.home-intro h1{font-size:48px}.home-feature{max-width:none}.home-community{flex-direction:column;align-items:flex-start;gap:22px}.social-links{flex-wrap:wrap;gap:15px}.home-footer{display:flex;gap:12px;flex-direction:column}}
.home-page{min-height:calc(100vh - 78px);display:flex;flex-direction:column;justify-content:center;max-width:760px;margin:auto;padding:9vh 30px 35px;gap:70px}
.home-intro{max-width:580px}
.home-intro h1{font:600 clamp(38px,5vw,64px)/1 'Space Grotesk';letter-spacing:-.06em;margin:0 0 24px}
.home-intro h1 span{color:inherit}
.home-note{align-self:center;border-left:1px solid var(--line);padding-left:24px;color:#aab1aa;font-size:14px;line-height:1.6}
.home-note .eyebrow{margin-bottom:20px}
.home-note p{margin:0 0 20px}
.home-count{font:10px 'DM Mono';color:var(--coral);text-transform:uppercase}
.home-community{width:100%;border-top:1px solid var(--line);padding-top:22px}
.home-footer{position:static;left:auto;right:auto;bottom:auto;border-top:0;padding-top:0}
.page-heading h2,.moderator-intro h2{font-size:42px}
@media(max-width:700px){.home-page{gap:38px}.home-note{align-self:stretch}.home-intro h1{font-size:44px}}
.list-page{max-width:none;min-height:calc(100vh - 78px);padding:48px max(25px,calc((100vw - 930px) / 2)) 70px;background-color:#101416;background-image:linear-gradient(45deg,rgba(241,239,231,.07) 1px,transparent 1px),linear-gradient(-45deg,rgba(241,239,231,.07) 1px,transparent 1px);background-size:24px 24px}
.list-search{display:flex;align-items:center;gap:10px;width:100%;height:48px;background:#182334;border:1px solid #33435a;border-radius:7px;padding:0 15px;margin-bottom:20px}
.list-search span{font:18px 'DM Mono';color:#71819b}.list-search input{flex:1;width:auto;color:var(--ink);font-size:13px}.list-search small{font:10px 'DM Mono';color:#8d9bb0}
.level-table{display:grid;gap:10px;background:transparent;border:0}
.level-row{display:grid;grid-template-columns:190px 38px minmax(0,1fr) 145px 25px;align-items:center;gap:18px;width:100%;border:1px solid #304056;border-radius:13px;background:#1b2738;padding:14px;text-align:left;box-shadow:0 4px 12px rgba(0,0,0,.14)}
.level-row:hover{width:100%;margin:0;background:#223149;border-color:#526c92;padding-left:14px;padding-right:14px}
.level-thumb{width:190px;height:92px;object-fit:cover;border-radius:8px;background:#111923}
.thumb-empty{display:block;background:linear-gradient(135deg,#27364a,#101820)}
.level-row .rank{font:13px 'DM Mono';color:#70a9ff}
.level-row .level-name strong{font-size:18px;letter-spacing:-.02em}.level-row .level-name small{font-size:11px;color:#9aabc2;margin-top:7px}
.level-rating{font:11px 'DM Mono';color:#72abff;text-align:right}.level-rating small{display:block;color:#91a0b5;margin-top:7px}
.level-row .arrow{color:#83b4ff;font-size:20px}
@media(max-width:700px){.list-page{padding:35px 15px 55px}.level-row{grid-template-columns:80px 30px minmax(0,1fr) 22px;gap:11px;padding:10px}.level-row:hover{padding-left:10px;padding-right:10px}.level-thumb{width:80px;height:58px}.level-row .level-name strong{font-size:15px}.level-row .level-name small{font-size:9px}.level-rating{display:none}}
.list-page{max-width:1180px;padding:55px 5vw 80px}
.ranking-head{display:flex;justify-content:space-between;align-items:flex-end;border-bottom:1px solid var(--line);padding-bottom:24px;margin-bottom:24px}
.section-kicker,.rail-label{font:10px 'DM Mono';letter-spacing:.11em;color:var(--muted)}
.ranking-head h2{font:500 38px 'Space Grotesk';letter-spacing:-.06em;margin:10px 0 0}
.ranking-summary{display:flex;align-items:center;gap:10px;text-align:left}
.ranking-summary strong{font:32px 'Space Grotesk';color:var(--coral)}
.ranking-summary span{font:10px/1.3 'DM Mono';color:var(--muted);text-transform:uppercase}
.ranking-layout{display:grid;grid-template-columns:220px 1fr;gap:30px;align-items:start}
.ranking-rail{padding-top:1px}
.ranking-rail .search-box{display:flex;border:1px solid var(--line);background:#151b1d;padding:11px 12px;min-height:42px}
.ranking-rail .search-box input{font-size:12px;width:100%}
.rail-rule{border-top:1px solid var(--line);margin:32px 0 24px}
.ranking-rail p{color:#aab1aa;font-size:12px;line-height:1.55;margin:11px 0 30px}
.rail-count{display:block;font:20px 'Space Grotesk';margin-top:9px}
.ranking-layout .level-table{border:1px solid var(--line);border-top:2px solid var(--coral);background:var(--line)}
.ranking-layout .level-table-head,.ranking-layout .level-row{grid-template-columns:55px minmax(220px,1fr) 125px 70px 25px;column-gap:15px}
.ranking-layout .level-table-head{background:#0c1011;padding:13px 16px}
.ranking-layout .level-row{background:#151b1d;padding:19px 16px}
.ranking-layout .level-row:hover{background:#202a2c}
.ranking-layout .level-name strong{font-size:17px}
.ranking-layout .level-name small{font-size:10px}
.ranking-layout .level-creator{font-size:12px;color:#c2c8c2}
.ranking-layout .level-creator small{font-size:14px}
.ranking-layout .level-rating{font-size:15px;color:var(--lime)}
.ranking-layout .level-rating small{font:9px 'DM Mono';color:var(--muted);margin-left:2px}
@media(max-width:700px){.list-page{padding:42px 20px 60px}.ranking-head h2{font-size:32px}.ranking-layout{display:block}.ranking-rail{margin-bottom:20px}.ranking-rail .rail-rule,.ranking-rail p,.ranking-rail .rail-label:not(:first-of-type),.ranking-rail .rail-count{display:none}.ranking-layout .level-table-head{display:none}.ranking-layout .level-row{grid-template-columns:38px 1fr 24px;padding:16px 12px}.ranking-layout .level-creator,.ranking-layout .level-rating{display:none}}
.list-page{max-width:1120px;padding-top:6vh}
.list-page .list-toolbar{margin-top:0}
.list-toolbar{border:1px solid var(--line);background:#151b1d;padding:7px 12px;margin-bottom:18px;border-radius:3px}
.search-box{flex:1;border:0;min-height:34px}
.search-box span{font:18px 'DM Mono';color:var(--muted)}
.search-box input{width:100%;font-size:13px;color:var(--ink)}
.toolbar-count{padding-left:15px}
.level-table{border-top:1px solid var(--line);border-left:1px solid var(--line);border-right:1px solid var(--line)}
.level-table-head,.level-row{display:grid;grid-template-columns:70px minmax(220px,1.7fr) minmax(130px,1fr) 150px 75px 24px;align-items:center;column-gap:18px}
.level-table-head{padding:12px 18px;color:var(--muted);font:10px 'DM Mono';text-transform:uppercase;letter-spacing:.08em}
.level-list{display:grid;gap:2px;background:var(--line);border:1px solid var(--line)}
.level-row{border:0;border-bottom:1px solid var(--line);background:#151b1d;padding:17px 18px;text-align:left}
.level-row:hover{width:100%;margin:0;background:#1b2527;padding-left:18px;padding-right:18px}
.level-name strong{font-size:18px}
.level-name small{font-size:10px}
.level-creator{font-size:13px;color:#bac0bb}
.level-type{font-size:9px}
.level-rating{font-size:13px}
.level-row .arrow{color:var(--coral)}
@media(max-width:700px){.list-page{padding-top:50px}.list-toolbar{margin-bottom:14px}.level-table-head{display:none}.level-row{grid-template-columns:39px 1fr 25px;padding:16px 12px}.level-row:hover{padding-left:12px;padding-right:12px}.level-name strong{font-size:17px}.level-creator,.level-type,.level-rating{display:none}}
/* Dark editorial theme */
:root{--ink:#f1efe7;--paper:#101416;--muted:#89918d;--line:#2d3638;--coral:#ff806c;--lime:#d9ed58;--blue:#1c3b42}
body,.app-shell{background:var(--paper);color:var(--ink)}
.brand-mark{display:none}
.intro{color:#aab1aa}
.key{background:#222b2d}
.button-dark{background:var(--ink);color:var(--paper)}
.button-dark:hover{background:#d9d6ce;color:#101416}
.button-coral{color:#171313}
.art-ring{border-color:var(--ink)}
.ring-two{border-color:#3b4848}
.art-card{box-shadow:9px 11px 0 #060809}
.card-back{background:var(--blue);color:#b9d8db}
.card-front{color:#171313}
.level-row:hover{background:#192022}
.drawer-backdrop,.modal-backdrop{background:rgba(3,7,8,.8)}
.level-drawer,.edit-modal{background:#151b1d}
.video-frame{background:#090c0d}
.tag-list span{color:#111712}
.moderator-bar{background:#e9e7dd;color:#111617}
.status-dot{background:#4b842f}
.admin-actions button:hover{background:#20282a}
.admin-actions .edit-action{color:var(--ink)}
.admin-actions .delete-action{color:#ff8c7a}
.form-grid input,.form-grid textarea{background:#0e1213;color:var(--ink)}
.list-page{max-width:980px;padding:48px 25px 70px}
.list-search{display:flex;align-items:center;gap:10px;width:100%;height:48px;background:#182334;border:1px solid #33435a;border-radius:7px;padding:0 15px;margin-bottom:20px}
.list-search span{font:18px 'DM Mono';color:#71819b}.list-search input{flex:1;width:auto;color:var(--ink);font-size:13px}.list-search small{font:10px 'DM Mono';color:#8d9bb0}
.level-table{display:grid;gap:10px;background:transparent;border:0}
.level-row{display:grid;grid-template-columns:190px 38px minmax(0,1fr) 145px 25px;align-items:center;gap:18px;width:100%;border:1px solid #304056;border-radius:13px;background:#1b2738;padding:14px;text-align:left;box-shadow:0 4px 12px rgba(0,0,0,.14)}
.level-row:hover{width:100%;margin:0;background:#223149;border-color:#526c92;padding-left:14px;padding-right:14px}
.level-thumb{width:190px;height:92px;object-fit:cover;border-radius:8px;background:#111923}
.thumb-empty{display:block;background:linear-gradient(135deg,#27364a,#101820)}
.level-row .rank{font:13px 'DM Mono';color:#70a9ff}
.level-row .level-name strong{font-size:18px;letter-spacing:-.02em}.level-row .level-name small{font-size:11px;color:#9aabc2;margin-top:7px}
.level-row .level-rating{font:11px 'DM Mono';color:#72abff;text-align:right}.level-row .level-rating small{display:block;color:#91a0b5;margin-top:7px}
.level-row .level-meta{font:11px 'DM Mono';color:#72abff;text-align:right}.level-row .level-meta small{display:block;color:#91a0b5;margin-top:7px}
.level-row .arrow{color:#83b4ff;font-size:20px}
@media(max-width:700px){.list-page{padding:35px 15px 55px}.level-row{grid-template-columns:80px 30px minmax(0,1fr) 22px;gap:11px;padding:10px}.level-row:hover{padding-left:10px;padding-right:10px}.level-thumb{width:80px;height:58px}.level-row .level-name strong{font-size:15px}.level-row .level-name small{font-size:9px}.level-row .level-rating{display:none}}
.app-shell{background-color:#101416}
.site-header{background:rgba(16,20,22,.88)}
.home-page{align-items:start;justify-items:start;justify-content:center;gap:55px;padding-top:15vh}
.home-intro,.home-community{width:100%;text-align:left}
.home-intro h1{margin-top:0}
.home-community{display:flex;justify-content:space-between;align-items:end}
.home-footer{display:none}
.social-links{justify-content:flex-end}
@media(max-width:700px){.home-page{padding-top:70px;gap:45px}.home-community{display:block}.social-links{justify-content:flex-start;margin-top:20px}}
.drawer-backdrop{align-items:center;justify-content:center;padding:28px}
.level-drawer{width:min(760px,100%);height:min(88vh,780px);border:1px solid #33445a;border-radius:14px;padding:0 32px 32px;background:#172233;box-shadow:0 18px 55px rgba(0,0,0,.45);overflow:auto}
.detail-hero{height:210px;margin:0 -32px 28px;position:relative;background:#101923;overflow:hidden}
.detail-hero img{width:100%;height:100%;object-fit:cover;opacity:.66}
.detail-hero-shade{position:absolute;inset:0;background:linear-gradient(180deg,rgba(8,14,22,.1),#172233)}
.detail-hero .drawer-top{position:absolute;inset:22px 28px auto;z-index:1;margin:0}
.level-drawer h2{font-size:42px;letter-spacing:-.055em;margin:0 0 11px}
.drawer-by{font:13px 'DM Mono';color:#a7b5c7;margin:0 0 27px}
.drawer-grid{border-top:1px solid #33445a;border-bottom:1px solid #33445a;padding:18px 0;margin-bottom:25px}
.drawer-grid strong{font-size:15px;color:#e7edf5}
.drawer-description{font:15px/1.55 'DM Sans';color:#b4c0cf;margin:0 0 25px}
.video-frame{border-radius:9px;overflow:hidden;margin:0;background:#0d141e}
.drawer-badge{background:#b9e7ff;border:0;color:#10202d;font-weight:700}
@media(max-width:700px){.drawer-backdrop{padding:10px}.level-drawer{height:94vh;padding:0 20px 24px}.detail-hero{margin:0 -20px 23px;height:155px}.detail-hero .drawer-top{inset:16px 18px auto}.level-drawer h2{font-size:32px}}
.home-page{width:min(100%,900px);height:calc(100vh - 150px);min-height:0;margin:0 auto;display:flex;flex-direction:column;justify-content:flex-start;align-items:stretch;padding:70px 40px 0;gap:70px;overflow:hidden}
.home-intro{width:100%;max-width:600px}
.home-community{width:100%;display:flex;justify-content:space-between;align-items:flex-end;border-top:1px solid var(--line);padding-top:22px}
.home-footer{display:block;margin-top:auto;border-top:1px solid var(--line);padding:18px 0 24px;color:var(--muted);font:10px 'DM Mono';width:100%}
.site-footer{height:72px;border-top:1px solid var(--line);display:flex;align-items:center;justify-content:space-between;padding:0 5vw;background:rgba(16,20,22,.9)}
.footer-brand{font:700 14px 'Space Grotesk'}.footer-links{display:flex;gap:25px}.footer-links a{font:10px 'DM Mono';color:var(--muted);text-decoration:none}.footer-links a:hover{color:var(--coral)}
.level-row{grid-template-columns:190px 38px minmax(0,1fr) 145px 25px}
.level-row .level-name small{display:block}
.level-meta{font:11px 'DM Mono';color:#72abff;text-align:right}.level-meta small{display:block;color:#91a0b5;margin-top:7px}
.admin-list{display:grid;gap:10px;border:0;background:transparent}
.admin-row{display:grid;grid-template-columns:150px 36px minmax(0,1fr) 125px auto;gap:16px;align-items:center;border:1px solid #304056;border-radius:12px;background:#1b2738;padding:12px 14px}
.admin-thumb{width:150px;height:72px;object-fit:cover;border-radius:7px;background:#111923}
.admin-title{display:grid;gap:6px;min-width:0}.admin-title strong{font-size:17px}.admin-title small{font-size:10px;color:#9aabc2}
.admin-meta{font:10px 'DM Mono';color:#72abff;text-align:right}.admin-meta small{display:block;color:#91a0b5;margin-top:7px}
.admin-actions{justify-content:flex-end}
.admin-row:hover{background:#223149;border-color:#526c92}
@media(max-width:700px){.home-page{width:100%;height:calc(100vh - 139px);padding:65px 25px 0;gap:50px}.site-footer{height:72px;padding:18px 25px;align-items:flex-start;gap:12px;flex-direction:column}.footer-links{gap:18px}.admin-row{grid-template-columns:80px 30px minmax(0,1fr);gap:10px;padding:10px}.admin-thumb{width:80px;height:58px}.admin-meta{display:none}.admin-actions{grid-column:3;justify-content:flex-start}.admin-row:hover{padding-left:10px;padding-right:10px}}
.home-footer{display:block}
.drawer-badge{background:#b9e7ff;border-color:#b9e7ff}
.site-header nav button.active:after{background:#8ed8ff}
.list-search:focus-within{border-color:#8ed8ff}
.button-coral{background:#8ed8ff;color:#10202d;border-radius:7px}
.rank,.admin-rank,.drawer-rank{color:#8ed8ff}
.dot{color:#8ed8ff}
.level-row .arrow,.admin-actions .delete-action{color:#8ed8ff}
.admin-actions button{border-radius:6px}
.moderator-intro{margin-bottom:28px}.moderator-intro h2{font-size:38px;letter-spacing:-.05em}.moderator-hint{color:var(--muted);font-size:12px;margin:10px 0 0}.moderator-actions-top{display:flex;gap:9px;align-items:center}.button-save{background:#26384b;color:#dcecff;border:1px solid #527296;border-radius:7px}.button-save:hover{background:#314b66}
.admin-list{padding-top:4px}.admin-rank input{width:42px;border:1px solid #526c92;border-radius:5px;background:#111923;color:#8ed8ff;padding:7px 5px;text-align:center;font:12px 'DM Mono'}
.admin-row{position:relative;cursor:grab;transition:transform .28s cubic-bezier(.2,.8,.2,1),opacity .28s ease,box-shadow .28s ease,border-color .28s ease}.admin-row:active{cursor:grabbing}.admin-row.is-dragging{opacity:1;transform:scale(1.02) translateY(-4px);z-index:2;box-shadow:0 14px 28px rgba(0,0,0,.38),0 0 0 2px #ef765f}.admin-row.is-drop-target{border-color:#ef765f;transform:translateY(5px);box-shadow:0 5px 0 rgba(239,118,95,.35)}
html{scrollbar-width:none}::-webkit-scrollbar{width:0;height:0}
.site-header nav button.active:after{background:#ef765f}
.list-search:focus-within{border-color:#ef765f}
.button-coral{background:#ef765f;color:#171313}
.rank,.admin-rank,.drawer-rank{color:#ef765f}
.level-row .arrow,.admin-actions .delete-action{color:#ef765f}
.level-table{border-top-color:#ef765f}
.admin-list{gap:12px}
.admin-row{grid-template-columns:180px 52px minmax(180px,1fr) 135px max-content;min-height:108px;padding:14px 16px;border-radius:12px}
.admin-thumb{width:180px;height:84px}
.admin-rank{display:flex;align-items:center;justify-content:center;font:13px 'DM Mono';color:#ef765f}
.admin-title strong{font-size:18px}
.admin-actions{display:flex;align-items:center;justify-content:flex-end;gap:6px;min-width:max-content;white-space:nowrap}
.admin-actions .edit-action{background:#ef765f;border-color:#ef765f;color:#171313}
.admin-actions button{min-width:30px;padding:8px 10px}
.admin-row.is-dragging{box-shadow:0 0 0 2px #ef765f}
.admin-row.is-drop-target{border-color:#ef765f;box-shadow:0 5px 0 rgba(239,118,95,.35)}
.placement-action{color:#dcecff!important;border-color:#527296!important;background:#26384b!important;white-space:nowrap}
.placement-action:hover{background:#314b66!important}
.placement-backdrop{position:fixed;inset:0;z-index:12;background:rgba(3,7,8,.78);display:grid;place-items:center;padding:20px}
.placement-modal{position:relative;width:min(520px,100%);background:#172233;border:1px solid #405775;border-radius:14px;padding:32px;box-shadow:0 24px 70px rgba(0,0,0,.5)}
.placement-modal .eyebrow{margin-bottom:14px}.placement-modal h2{font:600 30px 'Space Grotesk';letter-spacing:-.05em;margin:0 0 8px}
.placement-window{display:grid;gap:5px;max-height:250px;overflow:hidden;cursor:grab;user-select:none}
.placement-window:active{cursor:grabbing}
.placement-row{display:grid;grid-template-columns:45px 1fr auto;align-items:center;gap:10px;text-align:left;border:1px solid transparent;border-radius:7px;background:#1d2b3e;color:var(--ink);padding:11px 13px;min-height:42px;transition:background .25s ease,border-color .25s ease,transform .25s ease,opacity .25s ease}
.placement-row:hover{background:#263b54;border-color:#6e8eb7;transform:translateX(3px)}.placement-row.selected{background:#ef765f;border-color:#ff9b8b;color:#171313}.placement-row strong{font:12px 'DM Mono';color:#8ed8ff}.placement-row.selected strong{color:#171313}.placement-row span{font:14px 'Space Grotesk'}.placement-row small{font:9px 'DM Mono';text-transform:uppercase}
.placement-row.placement-edge{opacity:.48}.placement-row.placement-empty{visibility:hidden}.placement-position{margin:15px 0 0;color:#9aabc2;font:11px 'DM Mono'}.placement-position strong{color:#ef765f}.placement-actions{display:flex;justify-content:flex-end;gap:8px;margin-top:20px}.placement-actions .button{padding:10px 16px;font-size:11px}
@media(max-width:980px){
  .site-header{padding:0 24px}
  .site-header nav{gap:18px}
  .moderator-page{padding-left:24px;padding-right:24px}
  .admin-row{grid-template-columns:150px 45px minmax(150px,1fr) 110px max-content;gap:10px}
  .admin-thumb{width:150px;height:76px}
  .admin-actions button{padding:8px 8px}
}
@media(max-width:700px){
  .site-header{height:auto;min-height:78px;align-items:flex-start;gap:18px;padding:18px 16px}
  .brand{font-size:14px;line-height:1.15}
  .site-header nav{flex:1;justify-content:flex-end;gap:12px;flex-wrap:wrap}
  .site-header nav button{padding:10px 0 8px;font-size:11px}
  .list-page,.moderator-page{padding:32px 14px 52px}
  .list-search{margin-bottom:14px}
  .level-row{grid-template-columns:72px 26px minmax(0,1fr) 20px;gap:9px;padding:9px}
  .level-thumb{width:72px;height:54px}
  .level-row .level-name strong{font-size:14px;line-height:1.1}
  .level-row .level-name small{font-size:9px;line-height:1.25}
  .level-row .level-meta{display:none}
  .moderator-intro{display:flex;align-items:flex-start;gap:18px;margin-bottom:22px}
  .moderator-intro h2{font-size:30px}
  .moderator-hint{font-size:11px;line-height:1.4}
  .moderator-actions-top{flex:0 0 auto;display:grid;gap:6px}
  .moderator-actions-top .button{padding:10px 11px;font-size:11px;white-space:nowrap}
  .admin-row{grid-template-columns:82px 24px minmax(0,1fr);gap:9px;min-height:0;padding:10px}
  .admin-thumb{width:82px;height:58px}
  .admin-rank{font-size:10px}
  .admin-title strong{font-size:15px;line-height:1.1}
  .admin-title small{font-size:9px;line-height:1.25}
  .admin-meta{display:none}
  .admin-actions{grid-column:1 / -1;justify-content:flex-start;flex-wrap:wrap;min-width:0;padding-top:2px}
  .admin-actions button{font-size:10px;padding:8px 9px}
  .placement-modal{padding:26px 20px}
  .placement-modal h2{font-size:25px;padding-right:20px}
  .edit-modal{padding:26px 20px}
  .form-grid{grid-template-columns:1fr!important}
  .site-footer{height:auto;min-height:88px;padding:18px 16px;align-items:flex-start;gap:12px}
  .footer-links{flex-wrap:wrap;gap:10px 16px}
}
.drawer-badge,.site-header nav button.active:after,.list-search:focus-within,.button-coral,.level-table{border-color:#ef765f}
.site-header nav button.active:after{background:#ef765f}
.button-coral{background:#ef765f}
.rank,.admin-rank,.drawer-rank,.dot,.level-row .arrow,.admin-actions .delete-action{color:#ef765f}
.admin-row.is-dragging{box-shadow:0 0 0 2px #ef765f}
.admin-row.is-drop-target{border-color:#ef765f;box-shadow:0 5px 0 rgba(239,118,95,.35)}
.edit-modal{background:#172233;width:min(760px,100%);padding:32px 36px;position:relative;border:1px solid #344b68;border-radius:14px;box-shadow:0 20px 60px rgba(0,0,0,.48)}
.form-grid input,.form-grid textarea{border-radius:6px;border-color:#344b68;background:#111923}
.form-grid label{color:#91a5bf}
.modal-heading{border-color:#344b68}
.modal-heading h2{font-size:32px;letter-spacing:-.04em}
.admin-list-move{transition:transform .32s cubic-bezier(.2,.8,.2,1)}
.admin-row.is-dragging{opacity:1!important;transform:scale(1.025) translateY(-6px);z-index:5;box-shadow:0 18px 36px rgba(0,0,0,.48),0 0 0 2px var(--coral)}
.form-grid label small{color:#71819b;font:10px 'DM Mono';font-weight:400}.checkbox-field{display:flex;align-items:center;gap:8px}.checkbox-field input{width:auto!important;accent-color:#ef765f}
</style>
