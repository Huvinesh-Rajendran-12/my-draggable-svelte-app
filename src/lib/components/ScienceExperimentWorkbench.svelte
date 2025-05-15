<script lang="ts">
	import { writable, derived, type Writable, type Readable } from 'svelte/store';
	import { flip } from 'svelte/animate';
	import { v7 as uuidv7 } from 'uuid';

	// --- Type definitions ---
	interface PaletteBlock {
		type: string;
		label: string;
		icon: string; // Storing raw SVG string
		color: string; // Tailwind CSS class for background color
		defaultText: string;
		estimatedDuration: number; // in seconds for simulation
	}

	type PhaseStatus = 'pending' | 'running' | 'paused' | 'completed' | 'failed';

	interface ExperimentPhase {
		id: string;
		type: string;
		label: string;
		icon: string; // Raw SVG string
		color: string; // Tailwind CSS class for background color
		text: string;
		status: PhaseStatus;
		progress: number; // 0-100
		estimatedDuration: number; // in seconds
		actualDuration: string | null; // e.g., "15.23s"
		logs: string[];
		canBeRun: boolean; // Simple example flag, could be more complex
	}

	type DraggingItem = {
		type: 'paletteItem' | 'workflowItem';
		id?: string; // For workflow items
		data: PaletteBlock | ExperimentPhase;
	} | null;

	interface OverallProgress {
		averageProgress: number; // 0-100
		completedCount: number;
		totalCount: number;
		isComplete: boolean;
	}

	// --- Configuration for available experiment phase blocks in the palette ---
	const paletteBlocks: PaletteBlock[] = [
		{
			type: 'SAMPLE_PREP',
			label: 'Sample Preparation',
			icon: `<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6"><path stroke-linecap="round" stroke-linejoin="round" d="M19.5 14.25v-2.625a3.375 3.375 0 0 0-3.375-3.375h-1.5A1.125 1.125 0 0 1 13.5 7.125v-1.5a3.375 3.375 0 0 0-3.375-3.375H8.25m0 12.75h7.5m-7.5 3H12M10.5 2.25H5.625c-.621 0-1.125.504-1.125 1.125v17.25c0 .621.504 1.125 1.125 1.125h12.75c.621 0 1.125-.504 1.125-1.125V11.25a9 9 0 0 0-9-9Z" /></svg>`,
			color: 'bg-sky-500',
			defaultText: 'Prepare and calibrate samples...',
			estimatedDuration: 10 // in seconds for simulation
		},
		{
			type: 'DATA_ACQUISITION',
			label: 'Data Acquisition',
			icon: `<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6"><path stroke-linecap="round" stroke-linejoin="round" d="M3.75 4.875c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5A1.125 1.125 0 0 1 3.75 9.375v-4.5ZM3.75 14.625c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5a1.125 1.125 0 0 1-1.125-1.125v-4.5ZM13.5 4.875c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5a1.125 1.125 0 0 1-1.125-1.125v-4.5ZM13.5 14.625c0-.621.504-1.125 1.125-1.125h4.5c.621 0 1.125.504 1.125 1.125v4.5c0 .621-.504 1.125-1.125 1.125h-4.5a1.125 1.125 0 0 1-1.125-1.125v-4.5Z" /></svg>`,
			color: 'bg-indigo-500',
			defaultText: 'Collect data from sensors/instruments...',
			estimatedDuration: 20
		},
		{
			type: 'ANALYSIS_ALGORITHM',
			label: 'Analysis Algorithm',
			icon: `<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6"><path stroke-linecap="round" stroke-linejoin="round" d="M10.5 6h9.75M10.5 6a1.5 1.5 0 1 1-3 0m3 0a1.5 1.5 0 1 0-3 0M3.75 6H7.5m3 12h9.75m-9.75 0a1.5 1.5 0 0 1-3 0m3 0a1.5 1.5 0 0 0-3 0m-3.75 0H7.5m9-6h3.75m-3.75 0a1.5 1.5 0 0 1-3 0m3 0a1.5 1.5 0 0 0-3 0m-9.75 0h9.75" /></svg>`,
			color: 'bg-purple-600',
			defaultText: 'Run processing and analysis scripts...',
			estimatedDuration: 15
		},
		{
			type: 'VISUALIZE_RESULTS',
			label: 'Visualize Results',
			icon: `<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6"><path stroke-linecap="round" stroke-linejoin="round" d="M3 13.125C3 12.504 3.504 12 4.125 12h2.25c.621 0 1.125.504 1.125 1.125v6.75C7.5 20.496 6.996 21 6.375 21h-2.25A1.125 1.125 0 0 1 3 19.875v-6.75ZM9.75 8.625c0-.621.504-1.125 1.125-1.125h2.25c.621 0 1.125.504 1.125 1.125v11.25c0 .621-.504 1.125-1.125 1.125h-2.25A1.125 1.125 0 0 1 9.75 19.875V8.625ZM16.5 4.125c0-.621.504-1.125 1.125-1.125h2.25C20.496 3 21 3.504 21 4.125v15.75c0 .621-.504 1.125-1.125 1.125h-2.25a1.125 1.125 0 0 1-1.125-1.125V4.125Z" /></svg>`,
			color: 'bg-green-500',
			defaultText: 'Generate charts and visual outputs...',
			estimatedDuration: 8
		},
		{
			type: 'REPORT_GENERATION',
			label: 'Generate Report',
			icon: `<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke-width="1.5" stroke="currentColor" class="w-6 h-6"><path stroke-linecap="round" stroke-linejoin="round" d="M19.5 14.25v-2.625a3.375 3.375 0 0 0-3.375-3.375h-1.5A1.125 1.125 0 0 1 13.5 7.125v-1.5a3.375 3.375 0 0 0-3.375-3.375H8.25M9 17.25h6M16.03 17.25c.017-.005.034-.01.05-.015m-6.13 0c.017.005.034.01.05.015m5.98 0h.01M9.04 17.25h.01M12 3c-1.101 0-2.07.316-2.91.866M12 3c1.101 0 2.07.316 2.91.866m-5.82 0A21.016 21.016 0 0 0 3.75 6v10.5A2.25 2.25 0 0 0 6 18.75h12A2.25 2.25 0 0 0 20.25 16.5V6a21.016 21.016 0 0 0-2.41-.134M12 3v3.375c0 .621.504 1.125 1.125 1.125h1.5c.621 0 1.125-.504 1.125-1.125V3m-3.75 0V3.75M9.75 3v3.375c0 .621-.504 1.125-1.125 1.125h-1.5A1.125 1.125 0 0 1 6 6.375V3m5.25 9.75A1.5 1.5 0 0 1 10.5 15H9a1.5 1.5 0 0 1-1.5-1.5V9A1.5 1.5 0 0 1 9 7.5h1.5a1.5 1.5 0 0 1 1.5 1.5v3.75Z" /></svg>`,
			color: 'bg-amber-500',
			defaultText: 'Compile findings into a final report...',
			estimatedDuration: 12
		}
	];

	// --- Store for the current experiment phases on the canvas ---
	const experimentPhases: Writable<ExperimentPhase[]> = writable([]);

	// --- Drag and Drop State Variables ---
	let draggingItem: DraggingItem = null;
	let dragOverCanvas = false; // True if dragging over the canvas drop zone
	let dragOverPhaseId: string | null = null; // ID of the phase being dragged over (for reordering)

	// --- Simulation Interval Store ---
	// This will store interval IDs for each running phase to manage simulation
	let phaseIntervals: Record<string, number> = {};

	// --- Derived store for overall experiment progress ---
	const overallExperimentProgress: Readable<OverallProgress> = derived(
		experimentPhases,
		($phases) => {
			if ($phases.length === 0) {
				return {
					averageProgress: 0,
					completedCount: 0,
					totalCount: 0,
					isComplete: false
				};
			}
			const totalProgress = $phases.reduce((sum, phase) => sum + (phase.progress || 0), 0);
			const avgProgress = totalProgress / $phases.length;
			const completedPhases = $phases.filter((p) => p.status === 'completed').length;
			const totalPhases = $phases.length;
			return {
				averageProgress: Math.round(avgProgress),
				completedCount: completedPhases,
				totalCount: totalPhases,
				isComplete: completedPhases === totalPhases && totalPhases > 0
			};
		}
	);

	// --- Drag Handlers for Palette Items ---
	function handlePaletteDragStart(event: DragEvent, blockType: PaletteBlock) {
		if (event.dataTransfer) {
			event.dataTransfer.effectAllowed = 'copy';
			event.dataTransfer.setData('text/plain', blockType.type);
		}
		draggingItem = { type: 'paletteItem', data: blockType };
		dragOverCanvas = false;
		dragOverPhaseId = null;
	}

	// --- Drag Handlers for Canvas Area ---
	function handleCanvasDragEnter(event: DragEvent) {
		event.preventDefault();
		dragOverCanvas = true;
	}

	function handleCanvasDragOver(event: DragEvent) {
		event.preventDefault(); // Necessary to allow dropping
		if (event.dataTransfer) {
			if (draggingItem?.type === 'paletteItem') {
				event.dataTransfer.dropEffect = 'copy';
			} else if (draggingItem?.type === 'workflowItem') {
				// Allow moving workflow items within the canvas list
				event.dataTransfer.dropEffect = 'move';
			}
		}
		dragOverCanvas = true;
	}

	function handleCanvasDragLeave(event: DragEvent) {
		// Check if the mouse truly left the canvas or entered a child element
		if (event.currentTarget === event.target) {
			dragOverCanvas = false;
		}
	}

	function handleCanvasDrop(event: DragEvent) {
		event.preventDefault();
		if (draggingItem?.type === 'paletteItem') {
			// Add new item from palette
			const block = draggingItem.data as PaletteBlock; // Cast for type safety
			const newPhase: ExperimentPhase = {
				id: uuidv7(), // Generate a unique ID for the new phase
				type: block.type,
				label: block.label,
				icon: block.icon,
				color: block.color,
				text: block.defaultText || `New ${block.label} Phase`,
				status: 'pending', // Initial status
				progress: 0, // Initial progress
				estimatedDuration: block.estimatedDuration || 10, // seconds
				actualDuration: null,
				logs: [],
				canBeRun: true // Simple flag
			};
			experimentPhases.update((phases) => [...phases, newPhase]);
		}
		// Reordering logic is handled by individual phase item drop handlers
		dragOverCanvas = false;
		// draggingItem is reset in handleDragEnd
	}

	// --- Drag Handlers for Individual Experiment Phases (Reordering) ---
	function handlePhaseDragStart(event: DragEvent, phase: ExperimentPhase) {
		if (event.dataTransfer) {
			event.dataTransfer.effectAllowed = 'move';
			event.dataTransfer.setData('text/plain', phase.id); // Use phase.id for reordering
		}
		// Use type 'workflowItem' for items originating from the canvas itself
		draggingItem = { type: 'workflowItem', id: phase.id, data: phase };
		dragOverPhaseId = null;
	}

	function handlePhaseDragEnter(event: DragEvent, targetPhaseId: string) {
		event.preventDefault();
		if (draggingItem?.type === 'workflowItem' && draggingItem.id !== targetPhaseId) {
			dragOverPhaseId = targetPhaseId;
		}
	}

	function handlePhaseDragOver(event: DragEvent) {
		event.preventDefault();
		if (event.dataTransfer) {
			if (draggingItem?.type === 'workflowItem') {
				event.dataTransfer.dropEffect = 'move';
			} else {
				event.dataTransfer.dropEffect = 'none'; // Don't allow dropping palette items directly onto phases (only on canvas)
			}
		}
	}

	function handlePhaseDragLeave(event: DragEvent, targetPhaseId: string) {
		if (dragOverPhaseId === targetPhaseId) {
			dragOverPhaseId = null;
		}
	}

	function handlePhaseDrop(event: DragEvent, targetPhaseId: string) {
		event.preventDefault();
		event.stopPropagation(); // Prevent canvas drop from firing as well

		if (draggingItem?.type !== 'workflowItem' || !draggingItem.id) return;

		const sourcePhaseId = draggingItem.id;
		if (sourcePhaseId === targetPhaseId) {
			dragOverPhaseId = null;
			// draggingItem is reset in handleDragEnd
			return;
		}

		experimentPhases.update((currentPhases) => {
			const sourceIndex = currentPhases.findIndex((p) => p.id === sourcePhaseId);
			let targetIndex = currentPhases.findIndex((p) => p.id === targetPhaseId);

			if (sourceIndex === -1 || targetIndex === -1) return currentPhases;

			// Remove the dragged phase from its original position
			const [draggedPhase] = currentPhases.splice(sourceIndex, 1);

			// Insert the dragged phase at the target position
			// If dragging downwards, the targetIndex might have shifted. Re-find it or adjust.
			// A simpler way: splice removes the item, then we find the index to insert.
			// targetIndex might need adjustment if sourceIndex < targetIndex.
			// However, splice already accounts for this shift for elements *after* the source.
			// If sourceIndex < targetIndex, the elements at targetIndex and beyond shift left by 1.
			// If sourceIndex > targetIndex, elements don't shift relative to targetIndex.
			// The simplest reliable way is to find the new index of the target element after removal.
			const newTargetIndex = currentPhases.findIndex((p) => p.id === targetPhaseId);
			if (newTargetIndex !== -1) {
				currentPhases.splice(newTargetIndex + (sourceIndex < targetIndex ? 0 : 0), 0, draggedPhase); // Correction: insert *before* target
			} else {
				// Target might have been the removed item itself if it was the last one or something
				// This case should ideally not happen with the sourceId !== targetId check.
				// But as a fallback, re-add at the end.
				currentPhases.push(draggedPhase);
			}

			return currentPhases;
		});
		dragOverPhaseId = null;
		// draggingItem is reset in handleDragEnd
	}

	// --- General Drag End Handler ---
	function handleDragEnd() {
		draggingItem = null;
		dragOverCanvas = false;
		dragOverPhaseId = null;
	}

	// --- Function to remove a phase ---\
	function removePhase(phaseId: string) {
		experimentPhases.update((phases) => phases.filter((p) => p.id !== phaseId));
		// Clear simulation interval if it exists
		if (phaseIntervals[phaseId]) {
			clearInterval(phaseIntervals[phaseId]);
			delete phaseIntervals[phaseId];
		}
	}

	// --- Function to update phase text ---
	function updatePhaseText(phaseId: string, newText: string) {
		experimentPhases.update((phases) =>
			phases.map((p) => (p.id === phaseId ? { ...p, text: newText } : p))
		);
	}

	// --- Simulation Logic for Running a Phase ---
	function togglePhaseExecution(phaseId: string) {
		experimentPhases.update((phases) => {
			return phases.map((p) => {
				if (p.id === phaseId) {
					if (p.status === 'running') {
						// Pause functionality
						if (phaseIntervals[p.id]) clearInterval(phaseIntervals[p.id]);
						delete phaseIntervals[p.id];
						return { ...p, status: 'paused' };
					} else if (p.status === 'pending' || p.status === 'paused' || p.status === 'failed') {
						// Start or Resume
						// Clear any existing interval for this phase before starting a new one
						if (phaseIntervals[p.id]) clearInterval(phaseIntervals[p.id]);

						const startTime = Date.now();
						// Simulate progress every 200ms. Adjust for smoother/faster simulation.
						// Progress increment is based on estimated duration to make longer tasks take longer.
						// Divide estimated duration by interval to get number of steps. 100 / steps.
						const steps = p.estimatedDuration * (1000 / 200); // 200ms interval
						const progressIncrement = steps > 0 ? 100 / steps : 100; // Avoid division by zero

						phaseIntervals[p.id] = window.setInterval(() => {
							experimentPhases.update((current) => {
								return current.map((phaseToUpdate) => {
									if (phaseToUpdate.id === p.id && phaseToUpdate.status === 'running') {
										const newProgress = Math.min(100, phaseToUpdate.progress + progressIncrement);
										if (newProgress >= 100) {
											clearInterval(phaseIntervals[p.id]);
											delete phaseIntervals[p.id];
											const endTime = Date.now();
											const duration = ((endTime - startTime) / 1000).toFixed(2);
											return {
												...phaseToUpdate,
												progress: 100,
												status: 'completed',
												actualDuration: `${duration}s`,
												logs: [
													...phaseToUpdate.logs,
													`Phase completed in ${duration}s at ${new Date().toLocaleTimeString()}`
												]
											};
										}
										return { ...phaseToUpdate, progress: newProgress };
									}
									return phaseToUpdate;
								});
							});
						}, 200); // Run simulation step every 200ms
						return {
							...p,
							status: 'running',
							progress: p.status === 'paused' ? p.progress : 0, // Resume from paused progress
							logs: [...p.logs, `Phase execution started at ${new Date().toLocaleTimeString()}`]
						};
					} else if (p.status === 'completed') {
						// Reset and Re-run
						return {
							...p,
							status: 'pending',
							progress: 0,
							actualDuration: null,
							logs: [`Phase reset at ${new Date().toLocaleTimeString()}`]
						};
					}
				}
				return p;
			});
		});
	}

	// --- Function to run all pending phases sequentially ---
	async function runAllPhases() {
		const phasesToRun = $experimentPhases.filter(
			(p) => p.status === 'pending' || p.status === 'failed'
		);
		for (const phase of phasesToRun) {
			// Ensure phase is not already running or completed from a previous manual click or run
			const currentState = $experimentPhases.find((p) => p.id === phase.id);
			if (currentState && (currentState.status === 'pending' || currentState.status === 'failed')) {
				togglePhaseExecution(phase.id);
				// Wait for the current phase to complete or fail before starting the next one
				await new Promise<void>((resolve) => {
					const unsub = experimentPhases.subscribe((updatedPhases) => {
						const currentPhaseState = updatedPhases.find((p) => p.id === phase.id);
						// Resolve if the phase is completed or failed
						if (
							currentPhaseState?.status === 'completed' ||
							currentPhaseState?.status === 'failed'
						) {
							if (unsub) unsub(); // Unsubscribe once resolved
							resolve();
						}
					});
				});
			}
		}
	}

	// --- Helper to get status color ---
	function getStatusColor(status: PhaseStatus | undefined): string {
		switch (status) {
			case 'pending':
				return 'bg-gray-400';
			case 'running':
				return 'bg-blue-500 animate-pulse';
			case 'completed':
				return 'bg-green-500';
			case 'paused':
				return 'bg-yellow-500';
			case 'failed':
				return 'bg-red-500';
			default:
				return 'bg-gray-300'; // Should not happen with defined statuses
		}
	}

	// --- Helper to get button text for execution ---
	function getRunButtonText(status: PhaseStatus | undefined): string {
		switch (status) {
			case 'pending':
				return 'Run';
			case 'running':
				return 'Pause';
			case 'paused':
				return 'Resume';
			case 'completed':
				return 'Re-run';
			case 'failed':
				return 'Retry';
			default:
				return 'Run';
		}
	}

	// Cleanup intervals when component is destroyed
	import { onDestroy } from 'svelte';
	onDestroy(() => {
		for (const id in phaseIntervals) {
			clearInterval(phaseIntervals[id]);
		}
		phaseIntervals = {}; // Reset the object
	});
</script>

<!-- Main Layout -->
<div class="flex h-screen flex-col bg-gradient-to-br from-gray-900 via-gray-900 to-gray-800 font-sans text-white md:flex-row">
	<!-- Sidebar Palette -->
	<aside class="w-full space-y-3 overflow-y-auto bg-gray-800/90 p-6 shadow-xl border-r border-gray-700/30 backdrop-blur-sm md:w-72 lg:w-80">
		<h2 class="mb-6 border-b border-gray-700/70 pb-3 text-2xl font-bold text-sky-400 tracking-wide">
			Experiment Phases
		</h2>
		{#each paletteBlocks as block (block.type)}
			<div
				draggable="true"
				on:dragstart={(event) => handlePaletteDragStart(event, block)}
				on:dragend={handleDragEnd}
				class="p-4 from-{block.color.replace('bg-', '')} to-{block.color.replace('bg-', '')}/80 bg-gradient-to-br flex cursor-grab items-center space-x-3 rounded-xl text-white shadow-md transition-all duration-200 hover:shadow-lg hover:translate-y-[-2px] hover:scale-[1.02] border border-white/10"
				role="button"
				tabindex="0"
				aria-label={`Add ${block.label} phase`}
			>
				<div class="h-7 w-7 flex-shrink-0" aria-hidden="true">
					<!-- Using @html to render SVG string -->
					{@html block.icon}
				</div>
				<span class="font-medium">{block.label}</span>
			</div>
		{/each}
	</aside>

	<!-- Main Canvas Area -->
	<main
		class="flex-1 overflow-y-auto p-5 md:p-7 lg:p-10 bg-gray-900/60 backdrop-blur-sm"
		on:dragenter={handleCanvasDragEnter}
		on:dragover={handleCanvasDragOver}
		on:dragleave={handleCanvasDragLeave}
		on:drop={handleCanvasDrop}
	>
		<!-- Canvas Header -->
		<div class="mb-8 flex items-center justify-between border-b border-gray-700/60 pb-5">
			<h1 class="text-3xl font-extrabold text-sky-300 md:text-4xl tracking-wide">Experiment Canvas</h1>
			{#if $experimentPhases.length > 0}
				<button
					on:click={runAllPhases}
					class="rounded-lg px-5 py-2.5 font-semibold text-white transition-all duration-200 shadow-lg transform hover:translate-y-[-2px] hover:shadow-xl active:translate-y-0 active:shadow-md
                           {$experimentPhases.some((p) => p.status === 'running')
						? 'bg-gradient-to-r from-yellow-600 to-amber-600 hover:from-yellow-700 hover:to-amber-700'
						: 'bg-gradient-to-r from-green-600 to-emerald-600 hover:from-green-700 hover:to-emerald-700'}
                           disabled:cursor-not-allowed disabled:opacity-50"
					disabled={$experimentPhases.length === 0 || $overallExperimentProgress.isComplete}
				>
					{#if $overallExperimentProgress.isComplete}
						Experiment Complete!
					{:else if $experimentPhases.some((p) => p.status === 'running')}
						Running All...
					{:else}
						Run All Pending
					{/if}
				</button>
			{/if}
		</div>

		<!-- Overall Progress Bar (Visible only if phases exist) -->
		{#if $experimentPhases.length > 0}
			<div class="mb-8 rounded-xl bg-gray-800/80 p-5 shadow-lg border border-gray-700/50 backdrop-blur-sm">
				<h3 class="mb-3 text-xl font-bold text-sky-400">Overall Progress</h3>
				<div class="flex items-center space-x-4">
					<div class="flex-1">
						<div class="h-4 w-full rounded-full bg-gray-700/80 shadow-inner">
							<div
								class="h-4 rounded-full bg-gradient-to-r from-sky-600 to-sky-400 transition-all duration-500 ease-in-out shadow-lg"
								style="width: {$overallExperimentProgress.averageProgress}%;"
								aria-valuenow={$overallExperimentProgress.averageProgress}
								aria-valuemin="0"
								aria-valuemax="100"
								role="progressbar"
							></div>
						</div>
					</div>
					<span class="text-sm font-bold text-sky-300">
						{$overallExperimentProgress.averageProgress}%
					</span>
					<span class="text-sm font-medium text-gray-400">
						({$overallExperimentProgress.completedCount}/{$overallExperimentProgress.totalCount}
						completed)
					</span>
				</div>
				{#if $overallExperimentProgress.isComplete}
					<p class="mt-3 font-medium text-green-400 flex items-center"><span class="mr-2 text-lg">✓</span>Experiment successfully completed!</p>
				{/if}
			</div>
		{/if}

		<!-- Empty Canvas Message -->
		{#if $experimentPhases.length === 0}
			<div
				class="flex min-h-[300px] flex-col items-center justify-center rounded-xl border-2 border-dashed p-8 text-gray-500
               {dragOverCanvas && draggingItem?.type === 'paletteItem'
					? 'border-sky-500 bg-sky-900/30 shadow-inner'
					: 'border-gray-700/70 bg-gray-800/30'}
               transition-all duration-300"
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					class="mb-5 h-20 w-20 text-sky-800 opacity-70"
					fill="none"
					viewBox="0 0 24 24"
					stroke="currentColor"
					stroke-width="1"
					aria-hidden="true"
				>
					<path
						stroke-linecap="round"
						stroke-linejoin="round"
						d="M9.75h6.75m-6.75 0a3 3 0 01-3-3h12.75a3 3 0 01-3 3m0 0a3 3 0 00-3 3h.375m6.375 0h.375a3 3 0 003-3V6.75A3 3 0 0016.5 3.75h-9a3 3 0 00-3 3V15a3 3 0 003 3zm-3.75-9h15m-15 0v9A1.5 1.5 0 007.5 18h9a1.5 1.5 0 001.5-1.5v-9A1.5 1.5 0 0016.5 6H7.5A1.5 1.5 0 006 7.5z"
					/>
				</svg>
				<p class="text-xl font-bold text-sky-400 mb-2">Empty Canvas</p>
				<p class="text-base text-gray-400 max-w-md text-center">
					Drag phases from the left panel and drop them here to build your experiment workflow.
				</p>
			</div>
		{:else}
			<!-- Experiment Phases List -->
			<ul
				class="min-h-[300px] space-y-5 rounded-xl p-2
               {dragOverCanvas &&
				draggingItem?.type === 'paletteItem' &&
				$experimentPhases.length > 0
					? 'bg-sky-900/30 outline-2 outline-sky-500 outline-dashed shadow-inner'
					: ''}
               transition-all duration-300"
				role="list"
			>
				{#each $experimentPhases as phase (phase.id)}
					<li
						draggable="true"
						on:dragstart={(event) => handlePhaseDragStart(event, phase)}
						on:dragenter={(event) => handlePhaseDragEnter(event, phase.id)}
						on:dragover={handlePhaseDragOver}
						on:dragleave={(event) => handlePhaseDragLeave(event, phase.id)}
						on:drop={(event) => handlePhaseDrop(event, phase.id)}
						on:dragend={handleDragEnd}
						role="listitem"
						class="p-5 from-{phase.color.replace('bg-', '')} to-{phase.color.replace('bg-', '')}/90 bg-gradient-to-br group relative flex cursor-grab flex-col rounded-xl text-white shadow-lg border border-white/10 backdrop-blur-sm backdrop-saturate-150
                   transition-all duration-200 ease-in-out 
                   {draggingItem?.id === phase.id
							? 'scale-95 opacity-60 shadow-xl rotate-1'
							: 'scale-100 opacity-100 hover:shadow-xl hover:-translate-y-1'}
                   {dragOverPhaseId === phase.id && draggingItem?.id !== phase.id
							? 'ring-4 ring-sky-400 ring-offset-2 ring-offset-gray-900'
							: ''}"
						animate:flip={{ duration: 300 }}
					>
						<!-- Phase Header -->
						<div class="mb-3 flex items-center justify-between">
							<div class="flex items-center space-x-3">
								<div
									class="h-7 w-7 flex-shrink-0 opacity-80 transition-opacity group-hover:opacity-100"
									aria-hidden="true"
								>
									<!-- Using @html to render SVG string -->
									{#if phase.icon}{@html phase.icon}{/if}
								</div>
								<span class="text-lg font-semibold transition-colors group-hover:text-sky-300"
									>{phase.label}</span
								>
							</div>
							<button
								on:click={() => removePhase(phase.id)}
								class="rounded-full bg-black/20 p-1 text-white opacity-0 transition-opacity group-hover:opacity-100 hover:bg-black/40 hover:text-red-200"
								aria-label={`Remove ${phase.label} phase`}
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									class="h-5 w-5"
									fill="none"
									viewBox="0 0 24 24"
									stroke="currentColor"
									stroke-width="2"
									aria-hidden="true"
								>
									<path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
								</svg>
							</button>
						</div>

						<!-- Phase Description Input -->
						<textarea
							bind:value={phase.text}
							on:change={(e) => updatePhaseText(phase.id, (e.target as HTMLTextAreaElement).value)}
							class="mb-4 w-full resize-none rounded-lg border border-gray-600/50 bg-gray-700/50 p-3 text-sm text-gray-200 placeholder-gray-500 outline-none shadow-inner transition-all duration-200 focus:bg-gray-700/80 focus:ring-2 focus:ring-sky-500 focus:shadow-md"
							placeholder="Describe this phase or add notes..."
							rows="2"
							aria-label={`Description for ${phase.label} phase`}
						></textarea>

						<!-- Phase Progress Bar -->
						<div class="mb-4 flex items-center space-x-3">
							<div class="h-3 w-full rounded-full bg-gray-700/70 shadow-inner">
								<div
									class="h-3 rounded-full shadow-md {getStatusColor(phase.status).replace('bg-', 'bg-gradient-to-r from-').replace('-500', '-600 to-').replace('-500', '-400')}"
									style="width: {phase.progress}%;"
									aria-valuenow={phase.progress}
									aria-valuemin="0"
									aria-valuemax="100"
									role="progressbar"
								></div>
							</div>
							<span class="text-sm font-bold text-sky-300">{Math.round(phase.progress)}%</span>
						</div>

						<!-- Phase Status and Duration -->
						<div class="flex items-center justify-between text-xs text-gray-400">
							<div class="flex items-center space-x-2">
								<span
									class="rounded-full px-2.5 py-0.5 text-[10px] font-bold text-white uppercase ring-1 ring-white/20 shadow-sm {getStatusColor(
										phase.status
									).replace('bg-', 'bg-gradient-to-r from-').replace('-500', '-600 to-').replace('-500', '-400')}"
								>
									{phase.status}
								</span>
								{#if phase.actualDuration}
									<span class="flex items-center space-x-1.5">
									<span class="font-semibold text-sky-300">Actual:</span>
									<span class="font-medium">{phase.actualDuration}</span>
									</span>
								{:else if phase.status === 'pending'}
									<span class="flex items-center space-x-1.5">
										<span class="font-semibold text-emerald-300">Estimated:</span>
										<span class="font-medium">{phase.estimatedDuration}s</span>
									</span>
								{/if}
							</div>
							<!-- Run/Pause/Resume Button -->
							<button
								on:click={() => togglePhaseExecution(phase.id)}
								class="rounded-lg bg-gray-700 px-3.5 py-1.5 text-xs font-semibold text-white hover:bg-gray-600 shadow hover:shadow-md transition-all duration-200 hover:-translate-y-0.5 active:translate-y-0 disabled:cursor-not-allowed disabled:opacity-50 border border-gray-600/30"
								disabled={!phase.canBeRun && phase.status !== 'running'}
								aria-label="{getRunButtonText(phase.status)} {phase.label} phase"
							>
								{getRunButtonText(phase.status)}
							</button>
						</div>

						<!-- Phase Logs (Visible if logs exist) -->
						{#if phase.logs && phase.logs.length > 0}
							<div class="mt-4 border-t border-gray-700/30 pt-3">
								<details class="text-xs group">
									<summary class="cursor-pointer flex items-center font-medium text-gray-500 hover:text-sky-300 transition-colors duration-200">
										<svg xmlns="http://www.w3.org/2000/svg" class="h-3 w-3 mr-1.5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
											<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
										</svg>
										Logs ({phase.logs.length})
									</summary>
									<ul class="mt-2 max-h-24 space-y-1 overflow-y-auto pl-2 pr-1 py-1.5 bg-black/20 rounded-lg shadow-inner">
										{#each phase.logs as logMsg, i (i)}
											<li class="font-mono text-gray-400 text-[10px] leading-tight">{logMsg}</li>
										{/each}
									</ul>
								</details>
							</div>
						{/if}
					</li>
				{/each}
			</ul>
		{/if}

		<!-- Debug JSON Output (Visible if phases exist) -->
		{#if $experimentPhases.length > 0}
			<div class="mt-10 rounded-xl bg-gradient-to-br from-gray-800 to-gray-900 p-5 shadow-lg border border-gray-700/30">
				<h3 class="mb-3 text-xl font-bold text-sky-400">Experiment Data Structure (JSON)</h3>
				<!-- Using standard text binding for safer JSON output -->
				<pre
					class="max-h-60 overflow-y-auto rounded-lg bg-black/30 p-4 text-xs break-all whitespace-pre-wrap shadow-inner">{JSON.stringify(
						$experimentPhases,
						null,
						2
					)}</pre>
			</div>
		{/if}
	</main>
</div>
```

<!-- Styles -->
<style>
	/* Using Tailwind CSS classes primarily. */

	/* Minor style to improve drag image (browser dependent) */
	[draggable='true'] {
		user-select: none; /* Prevent text selection when dragging */
	}

	/* Custom scrollbar styles for log output */
	.max-h-20::-webkit-scrollbar {
		width: 8px;
	}

	.max-h-20::-webkit-scrollbar-track {
		background: #374151; /* gray-700 */
		border-radius: 4px;
	}

	.max-h-20::-webkit-scrollbar-thumb {
		background: #6b7280; /* gray-500 */
		border-radius: 4px;
	}

	.max-h-20::-webkit-scrollbar-thumb:hover {
		background: #9ca3af; /* gray-400 */
	}

	/* Custom scrollbar styles for JSON output */
	pre::-webkit-scrollbar {
		height: 8px;
		width: 8px;
	}

	pre::-webkit-scrollbar-track {
		background: #1f2937; /* gray-800 or gray-900 */
		border-radius: 4px;
	}

	pre::-webkit-scrollbar-thumb {
		background: #4b5563; /* gray-600 */
		border-radius: 4px;
	}

	pre::-webkit-scrollbar-thumb:hover {
		background: #6b7280; /* gray-500 */
	}
</style>
