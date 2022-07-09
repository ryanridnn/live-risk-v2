<script setup>
import { useAlertStore } from "../store";

const alert = useAlertStore();
</script>

<template>
	<div class="alert" :class="{ 'alert--show': alert.show }">
		<div class="alert__message">{{ alert.message }}</div>
	</div>
</template>

<style lang="scss">
.alert {
	position: fixed;
	top: 2rem;
	left: 50%;

	transform: translate(-50%, calc(-100% - 2rem));

	transition: all 0.2s ease;
	background: rgb(37, 99, 235);
	color: white;
	padding: 1rem;
	border-radius: 0.75rem;
	min-width: 600px;
	z-index: 100;

	&--show {
		transform: translate(-50%, 0);
	}
}
</style>
