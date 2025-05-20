<script>
	import { createEventDispatcher } from 'svelte';

	const dispatch = createEventDispatcher();

	function startGame() {
		dispatch('play');
	}
</script>

<div
	class="box-border flex h-screen w-screen items-center justify-center overflow-y-auto bg-gradient-to-br from-cyan-400 to-sky-600 p-5 text-center font-sans text-white"
>
	<div class="bg-opacity-60 w-11/12 max-w-2xl rounded-2xl bg-black p-7 shadow-xl md:p-10">
		<h1 class="text-khaki shadow-text mb-2.5 text-5xl">Minekit</h1>
		<p class="mb-6 text-lg leading-relaxed">
			A Minecraft-inspired sandbox game built with Svelte and Three.js. Explore, build, and craft
			directly in your browser!
		</p>

		<div class="mb-7 text-left">
			<h2 class="text-light-blue mb-2.5 text-2xl">Features:</h2>
			<ul class="list-none p-0">
				<li class="bg-opacity-10 mb-2 rounded-md bg-white p-2.5 text-base text-black">
					🌍 3D Sandbox Environment
				</li>
				<li class="bg-opacity-10 mb-2 rounded-md bg-white p-2.5 text-base text-black">
					🧱 Build & Destroy Blocks
				</li>
				<li class="bg-opacity-10 mb-2 rounded-md bg-white p-2.5 text-base text-black">
					🏃 Player Movement & Jump
				</li>
				<li class="bg-opacity-10 mb-2 rounded-md bg-white p-2.5 text-base text-black">
					🌲 Procedurally Generated Terrain with Mountains, Water & Trees
				</li>
				<li class="bg-opacity-10 mb-2 rounded-md bg-white p-2.5 text-base text-black">
					🎮 Third-Person Camera Control
				</li>
				<li class="bg-opacity-10 mb-2 rounded-md bg-white p-2.5 text-base text-black">
					⏸️ Pause Menu
				</li>
			</ul>
		</div>

		<button
			on:click={startGame}
			class="bg-tomato hover:bg-dark-tomato cursor-pointer rounded-lg border-none px-7 py-3.5 text-xl font-bold text-white shadow-lg transition-all duration-300 ease-in-out hover:-translate-y-0.5 active:translate-y-0"
			>Play Game</button
		>

		<div class="text-light-steel-blue mt-7 text-sm">Built with: Svelte, Three.js, WebGL</div>
	</div>
</div>

<style>
	/* Custom utility classes if needed, or for things Tailwind doesn't cover as easily */
	.text-khaki {
		color: #f0e68c; /* Khaki */
	}
	.shadow-text {
		text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
	}
	.text-light-blue {
		color: #add8e6; /* Light blue */
	}
	.bg-tomato {
		background-color: #ff6347; /* Tomato */
	}
	.hover\:bg-dark-tomato:hover {
		background-color: #e5533d; /* Darker Tomato */
	}
	.text-light-steel-blue {
		color: #b0c4de; /* Light steel blue */
	}
</style>
