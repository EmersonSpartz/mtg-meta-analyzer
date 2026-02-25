<script lang="ts">
	import WinRateDistributions from '$lib/components/WinRateDistributions.svelte';
	import FilterPanel from '$lib/components/FilterPanel.svelte';
	import {
		metagameData,
		filteredTournaments,
		tournamentList,
		availableFormats,
	} from '$lib/stores/tournaments';

	const tournamentCount = $derived($filteredTournaments.length);
</script>

<svelte:head>
	<title>Win Rate Distributions — MTG Meta Analyzer</title>
</svelte:head>

<h1>Win Rate Distributions</h1>

<FilterPanel tournaments={$tournamentList} formats={$availableFormats} />

{#if tournamentCount > 0}
	<p class="tournament-info">
		{tournamentCount} tournament{tournamentCount !== 1 ? 's' : ''}
	</p>
{/if}

{#if $metagameData}
	<WinRateDistributions stats={$metagameData.stats} />
{:else}
	<p class="no-data">No data available for the current filters.</p>
{/if}

<style>
	h1 {
		font-size: 1.5rem;
		margin-bottom: 1rem;
	}

	.tournament-info {
		color: var(--color-text-muted);
		font-size: 0.85rem;
		margin: 1rem 0;
	}

	.no-data {
		color: var(--color-text-muted);
		margin-top: 1rem;
	}
</style>
