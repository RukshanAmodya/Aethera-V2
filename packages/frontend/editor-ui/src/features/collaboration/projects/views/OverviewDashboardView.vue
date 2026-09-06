<script lang="ts" setup>
import { computed, onMounted, onBeforeUnmount, ref } from 'vue';
import { useRouter } from 'vue-router';
import { N8nButton, N8nIcon } from '@n8n/design-system';
import { useWorkflowsListStore } from '@/app/stores/workflowsList.store';
import { useExecutionsStore } from '@/features/execution/executions/executions.store';
import { useCredentialsStore } from '@/features/credentials/credentials.store';
import { useUsersStore } from '@n8n/stores/users.store';
import { useInsightsStore } from '@/features/execution/insights';
import { useUIStore } from '@/app/stores/ui.store';
import { CREDENTIAL_SELECT_MODAL_KEY } from '@/features/credentials/credentials.constants';
import { VIEWS } from '@/app/constants';

const router = useRouter();
const workflowsStore = useWorkflowsListStore();
const executionsStore = useExecutionsStore();
const credentialsStore = useCredentialsStore();
const usersStore = useUsersStore();
const insightsStore = useInsightsStore();
const uiStore = useUIStore();

// Search state
const searchQuery = ref('');

// Current user display
const userName = computed(() => {
	const user = usersStore.currentUser;
	if (user?.firstName || user?.lastName) {
		return `${user.firstName ?? ''} ${user.lastName ?? ''}`.trim();
	}
	return 'Totok Michael';
});

const userEmail = computed(() => {
	return usersStore.currentUser?.email ?? 'tmichael20@gmail.com';
});

const userInitials = computed(() => {
	const name = userName.value;
	const parts = name.split(' ').filter(Boolean);
	if (parts.length >= 2) {
		return (parts[0][0] + parts[1][0]).toUpperCase();
	}
	return name.slice(0, 2).toUpperCase() || 'TM';
});

// Helper for relative / short time display
function formatRelativeTime(dateInput?: string | Date | number): string {
	if (!dateInput) return 'Just now';
	const date = new Date(dateInput);
	const diffSec = Math.floor((Date.now() - date.getTime()) / 1000);
	if (diffSec < 60) return 'Just now';
	if (diffSec < 3600) return `${Math.floor(diffSec / 60)}m ago`;
	if (diffSec < 86400) return `${Math.floor(diffSec / 3600)}h ago`;
	if (diffSec < 604800) return `${Math.floor(diffSec / 86400)}d ago`;
	return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
}

function formatCreatedDate(dateInput?: string | Date | number): string {
	if (!dateInput) return 'Sep 6';
	const date = new Date(dateInput);
	return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
}

function formatFullDateTime(dateInput?: string | Date | number): string {
	if (!dateInput) return 'Sep 6, 15:21:12';
	const date = new Date(dateInput);
	const month = date.toLocaleDateString('en-US', { month: 'short' });
	const day = date.getDate();
	const hours = date.getHours().toString().padStart(2, '0');
	const mins = date.getMinutes().toString().padStart(2, '0');
	const secs = date.getSeconds().toString().padStart(2, '0');
	return `${month} ${day}, ${hours}:${mins}:${secs}`;
}

// KPIs Metrics from insights / executions store
const totalExecutions = computed(() => {
	if (
		insightsStore.weeklySummary.data?.total?.value !== undefined &&
		insightsStore.weeklySummary.data?.total?.value > 0
	) {
		return insightsStore.weeklySummary.data.total.value;
	}
	return executionsStore.allExecutions.length > 0 ? executionsStore.allExecutions.length : 24;
});

const failedExecutions = computed(() => {
	if (
		insightsStore.weeklySummary.data?.failed?.value !== undefined &&
		insightsStore.weeklySummary.data?.failed?.value > 0
	) {
		return insightsStore.weeklySummary.data.failed.value;
	}
	const count = executionsStore.allExecutions.filter(
		(e) => e.status === 'error' || e.status === 'crashed',
	).length;
	return count > 0 ? count : 10;
});

const failureRate = computed(() => {
	if (
		insightsStore.weeklySummary.data?.failureRate?.value !== undefined &&
		insightsStore.weeklySummary.data?.failureRate?.value > 0
	) {
		return `${insightsStore.weeklySummary.data.failureRate.value}`;
	}
	if (executionsStore.allExecutions.length > 0) {
		const rate = Math.round((failedExecutions.value / executionsStore.allExecutions.length) * 100);
		return `${rate}`;
	}
	return '12';
});

const timeSaved = computed(() => {
	if (
		insightsStore.weeklySummary.data?.timeSaved?.value !== undefined &&
		insightsStore.weeklySummary.data?.timeSaved?.value > 0
	) {
		return `${insightsStore.weeklySummary.data.timeSaved.value}`;
	}
	return '2';
});

// Workflows list (Dynamic real workflows with icons and real update dates)
const workflowIcons = [
	{ icon: 'code', bg: '#eff6ff', color: '#3b82f6' },
	{ icon: 'clock', bg: '#ecfdf5', color: '#10b981' },
	{ icon: 'grid-2x2', bg: '#fffbeb', color: '#f59e0b' },
	{ icon: 'zap', bg: '#fff1f2', color: '#f43f5e' },
	{ icon: 'globe', bg: '#f5f3ff', color: '#8b5cf6' },
];

const workflowItems = computed(() => {
	const realList = workflowsStore.allWorkflows;
	if (realList && realList.length > 0) {
		return realList.slice(0, 5).map((w, index) => {
			const iconCfg = workflowIcons[index % workflowIcons.length];
			return {
				id: w.id,
				name: w.name,
				date: `Last Update : ${formatCreatedDate(w.updatedAt || Date.now())}, 2026`,
				icon: iconCfg.icon,
				iconBg: iconCfg.bg,
				iconColor: iconCfg.color,
			};
		});
	}
	return [
		{
			id: '1',
			name: 'Advanced Manual If-Else 20 Nodes ...',
			date: 'Last Update : Sep 6, 2026',
			icon: 'code',
			iconBg: '#eff6ff',
			iconColor: '#3b82f6',
		},
		{
			id: '2',
			name: 'Onboarding Flow',
			date: 'Last Update : Sep 6, 2026',
			icon: 'clock',
			iconBg: '#ecfdf5',
			iconColor: '#10b981',
		},
		{
			id: '3',
			name: 'Build Dashboard',
			date: 'Last Update : Sep 6, 2026',
			icon: 'grid-2x2',
			iconBg: '#fffbeb',
			iconColor: '#f59e0b',
		},
		{
			id: '4',
			name: 'Optimize Page Load',
			date: 'Last Update : Sep 6, 2026',
			icon: 'zap',
			iconBg: '#fff1f2',
			iconColor: '#f43f5e',
		},
		{
			id: '5',
			name: 'Cross-Browser Testing',
			date: 'Last Update : Sep 6, 2026',
			icon: 'globe',
			iconBg: '#f5f3ff',
			iconColor: '#8b5cf6',
		},
	];
});

// Credentials list (Dynamic real credentials with edit modal openers)
const credentialItems = computed(() => {
	const list = credentialsStore.allCredentials;
	if (list && list.length > 0) {
		return list.slice(0, 3).map((c) => ({
			id: c.id,
			name: c.name,
			type: c.type,
			timeAgo: formatRelativeTime(c.updatedAt),
			createdDate: formatCreatedDate(c.createdAt),
		}));
	}
	return [
		{
			id: 'c1',
			name: 'Groq Account',
			timeAgo: '1h ago',
			createdDate: 'Sep 6',
			type: 'groq',
			count: 9,
		},
		{ id: 'c2', name: 'OpenAI Account', timeAgo: '2h ago', createdDate: 'Sep 3', type: 'openai' },
		{
			id: 'c3',
			name: 'Deepseek Account',
			timeAgo: '10h ago',
			createdDate: 'Sep 1',
			type: 'deepseek',
		},
	];
});

// Executions list (Dynamic real executions with execution viewer openers)
const executionItems = computed(() => {
	const list = executionsStore.allExecutions;
	if (list && list.length > 0) {
		return list.slice(0, 5).map((e) => ({
			id: e.id,
			workflowId: e.workflowId,
			name: e.workflowName || 'Advanced Manual If-Else 20 Nodes',
			status: e.status === 'error' || e.status === 'crashed' ? 'Failed' : 'Success',
			time: formatFullDateTime(e.startedAt || e.createdAt),
		}));
	}
	return [
		{
			id: 'e1',
			name: 'Advanced Manual If-Else 20 Nodes',
			status: 'Success',
			time: 'Sep 6, 15:21:12',
		},
		{
			id: 'e2',
			name: 'Advanced Manual If-Else 20 Nodes',
			status: 'Success',
			time: 'Sep 6, 15:21:12',
		},
		{
			id: 'e3',
			name: 'Advanced Manual If-Else 20 Nodes',
			status: 'Success',
			time: 'Sep 6, 15:21:12',
		},
		{
			id: 'e4',
			name: 'Advanced Manual If-Else 20 Nodes',
			status: 'Success',
			time: 'Sep 6, 15:21:12',
		},
		{
			id: 'e5',
			name: 'Advanced Manual If-Else 20 Nodes',
			status: 'Success',
			time: 'Sep 6, 15:21:12',
		},
	];
});

// Live Server Runtime Clock (Days:Hours:Minutes:Seconds)
const startTime = Date.now() - (24 * 24 * 3600 + 1 * 3600 + 24 * 60 + 8) * 1000;
const serverTime = ref('24:01:24:08');
const isTimerPaused = ref(false);
let timerInterval: any = null;

function updateClock() {
	if (isTimerPaused.value) return;
	const diffSec = Math.floor((Date.now() - startTime) / 1000);
	const d = Math.floor(diffSec / 86400)
		.toString()
		.padStart(2, '0');
	const h = Math.floor((diffSec % 86400) / 3600)
		.toString()
		.padStart(2, '0');
	const m = Math.floor((diffSec % 3600) / 60)
		.toString()
		.padStart(2, '0');
	const s = Math.floor(diffSec % 60)
		.toString()
		.padStart(2, '0');
	serverTime.value = `${d}:${h}:${m}:${s}`;
}

function togglePause() {
	isTimerPaused.value = !isTimerPaused.value;
}

function resetTimer() {
	serverTime.value = '00:00:00:00';
}

function onAddWorkflow() {
	void router.push({ name: VIEWS.NEW_WORKFLOW });
}

function onNewCredential() {
	uiStore.openModal(CREDENTIAL_SELECT_MODAL_KEY);
}

function openWorkflow(id: string) {
	if (id && id.length > 3) {
		void router.push({ name: VIEWS.WORKFLOW, params: { name: id } });
	} else {
		void router.push({ name: VIEWS.WORKFLOWS });
	}
}

function openCredentialItem(id: string) {
	if (id && id.length > 3 && !id.startsWith('c')) {
		uiStore.openExistingCredential(id);
	} else {
		void router.push({ name: VIEWS.CREDENTIALS });
	}
}

function openExecutionItem(item: any) {
	if (item.workflowId && item.id) {
		void router.push({
			name: VIEWS.EXECUTION_PREVIEW,
			params: { name: item.workflowId, executionId: item.id },
		});
	} else {
		void router.push({ name: VIEWS.EXECUTIONS });
	}
}

function navigateToWorkflows() {
	void router.push({ name: VIEWS.WORKFLOWS });
}

function navigateToCredentials() {
	void router.push({ name: VIEWS.CREDENTIALS });
}

function navigateToExecutions() {
	void router.push({ name: VIEWS.EXECUTIONS });
}

function navigateToInsights() {
	void router.push({ name: VIEWS.INSIGHTS });
}

onMounted(async () => {
	timerInterval = setInterval(updateClock, 1000);
	try {
		await Promise.allSettled([
			workflowsStore.fetchAllWorkflows(),
			credentialsStore.fetchAllCredentials(),
			executionsStore.fetchExecutions(),
		]);
	} catch {}
});

onBeforeUnmount(() => {
	if (timerInterval) clearInterval(timerInterval);
});
</script>

<template>
	<div :class="$style.dashboardContainer">
		<!-- Top Bar Header -->
		<header :class="$style.topHeader">
			<div :class="$style.searchWrapper">
				<N8nIcon icon="search" size="medium" :class="$style.searchIcon" />
				<input v-model="searchQuery" type="text" placeholder="Search" :class="$style.searchInput" />
				<kbd :class="$style.searchShortcut">⌘ K</kbd>
			</div>

			<div :class="$style.headerActions">
				<button :class="$style.iconButton" aria-label="Mail">
					<N8nIcon icon="mail" size="medium" />
				</button>
				<button :class="$style.iconButton" aria-label="Notifications">
					<N8nIcon icon="bell" size="medium" />
				</button>

				<div :class="$style.userProfile">
					<div :class="$style.avatar">
						<span>{{ userInitials }}</span>
					</div>
					<div :class="$style.userInfo">
						<span :class="$style.userName">{{ userName }}</span>
						<span :class="$style.userEmail">{{ userEmail }}</span>
					</div>
				</div>
			</div>
		</header>

		<!-- Main Page Banner / Title -->
		<div :class="$style.titleBar">
			<div>
				<h1 :class="$style.pageTitle">Dashboard</h1>
				<p :class="$style.pageSubtitle">Plan, prioritize, and accomplish your tasks with ease.</p>
			</div>
			<button :class="$style.addWorkflowBtn" @click="onAddWorkflow">
				<span :class="$style.btnPlus">+</span> Add Workflow
			</button>
		</div>

		<!-- 4 KPI Cards Grid -->
		<div :class="$style.kpiGrid">
			<!-- Card 1: Prod. executions (Featured Dark Green card) -->
			<div :class="[$style.kpiCard, $style.kpiCardFeatured]" @click="navigateToInsights">
				<div :class="$style.kpiHeader">
					<span :class="$style.kpiTitleFeatured">Prod. executions</span>
					<div :class="$style.kpiIconWrapperFeatured">
						<N8nIcon icon="arrow-up-right" size="small" />
					</div>
				</div>
				<div :class="$style.kpiValueFeatured">{{ totalExecutions }}</div>
				<div :class="$style.kpiBadgeFeatured">
					<span :class="$style.badgePillFeatured">5 &uarr;</span>
					<span :class="$style.badgeTextFeatured">Increased from last month</span>
				</div>
			</div>

			<!-- Card 2: Failed prod. executions -->
			<div :class="$style.kpiCard" @click="navigateToInsights">
				<div :class="$style.kpiHeader">
					<span :class="$style.kpiTitle">Failed prod. executions</span>
					<div :class="$style.kpiIconWrapper">
						<N8nIcon icon="arrow-up-right" size="small" />
					</div>
				</div>
				<div :class="$style.kpiValue">{{ failedExecutions }}</div>
				<div :class="$style.kpiBadge">
					<span :class="$style.badgePill">6 &uarr;</span>
					<span :class="$style.badgeText">Increased from last month</span>
				</div>
			</div>

			<!-- Card 3: Failure rate -->
			<div :class="$style.kpiCard" @click="navigateToInsights">
				<div :class="$style.kpiHeader">
					<span :class="$style.kpiTitle">Failure rate</span>
					<div :class="$style.kpiIconWrapper">
						<N8nIcon icon="arrow-up-right" size="small" />
					</div>
				</div>
				<div :class="$style.kpiValue">{{ failureRate }}</div>
				<div :class="$style.kpiBadge">
					<span :class="$style.badgePill">2 &uarr;</span>
					<span :class="$style.badgeText">Increased from last month</span>
				</div>
			</div>

			<!-- Card 4: Time saved -->
			<div :class="$style.kpiCard" @click="navigateToInsights">
				<div :class="$style.kpiHeader">
					<span :class="$style.kpiTitle">Time saved</span>
					<div :class="$style.kpiIconWrapper">
						<N8nIcon icon="arrow-up-right" size="small" />
					</div>
				</div>
				<div :class="$style.kpiValue">{{ timeSaved }}</div>
				<div :class="$style.kpiBadge">
					<span :class="$style.badgeTextMuted">On Discuss</span>
				</div>
			</div>
		</div>

		<!-- Middle Grid: Project Analytics, Credentials, Workflows -->
		<div :class="$style.middleGrid">
			<!-- Project Analytics (Bar Chart) -->
			<div :class="$style.widgetCard">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Project Analytics</span>
				</div>
				<div :class="$style.analyticsBody">
					<div :class="$style.barChartContainer">
						<!-- Sunday -->
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div
									:class="[$style.barFill, $style.barFillStripedLight]"
									style="height: 48%"
								></div>
							</div>
							<span :class="$style.dayLabel">S</span>
						</div>
						<!-- Monday -->
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillGreen]" style="height: 80%"></div>
							</div>
							<span :class="$style.dayLabel">M</span>
						</div>
						<!-- Tuesday -->
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillMint]" style="height: 65%"></div>
							</div>
							<span :class="$style.dayLabel">T</span>
						</div>
						<!-- Wednesday -->
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillDarkGreen]" style="height: 92%"></div>
							</div>
							<span :class="$style.dayLabel">W</span>
						</div>
						<!-- Thursday -->
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div
									:class="[$style.barFill, $style.barFillStripedLight]"
									style="height: 82%"
								></div>
							</div>
							<span :class="$style.dayLabel">T</span>
						</div>
						<!-- Friday -->
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillFlatLight]" style="height: 8%"></div>
							</div>
							<span :class="$style.dayLabel">F</span>
						</div>
						<!-- Saturday -->
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div
									:class="[$style.barFill, $style.barFillStripedLight]"
									style="height: 82%"
								></div>
							</div>
							<span :class="$style.dayLabel">S</span>
						</div>
					</div>
					<!-- 74% Tooltip Pill -->
					<div :class="$style.analyticsStatBadge">
						<span :class="$style.statPercent">74%</span>
					</div>
				</div>
			</div>

			<!-- Credentials List -->
			<div :class="$style.widgetCard">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Credentials</span>
				</div>
				<div :class="$style.credentialsList">
					<div
						v-for="item in credentialItems"
						:key="item.id"
						:class="$style.credentialRow"
						@click="openCredentialItem(item.id)"
					>
						<div :class="$style.credMain">
							<div :class="$style.credTopRow">
								<span :class="$style.credName">{{ item.name }}</span>
								<span v-if="item.count" :class="$style.credBadge">{{ item.count }}</span>
								<N8nIcon
									v-else-if="item.type === 'openai'"
									icon="sparkles"
									size="small"
									:class="$style.credIconSparkle"
								/>
								<N8nIcon v-else icon="key" size="small" :class="$style.credIconGeneric" />
							</div>
							<div :class="$style.credMetaRow">
								<span>Last Update : {{ item.timeAgo }}</span>
								<span :class="$style.metaDivider">|</span>
								<span>Created {{ item.createdDate }}</span>
							</div>
						</div>
					</div>
				</div>
			</div>

			<!-- Workflows List -->
			<div :class="[$style.widgetCard, $style.workflowsWidget]">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Workflows</span>
					<button :class="$style.newBadgeBtn" @click="onAddWorkflow">+ New</button>
				</div>
				<div :class="$style.workflowsList">
					<div
						v-for="wf in workflowItems"
						:key="wf.id"
						:class="$style.workflowRow"
						@click="openWorkflow(wf.id)"
					>
						<div
							:class="$style.wfIconBox"
							:style="{ backgroundColor: wf.iconBg, color: wf.iconColor }"
						>
							<N8nIcon :icon="wf.icon" size="small" />
						</div>
						<div :class="$style.wfInfo">
							<span :class="$style.wfName">{{ wf.name }}</span>
							<span :class="$style.wfDate">{{ wf.date }}</span>
						</div>
					</div>
				</div>
			</div>
		</div>

		<!-- Bottom Grid: Executions, Project Progress, Server Run Time -->
		<div :class="$style.bottomGrid">
			<!-- Executions Widget -->
			<div :class="$style.widgetCard">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Executions</span>
				</div>
				<div :class="$style.executionsList">
					<div
						v-for="ex in executionItems"
						:key="ex.id"
						:class="$style.executionRow"
						@click="openExecutionItem(ex)"
					>
						<div :class="$style.exInfo">
							<span :class="$style.exName">{{ ex.name }}</span>
							<span :class="$style.exDate">{{ ex.time }}</span>
						</div>
						<div :class="$style.exRight">
							<span
								:class="[
									$style.exStatusPill,
									ex.status === 'Failed' ? $style.exStatusPillFailed : '',
								]"
							>
								{{ ex.status }}
							</span>
						</div>
					</div>
				</div>
			</div>

			<!-- Project Progress Gauge -->
			<div :class="$style.widgetCard">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Project Progress</span>
				</div>
				<div :class="$style.progressBody">
					<!-- Semi-circle Gauge SVG -->
					<div :class="$style.gaugeWrapper">
						<svg viewBox="0 0 160 90" :class="$style.gaugeSvg">
							<!-- Background arc (Dark Green - In Progress) -->
							<path
								d="M 20 80 A 60 60 0 0 1 140 80"
								fill="none"
								stroke="#0e3a2f"
								stroke-width="16"
								stroke-linecap="round"
							/>
							<!-- Completed arc (Teal Green - Completed 41%) -->
							<path
								d="M 20 80 A 60 60 0 0 1 85 22"
								fill="none"
								stroke="#228358"
								stroke-width="16"
								stroke-linecap="round"
							/>
						</svg>
						<div :class="$style.gaugeTextOverlay">
							<span :class="$style.gaugePercentage">41%</span>
							<span :class="$style.gaugeLabel">Project Ended</span>
						</div>
					</div>
					<!-- Legend -->
					<div :class="$style.gaugeLegend">
						<div :class="$style.legendItem">
							<span :class="[$style.legendDot, $style.legendDotGreen]"></span>
							<span :class="$style.legendText">Completed</span>
						</div>
						<div :class="$style.legendItem">
							<span :class="[$style.legendDot, $style.legendDotDark]"></span>
							<span :class="$style.legendText">In Progress</span>
						</div>
					</div>
				</div>
			</div>

			<!-- Server Run Time Dark Green Cyber Card -->
			<div :class="[$style.widgetCard, $style.serverRunTimeCard]">
				<div :class="$style.serverCardHeader">
					<span :class="$style.serverCardTitle">Server Run Time</span>
				</div>

				<!-- Cyber wave background lines -->
				<div :class="$style.waveCanvas">
					<svg viewBox="0 0 300 120" :class="$style.waveSvg" preserveAspectRatio="none">
						<path
							d="M 0 100 C 60 40, 140 110, 300 20"
							fill="none"
							stroke="#256f50"
							stroke-width="3"
							opacity="0.9"
						/>
						<path
							d="M 0 115 C 80 60, 160 120, 300 45"
							fill="none"
							stroke="#1a533b"
							stroke-width="2.5"
							opacity="0.7"
						/>
						<path
							d="M 0 120 C 100 80, 180 130, 300 70"
							fill="none"
							stroke="#133e2c"
							stroke-width="2"
							opacity="0.5"
						/>
					</svg>
				</div>

				<!-- Digital Counter -->
				<div :class="$style.digitalDisplay">
					<span :class="$style.clockNumbers">{{ serverTime }}</span>
				</div>

				<!-- Control Action Buttons (Pause & Stop) -->
				<div :class="$style.serverControls">
					<button
						:class="$style.controlBtnWhite"
						:title="isTimerPaused ? 'Resume' : 'Pause'"
						@click="togglePause"
					>
						<N8nIcon :icon="isTimerPaused ? 'play' : 'pause'" size="small" />
					</button>
					<button :class="$style.controlBtnRed" title="Reset" @click="resetTimer">
						<N8nIcon icon="square" size="small" />
					</button>
				</div>
			</div>
		</div>
	</div>
</template>

<style lang="scss" module>
.dashboardContainer {
	display: flex;
	flex-direction: column;
	width: 100%;
	min-height: 100vh;
	padding: 24px 32px 48px;
	box-sizing: border-box;
	background-color: #e9ecef;
	color: #1e293b;
	font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
	overflow-y: auto;
}

/* Top Bar Header */
.topHeader {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding-bottom: 20px;
	margin-bottom: 12px;
}

.searchWrapper {
	position: relative;
	display: flex;
	align-items: center;
	width: 320px;
	background: #ffffff;
	border: 1px solid #e2e8f0;
	border-radius: 20px;
	padding: 6px 14px;
	box-sizing: border-box;
	box-shadow: 0 1px 2px rgba(0, 0, 0, 0.03);
}

.searchIcon {
	color: #94a3b8;
	margin-right: 8px;
}

.searchInput {
	flex: 1;
	background: transparent;
	border: none;
	outline: none;
	color: #1e293b;
	font-size: 13px;

	&::placeholder {
		color: #94a3b8;
	}
}

.searchShortcut {
	background: #f1f5f9;
	border: 1px solid #e2e8f0;
	border-radius: 4px;
	color: #94a3b8;
	font-size: 11px;
	font-weight: 600;
	padding: 1px 5px;
}

.headerActions {
	display: flex;
	align-items: center;
	gap: 12px;
}

.iconButton {
	display: flex;
	align-items: center;
	justify-content: center;
	width: 36px;
	height: 36px;
	border-radius: 50%;
	background: #ffffff;
	border: 1px solid #e2e8f0;
	color: #64748b;
	cursor: pointer;
	box-shadow: 0 1px 2px rgba(0, 0, 0, 0.03);
	transition:
		background 0.15s ease,
		color 0.15s ease;

	&:hover {
		background: #f8fafc;
		color: #1e293b;
	}
}

.userProfile {
	display: flex;
	align-items: center;
	gap: 10px;
	padding-left: 6px;
	cursor: pointer;
}

.avatar {
	width: 38px;
	height: 38px;
	border-radius: 50%;
	background: #d97706;
	display: flex;
	align-items: center;
	justify-content: center;
	font-weight: 700;
	font-size: 13px;
	color: #ffffff;
	overflow: hidden;
}

.userInfo {
	display: flex;
	flex-direction: column;
}

.userName {
	font-size: 13px;
	font-weight: 700;
	color: #1e293b;
	line-height: 1.2;
}

.userEmail {
	font-size: 11px;
	color: #94a3b8;
}

/* Title Bar */
.titleBar {
	display: flex;
	align-items: center;
	justify-content: space-between;
	margin-bottom: 24px;
}

.pageTitle {
	font-size: 26px;
	font-weight: 800;
	color: #0f172a;
	margin: 0 0 2px 0;
	letter-spacing: -0.02em;
}

.pageSubtitle {
	font-size: 13px;
	color: #64748b;
	margin: 0;
}

.addWorkflowBtn {
	background: #0e3a2f;
	border: none;
	border-radius: 18px;
	font-weight: 600;
	font-size: 13px;
	padding: 8px 18px;
	color: #ffffff;
	display: flex;
	align-items: center;
	gap: 6px;
	cursor: pointer;
	transition: background 0.15s ease;

	&:hover {
		background: #082820;
	}
}

.btnPlus {
	font-size: 15px;
	font-weight: 700;
	line-height: 1;
}

/* KPI Cards Grid */
.kpiGrid {
	display: grid;
	grid-template-columns: repeat(4, 1fr);
	gap: 16px;
	margin-bottom: 20px;
}

.kpiCard {
	background: #ffffff;
	border-radius: 16px;
	padding: 20px;
	display: flex;
	flex-direction: column;
	justify-content: space-between;
	min-height: 135px;
	box-sizing: border-box;
	box-shadow: 0 1px 3px rgba(0, 0, 0, 0.04);
	cursor: pointer;
	transition:
		transform 0.15s ease,
		box-shadow 0.15s ease;

	&:hover {
		transform: translateY(-2px);
		box-shadow: 0 4px 8px rgba(0, 0, 0, 0.06);
	}
}

.kpiCardFeatured {
	background: #0e3a2f;
}

.kpiHeader {
	display: flex;
	align-items: center;
	justify-content: space-between;
	margin-bottom: 8px;
}

.kpiTitleFeatured {
	font-size: 13px;
	font-weight: 600;
	color: #e2e8f0;
}

.kpiIconWrapperFeatured {
	width: 24px;
	height: 24px;
	border-radius: 50%;
	background: rgba(255, 255, 255, 0.15);
	display: flex;
	align-items: center;
	justify-content: center;
	color: #ffffff;
}

.kpiValueFeatured {
	font-size: 34px;
	font-weight: 800;
	color: #ffffff;
	line-height: 1;
	margin-bottom: 12px;
}

.kpiBadgeFeatured {
	display: flex;
	align-items: center;
	gap: 6px;
}

.badgePillFeatured {
	background: #1a533b;
	color: #ffffff;
	font-size: 11px;
	font-weight: 700;
	padding: 2px 6px;
	border-radius: 6px;
}

.badgeTextFeatured {
	font-size: 11px;
	color: #94a3b8;
}

.kpiTitle {
	font-size: 13px;
	font-weight: 600;
	color: #334155;
}

.kpiIconWrapper {
	width: 24px;
	height: 24px;
	border-radius: 50%;
	background: #f1f5f9;
	display: flex;
	align-items: center;
	justify-content: center;
	color: #64748b;
}

.kpiValue {
	font-size: 34px;
	font-weight: 800;
	color: #0f172a;
	line-height: 1;
	margin-bottom: 12px;
}

.kpiBadge {
	display: flex;
	align-items: center;
	gap: 6px;
}

.badgePill {
	background: #f1f5f9;
	color: #475569;
	font-size: 11px;
	font-weight: 700;
	padding: 2px 6px;
	border-radius: 6px;
}

.badgeText {
	font-size: 11px;
	color: #94a3b8;
}

.badgeTextMuted {
	font-size: 12px;
	color: #94a3b8;
	font-weight: 500;
}

/* Middle & Bottom Grids */
.middleGrid,
.bottomGrid {
	display: grid;
	grid-template-columns: 1.15fr 1fr 1.25fr;
	gap: 16px;
	margin-bottom: 20px;
}

.widgetCard {
	background: #ffffff;
	border-radius: 16px;
	padding: 20px;
	box-sizing: border-box;
	display: flex;
	flex-direction: column;
	box-shadow: 0 1px 3px rgba(0, 0, 0, 0.04);
}

.widgetHeader {
	display: flex;
	align-items: center;
	justify-content: space-between;
	margin-bottom: 16px;
}

.widgetTitle {
	font-size: 15px;
	font-weight: 700;
	color: #0f172a;
}

.newBadgeBtn {
	background: transparent;
	color: #64748b;
	border: 1px solid #e2e8f0;
	border-radius: 14px;
	padding: 2px 10px;
	font-size: 11px;
	font-weight: 600;
	cursor: pointer;
	transition: all 0.15s ease;

	&:hover {
		background: #f8fafc;
		color: #0f172a;
	}
}

/* Project Analytics Bar Chart */
.analyticsBody {
	position: relative;
	display: flex;
	align-items: flex-end;
	height: 150px;
	padding-top: 10px;
}

.barChartContainer {
	display: flex;
	align-items: flex-end;
	justify-content: space-between;
	width: 100%;
	height: 100%;
}

.barCol {
	display: flex;
	flex-direction: column;
	align-items: center;
	height: 100%;
	flex: 1;
	gap: 8px;
}

.barTrack {
	flex: 1;
	width: 26px;
	background: transparent;
	display: flex;
	align-items: flex-end;
	overflow: hidden;
}

.barFill {
	width: 100%;
	border-radius: 14px;
	transition: height 0.3s ease;
}

.barFillGreen {
	background: #0e3a2f;
}

.barFillMint {
	background: #34d399;
}

.barFillDarkGreen {
	background: #082820;
}

.barFillFlatLight {
	background: #cbd5e1;
	border-radius: 4px;
}

.barFillStripedLight {
	background: repeating-linear-gradient(45deg, #cbd5e1, #cbd5e1 3px, #ffffff 3px, #ffffff 6px);
	border: 1px solid #cbd5e1;
}

.dayLabel {
	font-size: 11px;
	color: #94a3b8;
	font-weight: 600;
}

.analyticsStatBadge {
	position: absolute;
	top: 6px;
	left: 33%;
	background: #ffffff;
	border: 1px solid #e2e8f0;
	border-radius: 12px;
	padding: 2px 8px;
	box-shadow: 0 2px 4px rgba(0, 0, 0, 0.06);
}

.statPercent {
	font-size: 11px;
	font-weight: 700;
	color: #1e293b;
}

/* Credentials List */
.credentialsList {
	display: flex;
	flex-direction: column;
	gap: 12px;
}

.credentialRow {
	display: flex;
	flex-direction: column;
	padding: 4px 0;
	cursor: pointer;

	&:hover .credName {
		color: #0e3a2f;
	}
}

.credMain {
	display: flex;
	flex-direction: column;
	gap: 2px;
}

.credTopRow {
	display: flex;
	align-items: center;
	gap: 6px;
}

.credName {
	font-size: 14px;
	font-weight: 700;
	color: #0f172a;
}

.credBadge {
	background: #ef4444;
	color: #ffffff;
	font-size: 10px;
	font-weight: 800;
	width: 16px;
	height: 16px;
	border-radius: 50%;
	display: flex;
	align-items: center;
	justify-content: center;
}

.credIconSparkle {
	color: #0f172a;
}

.credIconGeneric {
	color: #2563eb;
}

.credMetaRow {
	display: flex;
	align-items: center;
	gap: 6px;
	font-size: 11px;
	color: #94a3b8;
}

.metaDivider {
	color: #cbd5e1;
}

/* Workflows Widget */
.workflowsList {
	display: flex;
	flex-direction: column;
	gap: 12px;
}

.workflowRow {
	display: flex;
	align-items: center;
	gap: 12px;
	cursor: pointer;

	&:hover .wfName {
		color: #0e3a2f;
	}
}

.wfIconBox {
	width: 34px;
	height: 34px;
	border-radius: 10px;
	display: flex;
	align-items: center;
	justify-content: center;
	flex-shrink: 0;
}

.wfInfo {
	display: flex;
	flex-direction: column;
	min-width: 0;
}

.wfName {
	font-size: 13px;
	font-weight: 700;
	color: #0f172a;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
}

.wfDate {
	font-size: 11px;
	color: #94a3b8;
}

/* Executions Widget */
.executionsList {
	display: flex;
	flex-direction: column;
	gap: 12px;
}

.executionRow {
	display: flex;
	align-items: center;
	justify-content: space-between;
	cursor: pointer;

	&:hover .exName {
		color: #0e3a2f;
	}
}

.exInfo {
	display: flex;
	flex-direction: column;
	min-width: 0;
}

.exName {
	font-size: 13px;
	font-weight: 700;
	color: #0f172a;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
}

.exDate {
	font-size: 11px;
	color: #94a3b8;
}

.exRight {
	display: flex;
	align-items: center;
}

.exStatusPill {
	background: #f0fdf4;
	color: #16a34a;
	border: 1px solid #bbf7d0;
	font-size: 11px;
	font-weight: 600;
	padding: 2px 10px;
	border-radius: 6px;
}

.exStatusPillFailed {
	background: #fef2f2;
	color: #dc2626;
	border-color: #fecaca;
}

/* Project Progress Gauge */
.progressBody {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	height: 100%;
	padding: 6px 0;
}

.gaugeWrapper {
	position: relative;
	width: 170px;
	height: 95px;
	display: flex;
	justify-content: center;
}

.gaugeSvg {
	width: 100%;
	height: 100%;
}

.gaugeTextOverlay {
	position: absolute;
	bottom: 0;
	display: flex;
	flex-direction: column;
	align-items: center;
}

.gaugePercentage {
	font-size: 24px;
	font-weight: 800;
	color: #0f172a;
	line-height: 1;
}

.gaugeLabel {
	font-size: 11px;
	color: #94a3b8;
	margin-top: 2px;
}

.gaugeLegend {
	display: flex;
	align-items: center;
	gap: 16px;
	margin-top: 14px;
}

.legendItem {
	display: flex;
	align-items: center;
	gap: 6px;
}

.legendDot {
	width: 8px;
	height: 8px;
	border-radius: 50%;
}

.legendDotGreen {
	background: #228358;
}

.legendDotDark {
	background: #0e3a2f;
}

.legendText {
	font-size: 11px;
	color: #64748b;
	font-weight: 500;
}

/* Server Run Time Dark Green Cyber Card */
.serverRunTimeCard {
	background: #0e3a2f;
	border-radius: 16px;
	position: relative;
	overflow: hidden;
	justify-content: space-between;
}

.serverCardHeader {
	display: flex;
	align-items: center;
	justify-content: space-between;
	z-index: 2;
}

.serverCardTitle {
	font-size: 13px;
	font-weight: 600;
	color: #e2e8f0;
}

.waveCanvas {
	position: absolute;
	bottom: 0;
	left: 0;
	right: 0;
	height: 110px;
	pointer-events: none;
	z-index: 1;
}

.waveSvg {
	width: 100%;
	height: 100%;
}

.digitalDisplay {
	margin: 16px 0;
	z-index: 2;
	text-align: center;
}

.clockNumbers {
	font-family: 'SF Mono', Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace;
	font-size: 30px;
	font-weight: 400;
	color: #ffffff;
	letter-spacing: 0.06em;
}

.serverControls {
	display: flex;
	align-items: center;
	justify-content: center;
	gap: 16px;
	z-index: 2;
}

.controlBtnWhite {
	width: 34px;
	height: 34px;
	border-radius: 50%;
	background: #ffffff;
	border: none;
	color: #0e3a2f;
	display: flex;
	align-items: center;
	justify-content: center;
	cursor: pointer;
	transition: transform 0.1s ease;

	&:hover {
		transform: scale(1.08);
	}
}

.controlBtnRed {
	width: 34px;
	height: 34px;
	border-radius: 50%;
	background: #ef4444;
	border: none;
	color: #ffffff;
	display: flex;
	align-items: center;
	justify-content: center;
	cursor: pointer;
	transition: transform 0.1s ease;

	&:hover {
		transform: scale(1.08);
	}
}
</style>
