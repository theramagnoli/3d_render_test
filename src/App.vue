<script setup lang="ts">
import { onMounted, ref } from "vue";
import * as THREE from "three";
import { OBJLoader } from "three/addons/loaders/OBJLoader.js";
import { MTLLoader } from "three/addons/loaders/MTLLoader.js";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { XMLParser } from "fast-xml-parser";
import { gsap } from "gsap";

const imageSrc = ref<string | null>(null);

interface Camera {
    id: string;
    label: string;
    matrix: THREE.Matrix4;
    imagePath: string;
    referencePosition: THREE.Vector3;
    referenceYaw: number;
    referencePitch: number;
    referenceRoll: number;
}

function parseCamerasXml(xml: string): Camera[] {
    const parser = new XMLParser({
        ignoreAttributes: false,
        attributeNamePrefix: "",
    });
    const jsonObj = parser.parse(xml);
    const cameras: Camera[] = [];

    jsonObj.document.chunk.cameras.camera.forEach((camera: any) => {
        const transformValues = camera.transform.split(" ").map(parseFloat);
        const matrix = new THREE.Matrix4();
        matrix.fromArray(transformValues);
        matrix.transpose();

        const reference = camera.reference;
        const referencePosition = new THREE.Vector3(
            parseFloat(reference.x),
            parseFloat(reference.y),
            parseFloat(reference.z),
        );
        const referenceYaw = parseFloat(reference.yaw);
        const referencePitch = parseFloat(reference.pitch);
        const referenceRoll = parseFloat(reference.roll);

        cameras.push({
            id: camera.id,
            label: camera.label,
            matrix,
            imagePath: `/images/${camera.label}.jpg`,
            referencePosition,
            referenceYaw,
            referencePitch,
            referenceRoll,
        });
    });
    return cameras;
}

function addCameraIcons(cameras: Camera[], scene: THREE.Scene) {
    const iconGeometry = new THREE.SphereGeometry(0.1, 32, 32);
    const iconMaterial = new THREE.MeshBasicMaterial({ color: "#0000ff" });

    cameras.forEach((camera) => {
        const icon = new THREE.Mesh(iconGeometry, iconMaterial);

        icon.matrixAutoUpdate = false;
        icon.applyMatrix4(camera.matrix);

        icon.userData = { imagePath: camera.imagePath };
        scene.add(icon);
    });
}

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

    const mtlLoader = new MTLLoader();
    const objLoader = new OBJLoader();

    const response = await fetch("/cint_cameras.xml");
    const xml = await response.text();

    const cameras = parseCamerasXml(xml);
    // addCameraIcons(cameras, scene);

    mtlLoader.load("/cint_points_test.mtl", (material) => {
        material.preload();
        objLoader.setMaterials(material);
        objLoader.load("/cint_points_test.obj", (model) => {
            model.rotation.set(-0.8108, -2.1098, 2.1142);
            model.position.set(-1.8186, 2.2441, 1.2676);

            const box = new THREE.Box3().setFromObject(model);
            const center = box.getCenter(new THREE.Vector3());
            const size = box.getSize(new THREE.Vector3());
            const maxDim = Math.max(size.x, size.y, size.z);
            const fov = camera.fov * (Math.PI / 180);
            let cameraZ = Math.abs((maxDim / 4) * Math.tan(fov * 2));
            camera.position.z = cameraZ * 3;
            camera.position.x = center.x;
            camera.position.y = center.y - 10;
            camera.lookAt(center);
            controls.target = center;

            camera.updateProjectionMatrix();
            scene.add(model);
        });
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
</style>
