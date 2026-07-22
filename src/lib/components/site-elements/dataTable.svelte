<script>
	import * as d3 from 'd3';
	import { fade, fly } from 'svelte/transition';

	let {
		rows = [],
		view,
		onViewChange,
		sortState,
		onSort,
		currentPage,
		totalPages,
		onNextPage,
		onPreviousPage,
		rowsPerPage,
		numRows
	} = $props();

	let numResultsDisplay = $derived(
    	Math.min(currentPage * rowsPerPage, numRows)
	);

	const formatDate = d3.timeFormat('%b %d, %Y');

	// this function was written by chatGPT to help me format my money correctly because some instances of dollar amount < $1 were being displayed as "milli-dollars", (eg. $0.01 was $10.0m), which can be... confusing for the average viewer
	function formatMoney(value) {
		if (value >= 1_000_000) {
			return `$${(value / 1_000_000).toFixed(1)}M`;
		}

		if (value >= 1_000) {
			return `$${(value / 1_000).toFixed(1)}K`;
		}

		return `$${value.toFixed(2)}`;
	}

	const formatNumber = d3.format(',');

	const columns = {
		recent: [
			{ label: '#', key: 'index', width: '1%', sortable: false },
			{ label: 'Payment Date', key: 'paymentDate', width: '17%', sortable: true },
			{ label: 'Amount', key: 'amount', width: '15%', sortable: true },
			{ label: 'Department', key: 'department', width: '20%', sortable: true },
			{ label: 'Vendor Name', key: 'vendorName', width: '25%', sortable: true },
			{ label: 'Description', key: 'description', width: '20%', sortable: true }
		],

		department: [
			{ label: '#', key: 'index', width: '1%', sortable: false },
			{ label: 'Department', key: 'department', width: '30%', sortable: true },
			{ label: 'Total Amount', key: 'totalAmount', width: '', sortable: true },
			{ label: 'Reimbursements', key: 'totalReimbursements', width: '', sortable: true },
			{ label: 'Most Recent Payment', key: 'mostRecentPayment', width: '', sortable: true }
		]
	};
</script>

<div class="table-toolbar">
	<div></div>
	<div class="toggle-wrapper">
		<div class="toggle-highlight" class:department={view === 'department'}></div>

		<button class:active={view === 'recent'} onclick={() => onViewChange('recent')}>
			Most Recent
		</button>

		<button class:active={view === 'department'} onclick={() => onViewChange('department')}>
			By Department
		</button>
	</div>

	<div class="table-info">
		<p>
			Showing {currentPage * rowsPerPage - rowsPerPage + 1} - {numResultsDisplay} of {numRows}
			results
		</p>
	</div>
</div>

<div class="table-wrapper">
	{#if view === 'recent'}
		<table
			class="individual-reimbursements"
			in:fly={{ y: 50, duration: 500 }}
			out:fade={{ duration: 150 }}
		>
			<thead>
				<tr>
					{#each columns[view] as column}
						<th
							style="width: {column.width};"
							onclick={() => column.sortable && onSort(column.key)}
							class:sortable={column.sortable}
							class:not-sortable={!column.sortable}
						>
							{column.label}

							{#if sortState.column === column.key}
								{sortState.direction === 'desc' ? ' ↓' : ' ↑'}
							{/if}
						</th>
					{/each}
				</tr>
			</thead>
			<tbody>
				{#each rows as row, index}
					<tr>
						<td style="text-align: center;">{(currentPage - 1) * rowsPerPage + index + 1}</td>
						<td style="text-align: center;">{formatDate(new Date(row.paymentDate))}</td>
						<td class="column-amount">{formatMoney(row.amount)}</td>
						<td>{row.department}</td>
						<td>{row.vendorName}</td>
						<td>{row.description}</td>
					</tr>
				{/each}
			</tbody>
		</table>
	{:else if view === 'department'}
		<table
			class="individual-reimbursements"
			in:fly={{ y: 50, duration: 500 }}
			out:fade={{ duration: 150 }}
		>
			<thead>
				<tr>
					{#each columns[view] as column}
						<th
							style="width: {column.width};"
							onclick={() => column.sortable && onSort(column.key)}
							class:sortable={column.sortable}
							class:not-sortable={!column.sortable}
						>
							{column.label}

							{#if sortState.column === column.key}
								{sortState.direction === 'desc' ? ' ↓' : ' ↑'}
							{/if}
						</th>
					{/each}
				</tr>
			</thead>
			<tbody>
				{#each rows as row, index}
					<tr>
						<td style="text-align: center;">{(currentPage - 1) * rowsPerPage + index + 1}</td>
						<td>{row.department}</td>
						<td class="column-amount">{formatMoney(row.totalAmount)}</td>
						<td style="text-align:center;">{formatNumber(row.totalReimbursements)}</td>
						<td style="text-align:center;">
							{row.mostRecentPayment ? formatDate(row.mostRecentPayment) : 'No date'}
						</td>
					</tr>
				{/each}
			</tbody>
		</table>
	{/if}
</div>
{#if totalPages > 1}
	<div class="pagination">
		<button onclick={onPreviousPage} disabled={currentPage === 1}> ← Previous </button>

		<span>
			Page {currentPage} of {totalPages}
		</span>

		<button onclick={onNextPage} disabled={currentPage === totalPages}> Next → </button>
	</div>
{/if}

<style>

	.table-toolbar {
		display: grid;
		grid-template-columns: 1fr auto 1fr;
		align-items: center;
		margin: 20px 0;
	}

	.pagination {
		display: flex;
		justify-content: center;
		align-items: center;
		gap: 20px;

		margin-top: 20px;
	}

	.pagination button {
		padding: 8px 18px;
		border-radius: 8px;
		border: 1px solid #ccc;
		cursor: pointer;
	}

	.pagination button:disabled {
		opacity: 0.4;
		cursor: default;
	}

	.toggle-wrapper {
		justify-self: center;
		margin: 0;
		position: relative;
		display: flex;
		width: 320px;
		height: 48px;
		padding: 4px;

		background-color: #fff;
		border-radius: 999px;

		margin: 20px auto;
	}

	.table-info {
		justify-self: end;
	}

	.toggle-wrapper button {
		position: relative;
		z-index: 2;

		flex: 1;
		border: none;
		background: none;

		font-size: 16px;
		font-weight: 500;
		cursor: pointer;

		color: #555;

		transition: color 0.2s ease;
	}

	.toggle-wrapper button.active {
		color: white;
	}

	/* moving pill */
	.toggle-highlight {
		position: absolute;

		top: 4px;
		left: 4px;

		width: calc(50% - 4px);
		height: calc(100% - 8px);

		background-color: #41b6e6;
		border-radius: 999px;

		transition: transform 0.3s ease;
	}

	/* slide to right when department selected */
	.toggle-highlight.department {
		transform: translateX(100%);
	}

	.table-wrapper {
		height: 600px;
		overflow-y: auto;
		border: 1px solid #000;
		overflow-x: auto;
		border-radius: 10px;
	}

	table {
		width: 100%;
		border-collapse: collapse;
	}

	th {
		position: sticky;
		top: 0;
		background-color: #41b6e6;
		z-index: 1;
		color: #fff;
		padding: 20px;
		padding-top: 25px;
		padding-bottom: 25px;
		text-align: center;
		font-size: 20px;
		box-shadow: inset 0 -2px 0 #000;
	}

	td {
		padding: 10px;
		vertical-align: middle;
		color: #000;
	}

	.column-amount {
		text-align: center;
		/* font-weight: 300; */
		/* border-left: #a7a4a4 solid 1px;
        border-right: #a7a4a4 solid 1px; */
	}

	/* making the striped table rows */
	tr:nth-child(even) {
		background-color: #f2f2f2;
	}

	tr:nth-child(odd) {
		background-color: #ffffff;
	}

	/* incase I want some state changes on:hover */
	tr:hover td {
		background-color: #e40028;
		color: #fff;
	}

	.sortable:hover {
		background-color: #2e93bb;
	}
</style>
