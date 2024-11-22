<template>
	<div class="w-3/5 text-center">
		<div
			class="bg-card flex flex-col justify-center items-center p-8 pb-5 rounded-lg outline-dashed outline-2 outline-offset-[-2px] outline-red-400">
			<div>
				<Music class="mx-auto h-12 w-12 text-white" />

				<h2 class="mt-2 text-lg font-semibold text-white">Choose Your Music Folder</h2>
				<p class="mt-1 text-sm text-gray-400">
					Drag and drop your music folder here, or click the button below to select a folder.
				</p>
			</div>
			<Button class="mt-4 w-full" @click="handleFolderSelect()">
				<FolderOpen class="mr-2 h-4 w-4" />
				Open Folder
			</Button>
			<p class="mt-5 text-xs text-gray-400">Press any keys to select folder</p>
		</div>
		<span class="mt-5 block text-xs text-gray-400">
			To continue using the music player, please choose a folder where your songs are stored.
		</span>
	</div>
</template>

<script setup>
import { Button } from "../components/ui/button"
import { Music, FolderOpen } from "lucide-vue-next"
import { onMounted, onUnmounted } from "vue"

const emit = defineEmits(["files"])

const handleKeyDown = (e) => {
	const key = e.key
	if (/^[a-zA-Z0-9]$/.test(key) || key === " " || key === "Enter") {
		handleFolderSelect()
	}
}

onUnmounted(() => {
	window.removeEventListener("keydown", handleKeyDown)
})

onMounted(() => {
	window.addEventListener("keydown", handleKeyDown)
})

const handleFolderSelect = async () => {
	const folderHandle = await window.showDirectoryPicker()
	const supportedExtensions = [".mp3", ".ogg", ".wav", ".aac", ".m4a", ".flac"]

	const files = []
	for await (const entry of folderHandle.values()) {
		if (entry.kind === "file" && supportedExtensions.some((ext) => entry.name.endsWith(ext))) {
			const file = await entry.getFile()
			files.push(file)
		}
	}

	emit("files", files)
}
</script>
