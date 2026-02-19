<template>
	<div v-if="isOpen" id="gif-picker" ref="picker">
		<div class="gif-picker-header">
			<input
				ref="searchInput"
				v-model="query"
				type="search"
				placeholder="Search Giphy..."
				@input="onInput"
			/>
		</div>
		<div class="gif-picker-content" @scroll="onScroll">
			<div v-for="gif in gifs" :key="gif.id" class="gif-item" @click="selectGif(gif)">
				<img :src="gif.images.fixed_width_small.url" :alt="gif.title" />
			</div>
			<div v-if="loading" class="gif-loading">Loading...</div>
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
}

.gif-picker-header input {
	width: 100%;
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
import {defineComponent, ref, onMounted, onUnmounted, computed} from "vue";
import eventbus from "../js/eventbus";
import {useStore} from "../js/store";
import socket from "../js/socket";

export default defineComponent({
	name: "GifPicker",
	setup() {
		const store = useStore();
		const isOpen = ref(false);
		const query = ref("");
		const gifs = ref<any[]>([]);
		const loading = ref(false);
		const offset = ref(0);
		const searchInput = ref<HTMLInputElement>();
		const picker = ref<HTMLElement>();

		const giphyApiKey = computed(() => store.state.serverConfiguration?.giphyApiKey);

		const toggle = () => {
			if (!giphyApiKey.value) {
				return;
			}

			isOpen.value = !isOpen.value;

			if (isOpen.value) {
				gifs.value = [];
				offset.value = 0;
				query.value = "";
				void fetchGifs();
				setTimeout(() => searchInput.value?.focus(), 0);
			}
		};

		const fetchGifs = async () => {
			if (loading.value || !giphyApiKey.value) {
				return;
			}

			loading.value = true;
			const url = query.value
				? `https://api.giphy.com/v1/gifs/search?api_key=${
						giphyApiKey.value
				  }&q=${encodeURIComponent(query.value)}&limit=20&offset=${offset.value}`
				: `https://api.giphy.com/v1/gifs/trending?api_key=${giphyApiKey.value}&limit=20&offset=${offset.value}`;

			try {
				const response = await fetch(url);
				const data = await response.json();
				gifs.value = [...gifs.value, ...data.data];
				offset.value += 20;
			} catch (e) {
				// eslint-disable-next-line no-console
				console.error("Giphy API error", e);
			} finally {
				loading.value = false;
			}
		};

		const onInput = () => {
			gifs.value = [];
			offset.value = 0;
			void fetchGifs();
		};

		const onScroll = (e: Event) => {
			const target = e.target as HTMLElement;

			if (target.scrollTop + target.clientHeight >= target.scrollHeight - 50) {
				void fetchGifs();
			}
		};

		const selectGif = (gif: any) => {
			const activeChannel = store.state.activeChannel;

			if (activeChannel) {
				socket.emit("input", {
					target: activeChannel.channel.id,
					text: gif.images.original.url,
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

		onMounted(() => {
			eventbus.on("gif-picker:toggle", toggle);
			eventbus.on("escapekey", close);
			document.addEventListener("mousedown", onOutsideClick);
		});

		onUnmounted(() => {
			eventbus.off("gif-picker:toggle", toggle);
			eventbus.off("escapekey", close);
			document.removeEventListener("mousedown", onOutsideClick);
		});

		return {
			isOpen,
			query,
			gifs,
			loading,
			onInput,
			onScroll,
			selectGif,
			searchInput,
			picker,
		};
	},
});
</script>
