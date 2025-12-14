<script lang="ts">
	import { Button } from "$lib/components/ui/button";
	import { Input } from "$lib/components/ui/input";
	import { Separator } from "$lib/components/ui/separator";
	import * as Select from "$lib/components/ui/select";
	import * as Tooltip from "$lib/components/ui/tooltip";

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

	// Investment parameters
	let investmentRate = 7.0; // Expected annual return rate
	let investmentVolatility = 15; // Standard deviation (risk measure)
	let analysisYears = 40; // Total years to analyze (including post-loan)

	// Risk level classification
	$: riskLevel = investmentVolatility < 10 ? 'Low' : investmentVolatility < 20 ? 'Medium' : 'High';
	$: riskColor = investmentVolatility < 10 ? 'text-green-600' : investmentVolatility < 20 ? 'text-yellow-600' : 'text-red-600';

	// Reactive calculations
	$: principal = loanAmount - downPayment;
	$: monthlyRate = interestRate / 100 / 12;
	$: numberOfPayments = loanTermYears * 12;
	$: monthlyPayment =
		principal > 0 && monthlyRate > 0
			? (principal * (monthlyRate * Math.pow(1 + monthlyRate, numberOfPayments))) /
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
	};

	$: yearlyData = (() => {
		const data: YearlyData[] = [];
		let investmentValue = downPayment; // Start with down payment invested instead
		let loanBalance = principal;
		let totalPaid = downPayment;
		const monthlyInvestmentRate = investmentRate / 100 / 12;
		
		for (let year = 1; year <= analysisYears; year++) {
			const isLoanActive = year <= loanTermYears;
			
			// Monthly calculations for the year
			for (let month = 0; month < 12; month++) {
				// Investment grows with compound interest
				investmentValue *= (1 + monthlyInvestmentRate);
				
				if (isLoanActive) {
					// During loan: add monthly payment to investment (what you could have invested)
					investmentValue += monthlyPayment;
					totalPaid += monthlyPayment;
					
					// Calculate remaining loan balance
					const interestPayment = loanBalance * monthlyRate;
					const principalPayment = monthlyPayment - interestPayment;
					loanBalance = Math.max(0, loanBalance - principalPayment);
				}
			}
			
			data.push({
				year,
				investmentValue,
				loanBalance: isLoanActive ? loanBalance : 0,
				totalPaid,
				netPosition: investmentValue - totalPaid,
				isLoanActive
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
	$: loanEndX = scaleX(loanTermYears);
	$: lastYearData = yearlyData[yearlyData.length - 1];

	function resetValues() {
		loanAmount = 200000;
		interestRate = 4.5;
		loanTermYears = 30;
		downPayment = 40000;
		investmentRate = 7.0;
		investmentVolatility = 15;
		analysisYears = 40;
	}
</script>

<Tooltip.Provider>
	<div class="min-h-screen bg-background p-4 md:p-6 lg:p-8">
		<div class="space-y-4">
			<!-- Header -->
			<div class="space-y-2">
				<div class="flex items-start justify-between gap-4">
					<div class="flex-1">
						<h1 class="text-3xl font-bold tracking-tight md:text-4xl">
							Loan Profitability Simulator
						</h1>
						<p class="text-muted-foreground">
							Adjust the parameters below to see how they affect your monthly payments and total
							loan cost.
						</p>
					</div>
					<!-- Currency Selector & Reset Button -->
					<div class="flex items-center gap-3">
						<div class="flex items-center gap-2">
							<span class="text-sm font-medium text-muted-foreground">Currency:</span>
							<select 
								bind:value={selectedCurrencyCode}
								class="h-9 rounded-md border border-input bg-background px-3 py-1 text-sm shadow-sm focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring"
							>
								{#each currencies as currency}
									<option value={currency.code}>
										{currency.symbol} {currency.code}
									</option>
								{/each}
							</select>
						</div>
						<Button onclick={resetValues} variant="outline" size="sm">
							Reset
						</Button>
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
				<div class="sm:col-span-2 lg:col-span-3 rounded-lg border p-4 shadow-sm {isPositiveROI ? 'bg-green-50 dark:bg-green-950 border-green-200 dark:border-green-800' : 'bg-red-50 dark:bg-red-950 border-red-200 dark:border-red-800'}">
					<div class="flex items-center gap-3">
						<div class="text-3xl">
							{isPositiveROI ? '📈' : '📉'}
						</div>
						<div class="flex-1">
							<p class="font-semibold {isPositiveROI ? 'text-green-700 dark:text-green-300' : 'text-red-700 dark:text-red-300'}">
								{isPositiveROI ? 'Positive ROI' : 'Negative ROI'}
							</p>
							<p class="text-sm {isPositiveROI ? 'text-green-600 dark:text-green-400' : 'text-red-600 dark:text-red-400'}">
								{formatCurrency(finalNetPosition)} after {analysisYears} years
							</p>
							{#if breakEvenYear}
								<p class="text-xs text-muted-foreground">Break-even: Year {breakEvenYear}</p>
							{/if}
						</div>
					</div>
				</div>
			</div>

			<!-- Second Row: Summary Cards -->
			<div class="grid gap-3 grid-cols-2 lg:grid-cols-4 mb-4">
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Principal</p>
					<p class="text-lg font-semibold mt-1">{formatCurrency(principal)}</p>
				</div>
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Total Interest</p>
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
			<div class="grid gap-4 lg:grid-cols-12">
				<!-- LEFT: Input Parameters (Compact) -->
				<div class="lg:col-span-4 space-y-4">
					<!-- Loan Parameters -->
					<div class="rounded-lg border bg-card p-4 shadow-sm">
						<h2 class="mb-3 text-base font-semibold">Loan Parameters</h2>

						<!-- Loan Amount -->
						<div class="space-y-1.5">
							<div class="flex items-center gap-2">
								<label for="loan-amount" class="text-sm font-medium">Loan Amount</label>
								<Tooltip.Root>
									<Tooltip.Trigger>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											width="16"
											height="16"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											class="text-muted-foreground"
										>
											<circle cx="12" cy="12" r="10"></circle>
											<path d="M12 16v-4"></path>
											<path d="M12 8h.01"></path>
										</svg>
									</Tooltip.Trigger>
									<Tooltip.Content>
										<p class="max-w-xs text-sm">
											The total price of the property or item you want to purchase.
										</p>
									</Tooltip.Content>
								</Tooltip.Root>
								<span class="ml-auto text-sm text-muted-foreground">
									{formatCurrency(loanAmount)}
								</span>
							</div>
							<div class="flex items-center gap-3">
								<Input
									id="loan-amount"
									type="number"
									bind:value={loanAmount}
									min="0"
									step="1000"
									class="w-32"
								/>
								<input
									type="range"
									bind:value={loanAmount}
									min="10000"
									max="1000000"
									step="5000"
									class="flex-1"
								/>
							</div>
						</div>

						<Separator class="my-6" />

						<!-- Down Payment -->
						<div class="space-y-2">
							<div class="flex items-center gap-2">
								<label for="down-payment" class="text-sm font-medium">Down Payment</label>
								<Tooltip.Root>
									<Tooltip.Trigger>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											width="16"
											height="16"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											class="text-muted-foreground"
										>
											<circle cx="12" cy="12" r="10"></circle>
											<path d="M12 16v-4"></path>
											<path d="M12 8h.01"></path>
										</svg>
									</Tooltip.Trigger>
									<Tooltip.Content>
										<p class="max-w-xs text-sm">
											The initial payment you make upfront. A larger down payment reduces the
											loan amount and total interest paid.
										</p>
									</Tooltip.Content>
								</Tooltip.Root>
                                <span class="ml-auto text-sm text-muted-foreground">
									{formatCurrency(downPayment)}
								</span>
							</div>
							<div class="flex items-center gap-3">
								<Input
									id="down-payment"
									type="number"
									bind:value={downPayment}
									min="0"
									max={loanAmount}
									step="1000"
									class="w-32"
								/>
								
                                <input
                                    type="range"
                                    bind:value={downPayment}
                                    min="0"
                                    max={loanAmount}
                                    step="1000"
                                    class="flex-1"
                                />
							</div>
							<p class="text-xs text-muted-foreground">
								{((downPayment / loanAmount) * 100).toFixed(1)}% of loan amount
							</p>
						</div>

						<Separator class="my-6" />

						<!-- Interest Rate -->
						<div class="space-y-2">
							<div class="flex items-center gap-2">
								<label for="interest-rate" class="text-sm font-medium">Annual Interest Rate</label>
								<Tooltip.Root>
									<Tooltip.Trigger>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											width="16"
											height="16"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											class="text-muted-foreground"
										>
											<circle cx="12" cy="12" r="10"></circle>
											<path d="M12 16v-4"></path>
											<path d="M12 8h.01"></path>
										</svg>
									</Tooltip.Trigger>
									<Tooltip.Content>
										<p class="max-w-xs text-sm">
											The yearly percentage rate charged by the lender. A fixed rate remains the
											same throughout the loan term.
										</p>
									</Tooltip.Content>
								</Tooltip.Root>
                                <span class="ml-auto text-sm text-muted-foreground">
                                    {formatPercent(interestRate)}
                                </span>
							</div>
							<div class="flex items-center gap-3">
								<Input
									id="interest-rate"
									type="number"
									bind:value={interestRate}
									min="0"
									max="20"
									step="0.1"
									class="w-32"
								/>
                                <input
                                    type="range"
                                    bind:value={interestRate}
                                    min="0"
                                    max="15"
                                    step="0.1"
                                    class="flex-1"
                                />
							</div>
						</div>

						<Separator class="my-6" />

						<!-- Loan Term -->
						<div class="space-y-2">
							<div class="flex items-center gap-2">
								<label for="loan-term" class="text-sm font-medium">Loan Term (Years)</label>
								<Tooltip.Root>
									<Tooltip.Trigger>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											width="16"
											height="16"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											class="text-muted-foreground"
										>
											<circle cx="12" cy="12" r="10"></circle>
											<path d="M12 16v-4"></path>
											<path d="M12 8h.01"></path>
										</svg>
									</Tooltip.Trigger>
									<Tooltip.Content>
										<p class="max-w-xs text-sm">
											The length of time you have to repay the loan. Longer terms mean lower
											monthly payments but more interest paid overall.
										</p>
									</Tooltip.Content>
								</Tooltip.Root>
                                <span class="ml-auto text-sm text-muted-foreground">
                                    {loanTermYears} years
                                </span>
							</div>
							<div class="flex items-center gap-3">
								<Input
									id="loan-term"
									type="number"
									bind:value={loanTermYears}
									min="1"
									max="40"
									step="1"
									class="w-32"
								/>
                                <input
                                    type="range"
                                    bind:value={loanTermYears}
                                    min="5"
                                    max="40"
                                    step="1"
                                    class="flex-1"
                                />
							</div>
							<p class="text-xs text-muted-foreground">
								{numberOfPayments} monthly payments
							</p>
						</div>
					</div>

					<!-- Investment Parameters -->
					<div class="rounded-lg border bg-card p-6 shadow-sm">
						<h2 class="mb-6 text-xl font-semibold">Investment Parameters</h2>

						<!-- Expected Return Rate -->
						<div class="space-y-2">
							<div class="flex items-center gap-2">
								<label for="investment-rate" class="text-sm font-medium">Expected Annual Return</label>
								<Tooltip.Root>
									<Tooltip.Trigger>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											width="16"
											height="16"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											class="text-muted-foreground"
										>
											<circle cx="12" cy="12" r="10"></circle>
											<path d="M12 16v-4"></path>
											<path d="M12 8h.01"></path>
										</svg>
									</Tooltip.Trigger>
									<Tooltip.Content>
										<p class="max-w-xs text-sm">
											The expected annual return rate if you invested the money instead. S&P 500 historical average is ~7-10%.
										</p>
									</Tooltip.Content>
								</Tooltip.Root>
                                <span class="ml-auto text-right text-sm text-muted-foreground">
                                    {formatPercent(investmentRate)}
                                </span>
							</div>
							<div class="flex items-center gap-3">
								<Input
									id="investment-rate"
									type="number"
									bind:value={investmentRate}
									min="0"
									max="30"
									step="0.1"
									class="w-32"
								/>
                                <input
                                    type="range"
                                    bind:value={investmentRate}
                                    min="0"
                                    max="20"
                                    step="0.5"
                                    class="flex-1"
                                />
							</div>
						</div>

						<Separator class="my-6" />

						<!-- Volatility -->
						<div class="space-y-2">
							<div class="flex items-center gap-2">
								<label for="volatility" class="text-sm font-medium">Volatility (Risk)</label>
								<Tooltip.Root>
									<Tooltip.Trigger>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											width="16"
											height="16"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											class="text-muted-foreground"
										>
											<circle cx="12" cy="12" r="10"></circle>
											<path d="M12 16v-4"></path>
											<path d="M12 8h.01"></path>
										</svg>
									</Tooltip.Trigger>
									<Tooltip.Content>
										<p class="max-w-xs text-sm">
											Standard deviation of returns. Higher volatility means higher risk. Bonds: ~5%, Stocks: ~15-20%, Crypto: ~50%+.
										</p>
									</Tooltip.Content>
								</Tooltip.Root>
                                <span class="ml-auto text-sm {riskColor}">
                                    {riskLevel} Risk
                                </span>
							</div>
							<div class="flex items-center gap-3">
								<Input
									id="volatility"
									type="number"
									bind:value={investmentVolatility}
									min="0"
									max="100"
									step="1"
									class="w-32"
								/>
                                <input
                                    type="range"
                                    bind:value={investmentVolatility}
                                    min="0"
                                    max="50"
                                    step="1"
                                    class="flex-1"
                                />
							</div>
						</div>

						<Separator class="my-6" />

						<!-- Analysis Period -->
						<div class="space-y-2">
							<div class="flex items-center gap-2">
								<label for="analysis-years" class="text-sm font-medium">Analysis Period</label>
								<Tooltip.Root>
									<Tooltip.Trigger>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											width="16"
											height="16"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											class="text-muted-foreground"
										>
											<circle cx="12" cy="12" r="10"></circle>
											<path d="M12 16v-4"></path>
											<path d="M12 8h.01"></path>
										</svg>
									</Tooltip.Trigger>
									<Tooltip.Content>
										<p class="max-w-xs text-sm">
											Total years to analyze, including years after the loan term. Allows you to see long-term compound growth.
										</p>
									</Tooltip.Content>
								</Tooltip.Root>
                                <span class="ml-auto text-sm text-muted-foreground">
                                    {analysisYears} years
                                </span>
							</div>
							<div class="flex items-center gap-3">
								<Input
									id="analysis-years"
									type="number"
									bind:value={analysisYears}
									min={loanTermYears}
									max="60"
									step="1"
									class="w-32"
								/>
                                <input
                                    type="range"
                                    bind:value={analysisYears}
                                    min={loanTermYears}
                                    max="60"
                                    step="1"
                                    class="flex-1"
                                />
							</div>
							<p class="text-xs text-muted-foreground">
								{analysisYears - loanTermYears} years after loan ends
							</p>
						</div>
					</div>
				</div>

				<!-- CENTER & RIGHT: Analysis & Visualization -->
				<div class="lg:col-span-8 space-y-4">
					<!-- Volatility Scenarios & Key Metrics -->
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

					<!-- Growth Chart -->
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
	</div>
</Tooltip.Provider>

<style>
	input[type="range"] {
		-webkit-appearance: none;
		-moz-appearance: none;
		appearance: none;
		width: 100%;
		height: 0.75rem;
		background: transparent;
		cursor: pointer;
		outline: none;
		border-radius: 0.75rem;
	}

	/* WebKit (Chrome, Safari, Edge) - Track */
	input[type="range"]::-webkit-slider-runnable-track {
		width: 100%;
		height: 0.75rem;
		background: #e2e8f0;
		border-radius: 0.75rem;
		border: none;
	}

	/* WebKit - Thumb */
	input[type="range"]::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		height: 1.5rem;
		width: 1.5rem;
		border-radius: 50%;
		background: #3b82f6;
		cursor: pointer;
		margin-top: -0.375rem;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
		transition: all 0.2s ease;
	}

	input[type="range"]::-webkit-slider-thumb:hover {
		background: #2563eb;
		transform: scale(1.1);
		box-shadow: 0 3px 6px rgba(0, 0, 0, 0.3);
	}

	/* Firefox - Track */
	input[type="range"]::-moz-range-track {
		width: 100%;
		height: 0.75rem;
		background: #e2e8f0;
		border-radius: 0.75rem;
		border: none;
	}

	/* Firefox - Thumb */
	input[type="range"]::-moz-range-thumb {
		height: 1.5rem;
		width: 1.5rem;
		border-radius: 50%;
		background: #3b82f6;
		border: none;
		cursor: pointer;
		box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
		transition: all 0.2s ease;
	}

	input[type="range"]::-moz-range-thumb:hover {
		background: #2563eb;
		transform: scale(1.1);
		box-shadow: 0 3px 6px rgba(0, 0, 0, 0.3);
	}

	/* Focus states */
	input[type="range"]:focus {
		outline: none;
	}

	input[type="range"]:focus::-webkit-slider-thumb {
		outline: 2px solid #3b82f6;
		outline-offset: 2px;
	}

	input[type="range"]:focus::-moz-range-thumb {
		outline: 2px solid #3b82f6;
		outline-offset: 2px;
	}

	/* Dark mode support */
	:global(.dark) input[type="range"]::-webkit-slider-runnable-track {
		background: #334155;
	}

	:global(.dark) input[type="range"]::-moz-range-track {
		background: #334155;
	}
</style>
