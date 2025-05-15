<script>
	import ScienceExperimentWorkbench from '$lib/components/ScienceExperimentWorkbench.svelte';
</script>

<svelte:head>
	<title>Draggable Fun</title>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="flex min-h-screen flex-col justify-center bg-gray-100 py-6 sm:py-12">
	<ScienceExperimentWorkbench />
</div>

<style>
	/* Ensure your global styles or app.html includes Tailwind CSS */
	/* You might need a global.css file imported in your +layout.svelte */
	/* For example, in src/app.html, ensure you have Tailwind linked or set up via postcss */
</style>
