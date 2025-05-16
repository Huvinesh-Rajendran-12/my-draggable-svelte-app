<script lang="ts">
	import { writable, type Writable } from 'svelte/store';
	import { flip } from 'svelte/animate';
	import { v7 as uuidv7 } from 'uuid'; // For generating unique IDs for workflow steps

	// --- Type definitions ---
	interface WorkflowBlock {
		type: string;
		label: string;
		icon: string;
		color: string;
		defaultText: string;
	}

	interface WorkflowStep {
		id: string;
		type: string;
		label: string;
		icon: string;
		color: string;
		text: string;
	}

	type DraggingItem = {
		type: 'paletteItem' | 'workflowItem';
		id?: string;
		data: WorkflowBlock | WorkflowStep;
	} | null;

	// --- Configuration for available workflow blocks in the palette ---
	const paletteBlocks: WorkflowBlock[] = [
		{
			type: 'LOAD_DATA',
			label: 'Load Data',
			icon: '📊',
			color: 'bg-blue-500',
			defaultText: 'Load Data from Source'
		},
		{
			type: 'TRANSFORM_DATA',
			label: 'Transform Data',
			icon: '✨',
			color: 'bg-green-500',
			defaultText: 'Apply Transformation'
		},
		{
			type: 'RUN_MODEL',
			label: 'Run Model',
			icon: '🧠',
			color: 'bg-purple-500',
			defaultText: 'Execute ML Model'
		},
		{
			type: 'SEND_NOTIFICATION',
			label: 'Send Notification',
			icon: '📧',
			color: 'bg-orange-500',
			defaultText: 'Notify Stakeholders'
		},
		{
			type: 'SAVE_OUTPUT',
			label: 'Save Output',
			icon: '💾',
			color: 'bg-teal-500',
			defaultText: 'Save Results'
		}
	];

	// --- Store for the current workflow steps on the canvas ---
	const workflowSteps: Writable<WorkflowStep[]> = writable([]);

	// --- Drag and Drop State Variables ---
	let draggingItem: DraggingItem = null;
	let dragOverCanvas = false; // True if dragging over the workflow canvas
	let dragOverWorkflowItemId: string | null = null; // ID of the workflow item being dragged over (for reordering)

	// --- Drag Handlers for Palette Items ---
	function handlePaletteDragStart(event: DragEvent, blockType: WorkflowBlock) {
		if (event.dataTransfer) {
			event.dataTransfer.effectAllowed = 'copy'; // Indicate copying from palette
			event.dataTransfer.setData('text/plain', blockType.type); // Identify the block type
		}
		draggingItem = { type: 'paletteItem', data: blockType };
		dragOverCanvas = false;
		dragOverWorkflowItemId = null;
	}

	// --- Drag Handlers for Workflow Canvas Area ---
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
				event.dataTransfer.dropEffect = 'move';
			}
		}
		dragOverCanvas = true;
	}

	function handleCanvasDragLeave(event: DragEvent) {
		// Check if the mouse truly left the canvas or entered a child (workflow item)
		if (event.currentTarget === event.target) {
			dragOverCanvas = false;
		}
	}

	function handleCanvasDrop(event: DragEvent) {
		event.preventDefault();
		if (draggingItem?.type === 'paletteItem') {
			// Add new item from palette
			const block = draggingItem.data as WorkflowBlock;
			const newStep: WorkflowStep = {
				id: uuidv7(), // Generate a unique ID for the new step
				type: block.type,
				label: block.label,
				icon: block.icon,
				color: block.color,
				text: block.defaultText || `New ${block.label} Step`
				// You could add more default properties here
			};
			workflowSteps.update((steps) => [...steps, newStep]);
		}
		// Reordering logic is handled by individual workflow item drop handlers
		dragOverCanvas = false;
		// draggingItem is reset in handleDragEnd
	}

	// --- Drag Handlers for Individual Workflow Items (for reordering) ---
	function handleWorkflowItemDragStart(event: DragEvent, item: WorkflowStep) {
		if (event.dataTransfer) {
			event.dataTransfer.effectAllowed = 'move';
			event.dataTransfer.setData('text/plain', item.id); // Use item.id for reordering
		}
		draggingItem = { type: 'workflowItem', id: item.id, data: item };
		dragOverWorkflowItemId = null;
	}

	function handleWorkflowItemDragEnter(event: DragEvent, targetItemId: string) {
		event.preventDefault();
		if (draggingItem?.type === 'workflowItem' && draggingItem.id !== targetItemId) {
			dragOverWorkflowItemId = targetItemId;
		}
	}

	function handleWorkflowItemDragOver(event: DragEvent) {
		event.preventDefault();
		if (event.dataTransfer) {
			if (draggingItem?.type === 'workflowItem') {
				event.dataTransfer.dropEffect = 'move';
			} else {
				event.dataTransfer.dropEffect = 'none'; // Don't allow dropping palette items onto other workflow items directly (only on canvas)
			}
		}
	}

	function handleWorkflowItemDragLeave(event: DragEvent, targetItemId: string) {
		if (dragOverWorkflowItemId === targetItemId) {
			dragOverWorkflowItemId = null;
		}
	}

	function handleWorkflowItemDrop(event: DragEvent, targetItemId: string) {
		event.preventDefault();
		event.stopPropagation(); // Prevent canvas drop from firing as well

		if (draggingItem?.type !== 'workflowItem' || !draggingItem.id) return;

		const sourceItemId = draggingItem.id;
		if (sourceItemId === targetItemId) {
			dragOverWorkflowItemId = null;
			// draggingItem is reset in handleDragEnd
			return;
		}

		workflowSteps.update((currentSteps) => {
			const sourceIndex = currentSteps.findIndex((step) => step.id === sourceItemId);
			let targetIndex = currentSteps.findIndex((step) => step.id === targetItemId);

			if (sourceIndex === -1 || targetIndex === -1) return currentSteps;

			const [draggedStep] = currentSteps.splice(sourceIndex, 1) as [WorkflowStep];

			// If dragging downwards and source was before target, targetIndex effectively moved up by 1
			// If dragging upwards and source was after target, targetIndex is correct
			// The splice for removal handles index shifts correctly for items after sourceIndex
			// We need to find the new targetIndex based on the original targetItemId in the modified array
			// A simpler way: remove, then find the new index for insertion.
			// Let's re-find targetIndex after splice if sourceIndex < targetIndex
			// Actually, the targetIndex is the index *before* which we want to insert.
			// If sourceIndex < targetIndex, the original targetIndex is still valid for insertion *before* that element.
			// If sourceIndex > targetIndex, the original targetIndex is also valid.

			currentSteps.splice(targetIndex, 0, draggedStep);
			return currentSteps;
		});
		dragOverWorkflowItemId = null;
		// draggingItem is reset in handleDragEnd
	}

	// --- General Drag End Handler ---
	function handleDragEnd() {
		draggingItem = null;
		dragOverCanvas = false;
		dragOverWorkflowItemId = null;
	}

	// --- Function to remove a step from the workflow ---
	function removeStep(stepId: string) {
		workflowSteps.update((steps) => steps.filter((step) => step.id !== stepId));
	}

	// --- Function to allow editing text of a workflow step ---
	function updateStepText(stepId: string, newText: string) {
		workflowSteps.update((steps) =>
			steps.map((step) => (step.id === stepId ? { ...step, text: newText } : step))
		);
	}
</script>

<div
	class="flex h-screen flex-col bg-gradient-to-br from-gray-50 to-gray-100 font-sans text-gray-800 md:flex-row"
>
	<aside class="w-full space-y-3 overflow-y-auto bg-white p-5 shadow-lg md:w-64 lg:w-72">
		<h2 class="mb-5 border-b border-gray-100 pb-3 text-2xl font-bold text-gray-800">
			Workflow Blocks
		</h2>
		{#each paletteBlocks as block (block.type)}
			<div
				draggable="true"
				on:dragstart={(event) => handlePaletteDragStart(event, block)}
				on:dragend={handleDragEnd}
				class="p-4 from-{block.color.replace('bg-', '')} to-{block.color.replace(
					'bg-',
					''
				)}/90 flex cursor-grab items-center space-x-3 rounded-xl bg-gradient-to-br text-white shadow-md transition-all duration-200 hover:translate-y-[-2px] hover:scale-[1.02] hover:shadow-lg"
				role="button"
				tabindex="0"
			>
				<span class="text-2xl">{block.icon}</span>
				<span class="font-medium">{block.label}</span>
			</div>
		{/each}
	</aside>

	<main
		class="flex-1 overflow-y-auto p-5 md:p-8 lg:p-10"
		on:dragenter={handleCanvasDragEnter}
		on:dragover={handleCanvasDragOver}
		on:dragleave={handleCanvasDragLeave}
		on:drop={handleCanvasDrop}
	>
		<h1 class="mb-8 text-3xl font-extrabold text-gray-800 md:text-4xl">Workflow Canvas</h1>
		{#if $workflowSteps.length === 0}
			<div
				class="flex min-h-[300px] flex-col items-center justify-center rounded-xl border-2 border-dashed p-8 text-gray-500
               {dragOverCanvas && draggingItem?.type === 'paletteItem'
					? 'border-blue-500 bg-blue-50/70 shadow-inner'
					: 'border-gray-300 bg-white/50'}
               transition-all duration-300"
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					class="mb-3 h-12 w-12 text-gray-400"
					fill="none"
					viewBox="0 0 24 24"
					stroke="currentColor"
					stroke-width="1"
				>
					<path
						stroke-linecap="round"
						stroke-linejoin="round"
						d="M9.75h6.75m-6.75 0a3 3 0 01-3-3h12.75a3 3 0 01-3 3m0 0a3 3 0 00-3 3h.375m6.375 0h.375a3 3 0 003-3V6.75A3 3 0 0016.5 3.75h-9a3 3 0 00-3 3V15a3 3 0 003 3zm-3.75-9h15m-15 0v9A1.5 1.5 0 007.5 18h9a1.5 1.5 0 001.5-1.5v-9A1.5 1.5 0 0016.5 6H7.5A1.5 1.5 0 006 7.5z"
					/>
				</svg>
				<p>Drag blocks from the left panel and drop them here to build your workflow.</p>
			</div>
		{:else}
			<ul
				class="min-h-[300px] space-y-4 rounded-xl p-2
               {dragOverCanvas && draggingItem?.type === 'paletteItem' && $workflowSteps.length > 0
					? 'bg-blue-50/70 shadow-inner outline-2 outline-blue-500 outline-dashed'
					: ''}
               transition-all duration-300"
			>
				{#each $workflowSteps as step (step.id)}
					<li
						draggable="true"
						on:dragstart={(event) => handleWorkflowItemDragStart(event, step)}
						on:dragenter={(event) => handleWorkflowItemDragEnter(event, step.id)}
						on:dragover={handleWorkflowItemDragOver}
						on:dragleave={(event) => handleWorkflowItemDragLeave(event, step.id)}
						on:drop={(event) => handleWorkflowItemDrop(event, step.id)}
						on:dragend={handleDragEnd}
						role="listitem"
						class="p-5 from-{step.color.replace('bg-', '')} to-{step.color.replace(
							'bg-',
							''
						)}/90 group relative flex cursor-grab flex-col rounded-xl bg-gradient-to-br text-white shadow-lg
                   backdrop-blur-sm backdrop-saturate-150 transition-all duration-200 ease-in-out
                   {draggingItem?.id === step.id
							? 'scale-95 rotate-1 opacity-60 shadow-xl'
							: 'scale-100 opacity-100 hover:-translate-y-1 hover:shadow-xl'}
                   {dragOverWorkflowItemId === step.id && draggingItem?.id !== step.id
							? 'ring-4 ring-sky-400 ring-offset-2 ring-offset-gray-50'
							: ''}"
						animate:flip={{ duration: 300 }}
					>
						<div class="flex items-center justify-between">
							<div class="flex items-center space-x-3">
								<span class="text-2xl">{step.icon}</span>
								<span class="font-medium">{step.label}</span>
							</div>
							<button
								on:click={() => removeStep(step.id)}
								class="rounded-full bg-black/30 p-1.5 text-white opacity-0 transition-all duration-200 group-hover:opacity-100 hover:scale-110 hover:rotate-12 hover:bg-red-500/80 hover:text-white active:scale-95"
								aria-label="Remove step"
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									class="h-5 w-5"
									fill="none"
									viewBox="0 0 24 24"
									stroke="currentColor"
									stroke-width="2"
								>
									<path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
								</svg>
							</button>
						</div>
						<input
							type="text"
							bind:value={step.text}
							on:change={(e) => updateStepText(step.id, (e.target as HTMLInputElement).value)}
							class="mt-3 w-full rounded-lg border-none bg-white/25 p-3 text-white placeholder-white/70 shadow-inner transition-all duration-200 outline-none focus:bg-white/35 focus:shadow-md focus:ring-2 focus:ring-white/60"
							placeholder="Describe this step..."
						/>
					</li>
				{/each}
			</ul>
		{/if}

		{#if $workflowSteps.length > 0}
			<div
				class="mt-10 rounded-xl border border-gray-700/30 bg-gradient-to-br from-gray-800 to-gray-900 p-5 text-gray-100 shadow-lg"
			>
				<h3 class="mb-3 text-xl font-bold text-sky-300">Current Workflow (Data)</h3>
				<pre
					class="overflow-x-auto rounded-lg bg-black/30 p-4 text-xs break-all whitespace-pre-wrap shadow-inner">{JSON.stringify(
						$workflowSteps,
						null,
						2
					)}</pre>
			</div>
		{/if}
	</main>
</div>

<style>
	/* Minor style to improve drag image for palette items (browser dependent) */
	[draggable='true'] {
		user-select: none; /* Prevent text selection when dragging */
	}

	/* Style for the input field within workflow steps */
	input[type='text']::placeholder {
		color: rgba(255, 255, 255, 0.7);
	}
</style>
