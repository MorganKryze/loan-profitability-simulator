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

	function resetValues() {
		loanAmount = 200000;
		interestRate = 4.5;
		loanTermYears = 30;
		downPayment = 40000;
	}
</script>

<Tooltip.Provider>
	<div class="min-h-screen bg-background p-4 md:p-8">
		<div class="mx-auto max-w-6xl space-y-6">
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
					<!-- Currency Selector -->
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
				</div>
			</div>

			<div class="grid gap-6 lg:grid-cols-2">
				<!-- Input Section -->
				<div class="space-y-6">
					<div class="rounded-lg border bg-card p-6 shadow-sm">
						<h2 class="mb-6 text-xl font-semibold">Loan Parameters</h2>

						<!-- Loan Amount -->
						<div class="space-y-2">
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
							</div>
							<div class="flex items-center gap-4">
								<Input
									id="loan-amount"
									type="number"
									bind:value={loanAmount}
									min="0"
									step="1000"
									class="flex-1"
								/>
								<span class="w-32 text-right text-sm text-muted-foreground">
									{formatCurrency(loanAmount)}
								</span>
							</div>
							<input
								type="range"
								bind:value={loanAmount}
								min="10000"
								max="1000000"
								step="5000"
								class="w-full"
							/>
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
							</div>
							<div class="flex items-center gap-4">
								<Input
									id="down-payment"
									type="number"
									bind:value={downPayment}
									min="0"
									max={loanAmount}
									step="1000"
									class="flex-1"
								/>
								<span class="w-32 text-right text-sm text-muted-foreground">
									{formatCurrency(downPayment)}
								</span>
							</div>
							<input
								type="range"
								bind:value={downPayment}
								min="0"
								max={loanAmount}
								step="1000"
								class="w-full"
							/>
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
							</div>
							<div class="flex items-center gap-4">
								<Input
									id="interest-rate"
									type="number"
									bind:value={interestRate}
									min="0"
									max="20"
									step="0.1"
									class="flex-1"
								/>
								<span class="w-32 text-right text-sm text-muted-foreground">
									{formatPercent(interestRate)}
								</span>
							</div>
							<input
								type="range"
								bind:value={interestRate}
								min="0"
								max="15"
								step="0.1"
								class="w-full"
							/>
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
							</div>
							<div class="flex items-center gap-4">
								<Input
									id="loan-term"
									type="number"
									bind:value={loanTermYears}
									min="1"
									max="40"
									step="1"
									class="flex-1"
								/>
								<span class="w-32 text-right text-sm text-muted-foreground">
									{loanTermYears} years
								</span>
							</div>
							<input
								type="range"
								bind:value={loanTermYears}
								min="5"
								max="40"
								step="1"
								class="w-full"
							/>
							<p class="text-xs text-muted-foreground">
								{numberOfPayments} monthly payments
							</p>
						</div>

						<Separator class="my-6" />

					<Button onclick={resetValues} variant="outline" class="w-full">
							Reset to Default Values
						</Button>
					</div>
				</div>

				<!-- Results Section -->
				<div class="space-y-6">
					<!-- Monthly Payment Card -->
					<div class="rounded-lg border bg-card p-6 shadow-sm">
						<div class="space-y-2">
							<p class="text-sm font-medium text-muted-foreground">Monthly Payment</p>
							<p class="text-4xl font-bold tracking-tight">
								{formatCurrency(monthlyPayment)}
							</p>
						</div>
					</div>

					<!-- Summary Cards -->
					<div class="grid gap-4 sm:grid-cols-2">
						<div class="rounded-lg border bg-card p-6 shadow-sm">
							<div class="space-y-1">
								<p class="text-sm font-medium text-muted-foreground">Principal</p>
								<p class="text-2xl font-semibold">
									{formatCurrency(principal)}
								</p>
							</div>
						</div>

						<div class="rounded-lg border bg-card p-6 shadow-sm">
							<div class="space-y-1">
								<p class="text-sm font-medium text-muted-foreground">Total Interest</p>
								<p class="text-2xl font-semibold text-chart-1">
									{formatCurrency(totalInterest)}
								</p>
							</div>
						</div>

						<div class="rounded-lg border bg-card p-6 shadow-sm">
							<div class="space-y-1">
								<p class="text-sm font-medium text-muted-foreground">Total Payments</p>
								<p class="text-2xl font-semibold">
									{formatCurrency(totalPayment)}
								</p>
							</div>
						</div>

						<div class="rounded-lg border bg-card p-6 shadow-sm">
							<div class="space-y-1">
								<p class="text-sm font-medium text-muted-foreground">Total Cost</p>
								<p class="text-2xl font-semibold">
									{formatCurrency(totalCost)}
								</p>
								<p class="text-xs text-muted-foreground">Including down payment</p>
							</div>
						</div>
					</div>

					<!-- Breakdown -->
					<div class="rounded-lg border bg-card p-6 shadow-sm">
						<h3 class="mb-4 text-lg font-semibold">Cost Breakdown</h3>
						<div class="space-y-4">
							<div class="space-y-2">
								<div class="flex justify-between text-sm">
									<span class="text-muted-foreground">Down Payment</span>
									<span class="font-medium">{formatCurrency(downPayment)}</span>
								</div>
								<div class="h-2 w-full overflow-hidden rounded-full bg-secondary">
									<div
										class="h-full bg-primary transition-all"
										style="width: {(downPayment / totalCost) * 100}%"
									></div>
								</div>
								<p class="text-xs text-muted-foreground text-right">
									{((downPayment / totalCost) * 100).toFixed(1)}%
								</p>
							</div>

							<div class="space-y-2">
								<div class="flex justify-between text-sm">
									<span class="text-muted-foreground">Principal Payments</span>
									<span class="font-medium">{formatCurrency(principal)}</span>
								</div>
								<div class="h-2 w-full overflow-hidden rounded-full bg-secondary">
									<div
										class="h-full bg-chart-2 transition-all"
										style="width: {(principal / totalCost) * 100}%"
									></div>
								</div>
								<p class="text-xs text-muted-foreground text-right">
									{((principal / totalCost) * 100).toFixed(1)}%
								</p>
							</div>

							<div class="space-y-2">
								<div class="flex justify-between text-sm">
									<span class="text-muted-foreground">Interest Payments</span>
									<span class="font-medium">{formatCurrency(totalInterest)}</span>
								</div>
								<div class="h-2 w-full overflow-hidden rounded-full bg-secondary">
									<div
										class="h-full bg-chart-1 transition-all"
										style="width: {(totalInterest / totalCost) * 100}%"
									></div>
								</div>
								<p class="text-xs text-muted-foreground text-right">
									{((totalInterest / totalCost) * 100).toFixed(1)}%
								</p>
							</div>
						</div>
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
