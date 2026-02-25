<script lang="ts">
	import type { ArchetypeStats } from '$lib/types/metagame';

	let { stats }: { stats: ArchetypeStats[] } = $props();

	const MIN_MATCHES = 10;
	const ACCENT = '#4f46e5';
	const FILL = 'rgba(79, 70, 229, 0.1)';
	const STEPS = 300;

	// Lanczos approximation for ln(Γ(z))
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
			(a - 1) * Math.log(x) +
				(b - 1) * Math.log(1 - x) -
				lnGamma(a) -
				lnGamma(b) +
				lnGamma(a + b),
		);
	}

	// Wilson score 95% CI — used for x-axis range calculation
	function wilsonCI(wr: number, n: number): [number, number] {
		const z = 1.96;
		const denom = 1 + (z * z) / n;
		const center = (wr + z * z / (2 * n)) / denom;
		const margin = (z * Math.sqrt((wr * (1 - wr)) / n + (z * z) / (4 * n * n))) / denom;
		return [Math.max(0, center - margin), Math.min(1, center + margin)];
	}

	let sorted = $derived(
		[...stats]
			.filter((s) => s.totalMatches >= MIN_MATCHES)
			.sort((a, b) => b.overallWinrate - a.overallWinrate),
	);

	let xRange = $derived.by(() => {
		if (sorted.length === 0) return { xMin: 0.35, xMax: 0.65 };
		let lo = 1,
			hi = 0;
		for (const s of sorted) {
			const [ciLo, ciHi] = wilsonCI(s.overallWinrate, s.totalMatches);
			lo = Math.min(lo, ciLo);
			hi = Math.max(hi, ciHi);
		}
		const span = hi - lo;
		lo = Math.max(0, lo - span * 0.15);
		hi = Math.min(1, hi + span * 0.15);
		// Snap to nearest 5%
		lo = Math.floor(lo * 20) / 20;
		hi = Math.ceil(hi * 20) / 20;
		return { xMin: lo, xMax: hi };
	});

	function drawCanvas(
		canvas: HTMLCanvasElement,
		stat: ArchetypeStats,
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

		// Beta(wins+1, losses+draws+1): mode = wins/(wins+losses+draws) = overallWinrate
		const a = stat.wins + 1;
		const b = stat.losses + stat.draws + 1;

		const PL = 2,
			PR = 2,
			PT = 4,
			PB = 22;
		const plotW = W - PL - PR;
		const plotH = H - PT - PB;

		// Sample the PDF across the x-axis range
		const xs: number[] = [];
		const ys: number[] = [];
		for (let i = 0; i <= STEPS; i++) {
			const x = xMin + ((xMax - xMin) * i) / STEPS;
			xs.push(x);
			ys.push(betaPDF(x, a, b));
		}
		const maxY = Math.max(...ys);
		if (!maxY) return;

		const toX = (x: number) => PL + ((x - xMin) / (xMax - xMin)) * plotW;
		const toY = (y: number) => PT + plotH - (y / maxY) * plotH;

		// Fill under curve
		ctx.beginPath();
		ctx.moveTo(toX(xs[0]), toY(0));
		for (let i = 0; i < xs.length; i++) ctx.lineTo(toX(xs[i]), toY(ys[i]));
		ctx.lineTo(toX(xs[xs.length - 1]), toY(0));
		ctx.closePath();
		ctx.fillStyle = FILL;
		ctx.fill();

		// Curve
		ctx.beginPath();
		ctx.moveTo(toX(xs[0]), toY(ys[0]));
		for (let i = 1; i < xs.length; i++) ctx.lineTo(toX(xs[i]), toY(ys[i]));
		ctx.strokeStyle = ACCENT;
		ctx.lineWidth = 1.5;
		ctx.lineJoin = 'round';
		ctx.stroke();

		// Mode line (win rate)
		const modeX = toX(stat.overallWinrate);
		ctx.save();
		ctx.beginPath();
		ctx.moveTo(modeX, PT);
		ctx.lineTo(modeX, PT + plotH);
		ctx.strokeStyle = 'rgba(79, 70, 229, 0.4)';
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

		// X-axis ticks + labels
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
		params: { stat: ArchetypeStats; xMin: number; xMax: number },
	) {
		let currentParams = params;
		let ro: ResizeObserver | null = null;

		function redraw() {
			drawCanvas(canvas, currentParams.stat, currentParams.xMin, currentParams.xMax);
		}

		// Wait one frame for layout to settle
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
	<p class="empty">No archetypes with enough data (minimum {MIN_MATCHES} matches).</p>
{:else}
	<div class="grid">
		{#each sorted as stat}
			<div class="card">
				<div class="name" title={stat.name}>{stat.name}</div>
				<canvas
					class="dist-canvas"
					use:distAction={{ stat, xMin: xRange.xMin, xMax: xRange.xMax }}
				></canvas>
				<div class="meta">
					<span class="wr">{(stat.overallWinrate * 100).toFixed(1)}%</span>
					<span class="muted">
						· n={stat.totalMatches.toLocaleString()} · {(stat.metagameShare * 100).toFixed(1)}% meta
					</span>
				</div>
			</div>
		{/each}
	</div>
	<p class="footnote">
		Each curve is a Beta(wins+1, losses+draws+1) posterior — a Bayesian update from a uniform
		Beta(1,1) prior. Mirror matches, byes, and intentional draws are excluded from wins and losses.
		Draws count against the win rate (denominator). All curves share the same x-axis scale.
		Narrower = more certain; wider = less data.
	</p>
{/if}

<style>
	.grid {
		display: grid;
		grid-template-columns: repeat(auto-fill, minmax(210px, 1fr));
		gap: 0.75rem;
	}

	.card {
		background: var(--color-surface);
		border: 1px solid var(--color-border);
		border-radius: var(--radius);
		padding: 0.625rem 0.75rem 0.5rem;
	}

	.name {
		font-size: 0.8rem;
		font-weight: 600;
		color: var(--color-text);
		margin-bottom: 0.25rem;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
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
		color: var(--color-text);
	}

	.muted {
		color: var(--color-text-muted);
	}

	.footnote {
		font-size: 0.75rem;
		color: var(--color-text-muted);
		margin-top: 1.5rem;
		line-height: 1.6;
		max-width: 60ch;
	}

	.empty {
		color: var(--color-text-muted);
	}
</style>
