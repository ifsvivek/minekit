<script>
	import Scene from '$components/Scene.svelte';
	import MainMenu from '$components/MainMenu.svelte';

	let showMainMenu = $state(true);

	function handlePlayGame() {
		showMainMenu = false;
	}
</script>

{#if showMainMenu}
	<MainMenu on:play={handlePlayGame} />
{:else}
	<Scene />
{/if}
