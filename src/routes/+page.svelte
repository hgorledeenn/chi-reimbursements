<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';

	import Header from '$lib/components/site-elements/header.svelte';
	import DataTable from '$lib/components/site-elements/dataTable.svelte';
	import AmountWidget from '$lib/components/site-elements/amtWidget.svelte';
	import NumberWidget from '$lib/components/site-elements/numWidget.svelte';
	import ToDo from '$lib/components/site-elements/toDo.svelte';

	import '../style.css';

	let rawData = $state([]);

	onMount(async () => {
		rawData = await d3.csv('/data/employee_reimbursements.csv', (d) => ({
			voucherNumber: d.voucher_number,
			amount: +d.amount,
			paymentDate: d3.timeParse('%Y-%m-%d')(d.payment_date),
			vendorName: d.vendor_name,
			description: d.description,
			department: d.department
		}));
	});

	let byDepartment = $derived.by(() => {
		if (rawData.length === 0) return [];

		return Array.from(
			d3.rollup(
				rawData,
				(v) => ({
					totalAmount: d3.sum(v, (d) => d.amount),
					totalReimbursements: v.length,
					mostRecentPayment: d3.max(v, (d) => d.paymentDate)
				}),
				(d) => d.department
			),
			([department, values]) => ({
				department,
				...values
			})
		).sort((a, b) => b.totalAmount - a.totalAmount);
	});

	// to change between recent and by department table views
	let tableView = $state('recent');

	function changeView(view) {
		tableView = view;
		currentPage = 1;

		if (view === 'recent') {
			sortState.column = 'paymentDate';
			sortState.direction = 'desc';
		}

		if (view === 'department') {
			sortState.column = 'totalAmount';
			sortState.direction = 'desc';
		}
	}

	let displayedData = $derived(tableView === 'recent' ? rawData : byDepartment);

	let sortedRows = $derived.by(() => {
		let data = [...displayedData];

		data.sort((a, b) => {
			let aValue = a[sortState.column];
			let bValue = b[sortState.column];

			if (aValue < bValue) {
				return sortState.direction === 'asc' ? -1 : 1;
			}

			if (aValue > bValue) {
				return sortState.direction === 'asc' ? 1 : -1;
			}

			return 0;
		});

		return data;
	});

	let sortState = $state({
		column: 'paymentDate',
		direction: 'desc'
	});

	function handleSort(column) {
		currentPage = 1;
		if (sortState.column === column) {
			// clicking the same column reverses direction
			sortState.direction = sortState.direction === 'desc' ? 'asc' : 'desc';
		} else {
			// clicking a new column starts descending
			sortState.column = column;
			sortState.direction = 'desc';
		}
	}

	// table pagination
	let currentPage = $state(1);
	const rowsPerPage = 500;

	let pagedData = $derived.by(() => {
		const start = (currentPage - 1) * rowsPerPage;
		const end = start + rowsPerPage;

		return sortedRows.slice(start, end);
	});

	let totalPages = $derived(Math.ceil(sortedRows.length / rowsPerPage));

	let numRows = $derived(sortedRows.length);

	function nextPage() {
		if (currentPage < totalPages) {
			currentPage++;
		}
	}

	function previousPage() {
		if (currentPage > 1) {
			currentPage--;
		}
	}
</script>

<div class="dashboard">
	<div class="header">
		<Header />
	</div>
	<div class="table">
		<DataTable
			rows={pagedData}
			view={tableView}
			onViewChange={changeView}
			{sortState}
			onSort={handleSort}
			{currentPage}
			{totalPages}
			onNextPage={nextPage}
			onPreviousPage={previousPage}
			{rowsPerPage}
			{numRows}
		/>
	</div>
	<div class="widgets">
		<div class="amount-widget">
			<AmountWidget rows={rawData} />
		</div>

		<div class="number-widget">
			<NumberWidget rows={rawData} />
		</div>
	</div>
	<div class="to-do">
		<ToDo />
	</div>
</div>

<style>
	.widgets {
		display: grid;
		grid-template-columns: 2fr 1fr;
		gap: 20px;
		margin-top: 30px;
		margin-bottom: 100px;
	}

	.amount-widget {
		width: 100%;
		background: #ffffff;
		border-radius: 16px;
		padding: 20px;
		box-shadow:
			0 1px 3px rgba(0, 0, 0, 0.08),
			0 8px 20px rgba(0, 0, 0, 0.04);

		border: 1px solid #e8e8e8;

		box-sizing: border-box;
	}

	.number-widget {
		width: 100%;
	}

	.dashboard {
		max-width: 1200px;
		align-content: center;
		margin: auto;
	}
</style>
