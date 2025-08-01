<script lang="ts">
import { theme } from '$lib/stores/theme.svelte.js';
import { settings } from '$lib/stores/settings.svelte.js';
import { timeTravel } from '$lib/stores/timeTravel.svelte.js';
import { fontSize } from '$lib/stores/fontSize.svelte.js';
import { dataService, dataReloadService } from '$lib/services/dataService';
import { s } from '$lib/client/localization.svelte';
import { language } from '$lib/stores/language.svelte.js';
import { IconClock } from '@tabler/icons-svelte';
import ChaosIndex from './ChaosIndex.svelte';
import { experimental } from '$lib/stores/experimental.svelte.js';

// Props
interface Props {
	totalReadCount?: number;
	totalStoriesRead?: number;
	offlineMode?: boolean;
	getLastUpdated?: () => string;
	chaosIndex?: {
		score: number;
		summary: string;
		lastUpdated: string;
	};
}

let {
	totalReadCount = 0,
	totalStoriesRead = 0,
	offlineMode = false,
	getLastUpdated = () => 'Never',
	chaosIndex
}: Props = $props();

// Date click state for cycling through different stats
let dateClickCount = $state(0);

// Loading state for exiting time travel
let isExitingTimeTravel = $state(false);

// Theme-aware logo source
const logoSrc = $derived(
	theme.current === 'dark' ? '/svg/kite_dark.svg' : '/svg/kite.svg'
);

function handleLogoClick() {
	// Add animation class for 3 seconds
	const logo = document.querySelector('.logo');
	logo?.classList.add('kite-logo');
	setTimeout(() => {
		logo?.classList.remove('kite-logo');
	}, 3000);
}

function handleDateClick() {
	dateClickCount = (dateClickCount + 1) % 5;
}

// Handle date area keyboard events
function handleDateKeydown(event: KeyboardEvent) {
	if (event.key === 'Enter' || event.key === ' ') {
		event.preventDefault();
		handleDateClick();
	}
}

// Handle font size toggle
function toggleFontSize() {
	const fontSizes: Array<import('$lib/stores/fontSize.svelte.js').FontSize> = ["xs", "small", "normal", "large", "xl"];
	const currentIndex = fontSizes.indexOf(fontSize.current);
	const nextIndex = (currentIndex + 1) % fontSizes.length;
	fontSize.set(fontSizes[nextIndex]);
}

// Helper to capitalize first letter
function capitalizeFirst(str: string): string {
	return str.charAt(0).toUpperCase() + str.slice(1);
}

// Computed date/stats display
const dateDisplay = $derived.by(() => {
	// If in time travel mode, show the selected date
	if (timeTravel.selectedDate) {
		const dateStr = new Intl.DateTimeFormat(language.current, {
			weekday: 'long',
			month: 'long',
			day: 'numeric',
			year: 'numeric'
		}).format(timeTravel.selectedDate);
		return capitalizeFirst(dateStr);
	}
	
	if (dateClickCount === 0) {
		// Default date format
		const now = new Date();
		const dateStr = new Intl.DateTimeFormat(language.current, {
			weekday: 'long',
			month: 'long',
			day: 'numeric'
		}).format(now);
		return capitalizeFirst(dateStr);
	} else if (dateClickCount === 1) {
		return getLastUpdated();
	} else if (dateClickCount === 2) {
		return s('stats.newsToday', { count: totalReadCount.toString() }) || `News today: ${totalReadCount}`;
	} else if (dateClickCount === 3) {
		return s('stats.storiesRead', { count: totalStoriesRead.toString() }) || `Stories read: ${totalStoriesRead}`;
	} else {
		const now = new Date();
		const start = new Date(now.getFullYear(), 0, 1);
		const week = Math.ceil((now.getTime() - start.getTime()) / (1000 * 60 * 60 * 24 * 7));
		const day = Math.ceil((now.getTime() - start.getTime()) / (1000 * 60 * 60 * 24));
		return s('stats.weekDay', { week: week.toString(), day: day.toString() }) || `Week ${week}, Day ${day}`;
	}
});
</script>

<header class="mb-1">
	<div class="relative flex items-center">
		<div class="flex items-center">
			<img
				src={logoSrc}
				alt={s('app.logo.newsAlt') || 'Kite News'}
				class="mr-2 h-8 w-8 logo"
				onclick={handleLogoClick}
				role="presentation"
			/>
			<h1 class="text-xl font-bold text-gray-800 dark:text-gray-200">
				{s('app.title') || 'Kite'}
			</h1>
		</div>
		
		<div class="absolute left-1/2 transform -translate-x-1/2 flex items-center">
		{#if isExitingTimeTravel}
			<!-- Loading state when exiting time travel -->
			<div class="flex items-center gap-2 text-gray-600 dark:text-gray-400">
				<div class="animate-spin h-4 w-4 border-2 border-gray-300 dark:border-gray-600 border-t-blue-500 dark:border-t-blue-400 rounded-full"></div>
				<span class="text-sm">{s('timeTravel.returningToLive') || 'Returning to live...'}</span>
			</div>
		{:else if timeTravel.selectedDate}
			<div class="flex items-center gap-2 bg-blue-50 dark:bg-blue-900/30 px-2 py-1 rounded-lg">
				<svg
					xmlns="http://www.w3.org/2000/svg"
					class="h-4 w-4 text-blue-600 dark:text-blue-400"
					fill="none"
					viewBox="0 0 24 24"
					stroke="currentColor"
				>
					<path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="2"
						d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"
					/>
				</svg>
				<div class="text-sm font-medium text-blue-600 dark:text-blue-400">
					{dateDisplay}
				</div>
				<button
					onclick={async () => {
						// Immediately reset time travel to give instant feedback
						timeTravel.reset();
						dataService.setTimeTravelBatch(null);
						
						// Show loading state
						isExitingTimeTravel = true;
						
						try {
							// Trigger a reload of the data
							await dataReloadService.reloadData();
						} finally {
							isExitingTimeTravel = false;
						}
					}}
					class="ml-1 p-0.5"
					aria-label="Exit time travel mode"
					disabled={isExitingTimeTravel}
				>
					<svg class="w-3 h-3 text-blue-600 dark:text-blue-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
						<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
					</svg>
				</button>
			</div>
		{:else}
			<div
				class="cursor-pointer text-gray-600 dark:text-gray-400 focus-visible-ring rounded px-1 py-1"
				onclick={handleDateClick}
				onkeydown={handleDateKeydown}
				role="button"
				tabindex="0"
				aria-label="Cycle through date and statistics"
			>
				{dateDisplay}
			</div>
		{/if}
		{#if offlineMode}
			<svg
				xmlns="http://www.w3.org/2000/svg"
				class="ml-1 inline h-4 w-4 text-red-500"
				fill="none"
				viewBox="0 0 24 24"
				stroke="currentColor"
			>
				<path
					stroke-linecap="round"
					stroke-linejoin="round"
					stroke-width="2"
					d="M1 1l22 22M16.72 11.06a9 9 0 010 1.88M5.64 3.77A9.95 9.95 0 0112 2c2.35 0 4.5.78 6.22 2.08M12 18v-3.5M12 14H9"
				/>
			</svg>
		{/if}
		</div>
		
		<div class="ml-auto flex items-center space-x-2">
			<!-- Chaos Index - Hidden on mobile, shown in second row -->
			{#if experimental.showChaosIndex && chaosIndex && chaosIndex.score > 0}
				<div class="hidden md:block">
					<ChaosIndex 
						score={chaosIndex.score}
						summary={chaosIndex.summary}
						lastUpdated={chaosIndex.lastUpdated}
					/>
				</div>
			{/if}
			
			<button
				onclick={() => timeTravel.toggle()}
				title={s('header.timeTravel') || 'Time Travel'}
				aria-label={s('header.timeTravel') || 'Time Travel'}
				class="ml-2"
				type="button"
			>
				<IconClock 
					size={24}
					class="text-gray-600 dark:text-gray-400"
				/>
			</button>
			
			<button
				onclick={toggleFontSize}
				title={s('header.fontSize') || 'Font Size'}
				aria-label={s('header.fontSize') || 'Font Size'}
				class="ml-2"
				type="button"
			>
				<img
					src="/svg/font-size.svg"
					alt=""
					class="h-6 w-6 text-gray-600 dark:text-gray-400 dark:invert"
					aria-hidden="true"
				/>
			</button>
			
			<button
				onclick={() => settings.open()}
				title={s('header.settings') || 'Settings'}
				class="ml-2"
				type="button"
			>
				<img
					src="/svg/gear.svg"
					alt=""
					class="h-6 w-6 text-gray-600 dark:text-gray-400 dark:invert"
					aria-hidden="true"
				/>
			</button>
		</div>
	</div>
	
	<!-- Chaos Index on mobile - Second row -->
	{#if experimental.showChaosIndex && chaosIndex && chaosIndex.score > 0}
		<div class="mt-1 md:hidden">
			<ChaosIndex 
				score={chaosIndex.score}
				summary={chaosIndex.summary}
				lastUpdated={chaosIndex.lastUpdated}
			/>
		</div>
	{/if}
</header> 