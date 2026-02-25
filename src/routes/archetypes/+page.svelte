<script lang="ts">
	import { globalAttributionMatrix, globalMetagameData } from '$lib/stores/tournaments';
	import AttributionMatrix from '$lib/components/AttributionMatrix.svelte';
	import ArchetypeEditor from '$lib/components/ArchetypeEditor.svelte';
	import { base } from '$app/paths';

	const archetypes = $derived(
		($globalMetagameData?.stats ?? []).sort((a, b) => b.metagameShare - a.metagameShare),
	);
</script>

<svelte:head>
	<title>Archetypes — MTG Meta Analyzer</title>
</svelte:head>

<h1>Archetypes</h1>

{#if archetypes.length > 0}
	<section class="archetype-list-section">
		<ul class="archetype-list">
			{#each archetypes as s}
				<li>
					<a href="{base}/archetypes/{encodeURIComponent(s.name)}">{s.name}</a>
					<span class="meta">{(s.metagameShare * 100).toFixed(1)}% meta · {(s.overallWinrate * 100).toFixed(1)}% WR · {s.totalMatches} matches</span>
				</li>
			{/each}
		</ul>
	</section>
{/if}

<section class="editor-section">
	<h2>Archetype Configuration</h2>
	<p class="section-desc">
		View and edit archetype signature card definitions. Save custom configs
		and set one as active to re-run classification across all tournaments.
	</p>
	<ArchetypeEditor />
</section>

{#if $globalAttributionMatrix}
	<section class="attribution-section">
		<h2>Classification Attribution</h2>
		<p class="section-desc">
			Compares classifier-assigned archetypes (rows) vs player self-reported archetypes (columns). Cells show decklist counts.
		</p>
		<AttributionMatrix matrix={$globalAttributionMatrix} />
	</section>
{/if}

<style>
	h1 {
		font-size: 1.5rem;
		margin-bottom: 1rem;
	}

	.archetype-list-section {
		margin-bottom: 2rem;
	}

	.archetype-list {
		list-style: none;
		padding: 0;
		margin: 0;
		display: flex;
		flex-direction: column;
		gap: 0.15rem;
	}

	.archetype-list li {
		display: flex;
		align-items: baseline;
		gap: 0.75rem;
		padding: 0.3rem 0;
		border-bottom: 1px solid var(--color-border);
		font-size: 0.9rem;
	}

	.archetype-list a {
		color: var(--color-accent);
		text-decoration: none;
		font-weight: 500;
		min-width: 180px;
	}

	.archetype-list a:hover {
		text-decoration: underline;
	}

	.meta {
		color: var(--color-text-muted);
		font-size: 0.8rem;
	}

	.editor-section {
		margin-bottom: 2rem;
	}

	.attribution-section {
		margin-top: 0.5rem;
	}

	h2 {
		font-size: 1.15rem;
		font-weight: 600;
		margin-bottom: 0.5rem;
	}

	.section-desc {
		font-size: 0.85rem;
		color: var(--color-text-muted);
		margin-bottom: 1rem;
	}
</style>
