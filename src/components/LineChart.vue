<script setup>
import { defineProps, computed, watchEffect } from "vue";

import { Line } from "vue-chartjs";
import { Chart as ChartJS, registerables } from "chart.js";

ChartJS.register(...registerables);

const props = defineProps({
	label: String,
	content: Array,
	options: {
		type: Object,
		default: {
			showLabel: false,
			labels: [],
		},
	},
	colors: {
		type: Array,
		default: ["rgb(52 211 153)", "rgb(217 70 239)"],
	},
});

const chartData = computed(() => {
	return {
		labels: Object.keys(props.content[0]),
		datasets: props.content.map((cont, index) => {
			return {
				label: props.options.labels[index],
				data: Object.values(cont),
				borderColor: props.colors[index % props.colors.length],
				lineTension: 0.6,
			};
		}),
	};
});

const options = {
	responsive: true,
	maintainAspectRatio: true,
	interaction: {
		intersect: false,
		mode: "index",
	},
	scales: {
		y: {
			ticks: {
				color: "#bbb",
			},
			grid: {
				color: "#2f2f2f",
			},
		},
		x: {
			ticks: {
				color: "#bbb",
			},
			grid: {
				color: "#2f2f2f",
			},
		},
	},
	plugins: {
		title: {
			display: false,
		},
		legend: {
			position: "bottom",
			display: props.options.showLabel,
			labels: {
				color: "#ddd",
			},
			boxHeight: 200,
		},
	},
};
</script>

<template>
	<div class="line" v-if="Object.keys(props.content).length > 0">
		<h3 class="heading">
			{{ props.label }}
		</h3>
		<Line :chart-data="chartData" :chart-options="options" />
	</div>
</template>

<style lang="scss">
.line {
	h3.heading {
		margin-bottom: 1.25rem;
	}

	canvas {
		width: 48%;
	}
}
</style>
