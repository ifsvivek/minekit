<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

  let container;
  let controls;

  onMount(() => {
    // Create a new Three.js Scene
    const scene = new THREE.Scene();

    // Create a PerspectiveCamera
    const camera = new THREE.PerspectiveCamera(
      75, // fov
      window.innerWidth / window.innerHeight, // aspect ratio
      0.1, // near
      1000 // far
    );

    // Create a WebGLRenderer
    const renderer = new THREE.WebGLRenderer();

    // Set the renderer's size
    renderer.setSize(window.innerWidth, window.innerHeight);

    // Append the renderer's DOM element to the container div
    container.appendChild(renderer.domElement);

    // Define Block Properties
    const blockSize = 1;
    const gridWidth = 5;
    const gridDepth = 5;

    // Create reusable Block Geometry and Material
    const blockGeometry = new THREE.BoxGeometry(blockSize, blockSize, blockSize);
    const blockMaterial = new THREE.MeshBasicMaterial({ color: 0x00ff00 }); // Green color

    // Render Grid of Blocks
    for (let x = 0; x < gridWidth; x++) {
      for (let z = 0; z < gridDepth; z++) {
        const mesh = new THREE.Mesh(blockGeometry, blockMaterial);
        mesh.position.set(
          (x - gridWidth / 2 + 0.5) * blockSize,
          blockSize / 2,
          (z - gridDepth / 2 + 0.5) * blockSize
        );
        scene.add(mesh);
      }
    }

    // Adjust Camera Position
    camera.position.set(gridWidth, gridWidth * 1.5, gridDepth * 1.5); // Adjusted for better view
    // camera.lookAt(0, 0, 0); // OrbitControls will handle the target

    // Initialize OrbitControls
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.screenSpacePanning = false;
    controls.minDistance = 5;
    controls.maxDistance = 50;
    controls.maxPolarAngle = Math.PI / 2;


    // Basic animation loop
    function animate() {
      requestAnimationFrame(animate);

      // Update controls if damping is enabled
      if (controls.enableDamping) {
        controls.update();
      }

      // No per-block animation needed for this step
      // cube.rotation.x += 0.01;
      // cube.rotation.y += 0.01;

      renderer.render(scene, camera);
    }
    animate();

    // Handle window resize
    const handleResize = () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    };

    window.addEventListener('resize', handleResize);

    return () => {
      // Cleanup on component destroy
      window.removeEventListener('resize', handleResize);
      renderer.dispose();
      if (controls) {
        controls.dispose();
      }
      // Potentially remove renderer.domElement from container if necessary,
      // but Svelte should handle child removal when container is destroyed.
    };
  });
</script>

<div bind:this={container}></div>

<style>
  div {
    width: 100vw;
    height: 100vh;
    margin: 0;
    overflow: hidden;
  }
</style>
