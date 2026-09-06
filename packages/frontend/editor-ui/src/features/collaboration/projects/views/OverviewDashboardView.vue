<script lang="ts" setup>
import { computed, onMounted, onBeforeUnmount, ref } from 'vue';
import { useRouter } from 'vue-router';
import { N8nButton, N8nIcon } from '@n8n/design-system';
import { useWorkflowsListStore } from '@/app/stores/workflowsList.store';
import { useExecutionsStore } from '@/features/execution/executions/executions.store';
import { useCredentialsStore } from '@/features/credentials/credentials.store';
import { useUsersStore } from '@n8n/stores/users.store';
import { useInsightsStore } from '@/features/execution/insights';
import { VIEWS } from '@/app/constants';

const router = useRouter();
const workflowsStore = useWorkflowsListStore();
const executionsStore = useExecutionsStore();
const credentialsStore = useCredentialsStore();
const usersStore = useUsersStore();
const insightsStore = useInsightsStore();

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

// KPIs Metrics
const totalExecutions = computed(() => {
	const count = insightsStore.weeklySummary.data?.total?.value;
	return count !== undefined && count > 0 ? count : 24;
});

const failedExecutions = computed(() => {
	const count = insightsStore.weeklySummary.data?.failed?.value;
	return count !== undefined && count > 0 ? count : 10;
});

const failureRate = computed(() => {
	const rate = insightsStore.weeklySummary.data?.failureRate?.value;
	return rate !== undefined && rate > 0 ? `${rate}%` : '12%';
});

const timeSaved = computed(() => {
	const saved = insightsStore.weeklySummary.data?.timeSaved?.value;
	return saved !== undefined && saved > 0 ? `${saved}h` : '2h';
});

// Real or placeholder workflows
const workflowItems = computed(() => {
	const realList = workflowsStore.allWorkflows;
	if (realList && realList.length > 0) {
		const colors = ['#f59e0b', '#3b82f6', '#10b981', '#8b5cf6', '#ec4899'];
		return realList.slice(0, 5).map((w, index) => ({
			id: w.id,
			name: w.name,
			date: new Date(w.updatedAt || Date.now()).toLocaleDateString('en-US', {
				month: 'short',
				day: 'numeric',
				year: 'numeric',
			}),
			color: colors[index % colors.length],
		}));
	}
	return [
		{ id: '1', name: 'Advanced Manual If-Else 20 Nodes', date: 'Oct 17, 2024', color: '#10b981' },
		{ id: '2', name: 'Onboarding Flow', date: 'Oct 18, 2024', color: '#3b82f6' },
		{ id: '3', name: 'Build Dashboard', date: 'Oct 19, 2024', color: '#f59e0b' },
		{ id: '4', name: 'Optimize Page Load', date: 'Oct 20, 2024', color: '#ec4899' },
		{ id: '5', name: 'Cross-Browser Testing', date: 'Oct 21, 2024', color: '#8b5cf6' },
	];
});

// Real or placeholder credentials
const credentialItems = computed(() => {
	const list = credentialsStore.allCredentials;
	if (list && list.length > 0) {
		return list.slice(0, 3).map((c) => ({
			id: c.id,
			name: c.name,
			time: new Date(c.updatedAt || Date.now()).toLocaleTimeString([], {
				hour: '2-digit',
				minute: '2-digit',
			}),
			type: c.type,
		}));
	}
	return [
		{ id: 'c1', name: 'Groq Account', time: '11:45 AM', type: 'groq', count: 9 },
		{ id: 'c2', name: 'OpenAI Account', time: '10:15 AM', type: 'openai' },
		{ id: 'c3', name: 'Deepseek Account', time: '09:30 AM', type: 'deepseek' },
	];
});

// Real or placeholder executions
const executionItems = computed(() => {
	const list = executionsStore.allExecutions;
	if (list && list.length > 0) {
		return list.slice(0, 2).map((e) => ({
			id: e.id,
			name: e.workflowName || 'Workflow Execution',
			status: e.status === 'success' ? 'Success' : e.status || 'Success',
			time: new Date(e.startedAt || Date.now()).toLocaleDateString('en-US', {
				month: 'short',
				day: 'numeric',
				year: 'numeric',
			}),
		}));
	}
	return [
		{ id: 'e1', name: 'Advanced Manual If-Else 20 Nodes', status: 'Success', time: 'Oct 17, 2024' },
		{ id: 'e2', name: 'Advanced Manual If-Else 20 Nodes', status: 'Success', time: 'Oct 17, 2024' },
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

function openWorkflow(id: string) {
	if (id.length > 5) {
		void router.push({ name: VIEWS.WORKFLOW, params: { name: id } });
	} else {
		void router.push({ name: VIEWS.WORKFLOWS });
	}
}

function openCredentials() {
	void router.push({ name: VIEWS.CREDENTIALS });
}

function openExecutions() {
	void router.push({ name: VIEWS.EXECUTIONS });
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
	<div :class="$style.overviewContainer">
		<!-- Top Bar Header -->
		<header :class="$style.topHeader">
			<div :class="$style.searchWrapper">
				<N8nIcon icon="search" size="medium" :class="$style.searchIcon" />
				<input
					v-model="searchQuery"
					type="text"
					placeholder="Search anything here..."
					:class="$style.searchInput"
				/>
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
					<N8nIcon icon="chevron-down" size="small" :class="$style.chevron" />
				</div>
			</div>
		</header>

		<!-- Main Page Banner / Title -->
		<div :class="$style.titleBar">
			<div>
				<h1 :class="$style.pageTitle">Dashboard</h1>
				<p :class="$style.pageSubtitle">Plan, prioritize, and accomplish your tasks with ease.</p>
			</div>
			<N8nButton type="primary" size="medium" :class="$style.addWorkflowBtn" @click="onAddWorkflow">
				<span :class="$style.btnPlus">+</span> Add Workflow
			</N8nButton>
		</div>

		<!-- 4 KPI Cards Grid -->
		<div :class="$style.kpiGrid">
			<!-- Card 1: Prod. executions (Featured green) -->
			<div :class="[$style.kpiCard, $style.kpiCardFeatured]">
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
			<div :class="$style.kpiCard">
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
			<div :class="$style.kpiCard">
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
			<div :class="$style.kpiCard">
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
					<button :class="$style.moreBtn"><N8nIcon icon="ellipsis" size="medium" /></button>
				</div>
				<div :class="$style.analyticsBody">
					<div :class="$style.barChartContainer">
						<!-- Days bars -->
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillStriped]" style="height: 45%"></div>
							</div>
							<span :class="$style.dayLabel">S</span>
						</div>
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillSolid]" style="height: 70%"></div>
							</div>
							<span :class="$style.dayLabel">M</span>
						</div>
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillSolid]" style="height: 50%"></div>
							</div>
							<span :class="$style.dayLabel">T</span>
						</div>
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillStriped]" style="height: 85%"></div>
							</div>
							<span :class="$style.dayLabel">W</span>
						</div>
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillSolid]" style="height: 60%"></div>
							</div>
							<span :class="$style.dayLabel">T</span>
						</div>
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillSolid]" style="height: 90%"></div>
							</div>
							<span :class="$style.dayLabel">F</span>
						</div>
						<div :class="$style.barCol">
							<div :class="$style.barTrack">
								<div :class="[$style.barFill, $style.barFillStriped]" style="height: 35%"></div>
							</div>
							<span :class="$style.dayLabel">S</span>
						</div>
					</div>
					<div :class="$style.analyticsStatBadge">
						<span :class="$style.statPercent">74%</span>
					</div>
				</div>
			</div>

			<!-- Credentials List -->
			<div :class="$style.widgetCard">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Credentials</span>
					<button :class="$style.moreBtn" @click="openCredentials">
						<N8nIcon icon="ellipsis" size="medium" />
					</button>
				</div>
				<div :class="$style.credentialsList">
					<div
						v-for="item in credentialItems"
						:key="item.id"
						:class="$style.credentialRow"
						@click="openCredentials"
					>
						<div :class="$style.credLeft">
							<div :class="$style.credIconBox">
								<N8nIcon icon="key" size="small" />
							</div>
							<div :class="$style.credNameGroup">
								<span :class="$style.credName">{{ item.name }}</span>
								<span v-if="item.count" :class="$style.credCountBadge">{{ item.count }}</span>
							</div>
						</div>
						<span :class="$style.credTime">{{ item.time }}</span>
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
						<div :class="$style.wfLeft">
							<div :class="$style.wfDot" :style="{ backgroundColor: wf.color }"></div>
							<span :class="$style.wfName">{{ wf.name }}</span>
						</div>
						<span :class="$style.wfDate">{{ wf.date }}</span>
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
					<button :class="$style.moreBtn" @click="openExecutions">
						<N8nIcon icon="ellipsis" size="medium" />
					</button>
				</div>
				<div :class="$style.executionsList">
					<div
						v-for="ex in executionItems"
						:key="ex.id"
						:class="$style.executionRow"
						@click="openExecutions"
					>
						<div :class="$style.exLeft">
							<div :class="$style.exIconBox">
								<N8nIcon icon="play" size="small" />
							</div>
							<span :class="$style.exName">{{ ex.name }}</span>
						</div>
						<div :class="$style.exRight">
							<span :class="$style.exStatusPill">{{ ex.status }}</span>
							<span :class="$style.exDate">{{ ex.time }}</span>
						</div>
					</div>
				</div>
			</div>

			<!-- Project Progress Gauge -->
			<div :class="$style.widgetCard">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Project Progress</span>
					<button :class="$style.moreBtn"><N8nIcon icon="ellipsis" size="medium" /></button>
				</div>
				<div :class="$style.progressBody">
					<!-- Semi-circle Gauge SVG -->
					<div :class="$style.gaugeWrapper">
						<svg viewBox="0 0 160 90" :class="$style.gaugeSvg">
							<!-- Background arc -->
							<path
								d="M 20 80 A 60 60 0 0 1 140 80"
								fill="none"
								stroke="#1e293b"
								stroke-width="14"
								stroke-linecap="round"
							/>
							<!-- Filled progress arc (41%) -->
							<path
								d="M 20 80 A 60 60 0 0 1 85 22"
								fill="none"
								stroke="#10b981"
								stroke-width="14"
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

			<!-- Server Run Time Dark Digital Card -->
			<div :class="[$style.widgetCard, $style.serverRunTimeCard]">
				<div :class="$style.serverCardHeader">
					<span :class="$style.serverCardTitle">Server Run Time</span>
					<button :class="$style.moreBtnInverted"><N8nIcon icon="ellipsis" size="medium" /></button>
				</div>

				<!-- Cyber wave background lines -->
				<div :class="$style.waveCanvas">
					<svg viewBox="0 0 300 80" :class="$style.waveSvg" preserveAspectRatio="none">
						<path
							d="M 0 45 C 50 15, 100 70, 150 40 C 200 10, 250 65, 300 35 L 300 80 L 0 80 Z"
							fill="url(#waveGradient)"
							opacity="0.25"
						/>
						<path
							d="M 0 45 C 50 15, 100 70, 150 40 C 200 10, 250 65, 300 35"
							fill="none"
							stroke="#10b981"
							stroke-width="2"
							opacity="0.8"
						/>
						<defs>
							<linearGradient id="waveGradient" x1="0" y1="0" x2="0" y2="1">
								<stop offset="0%" stop-color="#10b981" />
								<stop offset="100%" stop-color="#10b981" stop-opacity="0" />
							</linearGradient>
						</defs>
					</svg>
				</div>

				<!-- Digital Counter -->
				<div :class="$style.digitalDisplay">
					<span :class="$style.clockNumbers">{{ serverTime }}</span>
				</div>

				<!-- Control Action Buttons (Pause & Stop) -->
				<div :class="$style.serverControls">
					<button
						:class="$style.controlBtn"
						:title="isTimerPaused ? 'Resume' : 'Pause'"
						@click="togglePause"
					>
						<N8nIcon :icon="isTimerPaused ? 'play' : 'pause'" size="small" />
					</button>
					<button :class="$style.controlBtn" title="Reset" @click="resetTimer">
						<N8nIcon icon="square" size="small" />
					</button>
				</div>
			</div>
		</div>
	</div>
</template>

<style lang="scss" module>
.overviewContainer {
	display: flex;
	flex-direction: column;
	width: 100%;
	min-height: 100vh;
	padding: 24px 32px 48px;
	box-sizing: border-box;
	background-color: #0b0f17;
	color: #f8fafc;
	font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
	overflow-y: auto;
}

/* Top Bar Header */
.topHeader {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding-bottom: 24px;
	border-bottom: 1px solid rgba(255, 255, 255, 0.05);
	margin-bottom: 24px;
}

.searchWrapper {
	position: relative;
	display: flex;
	align-items: center;
	width: 380px;
	background: #111827;
	border: 1px solid rgba(255, 255, 255, 0.08);
	border-radius: 24px;
	padding: 8px 16px;
	box-sizing: border-box;
}

.searchIcon {
	color: #64748b;
	margin-right: 10px;
}

.searchInput {
	flex: 1;
	background: transparent;
	border: none;
	outline: none;
	color: #f8fafc;
	font-size: 13px;

	&::placeholder {
		color: #64748b;
	}
}

.searchShortcut {
	background: rgba(255, 255, 255, 0.06);
	border: 1px solid rgba(255, 255, 255, 0.1);
	border-radius: 6px;
	color: #94a3b8;
	font-size: 11px;
	font-weight: 600;
	padding: 2px 6px;
}

.headerActions {
	display: flex;
	align-items: center;
	gap: 16px;
}

.iconButton {
	display: flex;
	align-items: center;
	justify-content: center;
	width: 38px;
	height: 38px;
	border-radius: 50%;
	background: #111827;
	border: 1px solid rgba(255, 255, 255, 0.08);
	color: #94a3b8;
	cursor: pointer;
	transition:
		background 0.15s ease,
		color 0.15s ease;

	&:hover {
		background: #1f2937;
		color: #fff;
	}
}

.userProfile {
	display: flex;
	align-items: center;
	gap: 10px;
	padding-left: 8px;
	cursor: pointer;
}

.avatar {
	width: 38px;
	height: 38px;
	border-radius: 50%;
	background: linear-gradient(135deg, #10b981, #047857);
	display: flex;
	align-items: center;
	justify-content: center;
	font-weight: 700;
	font-size: 13px;
	color: #fff;
}

.userInfo {
	display: flex;
	flex-direction: column;
}

.userName {
	font-size: 13px;
	font-weight: 600;
	color: #f8fafc;
	line-height: 1.2;
}

.userEmail {
	font-size: 11px;
	color: #64748b;
}

.chevron {
	color: #64748b;
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
	font-weight: 700;
	color: #fff;
	margin: 0 0 4px;
	letter-spacing: -0.02em;
}

.pageSubtitle {
	font-size: 13px;
	color: #94a3b8;
	margin: 0;
}

.addWorkflowBtn {
	background: #10b981 !important;
	border: none !important;
	border-radius: 8px !important;
	font-weight: 600 !important;
	padding: 8px 18px !important;
	color: #0b0f17 !important;
	display: flex;
	align-items: center;
	gap: 6px;

	&:hover {
		background: #059669 !important;
	}
}

.btnPlus {
	font-size: 16px;
	font-weight: 700;
	line-height: 1;
}

/* KPI Cards Grid */
.kpiGrid {
	display: grid;
	grid-template-columns: repeat(4, 1fr);
	gap: 16px;
	margin-bottom: 24px;
}

.kpiCard {
	background: #111827;
	border: 1px solid rgba(255, 255, 255, 0.05);
	border-radius: 14px;
	padding: 20px;
	display: flex;
	flex-direction: column;
	justify-content: space-between;
	min-height: 130px;
	box-sizing: border-box;
}

.kpiCardFeatured {
	background: #064e3b;
	border-color: #059669;
}

.kpiHeader {
	display: flex;
	align-items: center;
	justify-content: space-between;
	margin-bottom: 12px;
}

.kpiTitleFeatured {
	font-size: 13px;
	font-weight: 600;
	color: #a7f3d0;
}

.kpiIconWrapperFeatured {
	width: 28px;
	height: 28px;
	border-radius: 50%;
	background: rgba(255, 255, 255, 0.15);
	display: flex;
	align-items: center;
	justify-content: center;
	color: #fff;
}

.kpiValueFeatured {
	font-size: 32px;
	font-weight: 700;
	color: #fff;
	line-height: 1;
	margin-bottom: 12px;
}

.kpiBadgeFeatured {
	display: flex;
	align-items: center;
	gap: 8px;
}

.badgePillFeatured {
	background: #047857;
	color: #fff;
	font-size: 11px;
	font-weight: 700;
	padding: 2px 8px;
	border-radius: 12px;
}

.badgeTextFeatured {
	font-size: 11px;
	color: #a7f3d0;
}

.kpiTitle {
	font-size: 13px;
	font-weight: 600;
	color: #94a3b8;
}

.kpiIconWrapper {
	width: 28px;
	height: 28px;
	border-radius: 50%;
	background: rgba(255, 255, 255, 0.05);
	display: flex;
	align-items: center;
	justify-content: center;
	color: #94a3b8;
}

.kpiValue {
	font-size: 32px;
	font-weight: 700;
	color: #fff;
	line-height: 1;
	margin-bottom: 12px;
}

.kpiBadge {
	display: flex;
	align-items: center;
	gap: 8px;
}

.badgePill {
	background: rgba(16, 185, 129, 0.15);
	color: #10b981;
	font-size: 11px;
	font-weight: 700;
	padding: 2px 8px;
	border-radius: 12px;
}

.badgeText {
	font-size: 11px;
	color: #64748b;
}

.badgeTextMuted {
	font-size: 12px;
	color: #64748b;
	font-weight: 500;
}

/* Middle & Bottom Grids */
.middleGrid,
.bottomGrid {
	display: grid;
	grid-template-columns: 1.1fr 1fr 1.3fr;
	gap: 16px;
	margin-bottom: 24px;
}

.widgetCard {
	background: #111827;
	border: 1px solid rgba(255, 255, 255, 0.05);
	border-radius: 14px;
	padding: 20px;
	box-sizing: border-box;
	display: flex;
	flex-direction: column;
}

.widgetHeader {
	display: flex;
	align-items: center;
	justify-content: space-between;
	margin-bottom: 16px;
}

.widgetTitle {
	font-size: 15px;
	font-weight: 600;
	color: #fff;
}

.moreBtn {
	background: none;
	border: none;
	color: #64748b;
	cursor: pointer;
	padding: 4px;

	&:hover {
		color: #fff;
	}
}

.newBadgeBtn {
	background: rgba(16, 185, 129, 0.15);
	color: #10b981;
	border: 1px solid rgba(16, 185, 129, 0.3);
	border-radius: 16px;
	padding: 3px 10px;
	font-size: 12px;
	font-weight: 600;
	cursor: pointer;

	&:hover {
		background: rgba(16, 185, 129, 0.25);
	}
}

/* Project Analytics Bar Chart */
.analyticsBody {
	position: relative;
	display: flex;
	align-items: flex-end;
	height: 160px;
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
	width: 18px;
	background: rgba(255, 255, 255, 0.03);
	border-radius: 6px;
	display: flex;
	align-items: flex-end;
	overflow: hidden;
}

.barFill {
	width: 100%;
	border-radius: 6px;
	transition: height 0.3s ease;
}

.barFillSolid {
	background: #10b981;
}

.barFillStriped {
	background: repeating-linear-gradient(45deg, #047857, #047857 4px, #10b981 4px, #10b981 8px);
}

.dayLabel {
	font-size: 11px;
	color: #64748b;
	font-weight: 600;
}

.analyticsStatBadge {
	position: absolute;
	top: 10px;
	right: 10px;
	background: #047857;
	border-radius: 20px;
	padding: 4px 10px;
}

.statPercent {
	font-size: 12px;
	font-weight: 700;
	color: #fff;
}

/* Credentials List */
.credentialsList {
	display: flex;
	flex-direction: column;
	gap: 12px;
}

.credentialRow {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding: 10px 12px;
	background: rgba(255, 255, 255, 0.02);
	border-radius: 10px;
	cursor: pointer;
	transition: background 0.15s ease;

	&:hover {
		background: rgba(255, 255, 255, 0.06);
	}
}

.credLeft {
	display: flex;
	align-items: center;
	gap: 10px;
}

.credIconBox {
	width: 32px;
	height: 32px;
	border-radius: 8px;
	background: rgba(255, 255, 255, 0.05);
	display: flex;
	align-items: center;
	justify-content: center;
	color: #10b981;
}

.credNameGroup {
	display: flex;
	align-items: center;
	gap: 8px;
}

.credName {
	font-size: 13px;
	font-weight: 500;
	color: #f8fafc;
}

.credCountBadge {
	background: #1f2937;
	color: #94a3b8;
	font-size: 11px;
	font-weight: 600;
	padding: 1px 6px;
	border-radius: 10px;
}

.credTime {
	font-size: 11px;
	color: #64748b;
}

/* Workflows Widget */
.workflowsList {
	display: flex;
	flex-direction: column;
	gap: 8px;
}

.workflowRow {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding: 8px 10px;
	border-radius: 8px;
	cursor: pointer;
	transition: background 0.15s ease;

	&:hover {
		background: rgba(255, 255, 255, 0.04);
	}
}

.wfLeft {
	display: flex;
	align-items: center;
	gap: 10px;
}

.wfDot {
	width: 10px;
	height: 10px;
	border-radius: 3px;
	flex-shrink: 0;
}

.wfName {
	font-size: 13px;
	font-weight: 500;
	color: #f8fafc;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
	max-width: 220px;
}

.wfDate {
	font-size: 11px;
	color: #64748b;
}

/* Executions Widget */
.executionsList {
	display: flex;
	flex-direction: column;
	gap: 10px;
}

.executionRow {
	display: flex;
	align-items: center;
	justify-content: space-between;
	padding: 10px 12px;
	background: rgba(255, 255, 255, 0.02);
	border-radius: 10px;
	cursor: pointer;
	transition: background 0.15s ease;

	&:hover {
		background: rgba(255, 255, 255, 0.05);
	}
}

.exLeft {
	display: flex;
	align-items: center;
	gap: 10px;
}

.exIconBox {
	width: 28px;
	height: 28px;
	border-radius: 6px;
	background: rgba(16, 185, 129, 0.1);
	display: flex;
	align-items: center;
	justify-content: center;
	color: #10b981;
}

.exName {
	font-size: 13px;
	font-weight: 500;
	color: #f8fafc;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
	max-width: 180px;
}

.exRight {
	display: flex;
	align-items: center;
	gap: 12px;
}

.exStatusPill {
	background: rgba(16, 185, 129, 0.15);
	color: #10b981;
	font-size: 11px;
	font-weight: 600;
	padding: 2px 8px;
	border-radius: 10px;
}

.exDate {
	font-size: 11px;
	color: #64748b;
}

/* Project Progress Gauge */
.progressBody {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	height: 100%;
	padding: 10px 0;
}

.gaugeWrapper {
	position: relative;
	width: 180px;
	height: 100px;
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
	font-weight: 700;
	color: #fff;
	line-height: 1;
}

.gaugeLabel {
	font-size: 11px;
	color: #64748b;
	margin-top: 2px;
}

.gaugeLegend {
	display: flex;
	align-items: center;
	gap: 20px;
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
	background: #10b981;
}

.legendDotDark {
	background: #334155;
}

.legendText {
	font-size: 11px;
	color: #94a3b8;
}

/* Server Run Time Card */
.serverRunTimeCard {
	background: #090d16;
	border-color: rgba(16, 185, 129, 0.2);
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
	font-size: 15px;
	font-weight: 600;
	color: #fff;
}

.moreBtnInverted {
	background: none;
	border: none;
	color: #94a3b8;
	cursor: pointer;
	padding: 4px;

	&:hover {
		color: #fff;
	}
}

.waveCanvas {
	position: absolute;
	bottom: 0;
	left: 0;
	right: 0;
	height: 100px;
	pointer-events: none;
	z-index: 1;
}

.waveSvg {
	width: 100%;
	height: 100%;
}

.digitalDisplay {
	margin: 20px 0;
	z-index: 2;
	text-align: center;
}

.clockNumbers {
	font-family: 'SF Mono', Monaco, Consolas, 'Liberation Mono', 'Courier New', monospace;
	font-size: 28px;
	font-weight: 700;
	color: #fff;
	letter-spacing: 0.08em;
	text-shadow: 0 0 12px rgba(16, 185, 129, 0.4);
}

.serverControls {
	display: flex;
	align-items: center;
	justify-content: center;
	gap: 16px;
	z-index: 2;
}

.controlBtn {
	width: 36px;
	height: 36px;
	border-radius: 50%;
	background: rgba(255, 255, 255, 0.06);
	border: 1px solid rgba(255, 255, 255, 0.1);
	color: #f8fafc;
	display: flex;
	align-items: center;
	justify-content: center;
	cursor: pointer;
	transition:
		background 0.15s ease,
		transform 0.1s ease;

	&:hover {
		background: rgba(16, 185, 129, 0.2);
		border-color: #10b981;
		color: #10b981;
		transform: scale(1.05);
	}
}
</style>
