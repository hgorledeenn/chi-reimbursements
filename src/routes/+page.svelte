<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';

    import Header from '$lib/components/site-elements/header.svelte'
    import DataTable from '$lib/components/site-elements/dataTable.svelte';

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
		if (sortState.column === column) {
			// clicking the same column reverses direction
			sortState.direction = sortState.direction === 'desc' ? 'asc' : 'desc';
		} else {
			// clicking a new column starts descending
			sortState.column = column;
			sortState.direction = 'desc';
		}
	}
</script>


<div class="dashboard">
    <div class="header">
        <Header />
    </div>
	<div class="table">
		<DataTable
			rows={sortedRows}
			view={tableView}
			onViewChange={changeView}
			{sortState}
			onSort={handleSort}
		/>
	</div>
</div>

<style>
	.dashboard {
		max-width: 1200px;
		align-content: center;
		margin: auto;
	}
</style>