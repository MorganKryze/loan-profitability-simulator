<script lang="ts">
	import { Button } from "$lib/components/ui/button";
	import { Input } from "$lib/components/ui/input";
	import { Separator } from "$lib/components/ui/separator";
	import * as Select from "$lib/components/ui/select";
	import * as Tooltip from "$lib/components/ui/tooltip";
	import * as Sidebar from "$lib/components/ui/sidebar";

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

	// Input values
	let loanAmount = 20000;
	let interestRate = 4.5;
	let loanTermYears = 30;
	let downPayment = 1000;
	let deferralMonths = 0; // Grace period before loan repayment starts
	let deferralType: 'complete' | 'partial' = 'complete'; // complete = interest capitalizes, partial = interest paid monthly

	// Investment parameters
	let investmentRate = 7.0; // Expected annual return rate
	let investmentVolatility = 15; // Standard deviation (risk measure)
	let analysisYears = 40; // Total years to analyze (including post-loan)

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

	// Chart calculations
	const chartWidth = 100;
	const chartHeight = 60;
	
	$: chartMaxValue = Math.max(...yearlyData.map(d => Math.max(d.investmentValue, d.totalPaid)));
	$: chartMinValue = Math.min(...yearlyData.map(d => d.netPosition), 0);
	$: chartValueRange = chartMaxValue - chartMinValue;
	
	$: scaleY = (value: number) => chartHeight - ((value - chartMinValue) / chartValueRange) * chartHeight;
	$: scaleX = (year: number) => (year / analysisYears) * chartWidth;
	
	$: investmentLine = yearlyData.map(d => `${scaleX(d.year)},${scaleY(d.investmentValue)}`).join(' ');
	$: totalPaidLine = yearlyData.map(d => `${scaleX(d.year)},${scaleY(d.totalPaid)}`).join(' ');
	$: netPositionLine = yearlyData.map(d => `${scaleX(d.year)},${scaleY(d.netPosition)}`).join(' ');
	$: netPositionPolygon = `${scaleX(0)},${scaleY(0)} ${netPositionLine} ${scaleX(analysisYears)},${scaleY(0)}`;
	$: zeroLineY = scaleY(0);
	$: loanEndX = scaleX(totalLoanYears);
	$: lastYearData = yearlyData[yearlyData.length - 1];

	function resetValues() {
		loanAmount = 200000;
		interestRate = 4.5;
		loanTermYears = 30;
		downPayment = 40000;
		deferralMonths = 0;
		deferralType = 'complete';
		investmentRate = 7.0;
		investmentVolatility = 15;
		analysisYears = 40;
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
						<label for="loan-amount" class="text-sm font-medium">Loan Amount</label>
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
						<label for="down-payment" class="text-sm font-medium">Down Payment</label>
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
						<label for="interest-rate" class="text-sm font-medium">Annual Interest Rate (%)</label>
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
						<label for="loan-term" class="text-sm font-medium">Loan Term (Years)</label>
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
						<label for="deferral-months" class="text-sm font-medium">Deferral Period (Months)</label>
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
						<label for="investment-rate" class="text-sm font-medium">Expected Annual Return (%)</label>
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
						<label for="volatility" class="text-sm font-medium">Volatility (Risk %)</label>
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
						<label for="analysis-years" class="text-sm font-medium">Analysis Period (Years)</label>
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
									<Select.Trigger class="w-[120px]">
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
						<div class="relative">
							<svg viewBox="0 0 {chartWidth} {chartHeight + 10}" class="w-full h-64" preserveAspectRatio="xMidYMid meet">
								<!-- Zero line -->
								<line 
									x1="0" 
									y1={zeroLineY} 
									x2={chartWidth} 
									y2={zeroLineY} 
									stroke="currentColor" 
									stroke-opacity="0.2" 
									stroke-dasharray="2,2"
								/>
								
								<!-- Loan end marker -->
								<line 
									x1={loanEndX} 
									y1="0" 
									x2={loanEndX} 
									y2={chartHeight} 
									stroke="currentColor" 
									stroke-opacity="0.3" 
									stroke-dasharray="3,3"
								/>
								<text 
									x={loanEndX} 
									y={chartHeight + 8} 
									font-size="3" 
									text-anchor="middle" 
									fill="currentColor" 
									opacity="0.5"
								>
									Loan ends
								</text>
								
								<!-- Total Paid line (orange) -->
								<polyline
									fill="none"
									stroke="#f97316"
									stroke-width="0.8"
									stroke-linecap="round"
									stroke-linejoin="round"
									points={totalPaidLine}
								/>
								
								<!-- Investment Value line (green) -->
								<polyline
									fill="none"
									stroke="#22c55e"
									stroke-width="0.8"
									stroke-linecap="round"
									stroke-linejoin="round"
									points={investmentLine}
								/>
								
								<!-- Net Position area fill -->
								<polygon
									fill={isPositiveROI ? 'rgba(34, 197, 94, 0.1)' : 'rgba(239, 68, 68, 0.1)'}
									points={netPositionPolygon}
								/>
								
								<!-- Net Position line -->
								<polyline
									fill="none"
									stroke={isPositiveROI ? '#22c55e' : '#ef4444'}
									stroke-width="0.6"
									stroke-linecap="round"
									stroke-linejoin="round"
									stroke-dasharray="2,1"
									points={netPositionLine}
								/>
								
								<!-- Break-even point marker -->
								{#if breakEvenYear}
									<circle 
										cx={scaleX(breakEvenYear)} 
										cy={zeroLineY} 
										r="1.5" 
										fill="#22c55e"
									/>
								{/if}
								
								<!-- End point markers -->
								<circle 
									cx={scaleX(analysisYears)} 
									cy={scaleY(lastYearData.investmentValue)} 
									r="1.2" 
									fill="#22c55e"
								/>
								<circle 
									cx={scaleX(analysisYears)} 
									cy={scaleY(lastYearData.totalPaid)} 
									r="1.2" 
									fill="#f97316"
								/>
							</svg>
							
							<!-- Legend -->
							<div class="flex flex-wrap justify-center gap-4 mt-4 text-xs">
								<div class="flex items-center gap-1.5">
									<div class="w-3 h-0.5 bg-green-500 rounded"></div>
									<span class="text-muted-foreground">Investment Value</span>
								</div>
								<div class="flex items-center gap-1.5">
									<div class="w-3 h-0.5 bg-orange-500 rounded"></div>
									<span class="text-muted-foreground">Total Paid</span>
								</div>
								<div class="flex items-center gap-1.5">
									<div class="w-3 h-0.5 rounded" style="background: {isPositiveROI ? '#22c55e' : '#ef4444'}; opacity: 0.5;"></div>
									<span class="text-muted-foreground">Net Position</span>
								</div>
								{#if breakEvenYear}
									<div class="flex items-center gap-1.5">
										<div class="w-2 h-2 bg-green-500 rounded-full"></div>
										<span class="text-muted-foreground">Break-even (Year {breakEvenYear})</span>
									</div>
								{/if}
							</div>
						</div>
					</div>

					<!-- Investment Analysis (Right) -->
					<div class="rounded-lg border bg-card p-4 shadow-sm">
						<h3 class="mb-3 text-base font-semibold">Investment Analysis</h3>
						
						<!-- Volatility Scenarios -->
						<h4 class="font-medium mb-3">Volatility Scenarios</h4>
						<div class="space-y-3">
							<div class="flex justify-between items-center p-3 rounded-lg bg-green-50 dark:bg-green-950/50">
								<div>
									<p class="text-sm font-medium text-green-700 dark:text-green-300">Best Case ({formatPercent(bestCaseReturn)})</p>
									<p class="text-xs text-green-600 dark:text-green-400">+1 standard deviation</p>
								</div>
								<p class="text-lg font-semibold text-green-700 dark:text-green-300">{formatCurrency(bestCaseValue)}</p>
							</div>
							<div class="flex justify-between items-center p-3 rounded-lg bg-secondary/50">
								<div>
									<p class="text-sm font-medium">Expected ({formatPercent(investmentRate)})</p>
									<p class="text-xs text-muted-foreground">Base scenario</p>
								</div>
								<p class="text-lg font-semibold">{formatCurrency(finalInvestmentValue)}</p>
							</div>
							<div class="flex justify-between items-center p-3 rounded-lg bg-red-50 dark:bg-red-950/50">
								<div>
									<p class="text-sm font-medium text-red-700 dark:text-red-300">Worst Case ({formatPercent(worstCaseReturn)})</p>
									<p class="text-xs text-red-600 dark:text-red-400">-1 standard deviation</p>
								</div>
								<p class="text-lg font-semibold text-red-700 dark:text-red-300">{formatCurrency(worstCaseValue)}</p>
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
