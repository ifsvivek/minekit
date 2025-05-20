<script>
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import Inventory from './Inventory.svelte';

	let container;
	let inventoryComponent; // To bind to the Inventory component instance

	// Player movement variables
	const playerSpeed = 0.08; // Adjusted for third-person
	const playerLookSpeed = 0.002;
	let moveForward = false;
	let moveBackward = false;
	let moveLeft = false;
	let moveRight = false;
	let mouseX = 0;
	let mouseY = 0;

	// Player physics variables
	let playerVelocityY = 0;
	const gravity = 0.008;
	const jumpStrength = 0.15;
	let isOnGround = false;
	const playerHeight = 1.8; // Total height of the player capsule
	const playerRadius = 0.4;

	// Camera and scene variables
	let camera, scene, renderer;
	let playerMesh; // To hold the player's 3D model
	const clock = new THREE.Clock();

	// Third-person camera variables
	const cameraOffset = new THREE.Vector3(0, 2.5, 5); // Adjusted for potentially taller terrain
	let cameraPitch = 0.3;
	const cameraLookAtOffset = new THREE.Vector3(0, playerHeight * 0.6, 0);

	// UI and Game State variables
	let isPaused = false;
	let isInventoryOpen = false; // State for managing inventory visibility and game pause

	// World generation parameters
	const blockSize = 1;
	const worldWidth = 60; // Increased world size
	const worldDepth = 60; // Increased world size
	const worldMinY = -10; // Deeper world for water and mountains
	const waterLevel = 0; // Y-level for water surface

	onMount(() => {
		scene = new THREE.Scene();
		scene.background = new THREE.Color(0x87ceeb);

		camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);

		renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(window.innerWidth, window.innerHeight);
		container.appendChild(renderer.domElement);

		// Player Mesh Setup
		const playerCapsuleLength = playerHeight - 2 * playerRadius;
		const playerGeometry = new THREE.CapsuleGeometry(playerRadius, playerCapsuleLength, 4, 8);
		const playerMaterial = new THREE.MeshStandardMaterial({ color: 0xff0000 }); // Red color for player
		playerMesh = new THREE.Mesh(playerGeometry, playerMaterial);

		const materials = {
			grass: new THREE.MeshStandardMaterial({ color: 0x228b22 }),
			dirt: new THREE.MeshStandardMaterial({ color: 0x8b4513 }),
			stone: new THREE.MeshStandardMaterial({ color: 0x808080 }),
			water: new THREE.MeshStandardMaterial({
				color: 0x1ca3ec,
				transparent: true,
				opacity: 0.7
			}),
			wood: new THREE.MeshStandardMaterial({ color: 0x654321 }), // Brown for wood
			leaves: new THREE.MeshStandardMaterial({ color: 0x006400 }) // Dark green for leaves
		};
		const blockGeometry = new THREE.BoxGeometry(blockSize, blockSize, blockSize);
		const trunkGeometry = new THREE.BoxGeometry(blockSize, blockSize, blockSize);
		const leafBlockGeometry = new THREE.BoxGeometry(blockSize, blockSize, blockSize);

		const terrainData = generateProceduralTerrain(
			worldWidth,
			worldDepth,
			worldMinY,
			blockSize,
			waterLevel
		);

		terrainData.blocks.forEach((block) => {
			const material = materials[block.type] || materials.dirt;
			const mesh = new THREE.Mesh(blockGeometry, material);
			mesh.position.set(block.x, block.y, block.z);
			scene.add(mesh);
		});

		terrainData.trees.forEach((tree) => {
			// Trunk
			tree.trunk.forEach((segment) => {
				const trunkMesh = new THREE.Mesh(trunkGeometry, materials.wood);
				trunkMesh.position.set(segment.x, segment.y, segment.z);
				scene.add(trunkMesh);
			});
			// Leaves
			tree.leaves.forEach((segment) => {
				const leafMesh = new THREE.Mesh(leafBlockGeometry, materials.leaves);
				leafMesh.position.set(segment.x, segment.y, segment.z);
				scene.add(leafMesh);
			});
		});

		// Set initial player position
		const initialPlayerX = 0;
		const initialPlayerZ = 0;
		const initialGroundY =
			getTerrainHeightAt(initialPlayerX, initialPlayerZ, terrainData.blocks, blockSize) +
			blockSize / 2;
		playerMesh.position.set(
			initialPlayerX,
			initialGroundY + playerHeight / 2 - playerRadius,
			initialPlayerZ
		);
		scene.add(playerMesh);

		const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
		scene.add(ambientLight);
		const directionalLight = new THREE.DirectionalLight(0xffffff, 1.0);
		directionalLight.position.set(10, 15, 10);
		scene.add(directionalLight);

		document.addEventListener('keydown', onKeyDown);
		document.addEventListener('keyup', onKeyUp);
		document.addEventListener('mousemove', onMouseMove);
		container.addEventListener('click', () => {
			if (document.pointerLockElement !== container) {
				container.requestPointerLock().catch((err) => {
					console.error('Pointer lock failed:', err);
				});
			}
		});

		document.addEventListener('pointerlockchange', () => {
			if (document.pointerLockElement !== container) {
				mouseX = 0;
				mouseY = 0;
				// If pointer lock is lost and game is not intentionally paused, pause it.
				if (!isPaused && !isInventoryOpen) {
					isPaused = true;
				}
			}
		});

		function animate() {
			requestAnimationFrame(animate);
			if (isPaused || isInventoryOpen) {
				renderer.render(scene, camera);
				return;
			}
			const delta = clock.getDelta();

			updatePlayerPhysics(delta, terrainData, blockSize);
			updatePlayerMovement(delta);
			updateCamera();

			renderer.render(scene, camera);
		}
		animate();

		const handleResize = () => {
			camera.aspect = window.innerWidth / window.innerHeight;
			camera.updateProjectionMatrix();
			renderer.setSize(window.innerWidth, window.innerHeight);
		};

		window.addEventListener('resize', handleResize);

		return () => {
			window.removeEventListener('resize', handleResize);
			document.removeEventListener('keydown', onKeyDown);
			document.removeEventListener('keyup', onKeyUp);
			document.removeEventListener('mousemove', onMouseMove);
			document.removeEventListener('pointerlockchange', null);
			if (document.pointerLockElement === container) {
				document.exitPointerLock();
			}
			renderer.dispose();
			blockGeometry.dispose();
			trunkGeometry.dispose();
			leafBlockGeometry.dispose();
			playerGeometry.dispose();
			playerMaterial.dispose();
			Object.values(materials).forEach((material) => material.dispose());
		};
	});

	function generateProceduralTerrain(width, depth, minY, blockSize, waterLvl) {
		const blocks = [];
		const trees = [];
		const baseNoise = (x, z) => {
			let val = 0;
			val += (Math.sin(x * 0.03) + Math.cos(z * 0.04)) * 8; // Large scale features (mountains)
			val += (Math.sin(x * 0.1) + Math.cos(z * 0.12)) * 3; // Medium scale features
			val += (Math.random() - 0.5) * 2; // Small scale randomness
			return val;
		};

		const treePlacementChance = 0.02; // Chance to place a tree on a suitable block

		for (let i = -width / 2; i < width / 2; i++) {
			for (let k = -depth / 2; k < depth / 2; k++) {
				const x = i * blockSize;
				const z = k * blockSize;
				const terrainHeight = Math.round(baseNoise(i, k));
				const surfaceY = terrainHeight * blockSize;

				for (let j = minY; j <= terrainHeight; j++) {
					const y = j * blockSize;
					if (y > waterLvl) {
						let type = 'dirt';
						if (j === terrainHeight) type = 'grass';
						else if (j < terrainHeight - 5) type = 'stone';
						else if (j < terrainHeight - 2) type = 'dirt';
						blocks.push({ x, y, z, type });
					}
				}

				if (surfaceY < waterLvl) {
					for (
						let wY = Math.max(surfaceY + blockSize, minY * blockSize);
						wY <= waterLvl;
						wY += blockSize
					) {
						blocks.push({ x, y: wY, z, type: 'water' });
					}
				} else {
					if (terrainHeight > waterLvl && Math.random() < treePlacementChance) {
						const topBlock = blocks.find((b) => b.x === x && b.z === z && b.y === surfaceY);
						if (topBlock && topBlock.type === 'grass') {
							trees.push(generateTree(x, surfaceY + blockSize, z, blockSize));
						}
					}
				}
			}
		}
		return { blocks, trees };
	}

	function generateTree(baseX, baseY, baseZ, blockSize) {
		const tree = { trunk: [], leaves: [] };
		const trunkHeight = Math.floor(Math.random() * 3) + 3;

		for (let i = 0; i < trunkHeight; i++) {
			tree.trunk.push({ x: baseX, y: baseY + i * blockSize, z: baseZ });
		}

		const canopyRadius = Math.floor(Math.random() * 1) + 1;
		const canopyBaseY = baseY + trunkHeight * blockSize;

		for (let lx = -canopyRadius; lx <= canopyRadius; lx++) {
			for (let ly = 0; ly <= canopyRadius; ly++) {
				for (let lz = -canopyRadius; lz <= canopyRadius; lz++) {
					if (
						lx * lx + (ly - canopyRadius / 2) * (ly - canopyRadius / 2) + lz * lz <=
						canopyRadius * canopyRadius + 1
					) {
						if (lx === 0 && lz === 0 && ly < 1) continue;
						tree.leaves.push({
							x: baseX + lx * blockSize,
							y: canopyBaseY + ly * blockSize,
							z: baseZ + lz * blockSize
						});
					}
				}
			}
		}
		return tree;
	}

	function getTerrainHeightAt(xPos, zPos, terrainBlockData, bSize) {
		let maxHeight = -Infinity;
		const roundedX = Math.round(xPos / bSize) * bSize;
		const roundedZ = Math.round(zPos / bSize) * bSize;

		terrainBlockData.forEach((block) => {
			if (block.type !== 'water' && block.x === roundedX && block.z === roundedZ) {
				if (block.y > maxHeight) {
					maxHeight = block.y;
				}
			}
		});
		return maxHeight === -Infinity ? worldMinY * bSize - bSize : maxHeight;
	}

	function onKeyDown(event) {
		// Allow Inventory component to handle its own E, I, Escape keys if it's open
		if (isInventoryOpen && (event.key === 'e' || event.key === 'i' || event.key === 'Escape')) {
			return;
		}

		if (event.code === 'Escape') {
			isPaused = !isPaused;
			if (isPaused) {
				document.exitPointerLock();
			} else {
				if (container && !isInventoryOpen)
					container
						.requestPointerLock()
						.catch((err) => console.error('Pointer lock failed on resume:', err));
			}
			return;
		}

		if (isPaused || isInventoryOpen) return;

		switch (event.code) {
			case 'KeyS':
				moveForward = true;
				break;
			case 'KeyW':
				moveBackward = true;
				break;
			case 'KeyA':
				moveLeft = true;
				break;
			case 'KeyD':
				moveRight = true;
				break;
			case 'Space':
				if (isOnGround) {
					playerVelocityY = jumpStrength;
					isOnGround = false;
				}
				break;
		}
	}

	function onKeyUp(event) {
		switch (event.code) {
			case 'KeyS':
				moveForward = false;
				break;
			case 'KeyW':
				moveBackward = false;
				break;
			case 'KeyA':
				moveLeft = false;
				break;
			case 'KeyD':
				moveRight = false;
				break;
		}
	}

	function onMouseMove(event) {
		if (isPaused || isInventoryOpen || document.pointerLockElement !== container) return;

		mouseX = event.movementX || 0;
		mouseY = event.movementY || 0;

		playerMesh.rotation.y -= mouseX * playerLookSpeed;

		cameraPitch -= mouseY * playerLookSpeed;
		cameraPitch = Math.max(-Math.PI / 3, Math.min(Math.PI / 2.5, cameraPitch));
	}

	function updatePlayerPhysics(delta, terrainData, bSize) {
		if (!playerMesh || isPaused || isInventoryOpen) return;

		playerVelocityY -= gravity * delta * 60;
		playerMesh.position.y += playerVelocityY;

		const playerBottomY = playerMesh.position.y - playerHeight / 2 + playerRadius;
		const currentGroundY =
			getTerrainHeightAt(playerMesh.position.x, playerMesh.position.z, terrainData.blocks, bSize) +
			bSize / 2;

		if (playerBottomY < currentGroundY) {
			playerMesh.position.y = currentGroundY + playerHeight / 2 - playerRadius;
			playerVelocityY = 0;
			isOnGround = true;
		} else {
			isOnGround = false;
		}
	}

	function updatePlayerMovement(delta) {
		if (!playerMesh || isPaused || isInventoryOpen) return;
		const moveSpeed = playerSpeed * delta * 60;

		const forward = new THREE.Vector3();
		playerMesh.getWorldDirection(forward);
		forward.normalize();

		const right = new THREE.Vector3();
		right.crossVectors(new THREE.Vector3(0, 1, 0), forward).normalize();

		if (moveForward) {
			playerMesh.position.addScaledVector(forward, moveSpeed);
		}
		if (moveBackward) {
			playerMesh.position.addScaledVector(forward, -moveSpeed);
		}
		if (moveLeft) {
			playerMesh.position.addScaledVector(right, -moveSpeed);
		}
		if (moveRight) {
			playerMesh.position.addScaledVector(right, moveSpeed);
		}
	}

	function updateCamera() {
		if (!playerMesh || !camera || isPaused || isInventoryOpen) return;

		let desiredOffset = cameraOffset.clone();

		const pitchQuaternion = new THREE.Quaternion().setFromAxisAngle(
			new THREE.Vector3(1, 0, 0),
			cameraPitch
		);
		desiredOffset.applyQuaternion(pitchQuaternion);

		desiredOffset.applyQuaternion(playerMesh.quaternion);

		camera.position.copy(playerMesh.position).add(desiredOffset);

		const lookAtPosition = new THREE.Vector3().copy(playerMesh.position).add(cameraLookAtOffset);
		camera.lookAt(lookAtPosition);
	}

	function resumeGame() {
		isPaused = false;
		if (container && !isInventoryOpen)
			container
				.requestPointerLock()
				.catch((err) => console.error('Pointer lock failed on resume:', err));
	}

	function handleInventoryToggle(event) {
		isInventoryOpen = event.detail.isOpen;
		if (isInventoryOpen) {
			document.exitPointerLock();
		} else {
			if (!isPaused && container) {
				container.requestPointerLock().catch((err) => console.error('Pointer lock failed on inventory close:', err));
			}
		}
	}

	function handlePlaceBlock(event) {
		const item = event.detail.item;
		console.log('Scene received placeblock event for:', item.name);
		// TODO: Implement raycasting from player/camera to find placement position
		// TODO: Check if placement is valid (not inside player, on a valid surface)
		// TODO: Create a new block mesh and add it to the scene and terrainData.blocks
		// TODO: If successful, tell Inventory.svelte to decrement the item count
	}
</script>

<div bind:this={container} class="relative h-screen w-screen cursor-grab">
	<!-- Crosshair -->
	<div
		class="pointer-events-none absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 transform text-2xl text-white"
		style="text-shadow: 1px 1px 2px black;"
	>
		+
	</div>

	<!-- Pause Menu Overlay -->
	{#if isPaused && !isInventoryOpen}
		<div
			class="bg-opacity-50 absolute top-0 left-0 z-10 flex h-full w-full flex-col items-center justify-center bg-black text-white"
		>
			<h1 class="mb-4 text-4xl">Paused</h1>
			<button
				on:click={resumeGame}
				class="cursor-pointer rounded bg-blue-500 px-5 py-2.5 text-lg font-bold text-white hover:bg-blue-700"
				>Resume</button
			>
			<!-- Add other menu items here e.g., Settings, Quit -->
		</div>
	{/if}
	<Inventory bind:this={inventoryComponent} on:inventoryToggle={handleInventoryToggle} on:placeblock={handlePlaceBlock} />
</div>

<style>
	:global(body) {
		margin: 0;
		overflow: hidden;
	}
</style>
