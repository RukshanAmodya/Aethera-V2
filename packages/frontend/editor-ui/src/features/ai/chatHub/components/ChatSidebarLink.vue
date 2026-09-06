<script setup lang="ts" generic="T extends string">
import { N8nActionDropdown, N8nIcon, N8nIconButton, N8nText } from '@n8n/design-system';
import type { ActionDropdownItem, IconName } from '@n8n/design-system';
import { type RouteLocationRaw } from 'vue-router';

const {
	active = false,
	to,
	label,
	title,
	menuItems = [],
	icon,
	compact,
} = defineProps<{
	active?: boolean;
	to: RouteLocationRaw;
	label: string;
	title: string;
	menuItems?: Array<ActionDropdownItem<T>>;
	icon?: IconName;
	compact?: boolean;
}>();

const emit = defineEmits<{
	actionSelect: [action: T];
	click: [MouseEvent];
}>();

defineSlots<{
	default: unknown;
	icon: unknown;
}>();
</script>

<template>
	<div :class="[$style.menuItem, { [$style.active]: active }]">
		<slot v-if="$slots.default" />
		<template v-else>
			<RouterLink
				:to="to"
				:class="[$style.menuItemLink, { [$style.compact]: compact }]"
				:title="title"
				@click="emit('click', $event)"
			>
				<slot name="icon">
					<N8nIcon v-if="icon" size="large" :icon="icon" />
				</slot>
				<div v-if="!compact" :class="$style.textContainer">
					<N8nText :class="$style.label" size="small" color="text-light">{{ label }}</N8nText>
					<N8nText :class="$style.title" size="medium" color="text-dark">{{ title }}</N8nText>
				</div>
			</RouterLink>
			<N8nActionDropdown
				v-if="!compact && menuItems.length > 0"
				:items="menuItems"
				:class="$style.actionDropdown"
				placement="bottom-start"
				@select="emit('actionSelect', $event)"
				@click.stop
			>
				<template #activator>
					<N8nIconButton
						variant="ghost"
						icon="ellipsis-vertical"
						:class="$style.actionDropdownTrigger"
					/>
				</template>
			</N8nActionDropdown>
		</template>
	</div>
</template>

<style lang="scss" module>
.menuItem {
	display: flex;
	align-items: center;
	border-radius: 12px;
	padding-right: 0;
	margin-bottom: 2px;
	border: 1px solid transparent;
	transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);

	&:hover:not(.active) {
		background: rgba(255, 255, 255, 0.08);
		border-color: rgba(255, 255, 255, 0.1);
	}

	&.active,
	&:focus-within,
	&:has([aria-expanded='true']) {
		background: linear-gradient(
			135deg,
			rgba(255, 255, 255, 0.12) 0%,
			rgba(255, 255, 255, 0.04) 100%
		);
		border: 1px solid rgba(255, 255, 255, 0.18);
		box-shadow:
			0 4px 20px -2px rgba(0, 0, 0, 0.5),
			inset 0 1px 1px 0 rgba(255, 255, 255, 0.25),
			0 0 16px -4px rgba(255, 255, 255, 0.1);
		backdrop-filter: blur(16px);
	}
}

.menuItemLink {
	display: flex;
	align-items: center;
	padding: 8px 10px;
	gap: 10px;
	cursor: pointer;
	color: #cbd5e1;
	min-width: 0;
	flex: 1;
	text-decoration: none;
	outline: none;

	&.compact {
		padding: 8px;
		justify-content: center;
	}

	&:active {
		color: #fff;
	}
}

.textContainer {
	display: flex;
	flex-direction: column;
	min-width: 0;
	gap: 1px;
}

.label {
	white-space: nowrap;
	text-overflow: ellipsis;
	overflow: hidden;
	flex: 1;
	line-height: 1.2;
	min-width: 0;
	color: #94a3b8;
	font-size: 11px;
	font-weight: 500;
}

.title {
	white-space: nowrap;
	text-overflow: ellipsis;
	overflow: hidden;
	flex: 1;
	line-height: 1.3;
	min-width: 0;
	color: #f1f5f9;
	font-size: 13px;
	font-weight: 500;
}

.actionDropdown {
	opacity: 0;
	flex-shrink: 0;
	width: 0;
	overflow: hidden;
	transition: opacity 0.2s ease;

	.menuItem:has([aria-expanded='true']) &,
	.menuItem:has(:focus) &,
	.menuItem:hover &,
	.active & {
		width: auto;
		opacity: 1;
	}
}

.actionDropdownTrigger {
	box-shadow: none !important;
	outline: none !important;
	color: #94a3b8 !important;

	&:hover {
		color: #fff !important;
	}
}
</style>
