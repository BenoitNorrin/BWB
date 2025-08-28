<script setup lang="ts">
import { onMounted, onUnmounted, reactive, ref, watch, computed } from 'vue'

type Phase = 'idle' | 'filling' | 'emptying' | 'paused' | 'series-pause' | 'finished'

const settings = reactive({
	breathsPerMinute: 10, // breaths per minute
	numberOfSeries: 5, // number of series
	pauseSeconds: 30, // pause between series in seconds
	workTimeSeconds: 30, // work time in seconds per series
})

const phase = ref<Phase>('idle')
const fillPercent = ref(0) // 0 à 100
const currentSeries = ref(0)
const breathsInCurrentSeries = ref(0)
const remainingTime = ref(0) // temps restant en secondes
const currentTime = ref(0) // temps actuel pour forcer la mise à jour
const lastPhase = ref<Phase>('idle') // mémoriser la dernière phase avant pause

let rafId: number | null = null
let lastTs = 0
let seriesStartTs: number | null = null

// Calcul du temps restant pour l'affichage
const remainingTimeDisplay = computed(() => {
	if (phase.value === 'idle' || phase.value === 'finished') return ''
	
	// Pour la pause entre séries
	if (phase.value === 'series-pause') {
		return Math.ceil(remainingTime.value)
	}
	
	// Pour les phases de travail (filling/emptying)
	if ((phase.value === 'filling' || phase.value === 'emptying') && seriesStartTs != null) {
		if (settings.workTimeSeconds > 0) {
			const elapsed = (currentTime.value - seriesStartTs) / 1000
			const remaining = Math.max(0, settings.workTimeSeconds - elapsed)
			return Math.ceil(remaining)
		}
	}
	
	return ''
})

// Phase display translation
const phaseDisplay = computed(() => {
	switch (phase.value) {
		case 'filling': return 'Inhale'
		case 'emptying': return 'Exhale'
		case 'paused': return 'Paused'
		case 'series-pause': return 'Series Break'
		case 'finished': return 'Finished'
		case 'idle': return 'Ready'
		default: return phase.value
	}
})

// One breath = fill + empty
function getHalfCycleMs(): number {
	const breathsPerSecond = settings.breathsPerMinute / 60
	const cycleMs = 1000 / breathsPerSecond // ms pour 1 respiration complète
	return cycleMs / 2 // demi-cycle = remplissage OU vidage
}

// Simple beep via Web Audio API
let audioCtx: AudioContext | null = null
function beep(freq = 880, ms = 100): void {
	try {
		if (!audioCtx) audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)()
		const ctx = audioCtx
		if (!ctx) return
		const osc = ctx.createOscillator()
		const gain = ctx.createGain()
		osc.type = 'sine'
		osc.frequency.value = freq
		gain.gain.value = 0.05
		osc.connect(gain)
		gain.connect(ctx.destination)
		osc.start()
		setTimeout(() => {
			osc.stop()
			osc.disconnect()
			gain.disconnect()
		}, ms)
	} catch { /* ignore */ }
}

function start(): void {
	if (phase.value === 'finished') reset()
	if (settings.breathsPerMinute <= 0) return
	currentSeries.value = currentSeries.value || 1
	breathsInCurrentSeries.value = breathsInCurrentSeries.value || 0
	
	if (phase.value === 'paused') {
		// Reprendre la phase précédente
		phase.value = lastPhase.value
	} else if (phase.value === 'series-pause' || phase.value === 'idle') {
		// Commencer une nouvelle série
		phase.value = 'filling'
		fillPercent.value = 0
		// Initialiser le temps de début de série
		if (seriesStartTs == null) {
			seriesStartTs = performance.now()
		}
	}
	
	loop()
}

function pause(): void {
	if (phase.value === 'filling' || phase.value === 'emptying') {
		lastPhase.value = phase.value // mémoriser la phase actuelle
		phase.value = 'paused'
		stopLoop()
	}
}

function stopAllAudio(): void {
	if (audioCtx) {
		try { audioCtx.close() } catch { /* ignore */ }
		audioCtx = null
	}
}

function stop(): void {
	phase.value = 'idle'
	fillPercent.value = 0
	currentSeries.value = 0
	breathsInCurrentSeries.value = 0
	remainingTime.value = 0
	currentTime.value = 0
	lastPhase.value = 'idle'
	seriesStartTs = null
	stopLoop()
	stopAllAudio()
}

function reset(): void {
	fillPercent.value = 0
	currentSeries.value = 1
	breathsInCurrentSeries.value = 0
	remainingTime.value = 0
	currentTime.value = 0
	lastPhase.value = 'idle'
	seriesStartTs = null
	phase.value = 'idle'
}

function scheduleNextSeriesPause(): void {
	phase.value = 'series-pause'
	remainingTime.value = settings.pauseSeconds
	stopLoop()
	
	const interval = setInterval(() => {
		remainingTime.value -= 1
		if (remainingTime.value <= 0) {
			clearInterval(interval)
			if (phase.value !== 'series-pause') return
			phase.value = 'filling'
			loop()
		}
	}, 1000)
}

function loop(ts?: number): void {
	if (rafId != null) cancelAnimationFrame(rafId)
	rafId = requestAnimationFrame(loop)
	if (ts == null) return
	if (lastTs === 0) { lastTs = ts; return }
	const delta = ts - lastTs
	lastTs = ts

	// Mettre à jour le temps actuel pour forcer la réactivité
	currentTime.value = ts

	if (phase.value === 'paused' || phase.value === 'idle' || phase.value === 'series-pause' || phase.value === 'finished') return

	// Check time-bounded work per series
	checkWorkTime(ts)
	if (phase.value === 'series-pause' || phase.value === 'finished') return

	const halfMs = getHalfCycleMs()
	if (halfMs <= 0) return
	const step = (delta / halfMs) * 100

	if (phase.value === 'filling') {
		fillPercent.value = Math.min(100, fillPercent.value + step)
		if (fillPercent.value >= 100) {
			beep(880, 120)
			phase.value = 'emptying'
			lastTs = ts // reset phase time
		}
	} else if (phase.value === 'emptying') {
		fillPercent.value = Math.max(0, fillPercent.value - step)
		if (fillPercent.value <= 0) {
			beep(660, 120)
			breathsInCurrentSeries.value += 1
			const totalBreathsPerSeries = settings.breathsPerMinute
			if (breathsInCurrentSeries.value >= totalBreathsPerSeries) {
				currentSeries.value += 1
				breathsInCurrentSeries.value = 0
				seriesStartTs = null
				if (currentSeries.value > settings.numberOfSeries && settings.numberOfSeries > 0) {
					phase.value = 'finished'
					stopLoop()
					return
				}
				if (settings.pauseSeconds > 0) {
					scheduleNextSeriesPause()
					return
				} else {
					// Immediately continue next series
					seriesStartTs = performance.now()
				}
			}
			phase.value = 'filling'
			lastTs = ts
		}
	}
}

function stopLoop(): void {
	if (rafId != null) cancelAnimationFrame(rafId)
	rafId = null
	lastTs = 0
}

onMounted(() => {
	// rien
})

onUnmounted(() => {
	stopLoop()
	stopAllAudio()
})

// When settings change while running, keep consistency
watch(() => settings.breathsPerMinute, (v) => {
	if (v <= 0) stop()
})

// Optional: bound each series by work time, if > 0
watch(phase, (p) => {
	if (p === 'filling' || p === 'emptying') {
		if (seriesStartTs == null) seriesStartTs = performance.now()
	} else if (p === 'series-pause') {
		seriesStartTs = null
	}
})

function checkWorkTime(ts: number): void {
	if (settings.workTimeSeconds > 0 && seriesStartTs != null) {
		const elapsed = (ts - seriesStartTs) / 1000
		if (elapsed >= settings.workTimeSeconds) {
			currentSeries.value += 1
			breathsInCurrentSeries.value = 0
			seriesStartTs = null
			if (currentSeries.value > settings.numberOfSeries && settings.numberOfSeries > 0) {
				phase.value = 'finished'
				stopLoop()
				return
			}
			if (settings.pauseSeconds > 0) {
				scheduleNextSeriesPause()
				return
			}
			// Continue immediately into the next series
			seriesStartTs = performance.now()
		}
	}
}
</script>

<template>
	<div class="container">
		<div class="controls">
			<label>
				Frequency (per minute)
				<input type="number" min="1" v-model.number="settings.breathsPerMinute" />
			</label>
			<label>
				Number of series
				<input type="number" min="0" v-model.number="settings.numberOfSeries" />
			</label>
			<label>
				Pause between series (seconds)
				<input type="number" min="0" v-model.number="settings.pauseSeconds" />
			</label>
			<label>
				Work time per series (seconds)
				<input type="number" min="0" v-model.number="settings.workTimeSeconds" />
			</label>
			<div class="buttons">
				<button @click="start">Start</button>
				<button class="secondary" @click="pause">Pause</button>
				<button class="danger" @click="stop">Stop</button>
			</div>
		</div>

		<div class="gauge-wrap">
			<div class="gauge">
				<div class="fill" :style="{ height: fillPercent + '%' }" />
				<div class="time-display" v-if="remainingTimeDisplay !== ''">
					{{ remainingTimeDisplay }}
				</div>
				<div class="phase-display">
					{{ phaseDisplay }}
				</div>
			</div>
		</div>

		<div class="status">
			<div class="status-item">
				<span class="status-label">Series:</span>
				<span class="status-value">{{ currentSeries }}/{{ settings.numberOfSeries || '∞' }}</span>
			</div>
			<div class="status-item">
				<span class="status-label">Breaths:</span>
				<span class="status-value">{{ breathsInCurrentSeries }}</span>
			</div>
		</div>
	</div>
</template>

<style scoped>
</style>


