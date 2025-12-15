<script lang="ts">
	import { Button } from "$lib/components/ui/button";
	import { Input } from "$lib/components/ui/input";
	import { Separator } from "$lib/components/ui/separator";
	import * as Select from "$lib/components/ui/select";
	import * as Tooltip from "$lib/components/ui/tooltip";
	import * as Sidebar from "$lib/components/ui/sidebar";
	import * as Chart from "$lib/components/ui/chart";
	import { AreaChart } from "layerchart";
	import { scaleLinear } from "d3-scale";
	import { curveNatural } from "d3-shape";

	// Currency configuration
	type Currency = {
		code: string;
		symbol: string;
		name: string;
	};

	const currencies: Currency[] = [
		{ code: "USD", symbol: "$", name: "US Dollar" },
		{ code: "EUR", symbol: "€", name: "Euro" },
		{ code: "GBP", symbol: "£", name: "British Pound" },
		{ code: "CHF", symbol: "CHF", name: "Swiss Franc" },
	];

	let selectedCurrencyCode = "USD";
	
	$: selectedCurrency = currencies.find(c => c.code === selectedCurrencyCode) || currencies[0];

	// Default values (immutable)
	const DEFAULTS: {
		loanAmount: number;
		interestRate: number;
		loanTermYears: number;
		downPayment: number;
		deferralMonths: number;
		deferralType: 'complete' | 'partial';
		investmentRate: number;
		investmentVolatility: number;
		analysisYears: number;
	} = {
		loanAmount: 200000,
		interestRate: 4.5,
		loanTermYears: 30,
		downPayment: 40000,
		deferralMonths: 12,
		deferralType: 'complete',
		investmentRate: 7.0,
		investmentVolatility: 3,
		analysisYears: 40,
	};

	// Input values
	let loanAmount = DEFAULTS.loanAmount;
	let interestRate = DEFAULTS.interestRate;
	let loanTermYears = DEFAULTS.loanTermYears;
	let downPayment = DEFAULTS.downPayment;
	let deferralMonths = DEFAULTS.deferralMonths;
	let deferralType: 'complete' | 'partial' = DEFAULTS.deferralType;

	// Investment parameters
	let investmentRate = DEFAULTS.investmentRate;
	let investmentVolatility = DEFAULTS.investmentVolatility;
	let analysisYears = DEFAULTS.analysisYears;

	// Clamp values to respect min/max constraints
	$: loanAmount = Math.max(0, Math.min(10000000, loanAmount));
	$: downPayment = Math.max(0, Math.min(loanAmount, downPayment));
	$: interestRate = Math.max(0, Math.min(20, interestRate));
	$: loanTermYears = Math.max(1, Math.min(40, loanTermYears));
	$: deferralMonths = Math.max(0, Math.min(60, deferralMonths));
	$: investmentRate = Math.max(0, Math.min(30, investmentRate));
	$: investmentVolatility = Math.max(0, Math.min(100, investmentVolatility));
	$: analysisYears = Math.max(loanTermYears, Math.min(60, analysisYears));

	// Risk level classification
	$: riskLevel = investmentVolatility < 10 ? 'Low' : investmentVolatility < 20 ? 'Medium' : 'High';
	$: riskColor = investmentVolatility < 10 ? 'text-green-600' : investmentVolatility < 20 ? 'text-yellow-600' : 'text-red-600';

	// Reactive calculations
	// For complete deferral: interest accrues and is added to principal (capitalized interest)
	// For partial deferral: interest is paid monthly during deferral, principal stays the same
	$: principalAfterDeferral = (() => {
		let balance = loanAmount - downPayment;
		if (deferralType === 'complete') {
			// Interest capitalizes during deferral
			for (let month = 0; month < deferralMonths; month++) {
				balance *= (1 + monthlyRate);
			}
		}
		// For partial deferral, balance stays the same (interest paid separately)
		return balance;
	})();
	// Calculate total interest paid during partial deferral period
	$: deferralInterestPaid = deferralType === 'partial' ? (loanAmount - downPayment) * monthlyRate * deferralMonths : 0;
	$: principal = loanAmount - downPayment;
	$: monthlyRate = interestRate / 100 / 12;
	$: numberOfPayments = loanTermYears * 12;
	$: monthlyPayment =
		principalAfterDeferral > 0 && monthlyRate > 0
			? (principalAfterDeferral * (monthlyRate * Math.pow(1 + monthlyRate, numberOfPayments))) /
				(Math.pow(1 + monthlyRate, numberOfPayments) - 1)
			: 0;
	$: totalPayment = monthlyPayment * numberOfPayments;
	$: totalInterest = totalPayment - principal;
	$: totalCost = totalPayment + downPayment;

	// Investment calculations
	// Calculate year-by-year investment growth with compound interest
	type YearlyData = {
		year: number;
		investmentValue: number;
		loanBalance: number;
		totalPaid: number;
		netPosition: number;
		isLoanActive: boolean;
		isDeferralPeriod: boolean;
	};

	// Total loan duration including deferral period
	$: totalLoanMonths = deferralMonths + numberOfPayments;
	$: totalLoanYears = Math.ceil(totalLoanMonths / 12);

	$: yearlyData = (() => {
		const data: YearlyData[] = [];
		let investmentValue = downPayment; // Start with down payment invested instead
		let loanBalance = principal;
		let totalPaid = downPayment;
		const monthlyInvestmentRate = investmentRate / 100 / 12;
		let totalMonthsElapsed = 0;
		
		for (let year = 1; year <= analysisYears; year++) {
			// Monthly calculations for the year
			for (let month = 0; month < 12; month++) {
				totalMonthsElapsed++;
				const isDeferralMonth = totalMonthsElapsed <= deferralMonths;
				const isRepaymentMonth = totalMonthsElapsed > deferralMonths && totalMonthsElapsed <= totalLoanMonths;
				
				// Investment grows with compound interest
				investmentValue *= (1 + monthlyInvestmentRate);
				
				if (isDeferralMonth) {
					if (deferralType === 'complete') {
						// Complete deferral: interest accrues on loan balance (capitalized)
						loanBalance *= (1 + monthlyRate);
						// No payment during deferral
					} else {
						// Partial deferral: only interest is paid monthly, principal stays the same
						const interestOnly = loanBalance * monthlyRate;
						investmentValue += interestOnly; // Track what could have been invested
						totalPaid += interestOnly;
					}
				} else if (isRepaymentMonth) {
					// During repayment: add monthly payment to investment (what you could have invested)
					investmentValue += monthlyPayment;
					totalPaid += monthlyPayment;
					
					// Calculate remaining loan balance
					const interestPayment = loanBalance * monthlyRate;
					const principalPayment = monthlyPayment - interestPayment;
					loanBalance = Math.max(0, loanBalance - principalPayment);
				}
			}
			
			const isLoanActive = totalMonthsElapsed <= totalLoanMonths;
			const isDeferralPeriod = totalMonthsElapsed <= deferralMonths;
			
			data.push({
				year,
				investmentValue,
				loanBalance: isLoanActive ? loanBalance : 0,
				totalPaid,
				netPosition: investmentValue - totalPaid,
				isLoanActive,
				isDeferralPeriod
			});
		}
		return data;
	})();

	// Key metrics
	$: finalInvestmentValue = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].investmentValue : 0;
	$: finalNetPosition = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].netPosition : 0;
	$: isPositiveROI = finalNetPosition > 0;
	// Break-even: year when investment returns compensate for total interest paid
	// Investment returns = investmentValue - totalPaid (what you put in)
	// This should equal or exceed totalInterest (the cost of borrowing)
	$: breakEvenYear = yearlyData.findIndex(d => {
		const investmentReturns = d.investmentValue - d.totalPaid;
		return investmentReturns >= totalInterest;
	}) + 1 || null;
	
	// Volatility-adjusted returns (using simplified model)
	$: bestCaseReturn = investmentRate + investmentVolatility;
	$: worstCaseReturn = Math.max(0, investmentRate - investmentVolatility);
	
	// Calculate final values with volatility scenarios
	$: scenarioCalculation = (rate: number) => {
		let value = downPayment;
		const monthlyRate = rate / 100 / 12;
		for (let year = 1; year <= analysisYears; year++) {
			for (let month = 0; month < 12; month++) {
				value *= (1 + monthlyRate);
				if (year <= loanTermYears) {
					value += monthlyPayment;
				}
			}
		}
		return value;
	};
	
	$: bestCaseValue = scenarioCalculation(bestCaseReturn);
	$: worstCaseValue = scenarioCalculation(worstCaseReturn);

	// Reactive currency formatter
	$: formatCurrency = (value: number): string => {
		// Special handling for JPY (no decimals)
		const decimals = selectedCurrency.code === "JPY" ? 0 : 2;
		return new Intl.NumberFormat("en-US", {
			style: "currency",
			currency: selectedCurrency.code,
			minimumFractionDigits: decimals,
			maximumFractionDigits: decimals,
		}).format(value);
	};

	// Format percentage
	function formatPercent(value: number): string {
		return value.toFixed(2) + "%";
	}

	// Chart config for shadcn/layerchart
	const chartConfig = {
		investmentValue: {
			label: "Investment Value",
			color: "#22c55e"
		},
		totalPaid: {
			label: "Total Paid",
			color: "#f97316"
		},
		netPosition: {
			label: "Net Position",
			color: "#3b82f6"
		}
	} satisfies Chart.ChartConfig;

	// Scenarios chart config
	const scenariosChartConfig = {
		worst: {
			label: "Worst Case",
			color: "#ef4444"
		},
		expected: {
			label: "Expected",
			color: "#3b82f6"
		},
		best: {
			label: "Best Case",
			color: "#22c55e"
		}
	} satisfies Chart.ChartConfig;

	// Reactive scenario data - yearly values for all three scenarios
	$: scenarioYearlyData = (() => {
		const data: { year: number; worst: number; expected: number; best: number }[] = [];
		let worstValue = downPayment;
		let expectedValue = downPayment;
		let bestValue = downPayment;
		const worstMonthlyRate = worstCaseReturn / 100 / 12;
		const expectedMonthlyRate = investmentRate / 100 / 12;
		const bestMonthlyRate = bestCaseReturn / 100 / 12;
		
		for (let year = 1; year <= analysisYears; year++) {
			for (let month = 0; month < 12; month++) {
				worstValue *= (1 + worstMonthlyRate);
				expectedValue *= (1 + expectedMonthlyRate);
				bestValue *= (1 + bestMonthlyRate);
				if (year <= loanTermYears) {
					worstValue += monthlyPayment;
					expectedValue += monthlyPayment;
					bestValue += monthlyPayment;
				}
			}
			data.push({ year, worst: worstValue, expected: expectedValue, best: bestValue });
		}
		return data;
	})();

	// Keep lastYearData for metrics
	$: lastYearData = yearlyData[yearlyData.length - 1];

	function resetValues() {
		loanAmount = DEFAULTS.loanAmount;
		interestRate = DEFAULTS.interestRate;
		loanTermYears = DEFAULTS.loanTermYears;
		downPayment = DEFAULTS.downPayment;
		deferralMonths = DEFAULTS.deferralMonths;
		deferralType = DEFAULTS.deferralType;
		investmentRate = DEFAULTS.investmentRate;
		investmentVolatility = DEFAULTS.investmentVolatility;
		analysisYears = DEFAULTS.analysisYears;
	}
</script>

<Tooltip.Provider>
	<Sidebar.Provider>
		<Sidebar.Root side="left" collapsible="icon">
			<Sidebar.Header class="p-4 border-b">
				<div class="flex items-center justify-between gap-2">
					<h2 class="text-lg font-semibold group-data-[collapsible=icon]:hidden">Parameters</h2>
					<Sidebar.Trigger />
				</div>
			</Sidebar.Header>
			<Sidebar.Content class="p-4 space-y-6 group-data-[collapsible=icon]:hidden">
				<!-- Loan Parameters -->
				<div class="space-y-4">
					<h3 class="text-sm font-semibold text-muted-foreground uppercase tracking-wider">Loan Parameters</h3>
					
					<!-- Loan Amount -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="loan-amount" class="text-sm font-medium cursor-help">Loan Amount ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>The total amount you want to borrow</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="loan-amount"
							type="number"
							bind:value={loanAmount}
							min="0"
							max="10000000"
							step="1000"
						/>
						<p class="text-xs text-muted-foreground">{formatCurrency(loanAmount)}</p>
					</div>

					<!-- Down Payment -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="down-payment" class="text-sm font-medium cursor-help">Down Payment ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Initial payment reducing the borrowed amount</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="down-payment"
							type="number"
							bind:value={downPayment}
							min="0"
							max={loanAmount}
							step="1000"
						/>
						<p class="text-xs text-muted-foreground">
							{formatCurrency(downPayment)} ({((downPayment / loanAmount) * 100).toFixed(1)}%)
						</p>
					</div>

					<!-- Interest Rate -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="interest-rate" class="text-sm font-medium cursor-help">Annual Interest Rate (%) ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>The yearly interest rate charged on your loan</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="interest-rate"
							type="number"
							bind:value={interestRate}
							min="0"
							max="20"
							step="0.1"
						/>
						<p class="text-xs text-muted-foreground">{formatPercent(interestRate)}</p>
					</div>

					<!-- Loan Term -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="loan-term" class="text-sm font-medium cursor-help">Loan Term (Years) ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Duration over which you'll repay the loan</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="loan-term"
							type="number"
							bind:value={loanTermYears}
							min="1"
							max="40"
							step="1"
						/>
						<p class="text-xs text-muted-foreground">{numberOfPayments} monthly payments</p>
					</div>

					<!-- Deferral Period -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="deferral-months" class="text-sm font-medium cursor-help">Deferral Period (Months) ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Grace period before loan repayment begins</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="deferral-months"
							type="number"
							bind:value={deferralMonths}
							min="0"
							max="60"
							step="1"
						/>
						{#if deferralMonths > 0}
							<div class="space-y-2">
								<label for="deferral-type" class="text-sm font-medium">Deferral Type</label>
								<Select.Root type="single" bind:value={deferralType}>
									<Select.Trigger id="deferral-type" class="w-full">
										{deferralType === 'complete' ? 'Complete (no payment)' : 'Partial (interest only)'}
									</Select.Trigger>
									<Select.Content>
										<Select.Item value="complete">Complete (no payment)</Select.Item>
										<Select.Item value="partial">Partial (interest only)</Select.Item>
									</Select.Content>
								</Select.Root>
								<p class="text-xs text-muted-foreground">
									{#if deferralType === 'complete'}
										Interest capitalizes - no payments for {deferralMonths} months
									{:else}
										Interest-only payments of {selectedCurrency.symbol}{((loanAmount - downPayment) * monthlyRate).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })}/month
									{/if}
								</p>
							</div>
						{:else}
							<p class="text-xs text-muted-foreground">No deferral - repayment starts immediately</p>
						{/if}
					</div>
				</div>

				<Separator />

				<!-- Investment Parameters -->
				<div class="space-y-4">
					<h3 class="text-sm font-semibold text-muted-foreground uppercase tracking-wider">Investment Parameters</h3>
					
					<!-- Expected Return Rate -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="investment-rate" class="text-sm font-medium cursor-help">Expected Annual Return (%) ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Average yearly return on your investments (e.g., S&P 500 ≈ 7-10%)</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="investment-rate"
							type="number"
							bind:value={investmentRate}
							min="0"
							max="30"
							step="0.1"
						/>
						<p class="text-xs text-muted-foreground">{formatPercent(investmentRate)}</p>
					</div>

					<!-- Volatility -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="volatility" class="text-sm font-medium cursor-help">Volatility (Risk %) ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Standard deviation of returns - higher means more risk (S&P 500 ≈ 15-20%)</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="volatility"
							type="number"
							bind:value={investmentVolatility}
							min="0"
							max="100"
							step="1"
						/>
						<p class="text-xs {riskColor}">{riskLevel} Risk</p>
					</div>

					<!-- Analysis Period -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="analysis-years" class="text-sm font-medium cursor-help">Analysis Period (Years) ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Total time to simulate, including years after the loan ends</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="analysis-years"
							type="number"
							bind:value={analysisYears}
							min={loanTermYears}
							max="60"
							step="1"
						/>
						<p class="text-xs text-muted-foreground">{analysisYears - loanTermYears} years after loan ends</p>
					</div>
				</div>
			</Sidebar.Content>
			<Sidebar.Footer class="p-4 border-t group-data-[collapsible=icon]:hidden">
				<Button onclick={resetValues} variant="outline" class="w-full">
					Reset to Defaults
				</Button>
			</Sidebar.Footer>
		</Sidebar.Root>

		<Sidebar.Inset>
			<div class="min-h-screen bg-background p-4 md:p-6 lg:p-8">
				<div class="space-y-4">
					<!-- Header -->
					<div class="space-y-2">
						<div class="flex items-start justify-between gap-4">
							<div>
								<h1 class="text-3xl font-bold tracking-tight md:text-4xl">
									Loan Profitability Simulator
								</h1>
								<p class="text-muted-foreground">
									Adjust the parameters in the sidebar to see how they affect your results.
								</p>
							</div>
							<!-- Currency Selector -->
							<div class="flex items-center gap-2">
								<span class="text-sm font-medium text-muted-foreground">Currency:</span>
								<Select.Root type="single" bind:value={selectedCurrencyCode}>
									<Select.Trigger class="w-30">
										{selectedCurrency.symbol} {selectedCurrency.code}
									</Select.Trigger>
									<Select.Content>
										{#each currencies as currency}
											<Select.Item value={currency.code}>
												{currency.symbol} {currency.code}
											</Select.Item>
										{/each}
									</Select.Content>
								</Select.Root>
							</div>
						</div>
					</div>

					<!-- Dashboard: Key Metrics First -->
					<!-- Top Row: Most Important Info -->
					<div class="grid gap-4 sm:grid-cols-2 lg:grid-cols-5 mb-4">
				<!-- Monthly Payment - Most Prominent -->
				<div class="sm:col-span-2 lg:col-span-2 rounded-lg border bg-linear-to-br from-primary/10 to-primary/5 p-6 shadow-sm">
					<p class="text-sm font-medium text-muted-foreground">Monthly Payment</p>
					<p class="text-4xl font-bold tracking-tight mt-2">
						{formatCurrency(monthlyPayment)}
					</p>
					<p class="text-xs text-muted-foreground mt-1">{numberOfPayments} payments over {loanTermYears} years</p>
				</div>

				<!-- ROI Status -->
				<div class="sm:col-span-2 lg:col-span-3 rounded-lg border p-6 shadow-sm {isPositiveROI ? 'bg-green-50 dark:bg-green-950 border-green-200 dark:border-green-800' : 'bg-red-50 dark:bg-red-950 border-red-200 dark:border-red-800'}">
					<p class="text-sm font-medium text-muted-foreground flex items-center gap-2">
						{isPositiveROI ? '📈' : '📉'}
						{isPositiveROI ? 'Positive ROI' : 'Negative ROI'}
					</p>
					<p class="text-4xl font-bold tracking-tight mt-2 {isPositiveROI ? 'text-green-700 dark:text-green-300' : 'text-red-700 dark:text-red-300'}">
						{formatCurrency(finalNetPosition)}
					</p>
					<p class="text-xs text-muted-foreground mt-1">
						After {analysisYears} years{#if breakEvenYear} · Break-even: Year {breakEvenYear}{/if}
					</p>
				</div>
			</div>

			<!-- Second Row: Summary Cards -->
			<div class="grid gap-3 grid-cols-2 lg:grid-cols-4 mb-4">
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Principal</p>
					<p class="text-lg font-semibold mt-1">{formatCurrency(principal)}</p>
				</div>
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Total Loan Interest</p>
					<p class="text-lg font-semibold text-chart-1 mt-1">{formatCurrency(totalInterest)}</p>
				</div>
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Investment Value</p>
					<p class="text-lg font-semibold text-green-600 mt-1">{formatCurrency(finalInvestmentValue)}</p>
				</div>
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Total Cost</p>
					<p class="text-lg font-semibold mt-1">{formatCurrency(totalCost)}</p>
				</div>
			</div>

			<!-- Main Dashboard Grid -->
			<div class="space-y-4">
				<!-- Graph and Analysis Side by Side -->
				<div class="grid gap-4 lg:grid-cols-2">
					<!-- Growth Chart (Left) -->
					<div class="rounded-lg border bg-card p-6 shadow-sm">
						<h3 class="mb-4 text-lg font-semibold">Growth Over Time</h3>
						<Chart.Container config={chartConfig} class="h-80 w-full overflow-hidden">
							<AreaChart
								legend
								data={yearlyData}
								x="year"
								xScale={scaleLinear()}
								yPadding={[0, 25]}
								padding={{ left: 50, right: 16, top: 16, bottom: 40 }}
								series={[
									{
										key: "investmentValue",
										label: chartConfig.investmentValue.label,
										color: chartConfig.investmentValue.color,
									},
									{
										key: "totalPaid",
										label: chartConfig.totalPaid.label,
										color: chartConfig.totalPaid.color,
									},
									{
										key: "netPosition",
										label: chartConfig.netPosition.label,
										color: chartConfig.netPosition.color,
									},
								]}
								props={{
									area: {
										curve: curveNatural,
										"fill-opacity": 0.4,
										line: { class: "stroke-1" },
										motion: "tween",
									},
									xAxis: {
										format: (v: number) => `Y${v}`,
									},
									yAxis: {
										format: (v: number) => {
											if (v >= 1000000) return `${(v / 1000000).toFixed(1)}M`;
											if (v >= 1000) return `${(v / 1000).toFixed(0)}K`;
											return v.toString();
										},
									},
								}}
							>
								{#snippet tooltip()}
									<Chart.Tooltip
										labelFormatter={(v: number) => `Year ${v}`}
										indicator="line"
									/>
								{/snippet}
							</AreaChart>
						</Chart.Container>
						
						{#if breakEvenYear}
							<p class="text-xs text-muted-foreground mt-2 text-center">
								<span class="inline-block w-2 h-2 bg-green-500 rounded-full mr-1"></span>
								Break-even at Year {breakEvenYear}
							</p>
						{/if}
					</div>

					<!-- Investment Analysis (Right) -->
					<div class="rounded-lg border bg-card p-4 shadow-sm">
						<h3 class="mb-3 text-base font-semibold">Volatility Scenarios</h3>
						<p class="text-xs text-muted-foreground mb-4">Final investment value after {analysisYears} years based on ±1 standard deviation</p>
						
						<Chart.Container config={scenariosChartConfig} class="h-64 w-full overflow-hidden">
							<AreaChart
								legend
								data={scenarioYearlyData}
								x="year"
								xScale={scaleLinear()}
								yPadding={[0, 25]}
								padding={{ left: 60, right: 16, top: 16, bottom: 40 }}
								series={[
									{ key: "worst", label: `Pessimistic (${formatPercent(worstCaseReturn)})`, color: scenariosChartConfig.worst.color },
									{ key: "expected", label: `Expected (${formatPercent(investmentRate)})`, color: scenariosChartConfig.expected.color },
									{ key: "best", label: `Optimistic (${formatPercent(bestCaseReturn)})`, color: scenariosChartConfig.best.color },
								]}
								props={{
									area: {
										curve: curveNatural,
										"fill-opacity": 0.15,
										line: { class: "stroke-2" },
										motion: "tween",
									},
									xAxis: {
										format: (v: number) => `Y${v}`,
									},
									yAxis: {
										format: (v: number) => {
											if (v >= 1000000) return `${(v / 1000000).toFixed(1)}M`;
											if (v >= 1000) return `${(v / 1000).toFixed(0)}K`;
											return v.toString();
										},
									},
								}}
							>
								{#snippet tooltip()}
									<Chart.Tooltip
										labelFormatter={(v: number) => `Year ${v}`}
										indicator="line"
									/>
								{/snippet}
							</AreaChart>
						</Chart.Container>
						
						<!-- Legend with rates -->
						<div class="grid grid-cols-3 gap-2 mt-4 text-xs text-center">
							<div class="p-2 rounded bg-red-50 dark:bg-red-950/50">
								<p class="font-medium text-red-700 dark:text-red-300">{formatPercent(worstCaseReturn)}</p>
								<p class="text-red-600 dark:text-red-400">{formatCurrency(worstCaseValue)}</p>
							</div>
							<div class="p-2 rounded bg-blue-50 dark:bg-blue-950/50">
								<p class="font-medium text-blue-700 dark:text-blue-300">{formatPercent(investmentRate)}</p>
								<p class="text-blue-600 dark:text-blue-400">{formatCurrency(finalInvestmentValue)}</p>
							</div>
							<div class="p-2 rounded bg-green-50 dark:bg-green-950/50">
								<p class="font-medium text-green-700 dark:text-green-300">{formatPercent(bestCaseReturn)}</p>
								<p class="text-green-600 dark:text-green-400">{formatCurrency(bestCaseValue)}</p>
							</div>
						</div>
					</div>
				</div>

				<!-- Yearly Breakdown Table -->
					<div class="rounded-lg border bg-card p-6 shadow-sm">
						<h3 class="mb-4 text-lg font-semibold">Compound Growth Over Time</h3>
						<div class="overflow-x-auto">
							<table class="w-full text-sm">
								<thead>
									<tr class="border-b">
										<th class="text-left py-2 px-2">Year</th>
										<th class="text-right py-2 px-2">Investment Value</th>
										<th class="text-right py-2 px-2">Total Paid</th>
										<th class="text-right py-2 px-2">Net Position</th>
										<th class="text-center py-2 px-2">Status</th>
									</tr>
								</thead>
								<tbody>
									{#each yearlyData.filter((_, i) => i % 5 === 4 || i === 0 || i === yearlyData.length - 1 || i === loanTermYears - 1) as data}
										<tr class="border-b border-border/50 {data.year === loanTermYears ? 'bg-primary/5' : ''}">
											<td class="py-2 px-2 font-medium">
												{data.year}
												{#if data.year === loanTermYears}
													<span class="text-xs text-primary ml-1">(loan ends)</span>
												{/if}
											</td>
											<td class="text-right py-2 px-2">{formatCurrency(data.investmentValue)}</td>
											<td class="text-right py-2 px-2">{formatCurrency(data.totalPaid)}</td>
											<td class="text-right py-2 px-2 {data.netPosition >= 0 ? 'text-green-600' : 'text-red-600'}">
												{formatCurrency(data.netPosition)}
											</td>
											<td class="text-center py-2 px-2">
												{#if data.isLoanActive}
													<span class="text-xs px-2 py-1 rounded-full bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-200">Paying</span>
												{:else}
													<span class="text-xs px-2 py-1 rounded-full bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-200">Growing</span>
												{/if}
											</td>
										</tr>
									{/each}
								</tbody>
							</table>
						</div>
						<p class="text-xs text-muted-foreground mt-3">
							Showing key years: Year 1, every 5 years, loan end (year {loanTermYears}), and final year ({analysisYears})
						</p>
					</div>
				</div>
			</div>
		</div>
		</Sidebar.Inset>
	</Sidebar.Provider>
</Tooltip.Provider>
