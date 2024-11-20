<template>
	<Card class="w-full rounded-l-none flex flex-col justify-around">
		<CardHeader>
			<div class="xl:w-64 xl:h-64 md:w-60 md:h-60 w-full h-full aspect-square mx-auto">
				<div class="w-full h-full bg-red-50 rounded-full mx-auto disc" :class="{ active: isPlaying }"></div>
			</div>
		</CardHeader>

		<CardContent class="space-y-7">
			<div class="text-center space-y-1">
				<h3 class="text-xl text-white font-semibold title">{{ title }}</h3>
				<p class="text-gray-400">{{ artist }}</p>
			</div>

			<div>
				<Slider v-model="track" @update:modelValue="seek" class="w-full" :default-value="[0]" :max="100" :step="1" />
				<div class="flex justify-between text-sm text-gray-400 mt-2">
					<span>{{ currentTime }}</span>
					<span>{{ totalTime }}</span>
				</div>
			</div>
		</CardContent>

		<CardFooter class="flex justify-evenly">
			<Button size="icon" variant="icon" @click="emitMode()">
				<Shuffle v-if="currentMode == 'shuffle'" />
				<Repeat v-if="currentMode == 'repeat'" />
				<Repeat1 v-if="currentMode == 'repeatOne'" />
			</Button>

			<TooltipProvider>
				<Tooltip>
					<TooltipTrigger>
						<Button size="icon" @click="emitChangeSong('prev')" class="w-14 h-14" variant="icon">
							<SkipBack class="!w-6 !h-6" />
						</Button>
					</TooltipTrigger>
					<TooltipContent class="bg-zinc-700 flex items-center justify-center">
						<div class="flex items-center justify-center">
							<span>Ctrl + </span>
							<ChevronLeft class="w-5 h-5" />
						</div>
					</TooltipContent>
				</Tooltip>
			</TooltipProvider>

			<Button size="icon" class="w-14 h-14 rounded-full" @click="togglePlay()">
				<Play class="!w-6 !h-6" :stroke-width="2" v-if="!isPlaying" />
				<Pause class="!w-7 !h-7" :stroke-width="1.25" v-else />
			</Button>

			<TooltipProvider>
				<Tooltip>
					<TooltipTrigger>
						<Button size="icon" @click="emitChangeSong('next')" class="w-14 h-14" variant="icon">
							<SkipForward class="!w-6 !h-6" />
						</Button>
					</TooltipTrigger>
					<TooltipContent class="bg-zinc-700 flex items-center justify-center">
						<div class="flex items-center justify-center">
							<span>Ctrl + </span>
							<ChevronRight class="w-5 h-5" />
						</div>
					</TooltipContent>
				</Tooltip>
			</TooltipProvider>

			<Popover class="dark">
				<PopoverTrigger>
					<Button size="icon" variant="icon">
						<VolumeX v-if="volume[0] == 0" />
						<Volume v-if="volume[0] >= 1 && volume[0] <= 33" />
						<Volume1 v-if="volume[0] >= 34 && volume[0] <= 66" />
						<Volume2 v-if="volume[0] >= 67" />
					</Button>
				</PopoverTrigger>
				<PopoverContent class="bg-zinc-700 w-48">
					<Slider class="w-full" v-model="volume" :default-value="[50]" :max="100" :step="1" />
				</PopoverContent>
			</Popover>
		</CardFooter>
	</Card>
</template>

<script setup lang="ts">
import { Tooltip, TooltipContent, TooltipProvider, TooltipTrigger } from "@/components/ui/tooltip"
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover"
import {
	Play,
	SkipForward,
	SkipBack,
	Pause,
	Shuffle,
	Volume,
	VolumeX,
	Volume1,
	Volume2,
	Repeat,
	Repeat1,
	Disc3,
	ChevronRight,
	ChevronLeft,
} from "lucide-vue-next"
import { onMounted, ref, watch } from "vue"
import { Slider } from "@/components/ui/slider"
import { Button } from "@/components/ui/button"
import { Card, CardContent, CardFooter, CardHeader } from "@/components/ui/card"

const props = defineProps({
	current: {
		type: File,
		required: true,
	},
	currentMode: {
		type: String,
		required: true,
	},
})

const artist = ref("")
const title = ref("")

watch(
	() => props.current,
	(newVal) => {
		if (newVal) {
			let [newArtist, newTitle] = newVal.name.split("-")
			artist.value = newArtist.trim()
			title.value = newTitle.replace(".mp3", "").trim()
		}
	},
	{ immediate: true }
)

const emit = defineEmits(["mode", "changeSong"])

const currentTime = ref("0:00")
const totalTime = ref("00:00")
const isPlaying = ref(false)
const track = ref([0])
const audio = ref(<HTMLAudioElement | null>null)
const background = ref("")
const volume = ref([50])
volume.value = JSON.parse(localStorage.getItem("volume") || "[50]")

onMounted(() => {
	loadSong(true)
	window.addEventListener("keydown", (key) => {
		if (key.code == "Space") {
			togglePlay()
			return
		}

		if (key.ctrlKey && key.code == "ArrowRight") {
			emitChangeSong("next")
			return
		}

		if (key.ctrlKey && key.code == "ArrowLeft") {
			emitChangeSong("prev")
			return
		}

		if (key.code == "ArrowUp") {
			volume.value = [Math.min(100, volume.value[0] + 5)]
			return
		}

		if (key.code == "ArrowDown") {
			volume.value = [Math.max(0, volume.value[0] - 5)]
			return
		}
	})
})

const emitChangeSong = (event: any) => {
	emit("changeSong", event)
}

const loadSong = (isFirstLoad = false): void => {
	if (audio.value) {
		audio.value.pause() // Stop the current audio
		audio.value.currentTime = 0 // Reset playback position
		audio.value.removeEventListener("loadedmetadata", handleMetadata)
		audio.value.removeEventListener("timeupdate", handleTimeUpdate)
		audio.value.removeEventListener("ended", handleEnded)
		audio.value.removeEventListener("play", () => (isPlaying.value = true))
		audio.value.removeEventListener("pause", () => (isPlaying.value = false))

		audio.value = null // Clear the reference
	}

	// Create a new audio instance
	audio.value = new Audio(URL.createObjectURL(props.current))
	audio.value.volume = volume.value[0] / 100

	// Attach event listeners
	audio.value.addEventListener("loadedmetadata", handleMetadata)
	audio.value.addEventListener("timeupdate", handleTimeUpdate)
	audio.value.addEventListener("ended", handleEnded)
	audio.value.addEventListener("play", () => (isPlaying.value = true))
	audio.value.addEventListener("pause", () => (isPlaying.value = false))

	// Play or pause based on the current state
	if (isFirstLoad) changePlayStatus("pause")
	else changePlayStatus("play")
}

// Define handlers
const handleMetadata = () => {
	totalTime.value = formatTime(audio.value?.duration || 0)
}

const handleTimeUpdate = () => {
	currentTime.value = formatTime(audio.value?.currentTime || 0)
	track.value = [((audio.value?.currentTime || 0) / (audio.value?.duration || 1)) * 100]
}

const handleEnded = () => emitChangeSong("next")

const changePlayStatus = (status: String) => {
	if (!audio.value) return

	if (status == "pause") {
		isPlaying.value = false
		audio.value.pause()
		return
	}

	if (status == "play") {
		isPlaying.value = true
		audio.value.play()
		return
	}
}

const togglePlay = () => {
	if (audio.value) {
		if (isPlaying.value) {
			changePlayStatus("pause")
		} else {
			changePlayStatus("play")
		}
	}
}

const seek = () => {
	if (audio.value) {
		audio.value.currentTime = (track.value[0] / 100) * audio.value.duration
	}
}

watch(volume, () => {
	if (audio.value) {
		audio.value.volume = volume.value[0] / 100
	}
	localStorage.setItem("volume", JSON.stringify(volume.value))
})

// check if the song is changed or loaded
watch(
	() => props.current,
	() => {
		loadSong()
	}
)

//change Mode
const mode = ["shuffle", "repeat", "repeatOne"]
const modeIndex = ref(0)
const emitMode = () => {
	modeIndex.value = (modeIndex.value + 1) % mode.length
	emit("mode", mode[modeIndex.value])
}

// format time to mm:ss
const formatTime = (seconds: number) => {
	const mins = Math.floor(seconds / 60)
	const secs = Math.floor(seconds % 60)
		.toString()
		.padStart(2, "0")
	return `${mins}:${secs}`
}
</script>

<style scoped>
.disc {
	border: 1px solid grey;
	position: relative;
	animation: spin 12s linear infinite;
	animation-play-state: paused;
	background: conic-gradient(
		white,
		white,
		white,
		grey,
		grey,
		violet,
		deepskyblue,
		aqua,
		palegreen,
		yellow,
		orange,
		red,
		grey,
		grey,
		white,
		white,
		white,
		white,
		grey,
		grey,
		violet,
		deepskyblue,
		aqua,
		palegreen,
		yellow,
		orange,
		red,
		grey,
		grey,
		white
	);

	&.active {
		animation-play-state: running;
	}
}

@keyframes spin {
	to {
		transform: rotate(360deg);
	}
}

.disc::before,
.disc::after {
	content: "";
	position: absolute;
	top: 50%;
	left: 50%;
	border-radius: inherit;
	box-shadow: 0 0 1px grey;
	box-sizing: border-box;
}

.disc::before {
	width: 30%;
	height: 30%;
	margin: -15% 0 0 -15%;
	background: lightgrey;
	background-clip: padding-box;
	border: 10px solid rgba(0, 0, 0, 0.2);
}

.disc::after {
	width: 18%;
	height: 18%;
	margin: -9% 0 0 -9%;
	background: white;
	background-clip: padding-box;
	border: 10px solid rgba(0, 0, 0, 0.1);
	filter: drop-shadow(0 0 2px grey);
}
</style>
