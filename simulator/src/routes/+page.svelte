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
	import { onMount } from "svelte";
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
		insuranceCost: number;
		investmentRate: number;
		investmentVolatility: number;
		additionalYears: number;
		inflationRate: number;
	} = {
		loanAmount: 200000,
		interestRate: 4.5,
		loanTermYears: 30,
		downPayment: 40000,
		deferralMonths: 12,
		deferralType: 'complete',
		insuranceCost: 30,
		investmentRate: 7.0,
		investmentVolatility: 3,
		additionalYears: 10,
		inflationRate: 2.5,
	};

	// Investment presets
	type InvestmentPreset = {
		id: string;
		name: string;
		returnRate: number;
		volatility: number;
		description: string;
	};

	const investmentPresets: InvestmentPreset[] = [
		{ id: 'custom', name: 'Custom', returnRate: 7, volatility: 3, description: 'Set your own values' },
		{ id: 'savings', name: 'Savings Account', returnRate: 2, volatility: 0.5, description: 'Low risk, guaranteed returns' },
		{ id: 'bonds', name: 'Government Bonds', returnRate: 4, volatility: 5, description: 'Low-medium risk, stable income' },
		{ id: 'corporate-bonds', name: 'Corporate Bonds', returnRate: 5.5, volatility: 8, description: 'Medium risk, higher yield' },
		{ id: 'balanced', name: 'Balanced Fund (60/40)', returnRate: 6.5, volatility: 10, description: 'Mixed stocks and bonds' },
		{ id: 'sp500', name: 'S&P 500 Index', returnRate: 10, volatility: 15, description: 'US large-cap stocks' },
		{ id: 'world-stocks', name: 'World Stocks (MSCI)', returnRate: 8, volatility: 17, description: 'Global diversified equities' },
		{ id: 'real-estate', name: 'Real Estate Crowdfunding', returnRate: 8, volatility: 12, description: 'Property investments, illiquid' },
		{ id: 'reit', name: 'REITs', returnRate: 9, volatility: 20, description: 'Real estate investment trusts' },
		{ id: 'emerging', name: 'Emerging Markets', returnRate: 11, volatility: 25, description: 'Higher growth, higher risk' },
		{ id: 'crypto', name: 'Crypto (BTC/ETH)', returnRate: 30, volatility: 80, description: 'Extremely volatile, speculative' },
	];

	let selectedPresetId = 'custom';
	$: selectedPreset = investmentPresets.find(p => p.id === selectedPresetId) || investmentPresets[0];

	function applyPreset(presetId: string) {
		const preset = investmentPresets.find(p => p.id === presetId);
		if (preset && presetId !== 'custom') {
			investmentRate = preset.returnRate;
			investmentVolatility = preset.volatility;
		}
	}

	// Input values
	let loanAmount = DEFAULTS.loanAmount;
	let interestRate = DEFAULTS.interestRate;
	let loanTermYears = DEFAULTS.loanTermYears;
	let downPayment = DEFAULTS.downPayment;
	let deferralMonths = DEFAULTS.deferralMonths;
	let deferralType: 'complete' | 'partial' = DEFAULTS.deferralType;
	let insuranceCost = DEFAULTS.insuranceCost;

	// Investment parameters
	let investmentRate = DEFAULTS.investmentRate;
	let investmentVolatility = DEFAULTS.investmentVolatility;
	let additionalYears = DEFAULTS.additionalYears;
	let inflationRate = DEFAULTS.inflationRate;

	// Clamp values to respect min/max constraints
	$: loanAmount = Math.max(0, Math.min(10000000, loanAmount));
	$: downPayment = Math.max(0, Math.min(loanAmount, downPayment));
	$: interestRate = Math.max(0, Math.min(20, interestRate));
	$: loanTermYears = Math.max(1, Math.min(40, loanTermYears));
	// Deferral cannot exceed loan term minus 1 month (need at least 1 payment)
	$: deferralMonths = Math.max(0, Math.min(loanTermYears * 12 - 1, deferralMonths));
	$: insuranceCost = Math.max(0, Math.min(10000, insuranceCost));
	$: investmentRate = Math.max(0, Math.min(30, investmentRate));
	$: investmentVolatility = Math.max(0, Math.min(100, investmentVolatility));
	$: additionalYears = Math.max(0, Math.min(30, additionalYears));
	$: inflationRate = Math.max(0, Math.min(20, inflationRate));
	
	// Real return rate (inflation-adjusted)
	$: realInvestmentRate = investmentRate - inflationRate;
	
	// Total analysis period = loan term + additional years
	$: analysisYears = loanTermYears + additionalYears;

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
	// Deferral is part of the loan term, so repayment period = loan term - deferral
	$: totalLoanMonths = loanTermYears * 12;
	$: numberOfPayments = Math.max(1, totalLoanMonths - deferralMonths);
	$: monthlyPayment =
		principalAfterDeferral > 0 && monthlyRate > 0
			? (principalAfterDeferral * (monthlyRate * Math.pow(1 + monthlyRate, numberOfPayments))) /
				(Math.pow(1 + monthlyRate, numberOfPayments) - 1)
			: 0;
	$: totalPayment = monthlyPayment * numberOfPayments;
	// Total interest = payments during repayment - original principal + interest paid during partial deferral
	$: totalInterest = totalPayment - principal + deferralInterestPaid;
	$: totalCost = totalPayment + downPayment + deferralInterestPaid + totalInsuranceCost;
	$: totalInsuranceCost = insuranceCost * totalLoanMonths;

	// Investment calculations
	// Calculate year-by-year investment growth with compound interest
	type YearlyData = {
		year: number;
		investmentValue: number;
		investmentValueReal: number; // Inflation-adjusted investment value
		interestEarned: number; // Investment gains above principal
		interestEarnedReal: number; // Real (inflation-adjusted) gains
		loanBalance: number;
		totalPaid: number;
		totalPaidReal: number; // Real value of payments (cheaper over time due to inflation)
		totalBorrowingCost: number; // Cumulative interest + insurance paid
		totalBorrowingCostReal: number; // Real borrowing cost
		netPosition: number;
		netPositionReal: number; // Real net position
		isLoanActive: boolean;
		isDeferralPeriod: boolean;
	};

	// Total loan duration
	$: totalLoanYears = Math.ceil(totalLoanMonths / 12);

	$: yearlyData = (() => {
		// Explicit dependencies for reactivity
		const _inflationRate = inflationRate;
		const _investmentRate = investmentRate;
		
		const data: YearlyData[] = [];
		// Strategy: Take a loan, invest the principal immediately, pay off the loan over time
		// Investment starts with the borrowed amount (principal)
		let investmentValue = principal;
		let loanBalance = principal;
		let totalPaid = 0; // Track all payments made (principal + interest + insurance)
		let totalBorrowingCost = 0; // Track only the cost of borrowing (interest + insurance)
		const monthlyInvestmentRate = _investmentRate / 100 / 12;
		const monthlyInflationRate = _inflationRate / 100 / 12;
		let totalMonthsElapsed = 0;
		let cumulativeInflationFactor = 1; // Tracks cumulative inflation for real value calculations
		
		// Year 0: Initial state - loan taken, principal invested, no payments yet
		data.push({
			year: 0,
			investmentValue: principal,
			investmentValueReal: principal,
			interestEarned: 0,
			interestEarnedReal: 0,
			loanBalance: principal,
			totalPaid: 0,
			totalPaidReal: 0,
			totalBorrowingCost: 0,
			totalBorrowingCostReal: 0,
			netPosition: principal,
			netPositionReal: principal,
			isLoanActive: true,
			isDeferralPeriod: deferralMonths > 0
		});
		
		for (let year = 1; year <= analysisYears; year++) {
			// Monthly calculations for the year
			for (let month = 0; month < 12; month++) {
				totalMonthsElapsed++;
				const isDeferralMonth = totalMonthsElapsed <= deferralMonths;
				const isRepaymentMonth = totalMonthsElapsed > deferralMonths && totalMonthsElapsed <= totalLoanMonths;
				const isLoanActiveMonth = totalMonthsElapsed <= totalLoanMonths;
				
				// Update cumulative inflation factor
				cumulativeInflationFactor *= (1 + monthlyInflationRate);
				
				// Investment grows with compound interest
				investmentValue *= (1 + monthlyInvestmentRate);
				
				// Insurance is paid every month during the loan term (including deferral)
				if (isLoanActiveMonth) {
					totalPaid += insuranceCost;
					totalBorrowingCost += insuranceCost;
				}
				
				if (isDeferralMonth) {
					if (deferralType === 'complete') {
						// Complete deferral: interest accrues on loan balance (capitalized)
						loanBalance *= (1 + monthlyRate);
						// No payment during deferral (except insurance)
					} else {
						// Partial deferral: only interest is paid monthly, principal stays the same
						const interestOnly = loanBalance * monthlyRate;
						totalPaid += interestOnly;
						totalBorrowingCost += interestOnly;
					}
				} else if (isRepaymentMonth) {
					// During repayment: pay monthly payment
					totalPaid += monthlyPayment;
					
					// Calculate remaining loan balance and interest portion
					const interestPayment = loanBalance * monthlyRate;
					const principalPayment = monthlyPayment - interestPayment;
					loanBalance = Math.max(0, loanBalance - principalPayment);
					totalBorrowingCost += interestPayment; // Only the interest portion is a cost
				}
			}
			
			const isLoanActive = totalMonthsElapsed <= totalLoanMonths;
			const isDeferralPeriod = totalMonthsElapsed <= deferralMonths;
			const interestEarned = investmentValue - principal;
			
			// Calculate real (inflation-adjusted) values
			// Real value = Nominal value / cumulative inflation factor
			const investmentValueReal = investmentValue / cumulativeInflationFactor;
			const totalPaidReal = totalPaid / cumulativeInflationFactor;
			const totalBorrowingCostReal = totalBorrowingCost / cumulativeInflationFactor;
			const interestEarnedReal = investmentValueReal - principal;
			
			data.push({
				year,
				investmentValue,
				investmentValueReal,
				interestEarned,
				interestEarnedReal,
				loanBalance: isLoanActive ? loanBalance : 0,
				totalPaid,
				totalPaidReal,
				totalBorrowingCost,
				totalBorrowingCostReal,
				netPosition: investmentValue - totalPaid,
				netPositionReal: investmentValueReal - totalPaidReal,
				isLoanActive,
				isDeferralPeriod
			});
		}
		return data;
	})();

	// Key metrics (nominal)
	$: finalInvestmentValue = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].investmentValue : 0;
	$: finalInterestEarned = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].interestEarned : 0;
	$: finalBorrowingCost = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].totalBorrowingCost : 0;
	$: finalNetPosition = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].netPosition : 0;
	
	// Key metrics (real / inflation-adjusted)
	$: finalInvestmentValueReal = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].investmentValueReal : 0;
	$: finalInterestEarnedReal = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].interestEarnedReal : 0;
	$: finalNetPositionReal = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].netPositionReal : 0;
	
	// Total borrowing cost at loan end (interest + insurance for the entire loan)
	$: totalLoanBorrowingCost = totalInterest + totalInsuranceCost;
	// Real borrowing cost adjusted for inflation
	$: totalLoanBorrowingCostReal = yearlyData.length > 0 ? yearlyData[yearlyData.length - 1].totalBorrowingCostReal : 0;
	
	$: isPositiveROI = finalInterestEarned > totalLoanBorrowingCost;
	$: isPositiveROIReal = finalInterestEarnedReal > totalLoanBorrowingCostReal;
	
	// Break-even: year when investment interest earned exceeds the FINAL total borrowing cost
	// This is when your investment gains cover ALL the costs of the loan
	$: breakEvenYear = (() => {
		const found = yearlyData.find(d => d.interestEarned >= totalLoanBorrowingCost);
		return found ? found.year : null;
	})();
	
	// Real break-even (inflation-adjusted)
	$: breakEvenYearReal = (() => {
		const found = yearlyData.find(d => d.interestEarnedReal >= totalLoanBorrowingCostReal);
		return found ? found.year : null;
	})();
	
	// Volatility-adjusted returns (using simplified model)
	$: bestCaseReturn = investmentRate + investmentVolatility;
	$: worstCaseReturn = investmentRate - investmentVolatility;
	
	// Calculate final values with volatility scenarios
	// Same logic: borrow principal, invest it, let it grow
	$: scenarioCalculation = (rate: number) => {
		let value = principal;
		const monthlyRate = rate / 100 / 12;
		for (let year = 1; year <= analysisYears; year++) {
			for (let month = 0; month < 12; month++) {
				value *= (1 + monthlyRate);
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
			label: "Total Repayment",
			color: "#f97316"
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
	// Same logic: borrow principal, invest it, let it grow with different return rates
	$: scenarioYearlyData = (() => {
		const data: { year: number; worst: number; expected: number; best: number }[] = [];
		let worstValue = principal;
		let expectedValue = principal;
		let bestValue = principal;
		const worstMonthlyRate = worstCaseReturn / 100 / 12;
		const expectedMonthlyRate = investmentRate / 100 / 12;
		const bestMonthlyRate = bestCaseReturn / 100 / 12;
		
		// Year 0: Initial state
		data.push({ year: 0, worst: principal, expected: principal, best: principal });
		
		for (let year = 1; year <= analysisYears; year++) {
			for (let month = 0; month < 12; month++) {
				worstValue *= (1 + worstMonthlyRate);
				expectedValue *= (1 + expectedMonthlyRate);
				bestValue *= (1 + bestMonthlyRate);
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
		insuranceCost = DEFAULTS.insuranceCost;
		investmentRate = DEFAULTS.investmentRate;
		investmentVolatility = DEFAULTS.investmentVolatility;
		additionalYears = DEFAULTS.additionalYears;
		inflationRate = DEFAULTS.inflationRate;
		selectedPresetId = 'custom';
	}

	// Share functionality
	let showCopiedMessage = false;

	function generateShareUrl() {
		const params = new URLSearchParams({
			loanAmount: loanAmount.toString(),
			interestRate: interestRate.toString(),
			loanTermYears: loanTermYears.toString(),
			downPayment: downPayment.toString(),
			deferralMonths: deferralMonths.toString(),
			deferralType: deferralType,
			insuranceCost: insuranceCost.toString(),
			investmentRate: investmentRate.toString(),
			investmentVolatility: investmentVolatility.toString(),
			additionalYears: additionalYears.toString(),
			inflationRate: inflationRate.toString(),
			selectedPresetId: selectedPresetId,
			currency: selectedCurrencyCode,
		});
		return `${window.location.origin}${window.location.pathname}?${params.toString()}`;
	}

	async function copyShareLink() {
		const url = generateShareUrl();
		try {
			await navigator.clipboard.writeText(url);
			showCopiedMessage = true;
			setTimeout(() => {
				showCopiedMessage = false;
			}, 2000);
		} catch (err) {
			console.error('Failed to copy:', err);
		}
	}

	// Load parameters from URL on mount
	onMount(() => {
		const params = new URLSearchParams(window.location.search);

		if (params.has('loanAmount')) loanAmount = Number(params.get('loanAmount'));
		if (params.has('interestRate')) interestRate = Number(params.get('interestRate'));
		if (params.has('loanTermYears')) loanTermYears = Number(params.get('loanTermYears'));
		if (params.has('downPayment')) downPayment = Number(params.get('downPayment'));
		if (params.has('deferralMonths')) deferralMonths = Number(params.get('deferralMonths'));
		if (params.has('deferralType')) deferralType = params.get('deferralType') as 'complete' | 'partial';
		if (params.has('insuranceCost')) insuranceCost = Number(params.get('insuranceCost'));
		if (params.has('investmentRate')) investmentRate = Number(params.get('investmentRate'));
		if (params.has('investmentVolatility')) investmentVolatility = Number(params.get('investmentVolatility'));
		if (params.has('additionalYears')) additionalYears = Number(params.get('additionalYears'));
		if (params.has('inflationRate')) inflationRate = Number(params.get('inflationRate'));
		if (params.has('selectedPresetId')) selectedPresetId = params.get('selectedPresetId') || 'custom';
		if (params.has('currency')) selectedCurrencyCode = params.get('currency') || 'USD';
	});
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
						<p class="text-xs text-muted-foreground">
							{#if deferralMonths > 0}
								{deferralMonths} months deferral + {numberOfPayments} payments
							{:else}
								{numberOfPayments} monthly payments
							{/if}
						</p>
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
								<p>Grace period before loan repayment begins (included in loan term)</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="deferral-months"
							type="number"
							bind:value={deferralMonths}
							min="0"
							max={loanTermYears * 12 - 1}
							step="1"
						/>
						{#if deferralMonths > 0}
							<div class="space-y-2">
								<label for="deferral-type" class="text-sm font-medium">Deferral Type</label>
								<Select.Root type="single" bind:value={deferralType}>
									<Select.Trigger id="deferral-type" class="w-full">
										{deferralType === 'complete' ? 'Complete (insurance only)' : 'Partial (interest + insurance)'}
									</Select.Trigger>
									<Select.Content>
										<Select.Item value="complete">Complete (insurance only)</Select.Item>
										<Select.Item value="partial">Partial (interest + insurance)</Select.Item>
									</Select.Content>
								</Select.Root>
								<p class="text-xs text-muted-foreground">
									{#if deferralType === 'complete'}
										Interest capitalizes · Pay {formatCurrency(insuranceCost)}/month insurance for {deferralMonths} months
									{:else}
										Pay {formatCurrency((loanAmount - downPayment) * monthlyRate + insuranceCost)}/month (interest + insurance) for {deferralMonths} months
									{/if}
								</p>
							</div>
						{:else}
							<p class="text-xs text-muted-foreground">No deferral - repayment starts immediately</p>
						{/if}
					</div>

					<!-- Insurance Cost -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="insurance-cost" class="text-sm font-medium cursor-help">Monthly Insurance Cost ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Fixed monthly insurance premium paid during the loan term</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="insurance-cost"
							type="number"
							bind:value={insuranceCost}
							min="0"
							max="10000"
							step="10"
						/>
						<p class="text-xs text-muted-foreground">
							{formatCurrency(insuranceCost)}/month · Total: {formatCurrency(totalInsuranceCost)}
						</p>
					</div>
				</div>

				<Separator />

				<!-- Investment Parameters -->
				<div class="space-y-4">
					<h3 class="text-sm font-semibold text-muted-foreground uppercase tracking-wider">Investment Parameters</h3>
					
					<!-- Investment Preset Selector -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="investment-preset" class="text-sm font-medium cursor-help">Investment Type ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Select a preset or customize your own values</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Select.Root type="single" bind:value={selectedPresetId} onValueChange={applyPreset}>
							<Select.Trigger id="investment-preset" class="w-full">
								{selectedPreset.name}
							</Select.Trigger>
							<Select.Content>
								{#each investmentPresets as preset}
									<Select.Item value={preset.id}>
										<div class="flex flex-col">
											<span>{preset.name}</span>
											<span class="text-xs text-muted-foreground">{preset.returnRate}% · ±{preset.volatility}%</span>
										</div>
									</Select.Item>
								{/each}
							</Select.Content>
						</Select.Root>
						<p class="text-xs text-muted-foreground">{selectedPreset.description}</p>
					</div>

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

					<!-- Inflation Rate -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="inflation-rate" class="text-sm font-medium cursor-help">Inflation Rate (%) ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Expected annual inflation (historically ≈ 2-3%). Real returns = Nominal - Inflation</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="inflation-rate"
							type="number"
							bind:value={inflationRate}
							min="0"
							max="20"
							step="0.1"
						/>
						<p class="text-xs text-muted-foreground">Real return: {formatPercent(realInvestmentRate)}</p>
					</div>

					<!-- Analysis Period -->
					<div class="space-y-2">
						<Tooltip.Root>
							<Tooltip.Trigger>
								{#snippet child({ props })}
									<label {...props} for="additional-years" class="text-sm font-medium cursor-help">Years After Loan Ends ⓘ</label>
								{/snippet}
							</Tooltip.Trigger>
							<Tooltip.Content>
								<p>Additional years to simulate after the loan is paid off</p>
							</Tooltip.Content>
						</Tooltip.Root>
						<Input
							id="additional-years"
							type="number"
							bind:value={additionalYears}
							min="0"
							max="30"
							step="1"
						/>
						<p class="text-xs text-muted-foreground">Total analysis: {analysisYears} years ({loanTermYears} + {additionalYears})</p>
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
									Can your debt actually make you money?
								</p>
							</div>
							<!-- Currency Selector and Share Link -->
							<div class="flex items-center gap-6">
								<!-- Share Button -->
								<Tooltip.Root>
									<Tooltip.Trigger>
										{#snippet child({ props })}
											<Button
												{...props}
												variant="outline"
												size="icon"
												onclick={copyShareLink}
												class="relative"
											>
												{#if showCopiedMessage}
													<span class="text-xs">✓</span>
												{:else}
													<svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
														<path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"/>
														<path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"/>
													</svg>
												{/if}
											</Button>
										{/snippet}
									</Tooltip.Trigger>
									<Tooltip.Content>
										<p>{showCopiedMessage ? 'Link copied!' : 'Copy shareable link'}</p>
									</Tooltip.Content>
								</Tooltip.Root>

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
					</div>

					<!-- Dashboard: Key Metrics First -->
					<!-- Top Row: Most Important Info -->
					<div class="grid gap-4 sm:grid-cols-2 lg:grid-cols-5 mb-4">
				   <!-- Monthly Cost (Payment + Insurance) - Most Prominent -->
				   <div class="sm:col-span-2 lg:col-span-2 rounded-lg border bg-linear-to-br from-primary/10 to-primary/5 p-6 shadow-sm">
					   <p class="text-sm font-medium text-muted-foreground">Monthly Payment</p>
					   <p class="text-4xl font-bold tracking-tight mt-2">
						   {formatCurrency(monthlyPayment + insuranceCost)}
					   </p>
					   <p class="text-xs text-muted-foreground mt-1">{numberOfPayments} payments over {loanTermYears} years</p>
				   </div>

				<!-- ROI Status -->
				<div class="sm:col-span-2 lg:col-span-3 rounded-lg border p-6 shadow-sm {isPositiveROI ? 'bg-green-50 dark:bg-green-950 border-green-200 dark:border-green-800' : 'bg-red-50 dark:bg-red-950 border-red-200 dark:border-red-800'}">
					<p class="text-sm font-medium text-muted-foreground flex items-center gap-2">
						{isPositiveROI ? '📈' : '📉'}
						{isPositiveROI ? 'Profitable Strategy' : 'Unprofitable Strategy'}
					</p>
					<p class="flex items-baseline gap-2 mt-2">
						<span class="text-4xl font-bold tracking-tight {isPositiveROI ? 'text-green-700 dark:text-green-300' : 'text-red-700 dark:text-red-300'}">
							{formatCurrency(finalInterestEarned - totalLoanBorrowingCost)}
						</span>
						<span class="text-sm {isPositiveROIReal ? 'text-green-600 dark:text-green-400' : 'text-red-600 dark:text-red-400'}">
							(Real: {formatCurrency(finalInterestEarnedReal - totalLoanBorrowingCostReal)})
						</span>
					</p>
					<p class="text-xs text-muted-foreground mt-1">
						Gains: {formatCurrency(finalInterestEarned)} − Loan cost: {formatCurrency(totalLoanBorrowingCost)}{#if breakEvenYear} · Break-even: Year {breakEvenYear}{/if}
					</p>
				</div>
			</div>

			<!-- Second Row: Summary Cards -->
			<div class="grid gap-3 grid-cols-2 lg:grid-cols-5 mb-4">
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Principal</p>
					<p class="text-lg font-semibold mt-1">{formatCurrency(principal)}</p>
				</div>
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Total Loan Interest</p>
					<p class="text-lg font-semibold text-chart-1 mt-1">{formatCurrency(totalInterest)}</p>
				</div>
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Total Insurance Cost</p>
					<p class="text-lg font-semibold text-chart-1 mt-1">{formatCurrency(totalInsuranceCost)}</p>
				</div>
				<div class="rounded-lg border bg-card p-3 shadow-sm">
					<p class="text-xs text-muted-foreground">Investment Value</p>
					<p class="text-lg font-semibold text-green-600 mt-1">{formatCurrency(finalInvestmentValue)}</p>
					<p class="text-xs text-muted-foreground">Real: {formatCurrency(finalInvestmentValueReal)}</p>
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
					<div class="rounded-lg border bg-card p-4 shadow-sm flex flex-col">
						<h3 class="mb-4 text-lg font-semibold">Investment vs Total Repayment</h3>
						<div class="relative flex-1 min-h-64">
							<Chart.Container config={chartConfig} class="flex-1 min-h-48 w-full overflow-hidden">
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
									]}
									props={{
										area: {
											curve: curveNatural,
											"fill-opacity": 0.4,
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
							<!-- Vertical markers overlay -->
							<div class="absolute pointer-events-none" style="left: 50px; right: 16px; top: 16px; bottom: 40px;">
								<!-- Loan ends marker -->
								<div 
									class="absolute top-0 bottom-0 w-0.5 bg-primary/60" 
									style="left: {(totalLoanYears / analysisYears) * 100}%"
								>
									<div class="absolute -top-1 left-1/2 -translate-x-1/2 bg-primary text-primary-foreground text-[10px] px-1 rounded whitespace-nowrap">
										Loan ends
									</div>
								</div>
								<!-- Break-even marker -->
								{#if breakEvenYear}
									<div 
										class="absolute top-0 bottom-0 w-0.5 bg-green-500/60" 
										style="left: {(breakEvenYear / analysisYears) * 100}%"
									>
										<div class="absolute -top-1 left-1/2 -translate-x-1/2 bg-green-500 text-white text-[10px] px-1 rounded whitespace-nowrap">
											Break-even
										</div>
									</div>
								{/if}
							</div>
						</div>
						
						<!-- Chart legend -->
						<div class="flex flex-wrap gap-4 mt-3 text-xs text-muted-foreground justify-center">
							<div class="flex items-center gap-1">
								<span class="inline-block w-3 h-0.5 bg-primary"></span>
								<span>Loan ends: Year {totalLoanYears}</span>
							</div>
							{#if breakEvenYear}
								<div class="flex items-center gap-1">
									<span class="inline-block w-3 h-0.5 bg-green-500"></span>
									<span>Break-even: Year {breakEvenYear} (gains ≥ {formatCurrency(totalLoanBorrowingCost)})</span>
								</div>
							{:else}
								<div class="flex items-center gap-1">
									<span class="inline-block w-2 h-2 bg-red-500 rounded-full"></span>
									<span>No break-even within {analysisYears} years (need {formatCurrency(totalLoanBorrowingCost)})</span>
								</div>
							{/if}
						</div>
						
						<!-- Summary stats -->
						<div class="grid grid-cols-2 gap-2 mt-4 text-xs">
							<div class="p-2 rounded bg-green-50 dark:bg-green-950/50 text-center">
								<p class="text-muted-foreground">Investment Gains</p>
								<p class="font-semibold text-green-700 dark:text-green-300">{formatCurrency(finalInterestEarned)}</p>
							</div>
							<div class="p-2 rounded bg-orange-50 dark:bg-orange-950/50 text-center">
								<p class="text-muted-foreground">Total Loan Cost</p>
								<p class="font-semibold text-orange-700 dark:text-orange-300">{formatCurrency(totalLoanBorrowingCost)}</p>
							</div>
						</div>
					</div>

					<!-- Investment Analysis (Right) -->
					<div class="rounded-lg border bg-card p-4 shadow-sm flex flex-col">
						<h3 class="mb-2 text-base font-semibold">Volatility Scenarios</h3>
						<p class="text-xs text-muted-foreground mb-3">Investment value after {analysisYears} years based on ±1 standard deviation</p>
						<Chart.Container config={scenariosChartConfig} class="flex-1 min-h-48 w-full overflow-hidden">
							<AreaChart
								legend
								data={scenarioYearlyData}
								x="year"
								xScale={scaleLinear()}
								yPadding={[0, 25]}
								padding={{ left: 60, right: 16, top: 16, bottom: 40 }}
								series={[
									{ key: "best", label: `Optimistic (${formatPercent(bestCaseReturn)})`, color: scenariosChartConfig.best.color },
									{ key: "expected", label: `Expected (${formatPercent(investmentRate)})`, color: scenariosChartConfig.expected.color },
									{ key: "worst", label: `Pessimistic (${formatPercent(worstCaseReturn)})`, color: scenariosChartConfig.worst.color },
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
							<div class="p-2 rounded bg-green-50 dark:bg-green-950/50">
								<p class="font-medium text-green-700 dark:text-green-300">{formatPercent(bestCaseReturn)}</p>
								<p class="text-green-600 dark:text-green-400">{formatCurrency(bestCaseValue)}</p>
							</div>
							<div class="p-2 rounded bg-blue-50 dark:bg-blue-950/50">
								<p class="font-medium text-blue-700 dark:text-blue-300">{formatPercent(investmentRate)}</p>
								<p class="text-blue-600 dark:text-blue-400">{formatCurrency(finalInvestmentValue)}</p>
							</div>
							<div class="p-2 rounded bg-red-50 dark:bg-red-950/50">
								<p class="font-medium text-red-700 dark:text-red-300">{formatPercent(worstCaseReturn)}</p>
								<p class="text-red-600 dark:text-red-400">{formatCurrency(worstCaseValue)}</p>
							</div>
						</div>
					</div>
				</div>

				<!-- Yearly Breakdown Table -->
					<div class="rounded-lg border bg-card p-6 shadow-sm">
						<h3 class="mb-4 text-lg font-semibold">Profitability Over Time</h3>
						<p class="text-xs text-muted-foreground mb-3">Break-even when Investment Gains ≥ Total Loan Cost ({formatCurrency(totalLoanBorrowingCost)})</p>
						<div class="overflow-x-auto max-h-96 overflow-y-auto">
							<table class="w-full text-sm">
								<thead class="sticky top-0 bg-card">
									<tr class="border-b">
										<th class="text-left py-2 px-2">Year</th>
										<th class="text-right py-2 px-2">Investment Value</th>
										<th class="text-right py-2 px-2">Investment Gains</th>
										<th class="text-right py-2 px-2">Total Paid</th>
										<th class="text-right py-2 px-2">vs Loan Cost</th>
										<th class="text-center py-2 px-2">Status</th>
									</tr>
								</thead>
								<tbody>
									{#each yearlyData as data, i}
										{@const isBreakEven = breakEvenYear === data.year}
										{@const vsLoanCost = data.interestEarned - totalLoanBorrowingCost}
										<tr class="border-b border-border/50 {data.year === totalLoanYears ? 'bg-primary/5' : ''} {isBreakEven ? 'bg-green-100 dark:bg-green-900/30' : ''}">
											<td class="py-2 px-2 font-medium">
												{data.year}
												{#if data.year === totalLoanYears}
													<span class="text-xs text-primary ml-1">📋 loan ends</span>
												{/if}
												{#if isBreakEven}
													<span class="text-xs text-green-600 ml-1">✓ break-even</span>
												{/if}
											</td>
											<td class="text-right py-2 px-2">{formatCurrency(data.investmentValue)}</td>
											<td class="text-right py-2 px-2 text-green-600">{formatCurrency(data.interestEarned)}</td>
											<td class="text-right py-2 px-2 text-orange-600">{formatCurrency(data.totalPaid)}</td>
											<td class="text-right py-2 px-2 font-medium {vsLoanCost >= 0 ? 'text-green-600' : 'text-red-600'}">
												{vsLoanCost >= 0 ? '+' : ''}{formatCurrency(vsLoanCost)}
											</td>
											<td class="text-center py-2 px-2">
												{#if data.isLoanActive}
													<span class="text-xs px-2 py-1 rounded-full bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-200">Repaying</span>
												{:else if vsLoanCost >= 0}
													<span class="text-xs px-2 py-1 rounded-full bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-200">Profitable</span>
												{:else}
													<span class="text-xs px-2 py-1 rounded-full bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-200">Not yet</span>
												{/if}
											</td>
										</tr>
									{/each}
								</tbody>
							</table>
						</div>
						<p class="text-xs text-muted-foreground mt-3">
							vs Loan Cost = Investment Gains − Total Loan Cost (interest + insurance: {formatCurrency(totalLoanBorrowingCost)})
						</p>
					</div>
				</div>
			</div>
		</div>
		</Sidebar.Inset>
	</Sidebar.Provider>
</Tooltip.Provider>
