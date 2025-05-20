<script>
	export let item = null; // item is expected to be an object like { name: 'Dirt', count: 1, texture: 'path/to/img.png' }
</script>

<div class="inventory-slot border border-gray-500 w-16 h-16 flex items-center justify-center bg-gray-700 bg-opacity-50 relative text-white">
	{#if item}
		<!-- For now, just text. Later, could use item.texture for an image -->
		<span class="text-xs absolute top-1 left-1">{item.name}</span>
		<span class="text-lg">{item.count}</span>
	{/if}
</div>
