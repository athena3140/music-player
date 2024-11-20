<template>
	<div class="h-svh flex justify-center items-center">
		<div class="container xl:w-3/5 lg:w-4/5 md:w-4/5 h-[572px] flex justify-center items-center">
			<FolderPicker v-if="musicFiles.length == 0" @files="updateFiles($event)" />
			<template v-else>
				<List
					class="flex-1 h-full w-1/2"
					:files="musicFiles"
					:currentIndex="currentIndex"
					@updateCurrentSong="updateCurrent($event)" />
				<Player
					class="flex-1 h-full w-1/2"
					:current="currentFile"
					:currentMode="mode"
					@changeSong="changeSong($event)"
					@mode="updateMode($event)"
					@toggle="togglePlay()"
					@volume="updateVolume($event)" />
			</template>
		</div>
	</div>
</template>

<script setup>
import { ref } from "vue"
import Player from "./components/Player.vue"
import List from "./components/List.vue"
import FolderPicker from "./components/FolderPicker.vue"

const mode = ref("")
mode.value = localStorage.getItem("mode") || "shuffle"
const musicFiles = ref([])

const currentFile = ref()
const currentIndex = ref(0)
currentIndex.value = Number(localStorage.getItem("currentIndex")) || "0"

const updateCurrent = (event) => {
	currentIndex.value = event
	currentFile.value = musicFiles.value[event]
	localStorage.setItem("currentIndex", currentIndex.value)
}

const changeSong = (event) => {
	if (event == "next") {
		let index
		switch (mode.value) {
			case "shuffle":
				index = Math.floor(Math.random() * musicFiles.value.length)
				break
			case "repeat":
				if (musicFiles.value.length - 1 == currentIndex.value) index = 0
				else index = currentIndex.value + 1
				break
			case "repeatOne":
				index = currentIndex.value
				break
		}
		updateCurrent(index)
		return
	}

	if (event == "prev") {
		let index = currentIndex.value - 1
		if (currentIndex.value == 0) index = musicFiles.value.length - 1
		if (currentIndex.value == 0 && mode.value == "repeatOne") index = currentIndex.value
		updateCurrent(index)

		return
	}
}
const updateFiles = (event) => {
	musicFiles.value = event
	currentFile.value = musicFiles.value[currentIndex.value]
}

const updateMode = (event) => {
	mode.value = event
	localStorage.setItem("mode", mode.value)
}
</script>
