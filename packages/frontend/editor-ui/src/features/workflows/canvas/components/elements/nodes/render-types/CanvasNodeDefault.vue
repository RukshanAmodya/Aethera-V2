<script lang="ts" setup>
import { computed, ref, useCssModule, watch } from 'vue';
import { useNodeConnections } from '@/app/composables/useNodeConnections';
import { useI18n } from '@n8n/i18n';
import { useCanvasNode } from '../../../../composables/useCanvasNode';
import type { CanvasNodeDefaultRender } from '../../../../canvas.types';
import { injectCanvasRenderData } from '@/features/workflows/canvas/canvas.utils';
import { useCanvas } from '../../../../composables/useCanvas';
import { useZoomAdjustedValues } from '../../../../composables/useZoomAdjustedValues';
import CanvasNodeSettingsIcons from './parts/CanvasNodeSettingsIcons.vue';
import { useNodePrivateCredential } from '@/features/resolvers/composables/useNodePrivateCredential';
import { useNodeHelpers } from '@/app/composables/useNodeHelpers';
import { calculateNodeSize } from '@/app/utils/nodeViewUtils';
import ExperimentalInPlaceNodeSettings from '../../../../experimental/components/ExperimentalEmbeddedNodeDetails.vue';
import CanvasNodeTooltip from './parts/CanvasNodeTooltip.vue';
import CanvasNodeDisabledStrikeThrough from './parts/CanvasNodeDisabledStrikeThrough.vue';
import CanvasNodeStatusIcons from './parts/CanvasNodeStatusIcons.vue';
import NodeIcon from '@/app/components/NodeIcon.vue';
import { useRoute } from 'vue-router';
import { VIEWS } from '@/app/constants';
import { getNodeIconSize, type NodeIconSource } from '@/app/utils/nodeIcon';

const $style = useCssModule();
const i18n = useI18n();

const emit = defineEmits<{
	'open:contextmenu': [event: MouseEvent];
	activate: [id: string, event: MouseEvent];
	'replace:node': [id: string];
}>();

const { initialized, viewport, isExperimentalNdvActive } = useCanvas();
const { calculateNodeBorderOpacityStyle } = useZoomAdjustedValues(viewport);
const route = useRoute();
const {
	id,
	name,
	label,
	subtitle,
	connections,
	isDisabled,
	isReadOnly,
	isSelected,
	executionStatus,
	executionWaiting,
	executionWaitingForNext,
	executionRunning,
	hasRunData,
	runDataIterations,
	render,
	isNotInstalledCommunityNode,
	node,
} = useCanvasNode();
const { hasPrivateCredential, tooltipText: privateCredentialTooltip } =
	useNodePrivateCredential(name);
const renderData = injectCanvasRenderData();
const inputs = computed(() => renderData.value.nodeInputsByNodeId.get(id.value)?.value ?? []);
const outputs = computed(() => renderData.value.nodeOutputsByNodeId.get(id.value)?.value ?? []);
const hasExecutionErrors = computed(
	() => (renderData.value.executionIssuesByNodeId.get(id.value)?.value?.length ?? 0) > 0,
);
const hasPinnedData = computed(
	() =>
		!renderData.value.isExecutionDataDisplayed &&
		!!renderData.value.pinnedDataByNodeName[name.value],
);
const hasExecutionPinData = computed(
	() =>
		renderData.value.isExecutionDataDisplayed &&
		!!renderData.value.executionPinDataByNodeId.get(id.value)?.value,
);
const hasSubstitutedOutput = computed(() => hasPinnedData.value || hasExecutionPinData.value);
const { mainOutputs, mainOutputConnections, mainInputs, mainInputConnections, nonMainInputs } =
	useNodeConnections({
		inputs,
		outputs,
		connections,
	});

const nodeHelpers = useNodeHelpers();
const renderOptions = computed(() => render.value.options as CanvasNodeDefaultRender['options']);
const isDemoRoute = computed(() => route.name === VIEWS.DEMO);

const classes = computed(() => {
	const waiting = Boolean(executionWaiting.value || executionStatus.value === 'waiting');
	const running = Boolean(executionRunning.value || executionWaitingForNext.value);
	return {
		[$style.node]: true,
		[$style.selected]: isSelected.value,
		[$style.disabled]:
			isDisabled.value || (isNotInstalledCommunityNode.value && !isDemoRoute.value),
		[$style.success]: Boolean(
			hasRunData.value && executionStatus.value === 'success' && !hasExecutionPinData.value,
		),
		[$style.error]: hasExecutionErrors.value,
		[$style.running]: running,
		[$style.waiting]: waiting,
		[$style.pinned]: hasSubstitutedOutput.value,
		[$style.configurable]: renderOptions.value.configurable,
		[$style.configuration]: renderOptions.value.configuration,
		[$style.trigger]: renderOptions.value.trigger,
		[$style.warning]: renderOptions.value.dirtiness !== undefined,
		[$style.placeholder]: renderOptions.value.placeholder,
		waiting,
		running,
	};
});

const iconSize = computed(() => {
	const iconName = iconSource.value?.type === 'icon' ? iconSource.value.name : undefined;
	if (renderOptions.value.configuration) return getNodeIconSize('configuration', iconName);
	return getNodeIconSize('canvas', iconName);
});

const nodeSize = computed(() =>
	calculateNodeSize(
		renderOptions.value.configuration ?? false,
		renderOptions.value.configurable ?? false,
		mainInputs.value.length,
		mainOutputs.value.length,
		nonMainInputs.value.length,
		isExperimentalNdvActive.value,
	),
);

const nodeBorderOpacityStyle = calculateNodeBorderOpacityStyle();

const styles = computed(() => ({
	'--canvas-node--width': `${nodeSize.value.width}px`,
	'--canvas-node--height': `${nodeSize.value.height}px`,
	'--node--icon--size': `${iconSize.value}px`,
	...nodeBorderOpacityStyle.value,
}));

const dataTestId = computed(() => {
	let type = 'default';
	if (renderOptions.value.configurable) {
		type = 'configurable';
	} else if (renderOptions.value.configuration) {
		type = 'configuration';
	} else if (renderOptions.value.trigger) {
		type = 'trigger';
	}

	return `canvas-${type}-node`;
});

const isStrikethroughVisible = computed(() => {
	const isSingleMainInputNode =
		mainInputs.value.length === 1 && mainInputConnections.value.length <= 1;
	const isSingleMainOutputNode =
		mainOutputs.value.length === 1 && mainOutputConnections.value.length <= 1;

	return isDisabled.value && isSingleMainInputNode && isSingleMainOutputNode;
});

const iconSource = computed(() => {
	if (renderOptions.value.placeholder) {
		return {
			type: 'icon',
			name: 'plus',
		} as NodeIconSource;
	}

	const source = renderOptions.value.icon;
	// When the node uses a private credential, that icon takes over the node badge
	// slot, replacing any node-specific badge (e.g. the HTTP Request globe).
	if (hasPrivateCredential.value && source) {
		const badge: NodeIconSource['badge'] = {
			type: 'icon',
			name: 'user-round-key',
			tooltip: privateCredentialTooltip.value,
		};
		return { ...source, badge };
	}

	return source;
});

const showTooltip = ref(false);

watch(initialized, () => {
	if (initialized.value) {
		showTooltip.value = true;
	}
});

watch(viewport, () => {
	showTooltip.value = false;
	setTimeout(() => {
		showTooltip.value = true;
	}, 0);
});

const footerStatus = computed(() => {
	if (isDisabled.value) return { text: 'Disabled', type: 'disabled' };
	if (executionStatus.value === 'running') return { text: 'Running', type: 'running' };
	if (executionStatus.value === 'waiting') return { text: 'Waiting', type: 'waiting' };
	if (executionStatus.value === 'error') return { text: 'Error', type: 'error' };
	if (hasRunData.value || executionStatus.value === 'success') {
		return { text: 'Connected', type: 'connected' };
	}
	return { text: 'Ready', type: 'ready' };
});

const executionTimeMs = computed(() => {
	const tasks = renderData.value.executionRunDataByNodeId?.get(id.value)?.value;
	if (!tasks || tasks.length === 0) return null;
	// Calculate sum or last task execution time
	let totalMs = 0;
	let hasValid = false;
	for (const task of tasks) {
		if (typeof task.executionTime === 'number' && task.executionTime >= 0) {
			totalMs += task.executionTime;
			hasValid = true;
		}
	}
	return hasValid ? totalMs : null;
});

function formatExecutionTime(ms: number): string {
	if (ms < 1000) {
		return `${ms}ms`;
	}
	const totalSeconds = ms / 1000;
	if (totalSeconds < 60) {
		// e.g. 1.2s or 45s
		const sec = totalSeconds < 10 ? totalSeconds.toFixed(1) : Math.round(totalSeconds);
		return `${sec}s`;
	}
	const minutes = Math.floor(totalSeconds / 60);
	const remainingSec = Math.round(totalSeconds % 60);
	if (minutes < 60) {
		return remainingSec > 0 ? `${minutes}m ${remainingSec}s` : `${minutes}m`;
	}
	const hours = Math.floor(minutes / 60);
	const remainingMin = minutes % 60;
	return remainingMin > 0 ? `${hours}h ${remainingMin}m` : `${hours}h`;
}

const nodeCategory = computed(() => {
	const type = (node?.data?.value?.type || '').toLowerCase();
	const nameStr = (label.value || '').toLowerCase();

	if (
		renderOptions.value.trigger ||
		type.includes('trigger') ||
		type.includes('webhook') ||
		nameStr.includes('when') ||
		nameStr.includes('schedule')
	) {
		return 'Trigger';
	}
	if (
		type.includes('openai') ||
		type.includes('anthropic') ||
		type.includes('agent') ||
		type.includes('llm') ||
		type.includes('ai') ||
		nameStr.includes('ai') ||
		nameStr.includes('claude') ||
		nameStr.includes('gpt')
	) {
		return 'AI Agent';
	}
	if (
		type.includes('code') ||
		type.includes('function') ||
		type.includes('javascript') ||
		type.includes('python')
	) {
		return 'Script';
	}
	if (
		type.includes('if') ||
		type.includes('switch') ||
		type.includes('filter') ||
		type.includes('router') ||
		nameStr.includes('if') ||
		nameStr.includes('switch') ||
		nameStr.includes('score')
	) {
		return 'Logic';
	}
	if (
		type.includes('sheet') ||
		type.includes('airtable') ||
		type.includes('database') ||
		type.includes('postgres') ||
		type.includes('sql') ||
		type.includes('table')
	) {
		return 'Database';
	}
	if (
		type.includes('slack') ||
		type.includes('discord') ||
		type.includes('telegram') ||
		type.includes('email') ||
		type.includes('mail')
	) {
		return 'Message';
	}
	if (
		type.includes('edit') ||
		type.includes('set') ||
		nameStr.includes('set') ||
		nameStr.includes('edit')
	) {
		return 'Fields';
	}
	if (type.includes('wait') || nameStr.includes('wait')) {
		return 'Wait';
	}
	return 'Action';
});

const categoryClass = computed(() => {
	switch (nodeCategory.value) {
		case 'Trigger':
			return $style.catTrigger;
		case 'AI Agent':
			return $style.catAi;
		case 'Script':
			return $style.catScript;
		case 'Logic':
			return $style.catLogic;
		case 'Database':
			return $style.catDatabase;
		case 'Message':
			return $style.catMessage;
		case 'Fields':
			return $style.catFields;
		case 'Wait':
			return $style.catWait;
		default:
			return $style.catAction;
	}
});

const executionTimeDisplay = computed(() => {
	if (executionTimeMs.value !== null) {
		return formatExecutionTime(executionTimeMs.value);
	}
	if (executionStatus.value === 'running') {
		return 'running...';
	}
	return nodeCategory.value;
});

const footerLeftInfo = computed(() => {
	if (runDataIterations.value && runDataIterations.value > 1) {
		return `${runDataIterations.value} runs`;
	}
	if (hasRunData.value) {
		return '1 item';
	}
	return '';
});

function openContextMenu(event: MouseEvent) {
	emit('open:contextmenu', event);
}

function onActivate(event: MouseEvent) {
	if (renderOptions.value.placeholder) {
		emit('replace:node', id.value);
		return;
	}

	emit('activate', id.value, event);
}
</script>

<template>
	<ExperimentalInPlaceNodeSettings
		v-if="isExperimentalNdvActive"
		:node-id="id"
		:class="classes"
		:style="styles"
		:is-read-only="isReadOnly"
		:is-configurable="renderOptions.configurable ?? false"
	/>
	<div
		v-else
		:class="classes"
		:style="styles"
		:data-test-id="dataTestId"
		@contextmenu="openContextMenu"
		@dblclick.stop="onActivate"
	>
		<CanvasNodeTooltip v-if="renderOptions.tooltip" :visible="showTooltip" />
		<CanvasNodeSettingsIcons
			v-if="
				!renderOptions.configuration &&
				!isDisabled &&
				!(hasSubstitutedOutput && !nodeHelpers.isProductionExecutionPreview.value)
			"
		/>
		<CanvasNodeDisabledStrikeThrough v-if="isStrikethroughVisible" />

		<!-- Card Content: Header (Icon Box + Titles + Aligned Status Icon) -->
		<div :class="$style.header">
			<div :class="[$style.iconBox, categoryClass]">
				<NodeIcon
					:icon-source="iconSource"
					:size="22"
					:shrink="false"
					:disabled="isDisabled"
					:class="$style.icon"
				/>
			</div>
			<div :class="$style.titleWrapper">
				<h2 v-if="label" :class="$style.label" :title="label">
					{{ label }}
				</h2>
				<span v-if="subtitle" :class="$style.subtitle" :title="subtitle">
					{{ subtitle }}
				</span>
				<span v-else-if="isDisabled" :class="$style.disabledText">
					({{ i18n.baseText('node.disabled') }})
				</span>
			</div>
			<CanvasNodeStatusIcons v-if="!isDisabled" :class="$style.statusIcons" />
		</div>

		<!-- Card Content: Footer (Left Meta + Right Connected Status) -->
		<div :class="$style.footer">
			<div :class="$style.footerLeft">
				<span :class="[$style.categoryBadge, categoryClass]">
					<svg
						:class="$style.timeIcon"
						width="12"
						height="12"
						viewBox="0 0 24 24"
						fill="none"
						stroke="currentColor"
						stroke-width="2"
						stroke-linecap="round"
						stroke-linejoin="round"
					>
						<circle cx="12" cy="12" r="10" />
						<polyline points="12 6 12 12 16 14" />
					</svg>
					<span :class="$style.categoryLabel">{{ executionTimeDisplay }}</span>
				</span>
				<span v-if="footerLeftInfo" :class="$style.footerLeftText">{{ footerLeftInfo }}</span>
			</div>

			<div :class="[$style.statusBadge, $style[footerStatus.type]]">
				<span :class="$style.statusDot" />
				<span>{{ footerStatus.text }}</span>
			</div>
		</div>
	</div>
</template>

<style lang="scss" module>
@use './_canvasNodeStyles.scss' as styles;

.node {
	@include styles.canvas-node-border-defaults;
	--trigger-node--radius: 22px;
	--canvas-node--status-icons--margin: var(--spacing--3xs);
	--node--icon--color: var(--color--foreground--shade-1);

	position: relative;
	height: var(--canvas-node--height, 128px);
	width: var(--canvas-node--width, 288px);
	display: flex;
	flex-direction: column;
	justify-content: space-between;
	padding: 14px 16px;
	background: #141518;
	border: 1px solid #2a2c33;
	border-radius: 22px;
	box-shadow:
		0 16px 32px -4px rgba(0, 0, 0, 0.6),
		0 4px 8px -2px rgba(0, 0, 0, 0.4);
	backdrop-filter: blur(8px);
	-webkit-backdrop-filter: blur(8px);
	transition:
		transform 0.18s cubic-bezier(0.16, 1, 0.3, 1),
		box-shadow 0.18s cubic-bezier(0.16, 1, 0.3, 1),
		border-color 0.18s ease;

	&:hover {
		transform: translateY(-2px);
		border-color: #3f424e;
		box-shadow:
			0 20px 36px -4px rgba(0, 0, 0, 0.7),
			0 6px 12px -2px rgba(0, 0, 0, 0.5);
	}

	&.selected {
		border-color: #ff4d6d;
		box-shadow:
			0 0 0 2px #ff4d6d,
			0 16px 32px -4px rgba(255, 77, 109, 0.25);
	}

	&.running {
		border-color: #6366f1;
		box-shadow: 0 0 16px -2px rgba(99, 102, 241, 0.45);
	}

	&.error {
		border-color: #ff4d6d;
		box-shadow: 0 0 14px -2px rgba(255, 77, 109, 0.35);
	}

	&.disabled {
		opacity: 0.6;
		filter: grayscale(0.5);
	}
}

.header {
	display: flex;
	align-items: center;
	gap: 12px;
	min-width: 0;
	position: relative;
}

.iconBox {
	width: 44px;
	height: 44px;
	border-radius: 12px;
	background: #1e2027;
	border: 1px solid rgba(255, 255, 255, 0.08);
	display: flex;
	align-items: center;
	justify-content: center;
	flex-shrink: 0;
	color: #fff;
	box-shadow: 0 4px 12px rgba(0, 0, 0, 0.35);
	transition: all 0.2s cubic-bezier(0.16, 1, 0.3, 1);
	position: relative;
	overflow: hidden;

	/* Vibrancy & contrast for dark mode icons */
	img,
	svg {
		filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.5));
	}

	/* Category Distinct Styling for Icon Badges */
	&.catTrigger {
		background: radial-gradient(
			circle at top left,
			rgba(6, 182, 212, 0.28),
			rgba(20, 24, 33, 0.95)
		);
		border: 1px solid rgba(6, 182, 212, 0.4);
		box-shadow:
			0 4px 14px rgba(6, 182, 212, 0.2),
			0 0 0 1px rgba(6, 182, 212, 0.15) inset;
		--canvas-node--icon-color: #38bdf8;
		color: #38bdf8;
	}

	&.catAi {
		background: radial-gradient(
			circle at top left,
			rgba(168, 85, 247, 0.3),
			rgba(24, 20, 36, 0.95)
		);
		border: 1px solid rgba(168, 85, 247, 0.45);
		box-shadow:
			0 4px 14px rgba(168, 85, 247, 0.25),
			0 0 0 1px rgba(168, 85, 247, 0.2) inset;
		--canvas-node--icon-color: #c084fc;
		color: #c084fc;
	}

	&.catScript {
		background: radial-gradient(
			circle at top left,
			rgba(234, 179, 8, 0.25),
			rgba(28, 26, 20, 0.95)
		);
		border: 1px solid rgba(234, 179, 8, 0.38);
		box-shadow:
			0 4px 14px rgba(234, 179, 8, 0.2),
			0 0 0 1px rgba(234, 179, 8, 0.15) inset;
		--canvas-node--icon-color: #facc15;
		color: #facc15;
	}

	&.catLogic {
		background: radial-gradient(
			circle at top left,
			rgba(249, 115, 22, 0.25),
			rgba(30, 22, 18, 0.95)
		);
		border: 1px solid rgba(249, 115, 22, 0.38);
		box-shadow:
			0 4px 14px rgba(249, 115, 22, 0.2),
			0 0 0 1px rgba(249, 115, 22, 0.15) inset;
		--canvas-node--icon-color: #fb923c;
		color: #fb923c;
	}

	&.catDatabase {
		background: radial-gradient(
			circle at top left,
			rgba(59, 130, 246, 0.28),
			rgba(18, 23, 36, 0.95)
		);
		border: 1px solid rgba(59, 130, 246, 0.4);
		box-shadow:
			0 4px 14px rgba(59, 130, 246, 0.2),
			0 0 0 1px rgba(59, 130, 246, 0.15) inset;
		--canvas-node--icon-color: #60a5fa;
		color: #60a5fa;
	}

	&.catMessage {
		background: radial-gradient(
			circle at top left,
			rgba(34, 197, 94, 0.25),
			rgba(18, 28, 22, 0.95)
		);
		border: 1px solid rgba(34, 197, 94, 0.38);
		box-shadow:
			0 4px 14px rgba(34, 197, 94, 0.2),
			0 0 0 1px rgba(34, 197, 94, 0.15) inset;
		--canvas-node--icon-color: #4ade80;
		color: #4ade80;
	}

	&.catFields {
		background: radial-gradient(
			circle at top left,
			rgba(236, 72, 153, 0.25),
			rgba(32, 18, 26, 0.95)
		);
		border: 1px solid rgba(236, 72, 153, 0.38);
		box-shadow:
			0 4px 14px rgba(236, 72, 153, 0.2),
			0 0 0 1px rgba(236, 72, 153, 0.15) inset;
		--canvas-node--icon-color: #f472b6;
		color: #f472b6;
	}

	&.catWait {
		background: radial-gradient(
			circle at top left,
			rgba(148, 163, 184, 0.22),
			rgba(24, 26, 30, 0.95)
		);
		border: 1px solid rgba(148, 163, 184, 0.35);
		box-shadow:
			0 4px 14px rgba(148, 163, 184, 0.15),
			0 0 0 1px rgba(148, 163, 184, 0.1) inset;
		--canvas-node--icon-color: #cbd5e1;
		color: #cbd5e1;
	}

	&.catAction {
		background: radial-gradient(
			circle at top left,
			rgba(99, 102, 241, 0.25),
			rgba(20, 22, 32, 0.95)
		);
		border: 1px solid rgba(99, 102, 241, 0.35);
		box-shadow:
			0 4px 14px rgba(99, 102, 241, 0.18),
			0 0 0 1px rgba(99, 102, 241, 0.12) inset;
		--canvas-node--icon-color: #818cf8;
		color: #818cf8;
	}
}

.titleWrapper {
	display: flex;
	flex-direction: column;
	min-width: 0;
	flex: 1;
	overflow: hidden;
	padding-right: 24px;
}

.label {
	font-size: 15px;
	font-weight: 500;
	letter-spacing: -0.01em;
	line-height: 1.25;
	color: #ffffff;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
	margin: 0;
}

.subtitle,
.disabledText {
	font-size: 12px;
	color: #717682;
	font-weight: 400;
	margin-top: 2px;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
}

.disabledText {
	color: #f87171;
}

.footer {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding-top: 6px;
}

.footerLeft {
	display: flex;
	align-items: center;
	gap: 8px;
}

.categoryBadge {
	display: inline-flex;
	align-items: center;
	gap: 4.5px;
	padding: 2.5px 8px;
	border-radius: 7px;
	background: #1c1e24;
	border: 1px solid rgba(255, 255, 255, 0.06);
	color: #9ca3af;

	&.catTrigger {
		background: rgba(6, 182, 212, 0.1);
		border-color: rgba(6, 182, 212, 0.25);
		color: #38bdf8;
		.timeIcon {
			color: #38bdf8;
		}
		.categoryLabel {
			color: #bae6fd;
		}
	}

	&.catAi {
		background: rgba(168, 85, 247, 0.12);
		border-color: rgba(168, 85, 247, 0.28);
		color: #c084fc;
		.timeIcon {
			color: #c084fc;
		}
		.categoryLabel {
			color: #e9d5ff;
		}
	}

	&.catScript {
		background: rgba(234, 179, 8, 0.1);
		border-color: rgba(234, 179, 8, 0.25);
		color: #facc15;
		.timeIcon {
			color: #facc15;
		}
		.categoryLabel {
			color: #fef08a;
		}
	}

	&.catLogic {
		background: rgba(249, 115, 22, 0.1);
		border-color: rgba(249, 115, 22, 0.25);
		color: #fb923c;
		.timeIcon {
			color: #fb923c;
		}
		.categoryLabel {
			color: #fed7aa;
		}
	}

	&.catDatabase {
		background: rgba(59, 130, 246, 0.1);
		border-color: rgba(59, 130, 246, 0.25);
		color: #60a5fa;
		.timeIcon {
			color: #60a5fa;
		}
		.categoryLabel {
			color: #bfdbfe;
		}
	}

	&.catMessage {
		background: rgba(34, 197, 94, 0.1);
		border-color: rgba(34, 197, 94, 0.25);
		color: #4ade80;
		.timeIcon {
			color: #4ade80;
		}
		.categoryLabel {
			color: #bbf7d0;
		}
	}

	&.catFields {
		background: rgba(236, 72, 153, 0.1);
		border-color: rgba(236, 72, 153, 0.25);
		color: #f472b6;
		.timeIcon {
			color: #f472b6;
		}
		.categoryLabel {
			color: #fbcfe8;
		}
	}

	&.catWait {
		background: rgba(148, 163, 184, 0.1);
		border-color: rgba(148, 163, 184, 0.2);
		color: #94a3b8;
		.timeIcon {
			color: #94a3b8;
		}
		.categoryLabel {
			color: #cbd5e1;
		}
	}

	&.catAction {
		background: rgba(99, 102, 241, 0.1);
		border-color: rgba(99, 102, 241, 0.22);
		color: #818cf8;
		.timeIcon {
			color: #818cf8;
		}
		.categoryLabel {
			color: #c7d2fe;
		}
	}
}

.timeIcon {
	width: 11px;
	height: 11px;
	color: #94a3b8;
	opacity: 0.85;
}

.categoryLabel {
	font-size: 10.5px;
	font-weight: 500;
	letter-spacing: 0.02em;
	color: #cbd5e1;
	font-variant-numeric: tabular-nums;
}

.footerLeftText {
	font-size: 11px;
	color: #8c93a0;
	font-weight: 500;
}

.statusBadge {
	display: inline-flex;
	align-items: center;
	gap: 5px;
	padding: 3px 10px;
	border-radius: 9999px;
	font-size: 11px;
	font-weight: 500;
	letter-spacing: 0.01em;

	&.connected,
	&.success {
		background: #0d2617;
		border: 1px solid rgba(22, 78, 41, 0.6);
		color: #22c55e;

		.statusDot {
			background: #22c55e;
			box-shadow: 0 0 6px #22c55e;
		}
	}

	&.running {
		background: #141f38;
		border: 1px solid rgba(30, 58, 138, 0.6);
		color: #60a5fa;

		.statusDot {
			background: #60a5fa;
			box-shadow: 0 0 6px #60a5fa;
		}
	}

	&.waiting,
	&.ready {
		background: #1a1d24;
		border: 1px solid rgba(255, 255, 255, 0.08);
		color: #9ca3af;

		.statusDot {
			background: #9ca3af;
		}
	}

	&.error {
		background: #2a1318;
		border: 1px solid rgba(255, 77, 109, 0.3);
		color: #f87171;

		.statusDot {
			background: #ff4d6d;
			box-shadow: 0 0 6px #ff4d6d;
		}
	}

	&.disabled {
		background: #181a20;
		border: 1px solid rgba(255, 255, 255, 0.05);
		color: #6b7280;

		.statusDot {
			background: #6b7280;
		}
	}
}

.statusDot {
	width: 5px;
	height: 5px;
	border-radius: 50%;
	display: inline-block;
}

.statusIcons {
	position: absolute;
	top: 0;
	right: 0;
	z-index: 10;
}

.icon {
	flex-grow: 0;
	flex-shrink: 0;
	transition: transform 0.18s ease;

	.node:hover & {
		transform: scale(1.05);
	}
}
</style>
