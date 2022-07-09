<script setup>
import { computed, ref, onMounted, onUnmounted, watchEffect } from "vue";
import { useConnectionStore, useAlertStore } from "../store";
import { updateParam } from "../connection";
import { generateRandomNumber, useThrottle } from "../utils";

import LineChart from "../components/LineChart.vue";
import BarChart from "../components/BarChart.vue";
import Table from "../components/Table.vue";
import RangeInput from "../components/RangeInput.vue";

const connection = useConnectionStore();
const alert = useAlertStore();
const isRandomDataOn = ref(connection.randomizeData);
const randomInputInterval = ref(null);

const priceAndRisk = computed(() => connection.dagNodes[1]);
const portfolio = computed(() => connection.dagNodes[0]);

const marketRates = computed(() => {
	if (
		priceAndRisk.value &&
		portfolio.value &&
		priceAndRisk.value.output_params.MarketRates &&
		portfolio.value.output_params.OriginalMarketRates
	) {
		let currentMarketRates = {};
		let originalMarketRates = {};

		priceAndRisk.value.output_params.MarketRates.map((arr) => {
			currentMarketRates[arr[0]] = arr[1];
		});

		portfolio.value.output_params.OriginalMarketRates.map((arr) => {
			originalMarketRates[arr[0]] = arr[1];
		});

		return [currentMarketRates, originalMarketRates];
	} else {
		return [];
	}
});

const fittedValues = computed(() => {
	if (
		priceAndRisk.value &&
		priceAndRisk.value.output_params.fitted_values_forecast &&
		priceAndRisk.value.output_params.fitted_values_discount
	) {
		return [
			priceAndRisk.value.output_params.fitted_values_forecast,
			priceAndRisk.value.output_params.fitted_values_discount,
		];
	} else {
		return [];
	}
});

const risk = computed(() => {
	let risk = {};

	if (priceAndRisk.value && priceAndRisk.value.output_params.Risk) {
		priceAndRisk.value.output_params.Risk.map((arr) => {
			risk[arr[0]] = arr[1];
		});
		return risk;
	} else {
		return risk;
	}

	return (
		(priceAndRisk.value ? priceAndRisk.value.output_params.Risk : []) || []
	);
});

const portfolioNPV = computed(
	() =>
		(priceAndRisk.value
			? priceAndRisk.value.output_params.PortfolioNPV
			: 0) || 0
);

const currentValuationTime = computed(
	() =>
		(priceAndRisk.value ? priceAndRisk.value.output_params.CalcTime : 0) ||
		0
);

const irSwaps = computed(
	() => (portfolio.value ? portfolio.value.input_params.NumTrades : 0) || 0
);

const portfolioLoadTime = computed(
	() => (portfolio.value ? portfolio.value.output_params.CalcTime : 0) || 0
);

const tradesNPV = computed(() => {
	if (
		portfolio.value &&
		priceAndRisk.value &&
		portfolio.value.output_params.Trades &&
		priceAndRisk.value.output_params.TradeNPV
	) {
		let tradesNPV = priceAndRisk.value.output_params.TradeNPV.map(
			(npv, index) => {
				return { ...portfolio.value.output_params.Trades[index], npv };
			}
		);
		return tradesNPV;
	} else {
		return [];
	}
});

const validateSubmit = (e, callback) => {
	if (e.target.value !== "" && !!Number(e.target.value)) {
		if (
			isNaN(Number(e.target.value)) ||
			Number(e.target.value) < -3 ||
			Number(e.target.value) > 3
		) {
			alert.showAlert("Alert: Insert value between -0.05 and 0.05");
			return;
		} else {
			callback();
		}
	}
};

const handleUpdate = (e, key) => {
	validateSubmit(e, () => {
		updateParam(1, key, Number(e.target.value), true);
	});
};

const onRandomInput = (e) => {
	connection.setRandomizeData(!connection.randomizeData);
};

const throttle = useThrottle();

const riskBarRatio = ref(1);
const riskTableHeight = ref(500);
const tradesTableHeight = ref(500);

const handleSizing = (width) => {
	if (width >= 1200) {
		riskTableHeight.value = 500;
		tradesTableHeight.value = 500;
		riskBarRatio.value = 1;
	} else if (width >= 992) {
		riskBarRatio.value = 1.875;
	} else if (width >= 768) {
		riskTableHeight.value = 600;
		tradesTableHeight.value = 600;
		riskBarRatio.value = 1.5;
	} else {
		riskTableHeight.value = 500;
		tradesTableHeight.value = 500;
		riskBarRatio.value = 1;
	}
};

const onResize = (e) => {
	throttle(() => {
		const width = e.target.innerWidth;

		handleSizing(width);
	}, 100);
};

onMounted(() => {
	handleSizing(window.innerWidth);
	window.addEventListener("resize", onResize);
});

onUnmounted(() => {
	clearInterval(randomInputInterval.value);
	window.removeEventListener("resize", onResize);
});

watchEffect(() => {
	if (connection.randomizeData) {
		clearInterval(randomInputInterval.value);
		updateParam(1, "parallel_shift", generateRandomNumber(-3, 3), true);
		updateParam(1, "parallel_tilt", generateRandomNumber(-3, 3), true);
		updateParam(1, "parallel_twist", generateRandomNumber(-3, 3), true);

		randomInputInterval.value = setInterval(() => {
			updateParam(1, "parallel_shift", generateRandomNumber(-3, 3), true);
			updateParam(1, "parallel_tilt", generateRandomNumber(-3, 3), true);
			updateParam(1, "parallel_twist", generateRandomNumber(-3, 3), true);
		}, 2000);
	} else {
		clearInterval(randomInputInterval.value);
	}
});
</script>

<template>
	<div class="dashboard">
		<div class="dashboard__top">
			<h2 class="dashboard__heading heading">
				Market Data Stream Simulation
			</h2>
			<div class="dashboard__form">
				<label for="randomMarketData" class="dashboard__label"
					>Random Market Data</label
				>
				<div class="toggle">
					<input
						type="checkbox"
						v-model="isRandomDataOn"
						class="toggle__checkbox"
						id="randomMarketData"
						@input="onRandomInput"
					/>
					<label for="randomMarketData" class="toggle__wrapper">
						<div class="toggle__circle"></div>
					</label>
				</div>
				<label for="parallelShift" class="dashboard__label"
					>Parallel Shift</label
				>
				<RangeInput
					:onInput="(e) => handleUpdate(e, 'parallel_shift')"
					:options="{ min: -3, max: 3, step: 0.005 }"
					:value="
						priceAndRisk && priceAndRisk.input_params.parallel_shift
					"
				/>
				<label for="parallelTilt" class="dashboard__label"
					>Parallel Tilt</label
				>
				<RangeInput
					:onInput="(e) => handleUpdate(e, 'parallel_tilt')"
					:options="{ min: -3, max: 3, step: 0.005 }"
					:value="
						priceAndRisk && priceAndRisk.input_params.parallel_tilt
					"
				/>
				<label for="parallelTwist" class="dashboard__label"
					>Parallel Twist</label
				>
				<RangeInput
					:onInput="(e) => handleUpdate(e, 'parallel_twist')"
					:options="{ min: -3, max: 3, step: 0.005 }"
					:value="
						priceAndRisk && priceAndRisk.input_params.parallel_twist
					"
				/>
			</div>
		</div>
		<div class="dashboard__readonlies">
			<div class="dashboard__readonly readonly">
				<div class="readonly__label">PortfolioNPV</div>
				<div class="readonly__value">
					{{ portfolioNPV.toFixed(2) }}
				</div>
			</div>
			<div
				class="dashboard__readonly readonly dashboard__readonly--small"
			>
				<div class="readonly__label">IR SWAPS</div>
				<div class="readonly__value">
					{{ irSwaps }}
				</div>
			</div>
			<div class="dashboard__readonly readonly">
				<div class="readonly__label">Current valuation time</div>
				<div class="readonly__value">
					{{ currentValuationTime.toFixed(2) }}ms
				</div>
			</div>
			<div class="dashboard__readonly readonly">
				<div class="readonly__label">Portfolio Load Time</div>
				<div class="readonly__value">
					{{ portfolioLoadTime.toFixed(2) }}ms
				</div>
			</div>
		</div>
		<LineChart
			class="line--market-rates"
			label="Market Rates"
			:content="marketRates"
			:options="{
				showLabel: true,
				labels: ['Current Market Rates', 'Original Market Rates'],
			}"
			:colors="['rgb(52 211 153)', 'rgb(148 163 184)']"
		/>
		<LineChart
			class="line--fitted-values"
			label="Fitted Values"
			:content="fittedValues"
			:options="{
				showLabel: true,
				labels: ['Forecast', 'Discount'],
			}"
		/>
		<BarChart
			class="bar--risk"
			label="Risk Graph"
			:content="risk"
			:aspectRatio="riskBarRatio"
		/>
		<Table
			class="table--risk"
			label="Risk Table"
			:content="risk"
			:options="{ keyAndValue: true, headings: ['Key', 'Value'] }"
			:maxHeight="riskTableHeight"
		/>
		<Table
			class="table--trades"
			label="TradesNPV"
			:content="tradesNPV"
			:maxHeight="tradesTableHeight"
		/>
	</div>
</template>

<style lang="scss">
.dashboard {
	display: grid;
	gap: 2rem;
	grid-template-columns: repeat(6, 1fr);

	& > * {
		min-width: 0;
	}

	&__top {
		border: 2px dashed rgb(71 85 105);
		padding: 1.25rem;
		border-radius: 1rem;

		h2.heading {
			font-size: 1.125rem !important;
			@media screen and (min-width: 640px) {
				font-size: 1.25rem !important;
			}
		}

		@media screen and (min-width: 768px) {
			padding: 1.5rem;
		}
	}

	&__form {
		margin-top: 1.25rem;
		display: grid;
		gap: 0.875rem;

		& > * {
			min-width: 0;
		}

		@media screen and (min-width: 768px) {
			margin-top: 1.5rem;
			grid-template-columns: auto 1fr;
			align-items: center;
		}
	}

	&__input {
		display: flex;
		justify-content: space-between;
		align-items: center;
		width: 640px;
		padding: 0.25rem;
	}

	&__toggle-group {
		.toggle {
			width: 400px;
		}
	}

	&__label {
		font-weight: 500;
		font-size: 0.875rem;

		@media screen and (min-width: 768px) {
			font-size: 1rem;
		}
	}

	&__field {
		padding: 0.5rem;
		background: rgb(51 65 85);
		border-radius: 0.25rem;
		color: white;

		&:focus {
			outline: 1px solid rgb(226 232 240);
		}
	}

	&__bottom {
		margin-top: 2.5rem;
		display: flex;
		flex-direction: column;
		gap: 3rem;
	}

	&__readonlies {
		display: grid;
		grid-template-columns: 1fr;
		border: 2px dashed rgb(71 85 105);
		padding: 1.25rem;
		border-radius: 1rem;
		font-size: 0.875rem;
		gap: 1rem;
		justify-content: start;

		@media screen and (min-width: 640px) {
			padding: 1.5rem;
		}

		@media screen and (min-width: 768px) {
			grid-template-columns: 1fr 1fr;
			gap: 1.5rem;
		}

		@media screen and (min-width: 992px) {
			grid-template-columns: 1fr;
			gap: 1rem;
		}
	}

	&__readonly {
		flex: 2;
		&--small {
			flex: 1;
		}
	}

	&__outputs {
		display: flex;
		max-width: 100%;
		gap: 3rem;

		& > * {
			flex: 1;
		}
	}

	// grid childs

	&__top {
		grid-column: 1 / span 6;

		@media screen and (min-width: 992px) {
			grid-column: 1 / span 3;
		}
	}

	&__readonlies {
		grid-column: 1 / span 6;

		@media screen and (min-width: 992px) {
			grid-column: 4 / span 3;
			grid-row: 1 / span 1;
		}
	}

	.line--market-rates {
		grid-column: 1 / span 6;
		@media screen and (min-width: 768px) {
			grid-column: 1 / span 3;
		}

		@media screen and (min-width: 1200px) {
			grid-column: 1 / span 2;
		}
	}

	.line--fitted-values {
		grid-column: 1 / span 6;

		@media screen and (min-width: 768px) {
			grid-column: 4 / span 3;
		}

		@media screen and (min-width: 1200px) {
			grid-column: 3 / span 2;
		}
	}

	.bar--risk {
		grid-column: 1 / span 6;

		@media screen and (min-width: 1200px) {
			grid-column: 5 / span 2;
		}
	}

	.table--risk {
		grid-column: 1 / span 6;

		@media screen and (min-width: 768px) {
			grid-column: 5 / span 2;
		}

		@media screen and (min-width: 992px) {
			grid-column: 5 / span 2;
		}
	}

	.table--trades {
		grid-column: 1 / span 6;

		@media screen and (min-width: 768px) {
			grid-row: 5 / span 1;
			grid-column: 1 / span 4;
		}

		@media screen and (min-width: 992px) {
			grid-row: 4 / span 1;
		}

		@media screen and (min-width: 1200px) {
			grid-row: 3 / span 1;
		}
	}
}

.readonly {
	display: flex;
	justify-content: space-between;
	align-items: center;
	gap: 0.75rem;

	&__value {
		background: rgb(251, 191, 36);
		padding: 0.5rem;
		border-radius: 0.5rem;
		font-weight: bold;
		color: rgb(30, 41, 59);
	}
}

.toggle {
	&__checkbox {
		display: none;
	}

	&__checkbox:checked + &__wrapper {
		background: rgb(34 197 94);
		transition: all 0.2s ease;

		.toggle__circle {
			transform: translateX(1.75rem);
		}
	}

	&__wrapper {
		display: flex;
		align-items: center;
		height: 2rem;
		width: 3.5rem;
		border-radius: 0.5rem;
		background: rgb(51, 65, 85);
		cursor: pointer;
	}

	&__circle {
		height: 1.25rem;
		width: 1.25rem;
		background: #eee;
		border-radius: 0.25rem;
		transform: translateX(0.5rem);
		transition: all 0.2s ease;
	}
}
</style>
