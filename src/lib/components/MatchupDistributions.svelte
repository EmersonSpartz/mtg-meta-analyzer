<script lang="ts">
	import type { MatchupCell } from '$lib/types/metagame';
	import { base } from '$app/paths';

	let {
		matchups,
	}: {
		matchups: { opponent: string; cell: MatchupCell }[];
	} = $props();

	const MIN_MATCHES = 3;
	const STEPS = 300;

	function lnGamma(z: number): number {
		const c = [
			0.99999999999980993, 676.5203681218851, -1259.1392167224028, 771.32342877765313,
			-176.61502916214059, 12.507343278686905, -0.13857109526572012, 9.9843695780195716e-6,
			1.5056327351493116e-7,
		];
		if (z < 0.5) return Math.log(Math.PI / Math.sin(Math.PI * z)) - lnGamma(1 - z);
		z -= 1;
		let x = c[0];
		for (let i = 1; i <= 8; i++) x += c[i] / (z + i);
		const t = z + 7.5;
		return 0.5 * Math.log(2 * Math.PI) + (z + 0.5) * Math.log(t) - t + Math.log(x);
	}

	function betaPDF(x: number, a: number, b: number): number {
		if (x <= 0 || x >= 1) return 0;
		return Math.exp(
			(a - 1) * Math.log(x) + (b - 1) * Math.log(1 - x) - lnGamma(a) - lnGamma(b) + lnGamma(a + b),
		);
	}

	function wilsonCI(wr: number, n: number): [number, number] {
		const z = 1.96;
		const denom = 1 + (z * z) / n;
		const center = (wr + (z * z) / (2 * n)) / denom;
		const margin = (z * Math.sqrt((wr * (1 - wr)) / n + (z * z) / (4 * n * n))) / denom;
		return [Math.max(0, center - margin), Math.min(1, center + margin)];
	}

	// Game-level winrate (primary — more data)
	function gameWr(cell: MatchupCell): number {
		const n = cell.gameWins + cell.gameLosses;
		return n > 0 ? cell.gameWins / n : (cell.winrate ?? 0.5);
	}

	// Color by game win rate
	function colors(wr: number): { stroke: string; fill: string } {
		if (wr > 0.52) return { stroke: '#16a34a', fill: 'rgba(22, 163, 74, 0.12)' };
		if (wr < 0.48) return { stroke: '#dc2626', fill: 'rgba(220, 38, 38, 0.12)' };
		return { stroke: '#6b7280', fill: 'rgba(107, 114, 128, 0.12)' };
	}

	let sortBy = $state<'winrate' | 'popularity'>('popularity');

	let sorted = $derived(
		[...matchups]
			.filter((m) => m.cell.total >= MIN_MATCHES)
			.sort((a, b) =>
				sortBy === 'popularity'
					? b.cell.total - a.cell.total
					: gameWr(b.cell) - gameWr(a.cell),
			),
	);

	let sparse = $derived(
		[...matchups]
			.filter((m) => m.cell.total > 0 && m.cell.total < MIN_MATCHES)
			.sort((a, b) => gameWr(b.cell) - gameWr(a.cell)),
	);

	// Shared x-axis — use match-level CI so wider match curve is always visible
	let xRange = $derived.by(() => {
		if (sorted.length === 0) return { xMin: 0.3, xMax: 0.7 };
		let lo = 1,
			hi = 0;
		for (const m of sorted) {
			const wr = m.cell.winrate ?? 0.5;
			const [ciLo, ciHi] = wilsonCI(wr, m.cell.total);
			lo = Math.min(lo, ciLo);
			hi = Math.max(hi, ciHi);
		}
		const span = hi - lo;
		lo = Math.max(0, lo - span * 0.15);
		hi = Math.min(1, hi + span * 0.15);
		lo = Math.floor(lo * 20) / 20;
		hi = Math.ceil(hi * 20) / 20;
		return { xMin: lo, xMax: hi };
	});

	function drawCanvas(
		canvas: HTMLCanvasElement,
		cell: MatchupCell,
		xMin: number,
		xMax: number,
	) {
		const dpr = window.devicePixelRatio || 1;
		const W = canvas.offsetWidth;
		const H = canvas.offsetHeight;
		if (!W || !H) return;

		canvas.width = W * dpr;
		canvas.height = H * dpr;
		const ctx = canvas.getContext('2d')!;
		ctx.scale(dpr, dpr);
		ctx.clearRect(0, 0, W, H);

		// Game-level (primary — more data, narrower)
		const gn = cell.gameWins + cell.gameLosses;
		const gWr = gn > 0 ? cell.gameWins / gn : (cell.winrate ?? 0.5);
		const { stroke, fill } = colors(gWr);

		const PL = 2, PR = 2, PT = 4, PB = 22;
		const plotW = W - PL - PR;
		const plotH = H - PT - PB;

		// Compute game-level curve points (normalized to this max for height)
		const gameA = cell.gameWins + 1;
		const gameB = cell.gameLosses + 1;
		const gxs: number[] = [], gys: number[] = [];
		let maxY = 0;
		for (let i = 0; i <= STEPS; i++) {
			const x = xMin + ((xMax - xMin) * i) / STEPS;
			const y = betaPDF(x, gameA, gameB);
			gxs.push(x); gys.push(y);
			if (y > maxY) maxY = y;
		}
		if (!maxY) return;

		// Compute match-level curve (dashed gray, wider — normalized to same maxY so it appears shorter)
		const mA = cell.wins + 1;
		const mB = cell.losses + 1; // draws excluded — neither win nor loss
		const mxs: number[] = [], mys: number[] = [];
		for (let i = 0; i <= STEPS; i++) {
			const x = xMin + ((xMax - xMin) * i) / STEPS;
			mxs.push(x); mys.push(betaPDF(x, mA, mB));
		}

		const toX = (x: number) => PL + ((x - xMin) / (xMax - xMin)) * plotW;
		const toY = (y: number) => PT + plotH - (y / maxY) * plotH;

		// 50% reference line — solid, more visible
		const refX = toX(0.5);
		if (refX >= PL && refX <= PL + plotW) {
			ctx.beginPath();
			ctx.moveTo(refX, PT);
			ctx.lineTo(refX, PT + plotH);
			ctx.strokeStyle = 'rgba(100, 116, 139, 0.55)';
			ctx.lineWidth = 1.5;
			ctx.stroke();
		}

		// Match-level distribution (behind — dashed gray, wider)
		ctx.beginPath();
		ctx.moveTo(toX(mxs[0]), toY(0));
		for (let i = 0; i < mxs.length; i++) ctx.lineTo(toX(mxs[i]), toY(mys[i]));
		ctx.lineTo(toX(mxs[mxs.length - 1]), toY(0));
		ctx.closePath();
		ctx.fillStyle = 'rgba(150, 150, 150, 0.12)';
		ctx.fill();

		ctx.beginPath();
		ctx.moveTo(toX(mxs[0]), toY(mys[0]));
		for (let i = 1; i < mxs.length; i++) ctx.lineTo(toX(mxs[i]), toY(mys[i]));
		ctx.strokeStyle = 'rgba(150, 150, 150, 0.5)';
		ctx.lineWidth = 1;
		ctx.setLineDash([3, 3]);
		ctx.lineJoin = 'round';
		ctx.stroke();
		ctx.setLineDash([]);

		// Game-level distribution (front — solid colored, narrower)
		ctx.beginPath();
		ctx.moveTo(toX(gxs[0]), toY(0));
		for (let i = 0; i < gxs.length; i++) ctx.lineTo(toX(gxs[i]), toY(gys[i]));
		ctx.lineTo(toX(gxs[gxs.length - 1]), toY(0));
		ctx.closePath();
		ctx.fillStyle = fill;
		ctx.fill();

		// Curve
		ctx.beginPath();
		ctx.moveTo(toX(gxs[0]), toY(gys[0]));
		for (let i = 1; i < gxs.length; i++) ctx.lineTo(toX(gxs[i]), toY(gys[i]));
		ctx.strokeStyle = stroke;
		ctx.lineWidth = 1.5;
		ctx.lineJoin = 'round';
		ctx.stroke();

		// Mode line (game wr)
		const modeX = toX(gWr);
		ctx.save();
		ctx.beginPath();
		ctx.moveTo(modeX, PT);
		ctx.lineTo(modeX, PT + plotH);
		ctx.strokeStyle = stroke.replace(')', ', 0.35)').replace('rgb', 'rgba');
		ctx.lineWidth = 1;
		ctx.setLineDash([3, 3]);
		ctx.stroke();
		ctx.restore();

		// Baseline
		ctx.beginPath();
		ctx.moveTo(PL, PT + plotH);
		ctx.lineTo(PL + plotW, PT + plotH);
		ctx.strokeStyle = '#e5e7eb';
		ctx.lineWidth = 1;
		ctx.stroke();

		// X-axis ticks
		const span = xMax - xMin;
		const tickStep = span <= 0.15 ? 0.025 : span <= 0.3 ? 0.05 : 0.1;
		const firstTick = Math.ceil(xMin / tickStep) * tickStep;
		ctx.fillStyle = '#9ca3af';
		ctx.font = '9px system-ui, sans-serif';
		ctx.textAlign = 'center';
		ctx.textBaseline = 'top';
		for (
			let t = firstTick;
			t <= xMax + 0.0001;
			t = Math.round((t + tickStep) * 10000) / 10000
		) {
			const tx = toX(t);
			ctx.beginPath();
			ctx.moveTo(tx, PT + plotH);
			ctx.lineTo(tx, PT + plotH + 3);
			ctx.strokeStyle = '#e5e7eb';
			ctx.lineWidth = 1;
			ctx.stroke();
			ctx.fillStyle = '#9ca3af';
			ctx.fillText(`${Math.round(t * 100)}%`, tx, PT + plotH + 5);
		}
	}

	function distAction(
		canvas: HTMLCanvasElement,
		params: { cell: MatchupCell; xMin: number; xMax: number },
	) {
		let currentParams = params;
		let ro: ResizeObserver | null = null;

		function redraw() {
			drawCanvas(canvas, currentParams.cell, currentParams.xMin, currentParams.xMax);
		}

		requestAnimationFrame(() => {
			redraw();
			ro = new ResizeObserver(redraw);
			ro.observe(canvas);
		});

		return {
			update(newParams: typeof params) {
				currentParams = newParams;
				redraw();
			},
			destroy() {
				ro?.disconnect();
			},
		};
	}
</script>

{#if sorted.length === 0}
	<p class="empty">No matchups with enough data (minimum {MIN_MATCHES} matches).</p>
{:else}
	<details class="how-to-read">
		<summary>How to read these charts</summary>
		<div class="how-to-read-body">
			<p>
				Each card shows one matchup. The <strong>curve</strong> is a probability distribution —
				it represents where this deck's true win rate against that opponent probably falls,
				given the data we have. The peak is the best estimate; the width shows uncertainty.
			</p>
			<p>
				<strong>Narrow</strong> means we're confident — lots of matches played.
				<strong>Wide</strong> means we're not sure yet — only a few matches in the data.
				A wide curve crossing 50% means we genuinely can't tell if this matchup is favored or not.
			</p>
			<p>
				<strong style="color:#16a34a">Green</strong> = this deck wins more often than not.
				<strong style="color:#dc2626">Red</strong> = it loses more often than not.
				<strong style="color:#6b7280">Gray</strong> = too close to call.
				The vertical line marks 50%.
			</p>
		</div>
	</details>

	<div class="sort-bar">
		<span class="sort-label">Sort:</span>
		<button class="sort-btn" class:active={sortBy === 'winrate'} onclick={() => (sortBy = 'winrate')}>Win rate</button>
		<button class="sort-btn" class:active={sortBy === 'popularity'} onclick={() => (sortBy = 'popularity')}>Popularity</button>
	</div>
	<div class="grid">
		{#each sorted as m}
			{@const gn = m.cell.gameWins + m.cell.gameLosses}
			{@const wr = gn > 0 ? m.cell.gameWins / gn : (m.cell.winrate ?? 0)}
			{@const { stroke } = colors(wr)}
			<div class="card">
				<div class="opponent">
					<a href="{base}/archetypes/{encodeURIComponent(m.opponent)}" style="color:{stroke}">
						{m.opponent}
					</a>
				</div>
				<canvas
					class="dist-canvas"
					use:distAction={{ cell: m.cell, xMin: xRange.xMin, xMax: xRange.xMax }}
				></canvas>
				<div class="meta">
					<span class="wr" style="color:{stroke}">{(wr * 100).toFixed(1)}%</span>
					<span class="muted">
						· {m.cell.gameWins}–{m.cell.gameLosses} ({gn}g) · {m.cell.total}m
					</span>
				</div>
			</div>
		{/each}
	</div>

	{#if sparse.length > 0}
		<details class="sparse">
			<summary>Low-sample matchups ({sparse.length})</summary>
			<div class="sparse-list">
				{#each sparse as m}
					{@const gn = m.cell.gameWins + m.cell.gameLosses}
					{@const wr = gn > 0 ? m.cell.gameWins / gn : (m.cell.winrate ?? 0)}
					<span>
						<a href="{base}/archetypes/{encodeURIComponent(m.opponent)}">{m.opponent}</a>
						<span class="muted">{(wr * 100).toFixed(0)}% ({m.cell.gameWins}–{m.cell.gameLosses}, {gn}g / {m.cell.total}m)</span>
					</span>
				{/each}
			</div>
		</details>
	{/if}

	<div class="explainer">
		<p class="explainer-lead">
			Each curve estimates this deck's true win rate against the opponent.
			<strong>Solid</strong> = game wins/losses &nbsp;·&nbsp; <strong>Dashed gray</strong> = match wins/losses &nbsp;·&nbsp;
			<span style="color:#16a34a">green</span> = favored &nbsp;·&nbsp; <span style="color:#dc2626">red</span> = unfavored.
			Narrower = more confident. All curves share the same x-axis.
		</p>
		<details class="explainer-details">
			<summary>Why game wins instead of match wins?</summary>
			<div class="explainer-body">
				<p>
					Most win rate charts count match results — but in Best-of-3, each match contains 2–3
					individual games. Counting those gives roughly <strong>2.4× more data points</strong>,
					which tightens the estimate by ~35%. With only 10–30 matches against any given opponent
					in a tournament, that difference is meaningful.
				</p>
				<p>
					The solid colored curve is built from individual game wins and losses.
					The dashed gray curve behind it is the traditional match-level estimate.
					They should peak at nearly the same win rate — the width difference is just confidence.
					When the solid curve is noticeably narrower, the extra game data is doing real work.
				</p>
				<p>
					Games within a match aren't fully independent (same sideboard, same mental state), so
					this isn't identical to having 2.4× as many matches. But across hundreds of different
					players, those effects average out and the extra signal is real.
				</p>
			</div>
		</details>
	</div>
{/if}

<style>
	.how-to-read {
		font-size: 0.75rem;
		color: var(--color-text-muted);
		margin-bottom: 0.75rem;
	}

	.how-to-read summary {
		cursor: pointer;
		color: var(--color-text-muted);
		font-style: italic;
		user-select: none;
	}

	.how-to-read summary:hover {
		color: var(--color-accent);
	}

	.how-to-read-body {
		margin-top: 0.5rem;
		display: flex;
		flex-direction: column;
		gap: 0.4rem;
		line-height: 1.6;
		padding: 0.6rem 0.75rem;
		background: var(--color-surface);
		border: 1px solid var(--color-border);
		border-radius: var(--radius);
		max-width: 56ch;
	}

	.how-to-read-body p {
		margin: 0;
	}

	.sort-bar {
		display: flex;
		align-items: center;
		gap: 0.4rem;
		margin-bottom: 0.75rem;
	}

	.sort-label {
		font-size: 0.75rem;
		color: var(--color-text-muted);
	}

	.sort-btn {
		font-size: 0.75rem;
		padding: 0.2rem 0.6rem;
		border: 1px solid var(--color-border);
		border-radius: var(--radius);
		background: none;
		cursor: pointer;
		color: var(--color-text-muted);
	}

	.sort-btn.active {
		background: var(--color-accent);
		color: #fff;
		border-color: var(--color-accent);
	}

	.grid {
		display: grid;
		grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
		gap: 0.75rem;
	}

	.card {
		background: var(--color-surface);
		border: 1px solid var(--color-border);
		border-radius: var(--radius);
		padding: 0.625rem 0.75rem 0.5rem;
	}

	.opponent {
		font-size: 0.8rem;
		font-weight: 600;
		margin-bottom: 0.25rem;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.opponent a {
		text-decoration: none;
	}

	.opponent a:hover {
		text-decoration: underline;
	}

	.dist-canvas {
		width: 100%;
		height: 88px;
		display: block;
	}

	.meta {
		font-size: 0.725rem;
		margin-top: 0.125rem;
		line-height: 1.4;
	}

	.wr {
		font-weight: 600;
	}

	.muted {
		color: var(--color-text-muted);
	}

	.sparse {
		margin-top: 1rem;
		font-size: 0.8rem;
		color: var(--color-text-muted);
	}

	.sparse summary {
		cursor: pointer;
		margin-bottom: 0.5rem;
	}

	.sparse-list {
		display: flex;
		flex-direction: column;
		gap: 0.25rem;
		padding-left: 0.75rem;
	}

	.sparse-list a {
		color: var(--color-accent);
		text-decoration: none;
		margin-right: 0.4rem;
	}

	.explainer {
		margin-top: 1.5rem;
		max-width: 64ch;
	}

	.explainer-lead {
		font-size: 0.75rem;
		color: var(--color-text-muted);
		line-height: 1.6;
		margin-bottom: 0.5rem;
	}

	.explainer-details {
		font-size: 0.75rem;
		color: var(--color-text-muted);
	}

	.explainer-details summary {
		cursor: pointer;
		color: var(--color-accent);
		font-size: 0.75rem;
		user-select: none;
	}

	.explainer-details summary:hover {
		text-decoration: underline;
	}

	.explainer-body {
		margin-top: 0.6rem;
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		line-height: 1.65;
		padding-left: 0.75rem;
		border-left: 2px solid var(--color-border);
	}

	.explainer-body p {
		margin: 0;
	}

	.empty {
		color: var(--color-text-muted);
	}
</style>
