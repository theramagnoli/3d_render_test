<script setup lang="ts">
import { onMounted, ref } from "vue";
import * as THREE from "three";
import { FBXLoader } from "three/addons/loaders/FBXLoader.js";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { gsap } from "gsap";

const imageSrc = ref<string | null>(null);

onMounted(async () => {
    const canvas = document.getElementById("three-canvas") as HTMLCanvasElement;

    const renderer = new THREE.WebGLRenderer({
        antialias: true,
        canvas,
        depth: true,
    });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);

    const camera = new THREE.PerspectiveCamera(60, 2, 2.1, 500);
    const controls = new OrbitControls(camera, canvas);
    controls.update();

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x444444);

    const light = new THREE.AmbientLight(0xffffff, 2.5);
    scene.add(light);

    const fbxLoader = new FBXLoader();

    fbxLoader.load("/cint_points_test.fbx", (model) => {
        model.rotation.set(0, 4, 0);

        const box = new THREE.Box3().setFromObject(model);
        const center = box.getCenter(new THREE.Vector3());
        const size = box.getSize(new THREE.Vector3());
        const maxDim = Math.max(size.x, size.y, size.z);
        const fov = camera.fov * (Math.PI / 180);
        let cameraZ = Math.abs((maxDim / 4) * Math.tan(fov * 2));
        camera.position.z = cameraZ * 20;
        camera.position.x = center.x;
        camera.position.y = center.y - 10;
        camera.lookAt(center);
        controls.target = center;

        model.traverse((child) => {
            if (child.name === "Empty_test") {
                const tag = new THREE.Mesh(
                    new THREE.SphereGeometry(1, 64, 64),
                    new THREE.MeshBasicMaterial({ color: "#ff0000" }),
                );
                tag.position.setFromMatrixPosition(child.matrixWorld);
                scene.add(tag);
            }
        });

        camera.updateProjectionMatrix();
        scene.add(model);
    });

    function render() {
        renderer.render(scene, camera);
        requestAnimationFrame(render);
    }

    render();

    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();

    function onMouseClick(event: MouseEvent) {
        event.preventDefault();

        const rect = renderer.domElement.getBoundingClientRect();
        mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
        mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

        raycaster.setFromCamera(mouse, camera);
        const intersects = raycaster.intersectObjects(scene.children, true);

        if (intersects.length > 0) {
            const clickedObject = intersects[0].object;
            if (!clickedObject.userData.imagePath) return;
            gsap.to(camera.position, {
                x: clickedObject.position.x,
                y: clickedObject.position.y,
                z: clickedObject.position.z + 2,
                duration: 0.2,
                ease: "power2.out",
                onUpdate: () => {
                    camera.lookAt(clickedObject.position);
                },
                onComplete: () => {
                    if (clickedObject.userData.imagePath) {
                        imageSrc.value = clickedObject.userData.imagePath;
                    }
                },
            });
        }
    }

    window.addEventListener("click", onMouseClick);
});
</script>

<template>
    <canvas id="three-canvas"></canvas>
    <Teleport to="body">
        <div v-if="imageSrc" id="image-wrapper">
            <img :src="imageSrc" alt="" />
            <button @click="() => (imageSrc = null)">X</button>
        </div>
    </Teleport>
</template>

<style>
#image-wrapper {
    position: absolute;
    inset: 0;
    background-color: rgba(0, 0, 0, 0.8);

    img {
        width: 100%;
        height: 100%;
        object-fit: contain;
    }

    button {
        position: absolute;
        top: 0.5rem;
        right: 0.5rem;
        padding: 0.5rem;
        background-color: rgba(0, 0, 0, 0.8);
        color: white;
        border: none;
        cursor: pointer;
    }
}

canvas {
    display: block;
}
</style>
