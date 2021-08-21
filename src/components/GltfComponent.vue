<template>
  <div id="gltf_wrapper" style="width: 1400px; height: 1080px">
    <!--    <q-img src="~assets/quasar-logo-vertical.svg"></q-img>-->
  </div>
</template>

<script lang="ts">
  import {Vue} from 'vue-class-component';
  import * as THREE from 'three';

  import {MapControls} from 'three/examples/jsm/controls/OrbitControls';
  import {GLTFLoader} from 'three/examples/jsm/loaders/GLTFLoader';
  import {DRACOLoader} from 'three/examples/jsm/loaders/DRACOLoader.js';
  import {Color, Group} from "three";
  import Stats from 'three/examples/jsm/libs/stats.module.js';
  // import { OrbitControls } from './jsm/controls/OrbitControls.js';
  // import { GLTFLoader } from './jsm/loaders/GLTFLoader.js';
  // import { RGBELoader } from './jsm/loaders/RGBELoader.js';
  // import { RoughnessMipmapper } from './jsm/utils/RoughnessMipmapper.js';

  const Preset = {ASSET_GENERATOR: 'assetgenerator'};
  let camera: THREE.Camera;
  let scene: THREE.Scene;
  let
    renderer: THREE.WebGLRenderer;
  let statsFps: { domElement: any; update: () => void; };
  let statsMb: { domElement: any; update: () => void; };

  class Props {
  }

  export default class GltfComponent extends Vue.with(Props) {
    prevTime = 0;

    async mounted(): Promise<void> {
      const container = document.getElementById('gltf_wrapper');
      if (!container) throw new Error('container is undefined')
      camera = new THREE.PerspectiveCamera(40, container.clientWidth / container.clientHeight, 0.01, 100000);
      camera.position.set(50, 50, 50);
      const createFpsStats = () => {
        // @ts-ignore
        const stats = new Stats();
        stats.setMode(0);
        stats.domElement.style.position = 'absolute';
        stats.domElement.style.left = '0';
        stats.domElement.style.top = '0';

        return stats;
      }
      const createMbStats = () => {
        // @ts-ignore
        const stats = new Stats();
        stats.setMode(2);
        stats.domElement.style.position = 'absolute';
        stats.domElement.style.left = '75px';
        stats.domElement.style.top = '0';

        return stats;
      }
      statsFps = createFpsStats();
      statsMb = createMbStats();

      document.body.appendChild(statsFps.domElement);
      document.body.appendChild(statsMb.domElement);

      const light = new THREE.DirectionalLight(new THREE.Color('white'), 3);
      light.position.set(0.5, 0, 0.866); // ~60º
      // light.name = 'main_light';
      camera.add(light);
      camera.lookAt(new THREE.Vector3(0, 0, 0));

      const light1 = new THREE.AmbientLight(new THREE.Color('yellow'), 3);
      camera.add(light1);
      scene = new THREE.Scene();
      scene.background = new Color(new THREE.Color('#2f3136'))
      scene.add(new THREE.DirectionalLight(0xffffff, 0.3));
      // light.position.set(100, 0, 250);
      scene.add(camera);

      const raycaster = new THREE.Raycaster();
      const mouse = new THREE.Vector2();

      const render = () => {
        // update the picking ray with the camera and mouse position
        renderer.render(scene, camera);
        statsFps.update();
        statsMb.update();
      }
      const animate = (time: number) => {

        requestAnimationFrame(animate);

        const dt = (time - this.prevTime) / 1000;
        controls.update();
        statsFps.update();
        statsMb.update();

        this.prevTime = time;

      }
      requestAnimationFrame(animate)
      renderer = new THREE.WebGLRenderer({
        antialias: true,
        // alpha: true,
        depth: true,
        powerPreference: 'high-performance'
      });
      renderer.setPixelRatio(window.devicePixelRatio);
      renderer.setSize(container.clientWidth, container.clientHeight);
      renderer.outputEncoding = THREE.sRGBEncoding;
      // const pmremGenerator = new THREE.PMREMGenerator( renderer );
      // pmremGenerator.compileEquirectangularShader()
      container.appendChild(renderer.domElement);
      const controls = new MapControls(camera, renderer.domElement);
      controls.addEventListener('change', event => {
        render()
      });
      controls.autoRotate = false;
      controls.screenSpacePanning = true;
      controls.update();
      const manager = new THREE.LoadingManager();
      const loader = new GLTFLoader().setDRACOLoader(
        new DRACOLoader(manager).setDecoderPath('/draco/')
      );
      const groups: Record<string, Group> = {};
      loader.load('http://192.168.1.64:8081/kv_maximum.glb', (gltf) => {
        gltf.scene.traverse((node) => {
          // @ts-ignore
          if (node.isMesh) {
            // @ts-ignore
            // if (node.material.opacity === 0) {
            //   // @ts-ignore
            //   node.material.visible = false;
            //   node.visible = false;
            // }
          }
        })
        gltf.scene.children.forEach((node) => {
          // @ts-ignore
          if (!node.isMesh) return;
          if (!groups[node.userData.element_id]) {
            groups[node.userData.element_id] = new THREE.Group();
            groups[node.userData.element_id].add(node);
          } else {
            groups[node.userData.element_id].add(node);
          }
        })
        gltf.scene.children = Object.values(groups);
        // gltf.scene.castShadow = true;
        // gltf.scene.receiveShadow = true;

        // gltf.scene.traverse((child) => {
        //   if (child.isMesh) {
        //     // TODO(https://github.com/mrdoob/three.js/pull/18235): Clean up.
        //     child.material.depthWrite = !child.material.transparent;
        //   }
        //   // @ts-ignore
        //   // child.material.depthWrite = !child.material.transparent;
        //   // if (child.isMesh) {
        //   //   roughnessMipmapper.generateMipmaps(child.material);
        //   // }
        // });

        scene.add(gltf.scene);
        console.log(scene);

        // roughnessMipmapper.dispose();

        render();
      });
      const onWindowResize = () => {

        // @ts-ignore
        camera.aspect = container.clientWidth / container.clientHeight;
        // @ts-ignore
        camera.updateProjectionMatrix();
        if (!container) throw new Error('container is undefined')

        render();
      }
      window.addEventListener('resize', onWindowResize);
      let intersected: Record<string, { color: number[], opacity: number }> = {};
      const onMouseClick = (event: MouseEvent) => {
        if (event.button === 0) {
          const {top, left, width, height} = renderer.domElement.getBoundingClientRect();

          mouse.x = -1 + (2 * (event.clientX - left) / width);
          mouse.y = 1 - (2 * (event.clientY - top) / height);
          raycaster.setFromCamera(mouse, camera);

          // calculate objects intersecting the picking ray
          const currIntersects = raycaster.intersectObjects(scene.children, true);
          // currIntersects[0].object.material.color.set(0xff0000);
          if (currIntersects[0] && currIntersects[0].object.parent && Object.keys(intersected).length === 0) {
            currIntersects[0] && currIntersects[0].object.parent?.children.forEach((node) => {
              intersected[node.id] = {
                // @ts-ignore
                color: node.material.color.toArray(),
                // @ts-ignore
                opacity: node.material.opacity,
              }
                // @ts-ignore
                node.material.color.set(0xff0000);
                // @ts-ignore
                node.material.opacity = 1;
            })
            // currIntersects[0].object.parent?.children.forEach((node) => {

            // })
          } else {
            Object.keys(intersected).forEach((id) => {
              scene.traverse((node) => {
                if (node.id === Number(id)) {
                  //@ts-ignore
                  node.material.color.fromArray(intersected[node.id].color);
                  // @ts-ignore
                  node.material.opacity = intersected[node.id].opacity;
                  delete intersected[node.id];
                }
              })
              // const savedNodesColors = intersected[id];
              // console.log(savedNodesColors);
              // scene.children = scene.children.map((group) => {
              //   if (group.id === Number(id)) {
              //     return savedNodesColors;
              //   }
              //   return group;
              // });
            })
            render();
          }
          for (let i = 0; i < currIntersects.length; i++) {
            // if (currIntersects[i].object.visible) {
            //   if (intersected[currIntersects[i].object.uuid]) {
            //     scene.traverse((node) => {
            //       if (intersected[node.uuid]) {
            //         // @ts-ignore
            //         node.material.color.fromArray(intersected[node.uuid].color);
            //         // @ts-ignore
            //         node.material.opacity = intersected[node.uuid].opacity;
            //         delete intersected[node.uuid];
            //       }
            //     })
            //     break;
            //   }
            //   scene.traverse((node) => {
            //     if (node.userData.element_id === currIntersects[i].object.userData.element_id) {
            //       intersected[node.uuid] = {
            //         // @ts-ignore
            //         color: node.material.color.toArray(),
            //         // @ts-ignore
            //         opacity: node.material.opacity,
            //       }
            //       // @ts-ignore
            //       node.material.color.set(0xff0000);
            //       // @ts-ignore
            //       node.material.opacity = 1;
            //     }
            //   })
            //   break;
            // }
          }
          render();
        }
      }
      window.addEventListener('click', onMouseClick, false);
    }
  }
</script>
