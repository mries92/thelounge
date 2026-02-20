<template>
	<div v-if="isOpen" id="gif-picker" ref="picker">
		<div class="gif-picker-header">
			<select v-model="provider" class="gif-provider-selector" @change="onProviderChange">
				<option value="tenor">Tenor</option>
				<option value="giphy">Giphy</option>
			</select>
			<input
				ref="searchInput"
				v-model="query"
				type="search"
				:placeholder="'Search ' + (provider === 'giphy' ? 'Giphy' : 'Tenor') + '...'"
				@input="onInput"
			/>
		</div>
		<div class="gif-picker-content">
			<div v-for="gif in gifs" :key="gif.id" class="gif-item" @click="selectGif(gif)">
				<img
					:src="gif.preview"
					alt="GIF"
					referrerpolicy="no-referrer"
					loading="lazy"
					:title="gif.url"
				/>
			</div>
			<div v-if="loading" class="gif-loading">Loading...</div>
			<div v-if="!loading && gifs.length === 0 && query" class="gif-loading">
				No results found.
			</div>
		</div>
	</div>
</template>

<style>
#gif-picker {
	position: absolute;
	bottom: 70px;
	right: 10px;
	width: 675px;
	height: 750px;
	background: var(--window-bg-color);
	color: var(--body-color);
	border: 1px solid rgba(0, 0, 0, 0.2);
	border-radius: 5px;
	display: flex;
	flex-direction: column;
	z-index: 1001;
	box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);
}

@media (max-width: 768px) {
	#gif-picker {
		width: calc(100% - 10px);
		height: 60vh;
		left: 5px;
		right: 5px;
		bottom: 60px;
	}
}

.gif-picker-header {
	padding: 10px;
	border-bottom: 1px solid rgba(0, 0, 0, 0.2);
	display: flex;
	gap: 5px;
}

.gif-provider-selector {
	padding: 5px;
	border: 1px solid rgba(0, 0, 0, 0.2);
	border-radius: 3px;
	background: rgba(0, 0, 0, 0.05);
	color: var(--body-color);
	outline: none;
}

.gif-provider-selector option {
	background-color: var(--window-bg-color);
	color: var(--body-color);
}

.gif-picker-header input {
	flex: 1;
	padding: 5px;
	border: 1px solid rgba(0, 0, 0, 0.2);
	border-radius: 3px;
	background: rgba(0, 0, 0, 0.05);
	color: var(--body-color);
	outline: none;
}

.gif-picker-header input::placeholder {
	color: var(--body-color-muted);
	opacity: 0.8;
}

.gif-picker-content {
	flex: 1;
	overflow-y: auto;
	display: flex;
	flex-wrap: wrap;
	padding: 5px;
	align-content: flex-start;
}

.gif-item {
	width: 33.33%;
	padding: 2px;
	cursor: pointer;
}

.gif-item img {
	width: 100%;
	display: block;
	border-radius: 3px;
	transition: opacity 0.2s;
}

.gif-item:hover img {
	opacity: 0.8;
}

.gif-loading {
	width: 100%;
	text-align: center;
	padding: 10px;
	color: var(--body-color-muted);
}
</style>

<script lang="ts">
import {defineComponent, ref, onMounted, onUnmounted} from "vue";
import eventbus from "../js/eventbus";
import {useStore} from "../js/store";
import socket from "../js/socket";
import debounce from "lodash/debounce";

export default defineComponent({
	name: "GifPicker",
	setup() {
		const store = useStore();
		const isOpen = ref(false);
		const provider = ref("tenor");
		const query = ref("");
		const gifs = ref<any[]>([]);
		const loading = ref(false);
		const searchInput = ref<HTMLInputElement>();
		const picker = ref<HTMLElement>();

		const toggle = () => {
			isOpen.value = !isOpen.value;

			if (isOpen.value) {
				gifs.value = [];
				query.value = "";
				searchGifs();
				setTimeout(() => searchInput.value?.focus(), 0);
			}
		};

		const searchGifs = debounce(() => {
			loading.value = true;
			gifs.value = [];
			socket.emit("gif:search", {
				query: query.value,
				provider: provider.value,
			});
		}, 500);

		const onInput = () => {
			searchGifs();
		};

		const onProviderChange = () => {
			searchGifs();
		};

		const selectGif = (gif: any) => {
			const activeChannel = store.state.activeChannel;

			if (activeChannel) {
				socket.emit("input", {
					target: activeChannel.channel.id,
					text: gif.url,
				});
			}

			isOpen.value = false;
		};

		const close = () => {
			isOpen.value = false;
		};

		const onOutsideClick = (event: MouseEvent) => {
			if (
				isOpen.value &&
				picker.value &&
				!picker.value.contains(event.target as Node) &&
				!(event.target as HTMLElement).closest("#gif-picker-button")
			) {
				close();
			}
		};

		const onGifResults = (data: {results: any[]}) => {
			gifs.value = data.results;
			loading.value = false;
		};

		onMounted(() => {
			eventbus.on("gif-picker:toggle", toggle);
			eventbus.on("escapekey", close);
			document.addEventListener("mousedown", onOutsideClick);
			socket.on("gif:results", onGifResults);
		});

		onUnmounted(() => {
			eventbus.off("gif-picker:toggle", toggle);
			eventbus.off("escapekey", close);
			document.removeEventListener("mousedown", onOutsideClick);
			socket.off("gif:results", onGifResults);
		});

		return {
			isOpen,
			provider,
			query,
			gifs,
			loading,
			onInput,
			onProviderChange,
			selectGif,
			searchInput,
			picker,
		};
	},
});
</script>
