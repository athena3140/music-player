<template>
	<div
		v-if="!allLoaded"
		class="h-full bg-card rounded-md p-4 rounded-r-none flex border-r border-muted flex-col space-y-4 justify-center items-center">
		<template v-for="i in 8" :key="i">
			<div class="w-full flex justify-between items-start">
				<div class="space-y-2">
					<Skeleton class="h-6" :class="randomWidth()" />
					<Skeleton class="w-20 h-4" />
				</div>
				<Skeleton class="w-7 h-4" />
			</div>
		</template>
	</div>
	<div v-else class="h-full bg-card rounded-md rounded-r-none p-2">
		<RecycleScroller
			ref="scroller"
			class="h-full scrollBar pe-2"
			:items="songs"
			:item-size="null"
			:buffer="1"
			key-field="id"
			v-slot="{ item }">
			<div v-if="item.isFooter" class="px-4 py-2 text-xs text-end">
				<p>Total: {{ item.id }}</p>
			</div>
			<div
				class="item"
				:id="`item-${item.id}`"
				:class="{ active: item.id == currentIndex }"
				@click="updateCurrentSong(item)"
				v-else>
				<div class="truncate">
					<h3 class="title">{{ item.name }}</h3>
					<p class="artist">{{ item.artist }}</p>
				</div>
				<p class="duration">{{ item.duration }}</p>
			</div>
		</RecycleScroller>
	</div>
</template>

<script setup>
import "vue-virtual-scroller/dist/vue-virtual-scroller.css"
import { Skeleton } from "@/components/ui/skeleton"
import { RecycleScroller } from "vue-virtual-scroller"
import { ref, onMounted, useTemplateRef, watch, nextTick } from "vue"

const props = defineProps({
	currentIndex: {
		type: Number,
		required: true,
	},
	files: {
		type: Array,
		required: true,
	},
})
const emit = defineEmits(["updateCurrentSong"])
const songs = ref([])
const allLoaded = ref(false)
const scroller = useTemplateRef("scroller")
watch(
	() => props.currentIndex,
	() => scrollToCurrentSong()
)

const updateCurrentSong = (item) => {
	emit("updateCurrentSong", item.id)
}

const getDuration = (file) => {
	return new Promise((resolve) => {
		let audio = new Audio(URL.createObjectURL(file))
		audio.addEventListener("loadedmetadata", () => {
			resolve(audio.duration)
		})
	})
}

const formatDuration = (durationInSeconds) => {
	const minutes = Math.floor(durationInSeconds / 60)
	const seconds = Math.floor(durationInSeconds % 60)
		.toString()
		.padStart(2, "0")
	return `${minutes}:${seconds}`
}

const loadSongs = async (files) => {
	for (let [index, file] of files.entries()) {
		const [artist, song] = file.name.split("-")
		const duration = await getDuration(file)

		songs.value.push({
			id: index++,
			name: song.replace(".mp3", "").trim(),
			artist: artist.trim(),
			duration: formatDuration(duration),
			size: 60,
		})
	}
}

onMounted(() => {
	if (props.files && props.files.length > 0) {
		loadSongs(props.files).then(() => {
			allLoaded.value = true

			songs.value.push({
				id: songs.value.length,
				isFooter: true,
				size: 28,
			})

			setTimeout(() => {
				scroller.value.scrollToItem(props.currentIndex)
			}, 10)
		})
	} else {
		console.log("No files found in props.files")
	}
})

const isElementVisible = (el, callback) => {
	if (!el) {
		scroller.value.scrollToItem(props.currentIndex)
		return
	}

	const observer = new IntersectionObserver((entries) => {
		const entry = entries[0]
		callback(entry.isIntersecting) // `isIntersecting` is `true` if the element is visible in the viewport
		observer.disconnect() // Stop observing once visibility is determined
	})

	observer.observe(el)
}
const scrollToCurrentSong = () => {
	const element = document.getElementById(`item-${props.currentIndex}`)
	isElementVisible(element, (isVisible) => {
		if (!isVisible) {
			scroller.value.scrollToItem(props.currentIndex)
		}
	})
}
const randomWidth = () => {
	const widths = ["w-20", "w-28", "w-32", "w-36", "w-44"]
	return widths[Math.floor(Math.random() * widths.length)]
}
</script>

<style>
.item {
	@apply grid grid-cols-[1fr_auto] gap-3 cursor-pointer max-w-full hover:bg-muted rounded px-4 py-2;

	&.active {
		@apply bg-muted;
	}

	.title {
		@apply text-sm mb-1 text-white font-medium truncate;
	}

	.artist {
		@apply text-xs text-muted-foreground;
	}

	.duration {
		@apply text-xs font-medium text-white leading-none mt-1;
	}
}

.scrollBar::-webkit-scrollbar {
	@apply w-2;
}

.scrollBar::-webkit-scrollbar-track {
	@apply bg-transparent;
}

.scrollBar::-webkit-scrollbar-thumb {
	@apply bg-muted rounded-full;
}
</style>
