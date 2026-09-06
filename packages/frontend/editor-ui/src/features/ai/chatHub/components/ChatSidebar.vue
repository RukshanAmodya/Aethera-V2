<script lang="ts" setup>
import { nextTick, computed } from 'vue';
import { useI18n } from '@n8n/i18n';
import { useRouter } from 'vue-router';
import { VIEWS } from '@/app/constants';
import { type IMenuItem } from '@n8n/design-system';
import { useSettingsItems } from '@/app/composables/useSettingsItems';
import { useKeybindings } from '@/app/composables/useKeybindings';
import { useSidebarLayout } from '@/app/composables/useSidebarLayout';
import { N8nScrollArea } from '@n8n/design-system';
import BottomMenu from '@/app/components/BottomMenu.vue';
import MainSidebarHeader from '@/app/components/MainSidebarHeader.vue';
import ChatSidebarContent from '@/features/ai/chatHub/components/ChatSidebarContent.vue';

const i18n = useI18n();
const router = useRouter();

const { isCollapsed, toggleCollapse } = useSidebarLayout();

function openCommandBar(event: MouseEvent) {
	event.stopPropagation();

	void nextTick(() => {
		const keyboardEvent = new KeyboardEvent('keydown', {
			key: 'k',
			code: 'KeyK',
			metaKey: true,
			bubbles: true,
			cancelable: true,
		});
		document.dispatchEvent(keyboardEvent);
	});
}

const { settingsItems, handleSettingsItemSelect } = useSettingsItems();

const mainMenuItems = computed<IMenuItem[]>(() => [
	{
		id: 'settings',
		label: i18n.baseText('mainSidebar.settings'),
		icon: 'settings',
		available: true,
		children: settingsItems.value,
	},
]);

const visibleMenuItems = computed<IMenuItem[]>(() =>
	mainMenuItems.value.filter((item) => item.available !== false),
);

useKeybindings({
	['bracketleft']: () => toggleCollapse(),
});

const onLogout = () => {
	void router.push({ name: VIEWS.SIGNOUT });
};
</script>

<template>
	<aside
		id="side-menu"
		:class="{
			[$style.sideMenu]: true,
			[$style.sideMenuCollapsed]: isCollapsed,
		}"
	>
		<MainSidebarHeader
			hide-create
			:is-collapsed="isCollapsed"
			@collapse="toggleCollapse"
			@open-command-bar="openCommandBar"
		/>
		<div
			:class="{
				[$style.scrollAreaWrapper]: true,
			}"
		>
			<N8nScrollArea>
				<ChatSidebarContent :is-collapsed="isCollapsed" />
			</N8nScrollArea>
		</div>
		<BottomMenu
			:items="visibleMenuItems"
			:is-collapsed="isCollapsed"
			@select="handleSettingsItemSelect"
			@logout="onLogout"
		/>
	</aside>
</template>

<style lang="scss" module>
.sideMenu {
	position: relative;
	width: 240px;
	min-width: 240px;
	max-width: 240px;
	height: calc(100% - 16px);
	margin: 8px;
	border-radius: 24px;
	display: flex;
	flex-direction: column;
	background-color: #0b0c10;
	border: 1px solid rgba(255, 255, 255, 0.08);
	box-shadow:
		0 14px 38px -6px rgba(0, 0, 0, 0.7),
		0 0 0 1px rgba(255, 255, 255, 0.04);
	transition:
		width var(--duration--snappy) var(--easing--ease-out),
		min-width var(--duration--snappy) var(--easing--ease-out),
		max-width var(--duration--snappy) var(--easing--ease-out);
	will-change: width;
	overflow: hidden;
	box-sizing: border-box;

	&.sideMenuCollapsed {
		width: 58px;
		min-width: 58px;
		max-width: 58px;
		border-radius: 24px;
	}
}

.scrollAreaWrapper {
	position: relative;
	flex: 1;
	min-height: 0;
	display: flex;
	flex-direction: column;
	padding-top: var(--spacing--2xs);
}

/* 
 * Sleek Deep Black Glass Theme & Shader/Glow Effects for Sidebar items & Teleported Popovers
 */
:global(#side-menu),
:global([data-radix-popper-content-wrapper]),
:global(.el-popper) {
	/* Section headers / Labels */
	:global(.n8n-text) {
		color: #94a3b8;
	}

	/* Menu items base state */
	:global(a[role='menuitem']),
	:global(div[role='menuitem']) {
		color: #e2e8f0 !important;
		border: 1px solid transparent;
		transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
		background: transparent;

		:global(.n8n-text) {
			color: #e2e8f0 !important;
			font-weight: 500;
		}

		:global(svg) {
			color: #94a3b8 !important;
			transition: all 0.2s ease;
		}

		/* Hover state */
		&:hover:not(:global(.router-link-active)):not(:global(.active)) {
			background: rgba(255, 255, 255, 0.08) !important;
			border-color: rgba(255, 255, 255, 0.1) !important;

			:global(.n8n-text) {
				color: #fff !important;
			}

			:global(svg) {
				color: #fff !important;
				transform: scale(1.06);
			}
		}

		/* Active / Selected state (Deep glowing dark glass card with shader gradient) */
		&:global(.router-link-active),
		&:global(.active) {
			background: linear-gradient(
				135deg,
				rgba(255, 255, 255, 0.12) 0%,
				rgba(255, 255, 255, 0.04) 100%
			) !important;
			border: 1px solid rgba(255, 255, 255, 0.18) !important;
			box-shadow:
				0 4px 20px -2px rgba(0, 0, 0, 0.5),
				inset 0 1px 1px 0 rgba(255, 255, 255, 0.25),
				0 0 16px -4px rgba(255, 255, 255, 0.1) !important;
			backdrop-filter: blur(16px);

			:global(.n8n-text) {
				color: #fff !important;
				font-weight: 600;
				text-shadow: 0 1px 2px rgba(0, 0, 0, 0.4);
			}

			:global(svg) {
				color: #fff !important;
				filter: drop-shadow(0 0 6px rgba(255, 255, 255, 0.4));
			}
		}
	}
}
</style>
