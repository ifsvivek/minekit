<script>
	import InventorySlot from './InventorySlot.svelte';
	import { onMount, onDestroy, createEventDispatcher } from 'svelte'; // Added createEventDispatcher

	const dispatch = createEventDispatcher(); // For Svelte component events

	let inventory = $state(Array(9).fill(null)); // Hotbar
	let fullInventory = $state(Array(27).fill(null)); // 3x9 grid for main inventory
	let selectedSlot = $state(0);
	let isInventoryOpen = $state(false);

	// Function to add an item to the inventory
	function addItem(itemToAdd) {
		// Attempt to stack with existing items
		for (let i = 0; i < inventory.length; i++) {
			const slot = inventory[i];
			if (slot && slot.name === itemToAdd.name && slot.count < 64) {
				const canAdd = 64 - slot.count;
				const amountToAdd = Math.min(itemToAdd.count, canAdd);
				slot.count += amountToAdd;
				itemToAdd.count -= amountToAdd;
				if (itemToAdd.count === 0) {
					inventory[i] = {...slot}; // Trigger reactivity for the slot
					return true; // Item fully added
				}
			}
		}

		// If item still has count, try to add to an empty slot
		if (itemToAdd.count > 0) {
			for (let i = 0; i < inventory.length; i++) {
				if (!inventory[i]) {
					inventory[i] = { ...itemToAdd }; // Add a copy
					return true; // Item added to a new slot
				}
			}
		}
		
		return itemToAdd.count === 0; // Return true if fully added, false otherwise (e.g. inventory full)
	}

	// Function to remove an item from the selected slot
	function removeItem(count = 1) {
		if (inventory[selectedSlot]) {
			inventory[selectedSlot].count -= count;
			if (inventory[selectedSlot].count <= 0) {
				inventory[selectedSlot] = null;
			}
		} else {
			inventory[selectedSlot] = {...inventory[selectedSlot]}; // Trigger reactivity if only count changed
		}
	}

	onMount(() => {
		addItem({ name: 'Dirt', count: 32 });
		addItem({ name: 'Stone', count: 16 });
		addItem({ name: 'Wood', count: 5 });

		window.addEventListener('keydown', handleKeyDown);
		window.addEventListener('mouseup', handleMouseUp);
	});
	onDestroy(() => {
		window.removeEventListener('keydown', handleKeyDown);
		window.removeEventListener('mouseup', handleMouseUp);
	});

	function handleMouseUp(event) {
		if (isInventoryOpen || document.pointerLockElement === null) return;
		if (event.button === 2) { // Right click
			const currentItem = inventory[selectedSlot];
			if (currentItem) {
				console.log(`Attempting to place: ${currentItem.name}`);
				dispatch('placeblock', { item: currentItem }); 
				// removeItem(1); // Optionally remove item after placement attempt - Scene should confirm
			}
		}
	}

	function handleKeyDown(event) {
		if (event.target.tagName === 'INPUT' || event.target.tagName === 'TEXTAREA') {
			return;
		}

		if (event.key === 'e' || event.key === 'i') {
			event.preventDefault();
			isInventoryOpen = !isInventoryOpen;
			dispatch('inventoryToggle', { isOpen: isInventoryOpen }); // Dispatch Svelte event
			return;
		}

		if (event.key === 'Escape' && isInventoryOpen) {
			event.preventDefault();
			isInventoryOpen = false;
			dispatch('inventoryToggle', { isOpen: isInventoryOpen }); // Dispatch Svelte event
			return;
		}

		if (!isInventoryOpen && event.code.startsWith('Digit')) {
			const digit = parseInt(event.code.slice(-1));
			if (digit >= 1 && digit <= 9) {
				selectedSlot = digit - 1;
				event.preventDefault(); 
			}
		}
	}

</script>

{#if isInventoryOpen}
<div 
  class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-30"
  role="dialog"
  aria-modal="true"
  aria-labelledby="inventory-title"
  onclick={(event) => {
    if (event.target === event.currentTarget) {
      isInventoryOpen = false;
      dispatch('inventoryToggle', { isOpen: false });
    }
  }}
  onkeydown={(event) => {
    if (event.key === 'Escape') {
      isInventoryOpen = false;
      dispatch('inventoryToggle', { isOpen: false });
    }
  }}
>
    <div 
      class="bg-gray-800 p-4 rounded-lg shadow-xl text-white w-auto max-w-2xl" 
      role="document"
      onclick={(event) => event.stopPropagation()}
      onkeydown={(event) => event.stopPropagation()}
    >
      <h2 id="inventory-title" class="text-xl mb-4 text-center">Inventory</h2>
      <!-- Main Inventory Grid -->
      <div class="grid grid-cols-9 gap-1 mb-4">
        {#each fullInventory as item, i (i)} 
          <div class="w-16 h-16 border border-gray-600 bg-gray-700 rounded">
            <InventorySlot {item} />
          </div>
        {/each}
      </div>
      <!-- Hotbar displayed within the full inventory UI -->
      <div class="flex space-x-1 justify-center">
        {#each inventory as item, i (i)}
          <button 
            onclick={() => selectedSlot = i} 
            class="border-2 hover:border-gray-300 focus:outline-none w-16 h-16 rounded"
            class:border-yellow-400={selectedSlot === i}
            class:border-transparent={selectedSlot !== i} 
          >
            <InventorySlot {item} />
          </button>
        {/each}
      </div>
      <div class="flex justify-center mt-4">
        <button class="bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded" onclick={() => {
          isInventoryOpen = false;
          dispatch('inventoryToggle', { isOpen: false });
        }}>
          Close
        </button>
      </div>
    </div>
  </div>
{/if}

<!-- Hotbar (always visible when inventory is closed) -->
{#if !isInventoryOpen}
  <div class="inventory-hotbar fixed bottom-4 left-1/2 transform -translate-x-1/2 flex space-x-1 p-1 bg-black bg-opacity-75 rounded-md z-20">
    {#each inventory as item, i (i)}
	  <button 
		onclick={() => selectedSlot = i} 
		class="border-2 hover:border-gray-300 focus:outline-none w-16 h-16 rounded"
		class:border-yellow-400={selectedSlot === i}
		class:border-transparent={selectedSlot !== i} 
	  >
        <InventorySlot {item} />
      </button>
    {/each}
  </div>
{/if}
