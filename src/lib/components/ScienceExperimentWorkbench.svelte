<script lang="ts">
	import { writable, derived, type Writable, type Readable } from 'svelte/store';
	import { flip } from 'svelte/animate';
	import { v7 as uuidv7 } from 'uuid'; // User is using v7
	// Import the necessary type for components
	// import type { ComponentType } from 'svelte';

	// --- Icon Component Imports (assuming they are in a subfolder) ---
	import SamplePrepIcon from './icons/SamplePrepIcon.svelte';
	import DataAcquisitionIcon from './icons/DataAcquisitionIcon.svelte';
	import AnalysisAlgorithmIcon from './icons/AnalysisAlgorithmIcon.svelte';
	import VisualizeResultsIcon from './icons/VisualizeResultsIcon.svelte';
	import ReportGenerationIcon from './icons/ReportGenerationIcon.svelte';

	// --- Type definitions ---
	interface PaletteBlock {
		type: string;
		label: string;
		iconComponent: typeof SamplePrepIcon; // Using the actual component type
		color: string; // Tailwind CSS class for background color
		defaultText: string;
		estimatedDuration: number; // in seconds for simulation
	}

	type PhaseStatus = 'pending' | 'running' | 'paused' | 'completed' | 'failed';

	interface ExperimentPhase {
		id: string;
		type: string;
		label: string;
		iconComponent: typeof SamplePrepIcon; // Using the actual component type
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
		data: PaletteBlock | ExperimentPhase; // This will hold the full block or phase data
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
			iconComponent: SamplePrepIcon,
			color: 'bg-sky-500', // Used for gradient in template: from-sky-500
			defaultText: 'Prepare and calibrate samples...',
			estimatedDuration: 10
		},
		{
			type: 'DATA_ACQUISITION',
			label: 'Data Acquisition',
			iconComponent: DataAcquisitionIcon,
			color: 'bg-indigo-500', // from-indigo-500
			defaultText: 'Collect data from sensors/instruments...',
			estimatedDuration: 20
		},
		{
			type: 'ANALYSIS_ALGORITHM',
			label: 'Analysis Algorithm',
			iconComponent: AnalysisAlgorithmIcon,
			color: 'bg-purple-600', // from-purple-600
			defaultText: 'Run processing and analysis scripts...',
			estimatedDuration: 15
		},
		{
			type: 'VISUALIZE_RESULTS',
			label: 'Visualize Results',
			iconComponent: VisualizeResultsIcon,
			color: 'bg-green-500', // from-green-500
			defaultText: 'Generate charts and visual outputs...',
			estimatedDuration: 8
		},
		{
			type: 'REPORT_GENERATION',
			label: 'Generate Report',
			iconComponent: ReportGenerationIcon,
			color: 'bg-amber-500', // from-amber-500
			defaultText: 'Compile findings into a final report...',
			estimatedDuration: 12
		}
	];

	// --- Store for the current experiment phases on the canvas ---
	const experimentPhases: Writable<ExperimentPhase[]> = writable([]);

	// --- Drag and Drop State Variables ---
	let draggingItem: DraggingItem = null;
	let dragOverCanvas = false;
	let dragOverPhaseId: string | null = null;

	// --- Simulation Interval Store ---
	let phaseIntervals: Record<string, number> = {}; // Store interval IDs (number from setInterval)

	// --- Derived store for overall experiment progress ---
	const overallExperimentProgress: Readable<OverallProgress> = derived(
		experimentPhases,
		($phases) => {
			if ($phases.length === 0) {
				return { averageProgress: 0, completedCount: 0, totalCount: 0, isComplete: false };
			}
			const totalProgress = $phases.reduce((sum, phase) => sum + (phase.progress || 0), 0);
			const avgProgress = $phases.length > 0 ? totalProgress / $phases.length : 0;
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
			event.dataTransfer.setData('text/plain', blockType.type); // Keep type for potential other uses
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
		event.preventDefault();
		if (event.dataTransfer) {
			if (draggingItem?.type === 'paletteItem') {
				event.dataTransfer.dropEffect = 'copy';
			} else if (draggingItem?.type === 'workflowItem') {
				event.dataTransfer.dropEffect = 'move';
			}
		}
		dragOverCanvas = true;
	}

	function handleCanvasDragLeave(event: DragEvent) {
		if (event.currentTarget === event.target) {
			dragOverCanvas = false;
		}
	}

	function handleCanvasDrop(event: DragEvent) {
		event.preventDefault();
		if (draggingItem?.type === 'paletteItem') {
			const block = draggingItem.data as PaletteBlock;
			const newPhase: ExperimentPhase = {
				id: uuidv7(),
				type: block.type,
				label: block.label,
				iconComponent: block.iconComponent, // Use iconComponent
				color: block.color,
				text: block.defaultText || `New ${block.label} Phase`,
				status: 'pending',
				progress: 0,
				estimatedDuration: block.estimatedDuration || 10,
				actualDuration: null,
				logs: [],
				canBeRun: true
			};
			experimentPhases.update((phases) => [...phases, newPhase]);
		}
		dragOverCanvas = false;
		// draggingItem is reset in handleDragEnd
	}

	// --- Drag Handlers for Individual Experiment Phases (Reordering) ---
	function handlePhaseDragStart(event: DragEvent, phase: ExperimentPhase) {
		if (event.dataTransfer) {
			event.dataTransfer.effectAllowed = 'move';
			event.dataTransfer.setData('text/plain', phase.id);
		}
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
				event.dataTransfer.dropEffect = 'none';
			}
		}
	}

	function handlePhaseDragLeave(event: DragEvent, targetPhaseId: string) {
		if (dragOverPhaseId === targetPhaseId) {
			dragOverPhaseId = null;
		}
	}

	// Preserving user's reordering logic from their provided script
	function handlePhaseDrop(event: DragEvent, targetPhaseId: string) {
		event.preventDefault();
		event.stopPropagation();

		if (draggingItem?.type !== 'workflowItem' || !draggingItem.id) return;

		const sourcePhaseId = draggingItem.id;
		if (sourcePhaseId === targetPhaseId) {
			dragOverPhaseId = null;
			return;
		}

		experimentPhases.update((currentPhases) => {
			const sourceIndex = currentPhases.findIndex((p) => p.id === sourcePhaseId);
			let targetIndex = currentPhases.findIndex((p) => p.id === targetPhaseId); // Original target index

			if (sourceIndex === -1 || targetIndex === -1) return currentPhases;

			const [draggedPhase] = currentPhases.splice(sourceIndex, 1);

			// After removing, the targetIndex might need adjustment if sourceIndex was before it.
			// However, we want to insert *before* the original target item.
			// If sourceIndex < targetIndex, the original targetItem has shifted left by 1.
			// So, the new insertion index is the original targetIndex - 1 (if targetIndex was its original pos).
			// Or, more simply, find the index of targetPhaseId *again* in the modified array.

			const newTargetActualIndex = currentPhases.findIndex((p) => p.id === targetPhaseId);

			if (newTargetActualIndex !== -1) {
				// Determine if inserting before or after the target based on drag direction
				// This logic can be tricky. A common approach is to insert at the found newTargetActualIndex
				// if dragging downwards onto the top half of an item, or newTargetActualIndex + 1 for bottom half.
				// For simplicity here, we'll insert at the position of the target item, effectively placing it before.
				// If sourceIndex < original targetIndex, it means we dragged down.
				// If sourceIndex > original targetIndex, it means we dragged up.
				currentPhases.splice(newTargetActualIndex, 0, draggedPhase);
			} else {
				// This case implies the targetId was not found after splice, which is unusual
				// unless targetId was the same as sourceId (already handled).
				// Fallback: if the original targetIndex was at the end after removal.
				if (targetIndex > sourceIndex && targetIndex === currentPhases.length) {
					// If it was the last element
					currentPhases.push(draggedPhase);
				} else if (targetIndex < sourceIndex) {
					currentPhases.splice(targetIndex, 0, draggedPhase);
				} else {
					// Default to end if logic gets complicated
					currentPhases.push(draggedPhase);
				}
			}
			return currentPhases;
		});
		dragOverPhaseId = null;
	}

	// --- General Drag End Handler ---
	function handleDragEnd() {
		draggingItem = null;
		dragOverCanvas = false;
		dragOverPhaseId = null;
	}

	// --- Function to remove a phase ---
	function removePhase(phaseId: string) {
		experimentPhases.update((phases) => phases.filter((p) => p.id !== phaseId));
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
						if (phaseIntervals[p.id]) clearInterval(phaseIntervals[p.id]);
						delete phaseIntervals[p.id];
						return { ...p, status: 'paused' as PhaseStatus };
					} else if (p.status === 'pending' || p.status === 'paused' || p.status === 'failed') {
						if (phaseIntervals[p.id]) clearInterval(phaseIntervals[p.id]);
						const startTime = Date.now();
						const steps = p.estimatedDuration * (1000 / 200);
						const progressIncrement = steps > 0 ? 100 / steps : 100;

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
												status: 'completed' as PhaseStatus,
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
						}, 200);
						return {
							...p,
							status: 'running' as PhaseStatus,
							progress: p.status === 'paused' ? p.progress : 0,
							logs: [...p.logs, `Phase execution started at ${new Date().toLocaleTimeString()}`]
						};
					} else if (p.status === 'completed') {
						return {
							...p,
							status: 'pending' as PhaseStatus,
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
		// Get a snapshot of phases to run to avoid issues with store updates during iteration
		let phasesToRunSnapshot: ExperimentPhase[] = [];
		// Immediately invoke subscribe to get current value and unsubscribe
		experimentPhases.subscribe((currentPhases) => {
			phasesToRunSnapshot = currentPhases.filter(
				(p) => p.status === 'pending' || p.status === 'failed'
			);
		})(); // Immediately invoke to get current value and unsubscribe

		for (const phase of phasesToRunSnapshot) {
			// Check the *current* state of the phase from the store before toggling
			let currentPhaseState: ExperimentPhase | undefined;
			// Immediately invoke subscribe to get current value and unsubscribe
			experimentPhases.subscribe((updatedPhases) => {
				currentPhaseState = updatedPhases.find((p) => p.id === phase.id);
			})();

			if (
				currentPhaseState &&
				(currentPhaseState.status === 'pending' || currentPhaseState.status === 'failed')
			) {
				togglePhaseExecution(phase.id);
				// Wait for the phase to complete or fail
				await new Promise<void>((resolve) => {
					const unsubWait = experimentPhases.subscribe((updatedPhases) => {
						const phaseAfterToggle = updatedPhases.find((p) => p.id === phase.id);
						if (phaseAfterToggle?.status === 'completed' || phaseAfterToggle?.status === 'failed') {
							// Ensure we only unsubscribe if unsubWait is assigned (which it always will be here)
							unsubWait();
							resolve();
						}
					});
				});
			}
		}
	}

	// --- Helper to get status color ---
	function getStatusColor(status: PhaseStatus | undefined): string {
		// Palette colors are like 'bg-sky-500'. We need to transform them for gradient.
		// Example: 'bg-sky-500' -> 'from-sky-600 to-sky-400' (adjust shades as needed)
		const colorMap: Record<PhaseStatus, string> = {
			pending: 'from-gray-500 to-gray-400',
			running: 'from-blue-600 to-blue-400 animate-pulse', // Pulse on the gradient
			completed: 'from-green-600 to-green-400',
			paused: 'from-yellow-500 to-yellow-400',
			failed: 'from-red-600 to-red-400'
		};
		return status ? colorMap[status] || 'from-gray-400 to-gray-300' : 'from-gray-400 to-gray-300';
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
		phaseIntervals = {};
	});
</script>

<div
	class="flex h-screen flex-col bg-gradient-to-br from-gray-900 via-gray-900 to-gray-800 font-sans text-white md:flex-row"
>
	<aside
		class="w-full space-y-3 overflow-y-auto border-r border-gray-700/30 bg-gray-800/90 p-6 shadow-xl backdrop-blur-sm md:w-72 lg:w-80"
	>
		<h2 class="mb-6 border-b border-gray-700/70 pb-3 text-2xl font-bold tracking-wide text-sky-400">
			Experiment Phases
		</h2>
		{#each paletteBlocks as block (block.type)}
			<div
				draggable="true"
				on:dragstart={(event) => handlePaletteDragStart(event, block)}
				on:dragend={handleDragEnd}
				class="p-4 from-{block.color.replace('bg-', '')} to-{block.color.replace(
					'bg-',
					''
				)}/80 flex cursor-grab items-center space-x-3 rounded-xl border border-white/10 bg-gradient-to-br text-white shadow-md transition-all duration-200 hover:translate-y-[-2px] hover:scale-[1.02] hover:shadow-lg"
				role="button"
				tabindex="0"
				aria-label={`Add ${block.label} phase`}
			>
				<div class="h-7 w-7 flex-shrink-0" aria-hidden="true">
					<svelte:component this={block.iconComponent} />
				</div>
				<span class="font-medium">{block.label}</span>
			</div>
		{/each}
	</aside>

	<main
		class="flex-1 overflow-y-auto bg-gray-900/60 p-5 backdrop-blur-sm md:p-7 lg:p-10"
		on:dragenter={handleCanvasDragEnter}
		on:dragover={handleCanvasDragOver}
		on:dragleave={handleCanvasDragLeave}
		on:drop={handleCanvasDrop}
	>
		<div class="mb-8 flex items-center justify-between border-b border-gray-700/60 pb-5">
			<h1 class="text-3xl font-extrabold tracking-wide text-sky-300 md:text-4xl">
				Experiment Canvas
			</h1>
			{#if $experimentPhases.length > 0}
				<button
					on:click={runAllPhases}
					class="transform rounded-lg px-5 py-2.5 font-semibold text-white shadow-lg transition-all duration-200 hover:translate-y-[-2px] hover:shadow-xl active:translate-y-0 active:shadow-md
                           {$experimentPhases.some((p) => p.status === 'running')
						? 'bg-gradient-to-r from-yellow-600 to-amber-600 hover:from-yellow-700 hover:to-amber-700'
						: 'bg-gradient-to-r from-green-600 to-emerald-600 hover:from-green-700 hover:to-emerald-700'}
                           disabled:cursor-not-allowed disabled:opacity-50"
					disabled={$experimentPhases.some((p) => p.status === 'running') &&
						!$overallExperimentProgress.isComplete}
				>
					{#if $overallExperimentProgress.isComplete}
						Run All Again
					{:else if $experimentPhases.some((p) => p.status === 'running')}
						Experiment Running...
					{:else}
						Run All Pending
					{/if}
				</button>
			{/if}
		</div>

		{#if $experimentPhases.length > 0}
			<div
				class="mb-8 rounded-xl border border-gray-700/50 bg-gray-800/80 p-5 shadow-lg backdrop-blur-sm"
			>
				<h3 class="mb-3 text-xl font-bold text-sky-400">Overall Progress</h3>
				<div class="flex items-center space-x-4">
					<div class="flex-1">
						<div class="h-4 w-full rounded-full bg-gray-700/80 shadow-inner">
							<div
								class="h-4 rounded-full bg-gradient-to-r from-sky-600 to-sky-400 shadow-lg transition-all duration-500 ease-in-out"
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
				{#if $overallExperimentProgress.isComplete && $experimentPhases.length > 0}
					<p class="mt-3 flex items-center font-medium text-green-400">
						<span class="mr-2 text-lg">✓</span>Experiment successfully completed!
					</p>
				{/if}
			</div>
		{/if}

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
				<p class="mb-2 text-xl font-bold text-sky-400">Empty Canvas</p>
				<p class="max-w-md text-center text-base text-gray-400">
					Drag phases from the left panel and drop them here to build your experiment workflow.
				</p>
			</div>
		{:else}
			<ul
				class="min-h-[300px] space-y-5 rounded-xl p-2
               {dragOverCanvas &&
				draggingItem?.type === 'paletteItem' &&
				$experimentPhases.length > 0
					? 'bg-sky-900/30 shadow-inner outline-2 outline-sky-500 outline-dashed'
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
						class="p-5 from-{phase.color.replace('bg-', '')} to-{phase.color.replace(
							'bg-',
							''
						)}/90 group relative flex cursor-grab flex-col rounded-xl border border-white/10 bg-gradient-to-br text-white shadow-lg backdrop-blur-sm backdrop-saturate-150
                       transition-all duration-200 ease-in-out
                       {draggingItem?.id === phase.id
							? 'scale-95 rotate-1 opacity-60 shadow-xl'
							: 'scale-100 opacity-100 hover:-translate-y-1 hover:shadow-xl'}
                       {dragOverPhaseId === phase.id && draggingItem?.id !== phase.id
							? 'ring-4 ring-sky-400 ring-offset-2 ring-offset-gray-900'
							: ''}"
						animate:flip={{ duration: 300 }}
					>
						<div class="mb-3 flex items-center justify-between">
							<div class="flex items-center space-x-3">
								<div
									class="flex h-7 w-7 flex-shrink-0 items-center justify-center opacity-80 transition-opacity group-hover:opacity-100"
									aria-hidden="true"
								>
									<svelte:component this={phase.iconComponent} />
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

						<textarea
							bind:value={phase.text}
							on:change={(e) => updatePhaseText(phase.id, (e.target as HTMLTextAreaElement).value)}
							class="mb-4 w-full resize-none rounded-lg border border-gray-600/50 bg-gray-700/50 p-3 text-sm text-gray-200 placeholder-gray-500 shadow-inner transition-all duration-200 outline-none focus:bg-gray-700/80 focus:shadow-md focus:ring-2 focus:ring-sky-500"
							placeholder="Describe this phase or add notes..."
							rows="2"
							aria-label={`Description for ${phase.label} phase`}
						></textarea>

						<div class="mb-4 flex items-center space-x-3">
							<div class="h-3 w-full rounded-full bg-gray-700/70 shadow-inner">
								<div
									class="h-3 rounded-full bg-gradient-to-r shadow-md {getStatusColor(phase.status)}"
									style="width: {phase.progress}%;"
									aria-valuenow={phase.progress}
									aria-valuemin="0"
									aria-valuemax="100"
									role="progressbar"
								></div>
							</div>
							<span class="text-sm font-bold text-sky-300">{Math.round(phase.progress)}%</span>
						</div>

						<div class="flex items-center justify-between text-xs text-gray-400">
							<div class="flex items-center space-x-2">
								<span
									class="rounded-full bg-gradient-to-r px-2.5 py-0.5 text-[10px] font-bold text-white uppercase shadow-sm ring-1 ring-white/20 {getStatusColor(
										phase.status
									)}"
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
							<button
								on:click={() => togglePhaseExecution(phase.id)}
								class="rounded-lg border border-gray-600/30 bg-gray-700 px-3.5 py-1.5 text-xs font-semibold text-white shadow transition-all duration-200 hover:-translate-y-0.5 hover:bg-gray-600 hover:shadow-md active:translate-y-0 disabled:cursor-not-allowed disabled:opacity-50"
								disabled={!phase.canBeRun &&
									phase.status !== 'running' &&
									phase.status !== 'paused' &&
									phase.status !== 'failed'}
								aria-label="{getRunButtonText(phase.status)} {phase.label} phase"
							>
								{getRunButtonText(phase.status)}
							</button>
						</div>

						{#if phase.logs && phase.logs.length > 0}
							<div class="mt-4 border-t border-gray-700/30 pt-3">
								<details class="group text-xs">
									<summary
										class="flex cursor-pointer items-center font-medium text-gray-500 transition-colors duration-200 hover:text-sky-300"
									>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											class="mr-1.5 h-3 w-3"
											fill="none"
											viewBox="0 0 24 24"
											stroke="currentColor"
											aria-hidden="true"
										>
											<path
												stroke-linecap="round"
												stroke-linejoin="round"
												stroke-width="2"
												d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"
											/>
										</svg>
										Logs ({phase.logs.length})
									</summary>
									<ul
										class="mt-2 max-h-24 space-y-1 overflow-y-auto rounded-lg bg-black/20 py-1.5 pr-1 pl-2 shadow-inner"
									>
										{#each phase.logs as logMsg, i (i)}
											<li class="font-mono text-[10px] leading-tight text-gray-400">{logMsg}</li>
										{/each}
									</ul>
								</details>
							</div>
						{/if}
					</li>
				{/each}
			</ul>
		{/if}

		{#if $experimentPhases.length > 0}
			<div
				class="mt-10 rounded-xl border border-gray-700/30 bg-gradient-to-br from-gray-800 to-gray-900 p-5 shadow-lg"
			>
				<h3 class="mb-3 text-xl font-bold text-sky-400">Experiment Data Structure (JSON)</h3>
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

<style>
	/* Using Tailwind CSS classes primarily. */
	[draggable='true'] {
		user-select: none;
	}
	.max-h-20::-webkit-scrollbar {
		/* User had max-h-20, logs list is max-h-24, will adjust if needed or keep as is */
		width: 8px;
	}
	.max-h-20::-webkit-scrollbar-track {
		background: #374151;
		border-radius: 4px;
	}
	.max-h-20::-webkit-scrollbar-thumb {
		background: #6b7280;
		border-radius: 4px;
	}
	.max-h-20::-webkit-scrollbar-thumb:hover {
		background: #9ca3af;
	}

	/* For .max-h-24 used in logs */
	.max-h-24::-webkit-scrollbar {
		width: 8px;
	}
	.max-h-24::-webkit-scrollbar-track {
		background: rgba(0, 0, 0, 0.2); /* From user's original log scrollbar style */
		border-radius: 4px;
	}
	.max-h-24::-webkit-scrollbar-thumb {
		background: rgba(255, 255, 255, 0.2); /* From user's original log scrollbar style */
		border-radius: 4px;
	}
	.max-h-24::-webkit-scrollbar-thumb:hover {
		background: rgba(255, 255, 255, 0.4); /* From user's original log scrollbar style */
	}

	pre::-webkit-scrollbar {
		height: 8px;
		width: 8px;
	}
	pre::-webkit-scrollbar-track {
		background: #1f2937;
		border-radius: 4px;
	}
	pre::-webkit-scrollbar-thumb {
		background: #4b5563;
		border-radius: 4px;
	}
	pre::-webkit-scrollbar-thumb:hover {
		background: #6b7280;
	}
</style>
