<script lang="ts">
import { s } from '$lib/client/localization.svelte';
import CitationText from './CitationText.svelte';
import CitationTooltip from './CitationTooltip.svelte';
import { replaceWithNumberedCitations, type CitationMapping } from '$lib/utils/citationContext';
import { aggregateCitationsFromPoints } from '$lib/utils/citationAggregator';
import { parseStructuredText } from '$lib/utils/textParsing';
import type { Article } from '$lib/types';

// Props
interface Props {
	points?: string[];
	articles?: Article[];
	citationMapping?: CitationMapping;
}

let { points = [], articles = [], citationMapping }: Props = $props();

// Shared tooltip reference
let citationTooltip = $state<CitationTooltip | undefined>();

// Convert citations to numbered format if mapping is available
const displayPoints = $derived.by(() => {
	if (!citationMapping) return points;
	return points.map(point => replaceWithNumberedCitations(point, citationMapping));
});

// Get all cited articles from all highlight points
const allCitedArticles = $derived.by(() => {
	return aggregateCitationsFromPoints(displayPoints, citationMapping, articles);
});
</script>

<section class="mt-6">
	<h3 class="mb-4 text-xl font-semibold text-gray-800 dark:text-gray-200">
		{s('section.highlights') || 'Key Points'}
	</h3>
	<div class="border-t border-dashed border-gray-300 dark:border-gray-600">
		{#each displayPoints as point, index}
			{@const parsed = parseStructuredText(point)}
			<div class="relative border-b border-dashed border-gray-300 py-4 pl-10 dark:border-gray-600">
				<div class="absolute top-4 left-0">
					<div class="flex h-6 w-6 items-center justify-center rounded-full bg-[#F9D9B8]">
						<span class="text-sm font-semibold text-gray-800">{index + 1}</span>
					</div>
				</div>
				{#if parsed.hasTitle}
					<div>
						<h4 class="mb-2 font-semibold text-gray-800 dark:text-gray-200">
							<CitationText 
								text={parsed.title!} 
								articles={allCitedArticles.citedArticles} 
								{citationMapping}
								citationTooltip={citationTooltip}
							/>
						</h4>
						<p class="-ml-10 text-gray-700 dark:text-gray-300">
							<CitationText 
								text={parsed.content} 
								articles={allCitedArticles.citedArticles} 
								{citationMapping}
								citationTooltip={citationTooltip}
							/>
						</p>
					</div>
				{:else}
					<p class="text-gray-700 dark:text-gray-300">
						<CitationText 
							text={parsed.content} 
							articles={allCitedArticles.citedArticles} 
							{citationMapping}
							citationTooltip={citationTooltip}
						/>
					</p>
				{/if}
			</div>
		{/each}
	</div>
</section>

<!-- Shared Citation Tooltip -->
<CitationTooltip 
	bind:this={citationTooltip} 
	articles={allCitedArticles.citedArticles} 
	citationNumbers={allCitedArticles.citedNumbers} 
	hasCommonKnowledge={allCitedArticles.hasCommonKnowledge}
	citedItems={allCitedArticles.citedItems}
/> 