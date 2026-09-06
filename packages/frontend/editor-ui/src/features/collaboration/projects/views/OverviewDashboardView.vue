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
import CredentialIcon from '@/features/credentials/components/CredentialIcon.vue';

const router = useRouter();
const workflowsStore = useWorkflowsListStore();
const executionsStore = useExecutionsStore();
const credentialsStore = useCredentialsStore();
const usersStore = useUsersStore();
const insightsStore = useInsightsStore();
const uiStore = useUIStore();

// Search state
const searchQuery = ref('');

// Dialog Modals State
const activeModal = ref<'credentials' | 'workflows' | 'executions' | null>(null);
const modalSearchQuery = ref('');

function openModalView(type: 'credentials' | 'workflows' | 'executions') {
	modalSearchQuery.value = '';
	activeModal.value = type;
}

function closeModalView() {
	activeModal.value = null;
	modalSearchQuery.value = '';
}

// Current user display
const userName = computed(() => {
	const user = usersStore.currentUser;
	if (user?.firstName || user?.lastName) {
		return `${user.firstName ?? ''} ${user.lastName ?? ''}`.trim();
	}
	return user?.email?.split('@')[0] || 'User';
});

const userEmail = computed(() => {
	return usersStore.currentUser?.email ?? '';
});

const userInitials = computed(() => {
	const name = userName.value;
	const parts = name.split(' ').filter(Boolean);
	if (parts.length >= 2) {
		return (parts[0][0] + parts[1][0]).toUpperCase();
	}
	return name.slice(0, 2).toUpperCase() || 'U';
});

// Helpers
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
	if (!dateInput) return '';
	const date = new Date(dateInput);
	return date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
}

function formatFullDateTime(dateInput?: string | Date | number): string {
	if (!dateInput) return '';
	const date = new Date(dateInput);
	const month = date.toLocaleDateString('en-US', { month: 'short' });
	const day = date.getDate();
	const hours = date.getHours().toString().padStart(2, '0');
	const mins = date.getMinutes().toString().padStart(2, '0');
	const secs = date.getSeconds().toString().padStart(2, '0');
	return `${month} ${day}, ${hours}:${mins}:${secs}`;
}

// Real KPI Metrics
const totalExecutions = computed(() => {
	if (insightsStore.weeklySummary.data?.total?.value !== undefined) {
		return insightsStore.weeklySummary.data.total.value;
	}
	return executionsStore.allExecutions.length;
});

const failedExecutions = computed(() => {
	if (insightsStore.weeklySummary.data?.failed?.value !== undefined) {
		return insightsStore.weeklySummary.data.failed.value;
	}
	return executionsStore.allExecutions.filter((e) => e.status === 'error' || e.status === 'crashed')
		.length;
});

const failureRate = computed(() => {
	if (insightsStore.weeklySummary.data?.failureRate?.value !== undefined) {
		return `${insightsStore.weeklySummary.data.failureRate.value}`;
	}
	if (totalExecutions.value > 0) {
		return `${Math.round((failedExecutions.value / totalExecutions.value) * 100)}`;
	}
	return '0';
});

const timeSaved = computed(() => {
	if (insightsStore.weeklySummary.data?.timeSaved?.value !== undefined) {
		return `${insightsStore.weeklySummary.data.timeSaved.value}`;
	}
	// Estimate 1.5 minutes (0.025h) saved per successful execution
	const successfulCount = totalExecutions.value - failedExecutions.value;
	if (successfulCount > 0) {
		const hours = Math.round((successfulCount * 2) / 60);
		return `${hours > 0 ? hours : (successfulCount * 0.05).toFixed(1)}`;
	}
	return '0';
});

// Workflows list (100% Real Data)
const workflowIcons = [
	{ icon: 'code', bg: '#eff6ff', color: '#3b82f6' },
	{ icon: 'clock', bg: '#ecfdf5', color: '#10b981' },
	{ icon: 'grid-2x2', bg: '#fffbeb', color: '#f59e0b' },
	{ icon: 'zap', bg: '#fff1f2', color: '#f43f5e' },
	{ icon: 'globe', bg: '#f5f3ff', color: '#8b5cf6' },
];

const allRealWorkflows = computed(() => {
	const realList = workflowsStore.allWorkflows;
	if (!realList || realList.length === 0) return [];
	return realList.map((w, index) => {
		const iconCfg = workflowIcons[index % workflowIcons.length];
		return {
			id: w.id,
			name: w.name,
			date: `Last Update : ${formatCreatedDate(w.updatedAt || Date.now())}`,
			active: w.active,
			icon: iconCfg.icon,
			iconBg: iconCfg.bg,
			iconColor: iconCfg.color,
		};
	});
});

const workflowItems = computed(() => {
	return allRealWorkflows.value.slice(0, 5);
});

const modalFilteredWorkflows = computed(() => {
	const query = modalSearchQuery.value.trim().toLowerCase();
	if (!query) return allRealWorkflows.value;
	return allRealWorkflows.value.filter((w) => w.name.toLowerCase().includes(query));
});

// Credentials list (100% Real Data)
const allRealCredentials = computed(() => {
	const list = credentialsStore.allCredentials;
	if (!list || list.length === 0) return [];
	return list.map((c) => ({
		id: c.id,
		name: c.name,
		type: c.type,
		timeAgo: formatRelativeTime(c.updatedAt),
		createdDate: formatCreatedDate(c.createdAt),
	}));
});

const credentialItems = computed(() => {
	return allRealCredentials.value.slice(0, 3);
});

const modalFilteredCredentials = computed(() => {
	const query = modalSearchQuery.value.trim().toLowerCase();
	if (!query) return allRealCredentials.value;
	return allRealCredentials.value.filter(
		(c) => c.name.toLowerCase().includes(query) || c.type.toLowerCase().includes(query),
	);
});

// Executions list (100% Real Data)
const allRealExecutions = computed(() => {
	const list = executionsStore.allExecutions;
	if (!list || list.length === 0) return [];
	return list.map((e) => ({
		id: e.id,
		workflowId: e.workflowId,
		name: e.workflowName || 'Workflow Execution',
		status: e.status === 'error' || e.status === 'crashed' ? 'Failed' : 'Success',
		time: formatFullDateTime(e.startedAt || e.createdAt),
	}));
});

const executionItems = computed(() => {
	return allRealExecutions.value.slice(0, 5);
});

const modalFilteredExecutions = computed(() => {
	const query = modalSearchQuery.value.trim().toLowerCase();
	if (!query) return allRealExecutions.value;
	return allRealExecutions.value.filter(
		(e) => e.name.toLowerCase().includes(query) || e.status.toLowerCase().includes(query),
	);
});

// Real Analytics per Day of Week (Sun=0 to Sat=6)
const dayActivityStats = computed(() => {
	const counts = [
		{ day: 'S', count: 0, label: 'Sunday', updated: 0, created: 0 },
		{ day: 'M', count: 0, label: 'Monday', updated: 0, created: 0 },
		{ day: 'T', count: 0, label: 'Tuesday', updated: 0, created: 0 },
		{ day: 'W', count: 0, label: 'Wednesday', updated: 0, created: 0 },
		{ day: 'T', count: 0, label: 'Thursday', updated: 0, created: 0 },
		{ day: 'F', count: 0, label: 'Friday', updated: 0, created: 0 },
		{ day: 'S', count: 0, label: 'Saturday', updated: 0, created: 0 },
	];

	// Count real executions & workflow events by day of week
	executionsStore.allExecutions.forEach((e) => {
		if (e.createdAt || e.startedAt) {
			const d = new Date(e.createdAt || e.startedAt).getDay();
			counts[d].count += 1;
		}
	});

	workflowsStore.allWorkflows.forEach((w) => {
		if (w.updatedAt) {
			const d = new Date(w.updatedAt).getDay();
			counts[d].count += 1;
			counts[d].updated += 1;
		}
		if (w.createdAt) {
			const d = new Date(w.createdAt).getDay();
			counts[d].created += 1;
		}
	});

	const maxCount = Math.max(...counts.map((c) => c.count), 1);
	return counts.map((item, idx) => {
		const pct = item.count > 0 ? Math.max(Math.round((item.count / maxCount) * 90), 15) : 8;
		let styleClass = 'barFillGreen';
		if (item.count === 0) {
			styleClass = 'barFillFlatLight';
		} else if (item.updated > 0) {
			styleClass = 'barFillStripedLight';
		} else if (idx === 3 || idx === 1) {
			styleClass = 'barFillDarkGreen';
		} else {
			styleClass = 'barFillMint';
		}
		return {
			...item,
			height: `${pct}%`,
			styleClass,
		};
	});
});

const overallActivityRate = computed(() => {
	if (totalExecutions.value === 0 && workflowsStore.allWorkflows.length === 0) {
		return '0%';
	}
	const successRate = 100 - Number(failureRate.value || 0);
	return `${Math.max(successRate, 10)}%`;
});

// Real Project Progress (Active vs Inactive workflows)
const projectProgressData = computed(() => {
	const total = workflowsStore.allWorkflows.length;
	if (total === 0) {
		return {
			percentage: 0,
			completedCount: 0,
			inProgressCount: 0,
			arcPath: 'M 20 80 A 60 60 0 0 1 20 80',
		};
	}
	const activeCount = workflowsStore.allWorkflows.filter((w) => w.active).length;
	const inactiveCount = total - activeCount;
	const pct = Math.round((activeCount / total) * 100);

	// Angle for semi-circle arc: from PI (180deg - left) to (PI - angle)
	const angle = (pct / 100) * Math.PI;
	const cx = 80;
	const cy = 80;
	const r = 60;
	const x = cx - r * Math.cos(angle);
	const y = cy - r * Math.sin(angle);
	const arcPath =
		pct > 0
			? `M 20 80 A 60 60 0 0 1 ${x.toFixed(1)} ${y.toFixed(1)}`
			: 'M 20 80 A 60 60 0 0 1 20 80';

	return {
		percentage: pct,
		completedCount: activeCount,
		inProgressCount: inactiveCount,
		arcPath,
	};
});

// Server Runtime Clock: Real server uptime timer from backend
const serverStartTime = ref<number>(Date.now());
const serverTime = ref('00:00:00:00');
const isTimerPaused = ref(false);
let timerInterval: any = null;

async function fetchServerUptime() {
	try {
		const res = await fetch('/rest/debug/multi-main-setup');
		if (res.ok) {
			const data = await res.json();
			if (data.startTime) {
				serverStartTime.value = data.startTime;
			} else if (data.uptime) {
				serverStartTime.value = Date.now() - Math.floor(data.uptime * 1000);
			}
		}
	} catch {
		// Fallback to client session start time
		serverStartTime.value =
			Date.now() - (window.performance && performance.now ? Math.floor(performance.now()) : 0);
	}
}

function updateClock() {
	if (isTimerPaused.value) return;
	const diffSec = Math.max(0, Math.floor((Date.now() - serverStartTime.value) / 1000));
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
	serverStartTime.value = Date.now();
	serverTime.value = '00:00:00:00';
}

function onAddWorkflow() {
	closeModalView();
	void router.push({ name: VIEWS.NEW_WORKFLOW });
}

function onNewCredential() {
	closeModalView();
	uiStore.openModal(CREDENTIAL_SELECT_MODAL_KEY);
}

function openWorkflow(id: string) {
	closeModalView();
	if (id) {
		void router.push({ name: VIEWS.WORKFLOW, params: { name: id } });
	}
}

function openCredentialItem(id: string) {
	closeModalView();
	if (id) {
		uiStore.openExistingCredential(id);
	}
}

function openExecutionItem(item: any) {
	closeModalView();
	if (item.workflowId && item.id) {
		void router.push({
			name: VIEWS.EXECUTION_PREVIEW,
			params: { name: item.workflowId, executionId: item.id },
		});
	}
}

function navigateToInsights() {
	void router.push({ name: VIEWS.INSIGHTS });
}

function navigateToTab(routeName: string) {
	void router.push({ name: routeName });
}

onMounted(async () => {
	await fetchServerUptime();
	updateClock();
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
				<input
					v-model="searchQuery"
					type="text"
					placeholder="Search..."
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
						<span v-if="userEmail" :class="$style.userEmail">{{ userEmail }}</span>
					</div>
				</div>
			</div>
		</header>

		<!-- Main Page Banner / Title & Navigation Tabs -->
		<div :class="$style.titleBar">
			<div>
				<h1 :class="$style.pageTitle">Dashboard</h1>
				<p :class="$style.pageSubtitle">Plan, prioritize, and accomplish your tasks with ease.</p>
			</div>
			<div :class="$style.titleActions">
				<button :class="$style.addWorkflowBtn" @click="onAddWorkflow">
					<span :class="$style.btnPlus">+</span> Add Workflow
				</button>
			</div>
		</div>

		<!-- Quick Navigation Hub Bar (Workflows, Credentials, Executions, Variables, Data tables) -->
		<div :class="$style.quickNavHub">
			<button :class="[$style.navTabBtn, $style.navTabActive]">
				<N8nIcon icon="grid-2x2" size="small" :class="$style.navTabIcon" />
				<span>Overview</span>
			</button>
			<button :class="$style.navTabBtn" @click="navigateToTab(VIEWS.WORKFLOWS)">
				<N8nIcon icon="project-diagram" size="small" :class="$style.navTabIcon" />
				<span>Workflows</span>
				<span v-if="allRealWorkflows.length > 0" :class="$style.navTabCount">{{
					allRealWorkflows.length
				}}</span>
			</button>
			<button :class="$style.navTabBtn" @click="navigateToTab(VIEWS.CREDENTIALS)">
				<N8nIcon icon="key" size="small" :class="$style.navTabIcon" />
				<span>Credentials</span>
				<span v-if="allRealCredentials.length > 0" :class="$style.navTabCount">{{
					allRealCredentials.length
				}}</span>
			</button>
			<button :class="$style.navTabBtn" @click="navigateToTab(VIEWS.EXECUTIONS)">
				<N8nIcon icon="history" size="small" :class="$style.navTabIcon" />
				<span>Executions</span>
				<span v-if="allRealExecutions.length > 0" :class="$style.navTabCount">{{
					allRealExecutions.length
				}}</span>
			</button>
			<button :class="$style.navTabBtn" @click="navigateToTab(VIEWS.HOME_VARIABLES)">
				<N8nIcon icon="variable" size="small" :class="$style.navTabIcon" />
				<span>Variables</span>
			</button>
			<button :class="$style.navTabBtn" @click="navigateToTab('data-tables')">
				<N8nIcon icon="table" size="small" :class="$style.navTabIcon" />
				<span>Data tables</span>
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
					<span :class="$style.badgePillFeatured">{{ totalExecutions }} Total</span>
					<span :class="$style.badgeTextFeatured">All recorded executions</span>
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
					<span :class="$style.badgePill">{{ failedExecutions }} Error</span>
					<span :class="$style.badgeText">Failed or crashed runs</span>
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
				<div :class="$style.kpiValue">{{ failureRate }}%</div>
				<div :class="$style.kpiBadge">
					<span :class="$style.badgePill">{{ 100 - Number(failureRate) }}% Success</span>
					<span :class="$style.badgeText">Execution reliability</span>
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
				<div :class="$style.kpiValue">{{ timeSaved }}h</div>
				<div :class="$style.kpiBadge">
					<span :class="$style.badgeTextMuted">Automated runtime saved</span>
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
						<div v-for="(dayStat, dIndex) in dayActivityStats" :key="dIndex" :class="$style.barCol">
							<div :class="$style.barTrack">
								<div
									:class="[$style.barFill, $style[dayStat.styleClass]]"
									:style="{ height: dayStat.height }"
									:title="`${dayStat.label}: ${dayStat.count} events`"
								></div>
							</div>
							<span :class="$style.dayLabel">{{ dayStat.day }}</span>
						</div>
					</div>
					<!-- Tooltip Pill -->
					<div :class="$style.analyticsStatBadge">
						<span :class="$style.statPercent">{{ overallActivityRate }}</span>
					</div>
				</div>
			</div>

			<!-- Credentials List -->
			<div :class="$style.widgetCard">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Credentials ({{ allRealCredentials.length }})</span>
					<div :class="$style.headerActionsSmall">
						<button
							v-if="allRealCredentials.length > 0"
							:class="$style.viewAllBtn"
							@click="openModalView('credentials')"
						>
							View All
						</button>
						<button :class="$style.newBadgeBtn" @click="onNewCredential">+ New</button>
					</div>
				</div>
				<div :class="$style.credentialsList">
					<div
						v-for="item in credentialItems"
						:key="item.id"
						:class="$style.credentialRow"
						@click="openCredentialItem(item.id)"
					>
						<div :class="$style.credIconContainer">
							<CredentialIcon :credential-type-name="item.type" :size="24" />
						</div>
						<div :class="$style.credMain">
							<div :class="$style.credTopRow">
								<span :class="$style.credName">{{ item.name }}</span>
							</div>
							<div :class="$style.credMetaRow">
								<span>Last Update : {{ item.timeAgo }}</span>
								<span :class="$style.metaDivider">|</span>
								<span>Created {{ item.createdDate }}</span>
							</div>
						</div>
					</div>
					<!-- Clean Real Empty State -->
					<div v-if="allRealCredentials.length === 0" :class="$style.cardEmptyState">
						<N8nIcon icon="key" size="medium" :class="$style.emptyStateIcon" />
						<span :class="$style.emptyStateText">No credentials created yet</span>
						<button :class="$style.emptyStateActionBtn" @click="onNewCredential">
							Create Credential
						</button>
					</div>
				</div>
			</div>

			<!-- Workflows List -->
			<div :class="[$style.widgetCard, $style.workflowsWidget]">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Workflows ({{ allRealWorkflows.length }})</span>
					<div :class="$style.headerActionsSmall">
						<button
							v-if="allRealWorkflows.length > 0"
							:class="$style.viewAllBtn"
							@click="openModalView('workflows')"
						>
							View All
						</button>
						<button :class="$style.newBadgeBtn" @click="onAddWorkflow">+ New</button>
					</div>
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
					<!-- Clean Real Empty State -->
					<div v-if="allRealWorkflows.length === 0" :class="$style.cardEmptyState">
						<N8nIcon icon="project-diagram" size="medium" :class="$style.emptyStateIcon" />
						<span :class="$style.emptyStateText">No workflows created yet</span>
						<button :class="$style.emptyStateActionBtn" @click="onAddWorkflow">
							Create Workflow
						</button>
					</div>
				</div>
			</div>
		</div>

		<!-- Bottom Grid: Executions, Project Progress, Server Run Time -->
		<div :class="$style.bottomGrid">
			<!-- Executions Widget -->
			<div :class="$style.widgetCard">
				<div :class="$style.widgetHeader">
					<span :class="$style.widgetTitle">Executions ({{ allRealExecutions.length }})</span>
					<button
						v-if="allRealExecutions.length > 0"
						:class="$style.viewAllBtn"
						@click="openModalView('executions')"
					>
						View All
					</button>
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
					<!-- Clean Real Empty State -->
					<div v-if="allRealExecutions.length === 0" :class="$style.cardEmptyState">
						<N8nIcon icon="history" size="medium" :class="$style.emptyStateIcon" />
						<span :class="$style.emptyStateText">No executions recorded yet</span>
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
							<!-- Background arc (Dark Green - Total/Inactive) -->
							<path
								d="M 20 80 A 60 60 0 0 1 140 80"
								fill="none"
								stroke="#0e3a2f"
								stroke-width="16"
								stroke-linecap="round"
							/>
							<!-- Completed arc (Teal Green - Active workflows) -->
							<path
								:d="projectProgressData.arcPath"
								fill="none"
								stroke="#228358"
								stroke-width="16"
								stroke-linecap="round"
							/>
						</svg>
						<div :class="$style.gaugeTextOverlay">
							<span :class="$style.gaugePercentage">{{ projectProgressData.percentage }}%</span>
							<span :class="$style.gaugeLabel">Active Workflows</span>
						</div>
					</div>
					<!-- Legend -->
					<div :class="$style.gaugeLegend">
						<div :class="$style.legendItem">
							<span :class="[$style.legendDot, $style.legendDotGreen]"></span>
							<span :class="$style.legendText"
								>Active ({{ projectProgressData.completedCount }})</span
							>
						</div>
						<div :class="$style.legendItem">
							<span :class="[$style.legendDot, $style.legendDotDark]"></span>
							<span :class="$style.legendText"
								>Inactive ({{ projectProgressData.inProgressCount }})</span
							>
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

				<!-- Control Action Buttons (Pause & Reset) -->
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

		<!-- Dialog Popups for Sections (Credentials / Workflows / Executions) -->
		<div v-if="activeModal" :class="$style.modalBackdrop" @click.self="closeModalView">
			<div :class="$style.modalContainer">
				<div :class="$style.modalHeader">
					<div :class="$style.modalTitleBox">
						<h2 :class="$style.modalTitle">
							{{
								activeModal === 'credentials'
									? 'All Credentials'
									: activeModal === 'workflows'
										? 'All Workflows'
										: 'All Executions'
							}}
						</h2>
						<p :class="$style.modalSubtitle">
							{{
								activeModal === 'credentials'
									? 'Manage and search through your authenticated credentials'
									: activeModal === 'workflows'
										? 'Quickly find, run, and edit your project workflows'
										: 'View historical executions, execution logs, and statuses'
							}}
						</p>
					</div>
					<button :class="$style.modalCloseBtn" @click="closeModalView">
						<N8nIcon icon="times" size="medium" />
					</button>
				</div>

				<!-- Search Input in Dialog -->
				<div :class="$style.modalSearchBar">
					<N8nIcon icon="search" size="medium" :class="$style.modalSearchIcon" />
					<input
						v-model="modalSearchQuery"
						type="text"
						:placeholder="`Search ${activeModal}...`"
						:class="$style.modalSearchInput"
					/>
					<button
						v-if="activeModal === 'credentials'"
						:class="$style.modalActionBtn"
						@click="onNewCredential"
					>
						+ New Credential
					</button>
					<button
						v-else-if="activeModal === 'workflows'"
						:class="$style.modalActionBtn"
						@click="onAddWorkflow"
					>
						+ New Workflow
					</button>
				</div>

				<!-- Modal Content Body -->
				<div :class="$style.modalBody">
					<!-- Credentials List Modal -->
					<div v-if="activeModal === 'credentials'" :class="$style.modalListGrid">
						<div
							v-for="item in modalFilteredCredentials"
							:key="item.id"
							:class="$style.modalItemCard"
							@click="openCredentialItem(item.id)"
						>
							<div :class="$style.modalItemIconContainer">
								<CredentialIcon :credential-type-name="item.type" :size="32" />
							</div>
							<div :class="$style.modalItemInfo">
								<span :class="$style.modalItemName">{{ item.name }}</span>
								<span :class="$style.modalItemMeta">
									Type: {{ item.type }} &bull; Updated {{ item.timeAgo }} &bull; Created
									{{ item.createdDate }}
								</span>
							</div>
							<N8nIcon icon="chevron-right" size="small" :class="$style.modalItemArrow" />
						</div>
						<div v-if="modalFilteredCredentials.length === 0" :class="$style.emptyModal">
							No credentials found
						</div>
					</div>

					<!-- Workflows List Modal -->
					<div v-else-if="activeModal === 'workflows'" :class="$style.modalListGrid">
						<div
							v-for="wf in modalFilteredWorkflows"
							:key="wf.id"
							:class="$style.modalItemCard"
							@click="openWorkflow(wf.id)"
						>
							<div
								:class="$style.wfIconBox"
								:style="{
									backgroundColor: wf.iconBg,
									color: wf.iconColor,
									width: '38px',
									height: '38px',
								}"
							>
								<N8nIcon :icon="wf.icon" size="medium" />
							</div>
							<div :class="$style.modalItemInfo">
								<span :class="$style.modalItemName">{{ wf.name }}</span>
								<span :class="$style.modalItemMeta">{{ wf.date }}</span>
							</div>
							<div :class="$style.modalItemRight">
								<span :class="[wf.active ? $style.badgeActive : $style.badgeInactive]">
									{{ wf.active ? 'Active' : 'Inactive' }}
								</span>
								<N8nIcon icon="chevron-right" size="small" :class="$style.modalItemArrow" />
							</div>
						</div>
						<div v-if="modalFilteredWorkflows.length === 0" :class="$style.emptyModal">
							No workflows found
						</div>
					</div>

					<!-- Executions List Modal -->
					<div v-else-if="activeModal === 'executions'" :class="$style.modalListGrid">
						<div
							v-for="ex in modalFilteredExecutions"
							:key="ex.id"
							:class="$style.modalItemCard"
							@click="openExecutionItem(ex)"
						>
							<div :class="$style.modalItemInfo">
								<span :class="$style.modalItemName">{{ ex.name }}</span>
								<span :class="$style.modalItemMeta">{{ ex.time }} &bull; ID: {{ ex.id }}</span>
							</div>
							<div :class="$style.modalItemRight">
								<span
									:class="[
										$style.exStatusPill,
										ex.status === 'Failed' ? $style.exStatusPillFailed : '',
									]"
								>
									{{ ex.status }}
								</span>
								<N8nIcon icon="chevron-right" size="small" :class="$style.modalItemArrow" />
							</div>
						</div>
						<div v-if="modalFilteredExecutions.length === 0" :class="$style.emptyModal">
							No executions found
						</div>
					</div>
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
	background: #fff;
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

.headerActionsSmall {
	display: flex;
	align-items: center;
	gap: 8px;
}

.viewAllBtn {
	background: transparent;
	border: none;
	color: #0e3a2f;
	font-size: 12px;
	font-weight: 600;
	cursor: pointer;
	padding: 2px 6px;
	border-radius: 6px;

	&:hover {
		background: #f1f5f9;
		text-decoration: underline;
	}
}

.iconButton {
	display: flex;
	align-items: center;
	justify-content: center;
	width: 36px;
	height: 36px;
	border-radius: 50%;
	background: #fff;
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
	color: #fff;
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
	margin-bottom: 16px;
}

.titleActions {
	display: flex;
	align-items: center;
	gap: 10px;
}

.pageTitle {
	font-size: 26px;
	font-weight: 800;
	color: #0f172a;
	margin: 0 0 2px;
	letter-spacing: -0.02em;
}

.pageSubtitle {
	font-size: 13px;
	color: #64748b;
	margin: 0;
}

/* Quick Navigation Hub Bar */
.quickNavHub {
	display: flex;
	align-items: center;
	gap: 8px;
	margin-bottom: 24px;
	overflow-x: auto;
	padding-bottom: 4px;
}

.navTabBtn {
	display: flex;
	align-items: center;
	gap: 8px;
	background: #ffffff;
	border: 1px solid #e2e8f0;
	border-radius: 12px;
	padding: 8px 16px;
	font-size: 13px;
	font-weight: 600;
	color: #475569;
	cursor: pointer;
	transition: all 0.15s ease;
	white-space: nowrap;

	&:hover {
		background: #f8fafc;
		border-color: #cbd5e1;
		color: #0f172a;
	}
}

.navTabActive {
	background: #0e3a2f;
	border-color: #0e3a2f;
	color: #ffffff;

	&:hover {
		background: #082820;
		border-color: #082820;
		color: #ffffff;
	}

	.navTabIcon {
		color: #34d399;
	}
}

.navTabIcon {
	color: #64748b;
	display: flex;
	align-items: center;
}

.navTabCount {
	background: rgba(0, 0, 0, 0.08);
	color: inherit;
	font-size: 11px;
	font-weight: 700;
	padding: 1px 6px;
	border-radius: 10px;
}

.addWorkflowBtn {
	background: #0e3a2f;
	border: none;
	border-radius: 18px;
	font-weight: 600;
	font-size: 13px;
	padding: 8px 18px;
	color: #fff;
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
	background: #fff;
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
	color: #fff;
}

.kpiValueFeatured {
	font-size: 34px;
	font-weight: 800;
	color: #fff;
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
	color: #fff;
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
	background: #fff;
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
	background: repeating-linear-gradient(45deg, #cbd5e1, #cbd5e1 3px, #fff 3px, #fff 6px);
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
	background: #fff;
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
	align-items: center;
	gap: 12px;
	padding: 4px 0;
	cursor: pointer;

	&:hover .credName {
		color: #0e3a2f;
	}
}

.credIconContainer {
	display: flex;
	align-items: center;
	justify-content: center;
	width: 32px;
	height: 32px;
	border-radius: 8px;
	background: #f8fafc;
	border: 1px solid #f1f5f9;
	flex-shrink: 0;
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

/* Real Empty State in cards */
.cardEmptyState {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: center;
	padding: 24px 12px;
	text-align: center;
	background: #fafbfc;
	border: 1px dashed #e2e8f0;
	border-radius: 12px;
}

.emptyStateIcon {
	color: #cbd5e1;
	margin-bottom: 8px;
}

.emptyStateText {
	font-size: 12px;
	color: #94a3b8;
	font-weight: 500;
	margin-bottom: 8px;
}

.emptyStateActionBtn {
	background: #0e3a2f;
	color: #fff;
	border: none;
	border-radius: 12px;
	padding: 5px 12px;
	font-size: 11px;
	font-weight: 600;
	cursor: pointer;

	&:hover {
		background: #082820;
	}
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
	color: #fff;
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
	background: #fff;
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
	color: #fff;
	display: flex;
	align-items: center;
	justify-content: center;
	cursor: pointer;
	transition: transform 0.1s ease;

	&:hover {
		transform: scale(1.08);
	}
}

/* Modal Backdrop & Dialog Box */
.modalBackdrop {
	position: fixed;
	top: 0;
	left: 0;
	right: 0;
	bottom: 0;
	background: rgba(15, 23, 42, 0.45);
	backdrop-filter: blur(4px);
	z-index: 9999;
	display: flex;
	align-items: center;
	justify-content: center;
	padding: 24px;
}

.modalContainer {
	background: #fff;
	border-radius: 20px;
	width: 100%;
	max-width: 680px;
	max-height: 85vh;
	display: flex;
	flex-direction: column;
	box-shadow:
		0 20px 25px -5px rgba(0, 0, 0, 0.1),
		0 10px 10px -5px rgba(0, 0, 0, 0.04);
	border: 1px solid #e2e8f0;
	overflow: hidden;
	animation: modalSlideUp 0.2s ease-out;
}

@keyframes modalSlideUp {
	from {
		opacity: 0;
		transform: translateY(12px) scale(0.98);
	}
	to {
		opacity: 1;
		transform: translateY(0) scale(1);
	}
}

.modalHeader {
	display: flex;
	align-items: flex-start;
	justify-content: space-between;
	padding: 20px 24px 16px;
	border-bottom: 1px solid #f1f5f9;
}

.modalTitleBox {
	display: flex;
	flex-direction: column;
}

.modalTitle {
	font-size: 18px;
	font-weight: 800;
	color: #0f172a;
	margin: 0;
}

.modalSubtitle {
	font-size: 12px;
	color: #64748b;
	margin: 4px 0 0;
}

.modalCloseBtn {
	background: transparent;
	border: none;
	color: #94a3b8;
	cursor: pointer;
	padding: 4px;
	border-radius: 6px;
	display: flex;
	align-items: center;
	justify-content: center;

	&:hover {
		background: #f1f5f9;
		color: #0f172a;
	}
}

.modalSearchBar {
	display: flex;
	align-items: center;
	gap: 12px;
	padding: 12px 24px;
	background: #f8fafc;
	border-bottom: 1px solid #e2e8f0;
}

.modalSearchIcon {
	color: #94a3b8;
}

.modalSearchInput {
	flex: 1;
	background: #fff;
	border: 1px solid #cbd5e1;
	border-radius: 8px;
	padding: 8px 12px;
	font-size: 13px;
	color: #0f172a;
	outline: none;

	&:focus {
		border-color: #0e3a2f;
	}
}

.modalActionBtn {
	background: #0e3a2f;
	color: #fff;
	border: none;
	border-radius: 8px;
	padding: 8px 14px;
	font-size: 12px;
	font-weight: 600;
	cursor: pointer;
	white-space: nowrap;

	&:hover {
		background: #082820;
	}
}

.modalBody {
	padding: 16px 24px 24px;
	overflow-y: auto;
	flex: 1;
}

.modalListGrid {
	display: flex;
	flex-direction: column;
	gap: 10px;
}

.modalItemCard {
	display: flex;
	align-items: center;
	gap: 14px;
	padding: 12px 16px;
	background: #fff;
	border: 1px solid #e2e8f0;
	border-radius: 12px;
	cursor: pointer;
	transition: all 0.15s ease;

	&:hover {
		border-color: #0e3a2f;
		background: #f8fafc;
		transform: translateY(-1px);
	}
}

.modalItemIconContainer {
	width: 40px;
	height: 40px;
	display: flex;
	align-items: center;
	justify-content: center;
	background: #f1f5f9;
	border-radius: 10px;
	flex-shrink: 0;
}

.modalItemInfo {
	display: flex;
	flex-direction: column;
	flex: 1;
	min-width: 0;
}

.modalItemName {
	font-size: 14px;
	font-weight: 700;
	color: #0f172a;
	white-space: nowrap;
	overflow: hidden;
	text-overflow: ellipsis;
}

.modalItemMeta {
	font-size: 11px;
	color: #64748b;
	margin-top: 2px;
}

.modalItemRight {
	display: flex;
	align-items: center;
	gap: 12px;
}

.modalItemArrow {
	color: #cbd5e1;
}

.badgeActive {
	background: #f0fdf4;
	color: #16a34a;
	border: 1px solid #bbf7d0;
	font-size: 11px;
	font-weight: 600;
	padding: 2px 8px;
	border-radius: 6px;
}

.badgeInactive {
	background: #f1f5f9;
	color: #64748b;
	border: 1px solid #e2e8f0;
	font-size: 11px;
	font-weight: 600;
	padding: 2px 8px;
	border-radius: 6px;
}

.emptyModal {
	text-align: center;
	padding: 32px 0;
	color: #94a3b8;
	font-size: 13px;
}

/* Full Responsive Breakdown */
@media (max-width: 1200px) {
	.kpiGrid {
		grid-template-columns: repeat(2, 1fr);
	}

	.middleGrid,
	.bottomGrid {
		grid-template-columns: 1fr;
	}
}

@media (max-width: 768px) {
	.dashboardContainer {
		padding: 16px;
	}

	.topHeader {
		flex-direction: column;
		align-items: stretch;
		gap: 12px;
	}

	.searchWrapper {
		width: 100%;
	}

	.kpiGrid {
		grid-template-columns: 1fr;
	}

	.titleBar {
		flex-direction: column;
		align-items: flex-start;
		gap: 12px;
	}
}
</style>
