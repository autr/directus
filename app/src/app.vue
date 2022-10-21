<template>
	<div id="directus" :style="brandStyle">
		<transition name="fade">
			<div v-if="hydrating" class="hydrating">
				<v-progress-circular indeterminate />
			</div>
		</transition>

		<v-info v-if="error" type="danger" :title="t('unexpected_error')" icon="error" center>
			{{ t('unexpected_error_copy') }}

			<template #append>
				<v-error :error="error" />
			</template>
		</v-info>

		<router-view v-else-if="!hydrating" />

		<teleport to="#custom-css">{{ customCSS }}</teleport>
	</div>
</template>

<script lang="ts">
import { useI18n } from 'vue-i18n';
import { defineComponent, toRefs, watch, computed, onMounted, onUnmounted, StyleValue } from 'vue';
import { useAppStore } from '@/stores/app';
import { useUserStore } from '@/stores/user';
import { useServerStore } from '@/stores/server';
import { startIdleTracking, stopIdleTracking } from './idle';
import { useSystem } from '@/composables/use-system';

import { setFavicon } from '@/utils/set-favicon';
import { User } from '@directus/shared/types';

export default defineComponent({
	setup() {
		const { t } = useI18n();

		const appStore = useAppStore();
		const userStore = useUserStore();
		const serverStore = useServerStore();

		const { hydrating } = toRefs(appStore);

		const hex2HSL = (hex) => {
			let result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
			let r = parseInt(result[1], 16);
			let g = parseInt(result[2], 16);
			let b = parseInt(result[3], 16);
			r /= 255;
			g /= 255;
			b /= 255;
			let max = Math.max(r, g, b),
				min = Math.min(r, g, b);
			let h,
				s,
				l = (max + min) / 2;
			if (max == min) {
				h = s = 0; // achromatic
			} else {
				let d = max - min;
				s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
				switch (max) {
					case r:
						h = (g - b) / d + (g < b ? 6 : 0);
						break;
					case g:
						h = (b - r) / d + 2;
						break;
					case b:
						h = (r - g) / d + 4;
						break;
				}
				h /= 6;
			}
			let HSL = new Object();
			HSL['h'] = h;
			HSL['s'] = s;
			HSL['l'] = l;
			return HSL;
		};

		const brandStyle = computed(() => {
			const col = hex2HSL(serverStore.info?.project?.project_color || '#8866ff');
			const adjust = (h, s, l) => `hsl(${col.h * 360 + h},${col.s * 100 + s}%,${60 + l / 2.5}%)`;

			const colors = {
				'--primary-10': adjust(0, 0, 90),
				'--primary-25': adjust(0, 0, 75),
				'--primary-50': adjust(0, 0, 50),
				'--primary-75': adjust(0, 0, 25),
				'--primary-90': adjust(0, 0, 10),
				'--primary-100': adjust(0, 0, 0),
				'--primary-110': adjust(0, 0, -10),
				'--primary-125': adjust(0, -5, -25),
				'--primary-150': adjust(0, -10, -50),
				'--primary-175': adjust(0, -15, -75),
				'--primary-190': adjust(0, -20, -90),
				'--primary': adjust(0, 0, 0),
				'--brand': serverStore.info?.project?.project_color || 'var(--primary)',
			} as StyleValue;

			/*
				// debug colour output into console:

				Object.keys(colors).forEach( key => {
					return console.log(`%c${key}: ${colors[key]}`, `background-color:${colors[key]}`)
				})
			*/

			return colors;
		});

		onMounted(() => startIdleTracking());
		onUnmounted(() => stopIdleTracking());

		watch(
			[() => serverStore.info?.project?.project_color ?? null, () => serverStore.info?.project?.project_logo ?? null],
			() => {
				const hasCustomLogo = !!serverStore.info?.project?.project_logo;
				setFavicon(serverStore.info?.project?.project_color, hasCustomLogo);
			},
			{ immediate: true }
		);

		watch(
			() => (userStore.currentUser as User)?.theme,
			(theme) => {
				document.body.classList.remove('dark');
				document.body.classList.remove('light');
				document.body.classList.remove('auto');

				if (theme) {
					document.body.classList.add(theme);
					document
						.querySelector('head meta[name="theme-color"]')
						?.setAttribute('content', theme === 'light' ? '#ffffff' : '#263238');
				} else {
					// Default to auto mode
					document.body.classList.add('auto');
				}
			},
			{ immediate: true }
		);

		watch(
			() => serverStore.info?.project?.project_name,
			(projectName) => {
				document.title = projectName || 'Directus';
			},
			{ immediate: true }
		);

		const customCSS = computed(() => {
			return serverStore.info?.project?.custom_css || '';
		});

		const error = computed(() => appStore.error);

		useSystem();

		return { t, hydrating, brandStyle, error, customCSS };
	},
});
</script>

<style lang="scss" scoped>
:global(#app) {
	height: 100%;
}

#directus {
	height: 100%;
}

.hydrating {
	position: fixed;
	z-index: 1000;
	display: flex;
	align-items: center;
	justify-content: center;
	width: 100%;
	height: 100%;
	backdrop-filter: blur(10px);
}

.fade-enter-active,
.fade-leave-active {
	transition: opacity var(--medium) var(--transition);
}

.fade-enter-from,
.fade-leave-to {
	opacity: 0;
}
</style>
