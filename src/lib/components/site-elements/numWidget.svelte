<script>
	let { rows = [] } = $props();


	let stats = $derived.by(() => {

		const today = new Date();

		const currentYear = today.getFullYear();
		const previousYear = currentYear - 1;

		const cutoffPreviousYear = new Date(
			previousYear,
			today.getMonth(),
			today.getDate()
		);


		const currentCount = rows.filter(d => {

			if (!d.paymentDate) return false;

			return (
				d.paymentDate.getFullYear() === currentYear &&
				d.paymentDate <= today
			);

		}).length;



		const previousCount = rows.filter(d => {

			if (!d.paymentDate) return false;

			return (
				d.paymentDate.getFullYear() === previousYear &&
				d.paymentDate <= cutoffPreviousYear
			);

		}).length;



		const change = previousCount === 0
			? 0
			: ((currentCount - previousCount) / previousCount) * 100;



		return {
			currentYear,
			currentCount,
			previousCount,
			change
		};

	});


	function formatChange(value) {

		if (value > 0) {
			return `↑ ${value.toFixed(1)}%`;
		}

		if (value < 0) {
			return `↓ ${Math.abs(value).toFixed(1)}%`;
		}

		return `0%`;

	}

</script>


<div class="widget">

	<h2>
		Employee Reimbursements
	</h2>


	<div class="main-number">
		{stats.currentCount.toLocaleString()}
	</div>


	<div class="subtitle">
		paid in {stats.currentYear} YTD
	</div>



	<div class="comparison">

		{stats.previousCount.toLocaleString()}
		paid at this point in {stats.currentYear - 1}


		<span
			class:positive={stats.change > 0}
			class:negative={stats.change < 0}
		>
			({formatChange(stats.change)})
		</span>

	</div>


</div>



<style>

.widget {

	background:white;

	border-radius:20px;

	padding:25px;

	box-shadow:0 4px 12px rgba(0,0,0,0.08);

}


h2 {

	margin:0 0 15px 0;

	font-size:18px;

	font-weight:600;

}


.main-number {

	font-size:52px;

	font-weight:700;

	line-height:1;

}


.subtitle {

	color:#666;

	margin-top:8px;

	font-size:16px;

}



.comparison {

	margin-top:25px;

	color:#555;

	font-size:16px;

}



.positive {

	color:#2ca02c;

	font-weight:600;

	margin-left:5px;

}



.negative {

	color:#d62728;

	font-weight:600;

	margin-left:5px;

}

</style>