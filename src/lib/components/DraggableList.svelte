<script lang="ts">
	import { writable, type Writable } from 'svelte/store';
	import { flip } from 'svelte/animate'; // For smooth reordering animation

	interface Item {
		id: number;
		text: string;
		color: string;
	}

	// Sample initial items. In a real app, this might come from props or an API.
	let items: Writable<Item[]> = writable([
		{ id: 1, text: '🍎 Apple (Drag me!)', color: 'bg-red-400' },
		{ id: 2, text: '🍌 Banana (Drag me!)', color: 'bg-yellow-400' },
		{ id: 3, text: '🍇 Grapes (Drag me!)', color: 'bg-purple-400' },
		{ id: 4, text: '🍓 Strawberry (Drag me!)', color: 'bg-pink-400' },
		{ id: 5, text: '🥝 Kiwi (Drag me!)', color: 'bg-green-400' },
		{ id: 6, text: '🍊 Orange (Drag me!)', color: 'bg-orange-400' }
	]);

	let draggingItemId: number | null = null; // ID of the item currently being dragged
	let dragOverItemId: number | null = null; // ID of the item we are dragging over

	// Function to handle the start of a drag operation
	function handleDragStart(event: DragEvent, itemId: number) {
		event.dataTransfer!.effectAllowed = 'move';
		event.dataTransfer!.setData('text/plain', String(itemId)); // Required for Firefox
		draggingItemId = itemId;
		dragOverItemId = null; // Reset on new drag
	}

	// Function to handle when a dragged item enters a potential drop target
	function handleDragEnter(event: DragEvent, targetItemId: number) {
		event.preventDefault();
		if (draggingItemId !== targetItemId) {
			dragOverItemId = targetItemId;
		}
	}

	// Function to handle when a dragged item is over a potential drop target
	function handleDragOver(event: DragEvent) {
		event.preventDefault(); // Necessary to allow dropping
		event.dataTransfer!.dropEffect = 'move';
	}

	// Function to handle when a dragged item leaves a potential drop target
	function handleDragLeave(event: DragEvent, targetItemId: number) {
		if (dragOverItemId === targetItemId) {
			// Only clear if leaving the item we thought we were over
			// This prevents flickering when moving over child elements
			dragOverItemId = null;
		}
	}

	// Function to handle the drop operation
	function handleDrop(event: DragEvent, targetItemId: number) {
		event.preventDefault();
		const sourceItemId = draggingItemId;

		if (sourceItemId === targetItemId || sourceItemId === null) {
			draggingItemId = null;
			dragOverItemId = null;
			return;
		}

		items.update((currentItems) => {
			const sourceIndex = currentItems.findIndex((item) => item.id === sourceItemId);
			let targetIndex = currentItems.findIndex((item) => item.id === targetItemId);

			if (sourceIndex === -1 || targetIndex === -1) return currentItems; // Should not happen

			const [draggedItem] = currentItems.splice(sourceIndex, 1); // Remove dragged item

			// Adjust targetIndex if source was before target and items shifted
			if (sourceIndex < targetIndex) {
				// No adjustment needed as splice already shifted items
			}

			currentItems.splice(targetIndex, 0, draggedItem); // Insert at new position
			return currentItems;
		});

		draggingItemId = null;
		dragOverItemId = null;
	}

	// Function to handle the end of a drag operation (cleanup)
	function handleDragEnd(): void {
		draggingItemId = null;
		dragOverItemId = null;
	}
</script>

<div class="container mx-auto max-w-2xl p-4 md:p-8">
	<h1 class="mb-6 text-center text-2xl font-bold text-gray-700 md:text-3xl">Draggable List</h1>
	<p class="mb-8 text-center text-gray-500">Drag and drop the items below to reorder them.</p>

	<ul class="list-none space-y-3 p-0">
		{#each $items as item (item.id)}
			<li
				draggable="true"
				on:dragstart={(event) => handleDragStart(event, item.id)}
				on:dragenter={(event) => handleDragEnter(event, item.id)}
				on:dragover={handleDragOver}
				on:dragleave={(event) => handleDragLeave(event, item.id)}
				on:drop={(event) => handleDrop(event, item.id)}
				on:dragend={handleDragEnd}
				class="cursor-grab rounded-lg p-4 font-medium text-white shadow-md transition-all duration-150 ease-in-out
               {item.color}
               {draggingItemId === item.id
					? 'scale-95 opacity-50 shadow-xl'
					: 'scale-100 opacity-100'}
               {dragOverItemId === item.id && draggingItemId !== item.id
					? 'ring-4 ring-blue-500 ring-offset-2'
					: ''}"
				animate:flip={{ duration: 300 }}
			>
				{item.text}
			</li>
		{/each}
	</ul>

	{#if $items.length === 0}
		<p class="mt-8 text-center text-gray-500">The list is empty. Try adding some items!</p>
	{/if}
</div>

<style>
	/* Using Tailwind CSS classes directly in the markup, so minimal global styles needed. */
	/* Ensure Tailwind is set up in your SvelteKit project. */

	/* Style for when an item is being dragged */
	/* The opacity and scale are handled inline with conditional classes */

	/* General container styling for responsiveness */
	.container {
		font-family: 'Inter', sans-serif; /* Assuming Inter font is available */
	}

	/* Add a subtle hover effect to draggable items */
	li[draggable='true']:hover {
		transform: translateY(-2px);
		box-shadow: 0 6px 12px rgba(0, 0, 0, 0.1);
	}
</style>
