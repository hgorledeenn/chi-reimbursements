<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';

	let { rows = [] } = $props();


	// ----------------------------
	// Responsive sizing
	// ----------------------------

	let container;
	let containerWidth = $state(0);

	onMount(() => {
		const observer = new ResizeObserver(entries => {
			containerWidth = entries[0].contentRect.width;
		});

		observer.observe(container);

		return () => observer.disconnect();
	});


	const height = 350;

	const margin = {
		top: 25,
		right: 40,
		bottom: 50,
		left: 80
	};


	let width = $derived(
		Math.max(containerWidth - margin.left - margin.right, 100)
	);

	let chartHeight = $derived(
		height - margin.top - margin.bottom
	);



	// ----------------------------
	// Hover state
	// ----------------------------

	const today = new Date();

	let hoverDate = $state(
		new Date(
			2000,
			today.getMonth(),
			today.getDate()
		)
	);

	let hovering = $state(false);



	// ----------------------------
	// Build cumulative yearly data
	// ----------------------------

	let cumulativeData = $derived.by(() => {

		if (!rows.length) return [];


		const currentYear = today.getFullYear();


		return [
			currentYear - 2,
			currentYear - 1,
			currentYear
		].map(year => {


			const yearlyRows = rows
				.filter(d =>
					d.paymentDate.getFullYear() === year
				)
				.sort((a,b) =>
					a.paymentDate - b.paymentDate
				);


			const dailyTotals = d3.rollup(
				yearlyRows,
				v => d3.sum(v, d => d.amount),
				d => d3.timeDay(d.paymentDate)
			);


			let runningTotal = 0;

			const values = [];


			for (
				let date = new Date(year,0,1);
				date <= new Date(year,11,31);
				date = d3.timeDay.offset(date,1)
			) {


				if (
					year === currentYear &&
					date > today
				) {
					break;
				}


				runningTotal +=
					dailyTotals.get(+date) ?? 0;


				values.push({
					date: new Date(date),
					total: runningTotal
				});

			}


			return {
				year,
				values
			};

		});
	});



	// ----------------------------
	// Scales
	// ----------------------------

	const fakeStart = new Date(2000,0,1);
	const fakeEnd = new Date(2000,11,31);


	let xScale = $derived(
		d3.scaleTime()
			.domain([
				fakeStart,
				fakeEnd
			])
			.range([
				0,
				width
			])
	);



	let yScale = $derived(
		d3.scaleLinear()
			.domain([
				0,
				d3.max(
					cumulativeData,
					year =>
						d3.max(
							year.values,
							d => d.total
						)
				) ?? 0
			])
			.nice()
			.range([
				chartHeight,
				0
			])
	);



	// ----------------------------
	// Line
	// ----------------------------

	const line = d3.line()
		.x(d =>
			xScale(
				new Date(
					2000,
					d.date.getMonth(),
					d.date.getDate()
				)
			)
		)
		.y(d =>
			yScale(d.total)
		)
		.curve(
			d3.curveStepAfter
		);



	// ----------------------------
	// Axes
	// ----------------------------

	let svg;


	$effect(() => {

		if (!svg) return;


		d3.select(svg)
			.select(".x-axis")
			.call(
				d3.axisBottom(xScale)
					.ticks(12)
					.tickFormat(
						d3.timeFormat("%b")
					)
			);


		d3.select(svg)
			.select(".y-axis")
			.call(
				d3.axisLeft(yScale)
					.ticks(5)
					.tickFormat(
						d3.format("$,.2s")
					)
			);

	});

		// ----------------------------
	// Hover behavior
	// ----------------------------

	function handleMouseMove(event) {

		const [x] = d3.pointer(event);

		const date = xScale.invert(x);


		hoverDate = new Date(
			2000,
			date.getMonth(),
			date.getDate()
		);

	}


	function handleMouseEnter() {
		hovering = true;
	}


	function handleMouseLeave() {
		hovering = false;
	}



	function valueAtHover(values) {

		if (!values.length) return null;


		return values.reduce((closest,current) => {


			const closestDate =
				new Date(
					2000,
					closest.date.getMonth(),
					closest.date.getDate()
				);


			const currentDate =
				new Date(
					2000,
					current.date.getMonth(),
					current.date.getDate()
				);



			return Math.abs(currentDate - hoverDate)
				<
				Math.abs(closestDate - hoverDate)
				?
				current
				:
				closest;

		});

	}



	let todayMarker = $derived(
		new Date(
			2000,
			today.getMonth(),
			today.getDate()
		)
	);



	const colors = [
		"#1f77b4",
		"#ff7f0e",
		"#2ca02c"
	];

</script>



<div class="chart-container" bind:this={container}>


	<!-- Persistent tooltip / legend -->

	<div class="tooltip">

		<strong>
			{d3.timeFormat("%B %d")(hoverDate)}
		</strong>


		{#each cumulativeData as year,i}

			{@const point=valueAtHover(year.values)}

			{#if point}

				<div class="tooltip-row">

					<span
						class="dot"
						style="background:{colors[i]}"
					></span>


					<span>
						{year.year}:
						{d3.format("$,.3s")(point.total)}
					</span>

				</div>

			{/if}

		{/each}

	</div>



	<svg
		bind:this={svg}
		width={containerWidth}
		height={height}
	>

		<g transform="translate({margin.left},{margin.top})">


			<g
				class="x-axis"
				transform="translate(0,{chartHeight})"
			/>

			<g class="y-axis"/>



			{#each cumulativeData as year,i}

				<path
					d={line(year.values)}
					fill="none"
					stroke={colors[i]}
					stroke-width="3"
				/>

			{/each}



			<!-- Today marker -->

			<line
				x1={xScale(todayMarker)}
				x2={xScale(todayMarker)}
				y1="0"
				y2={chartHeight}
				stroke="gray"
				stroke-dasharray="6 5"
			/>



			<!-- Hover line -->

			{#if hovering}

				<line
					x1={xScale(hoverDate)}
					x2={xScale(hoverDate)}
					y1="0"
					y2={chartHeight}
					stroke="black"
					stroke-dasharray="4"
				/>

			{/if}



			<!-- Hover dots -->

			{#if hovering}

				{#each cumulativeData as year,i}

					{@const point=valueAtHover(year.values)}

					{#if point}

						<circle
							cx={xScale(hoverDate)}
							cy={yScale(point.total)}
							r="5"
							fill={colors[i]}
							opacity={
								year.year === today.getFullYear()
								&& hoverDate > todayMarker
								? 0
								: 1
							}
						/>

					{/if}

				{/each}

			{/if}



			<!-- Interaction layer -->

			<rect
				width={width}
				height={chartHeight}
				fill="transparent"
				onmousemove={handleMouseMove}
				onmouseenter={handleMouseEnter}
				onmouseleave={handleMouseLeave}
			/>


		</g>

	</svg>


</div>



<style>

.chart-container {

	width:100%;

	position:relative;

}



svg {

	display:block;

	width:100%;

}



.tooltip {

	position:absolute;

	top:20px;

	left:95px;

	padding:10px 15px;

	border-radius:10px;

	background:white;

	border:1px solid #ddd;

	box-shadow:0 2px 8px rgba(0,0,0,0.15);

	font-size:14px;

	pointer-events:none;

	z-index:2;

}



.tooltip-row {

	display:flex;

	align-items:center;

	gap:8px;

	margin-top:5px;

}



.dot {

	height:12px;

	width:12px;

	border-radius:50%;

	display:inline-block;

	flex-shrink:0;

}

</style>