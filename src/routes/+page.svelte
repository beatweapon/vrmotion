<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
	import { GLTFLoader, type GLTF } from 'three/addons/loaders/GLTFLoader.js';
	import { FilesetResolver, FaceLandmarker, type Classifications } from '@mediapipe/tasks-vision';
	import { VRMLoaderPlugin, VRMUtils, type VRM } from '@pixiv/three-vrm';
	import '../global.css';

	let roll = 0;
	let yaw = 0;
	let pitch = 0;

	onMount(() => {
		// /**
		//  * Returns the world-space dimensions of the viewport at `depth` units away from
		//  * the camera.
		//  */
		// function getViewportSizeAtDepth(camera: THREE.PerspectiveCamera, depth: number): THREE.Vector2 {
		// 	const viewportHeightAtDepth =
		// 		2 * depth * Math.tan(THREE.MathUtils.degToRad(0.5 * camera.fov));
		// 	const viewportWidthAtDepth = viewportHeightAtDepth * camera.aspect;
		// 	return new THREE.Vector2(viewportWidthAtDepth, viewportHeightAtDepth);
		// }

		// /**
		//  * Creates a `THREE.Mesh` which fully covers the `camera` viewport, is `depth`
		//  * units away from the camera and uses `material`.
		//  */
		// function createCameraPlaneMesh(
		// 	camera: THREE.PerspectiveCamera,
		// 	depth: number,
		// 	material: THREE.Material
		// ): THREE.Mesh {
		// 	if (camera.near > depth || depth > camera.far) {
		// 		console.warn('Camera plane geometry will be clipped by the `camera`!');
		// 	}
		// 	const viewportSize = getViewportSizeAtDepth(camera, depth);
		// 	const cameraPlaneGeometry = new THREE.PlaneGeometry(viewportSize.width, viewportSize.height);
		// 	cameraPlaneGeometry.translate(0, 0, -depth);

		// 	return new THREE.Mesh(cameraPlaneGeometry, material);
		// }

		type RenderCallback = (delta: number) => void;

		class BasicScene {
			scene: THREE.Scene;
			width: number;
			height: number;
			camera: THREE.PerspectiveCamera;
			renderer: THREE.WebGLRenderer;
			controls: OrbitControls;
			lastTime: number = 0;
			callbacks: RenderCallback[] = [];

			constructor() {
				// Initialize the canvas with the same aspect ratio as the video input
				this.height = window.innerHeight;
				this.width = (this.height * 1280) / 720;
				// Set up the Three.js scene, camera, and renderer
				this.scene = new THREE.Scene();
				this.scene.background = new THREE.Color(0xff00ff);
				this.camera = new THREE.PerspectiveCamera(35, this.width / this.height, 0.1, 2000);
				this.camera.position.set(0.0, 1.35, 0.7);

				this.renderer = new THREE.WebGLRenderer({ antialias: true });
				this.renderer.setSize(this.width, this.height);
				// THREE.ColorManagement.enabled = true;
				// this.renderer.outputColorSpace = THREE.SRGBColorSpace;
				document.body.appendChild(this.renderer.domElement);

				// Set up the basic lighting for the scene
				// const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
				// this.scene.add(ambientLight);
				// const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
				// directionalLight.position.set(0, 1, 0);
				// this.scene.add(directionalLight);

				// Set up the camera position and controls
				// this.camera.position.z = 0;
				this.controls = new OrbitControls(this.camera, this.renderer.domElement);
				let orbitTarget = this.camera.position.clone();
				orbitTarget.z -= 5;
				this.controls.target = orbitTarget;
				this.controls.update();

				// light
				const light = new THREE.AmbientLight(0xffffff, Math.PI);
				this.scene.add(light);

				// Add a video background
				// const video = document.getElementById('video') as HTMLVideoElement;
				// const inputFrameTexture = new THREE.VideoTexture(video);
				// if (!inputFrameTexture) {
				// 	throw new Error("Failed to get the 'input_frame' texture!");
				// }
				// inputFrameTexture.colorSpace = THREE.SRGBColorSpace;
				// const inputFramesDepth = 500;
				// const inputFramesPlane = createCameraPlaneMesh(
				// 	this.camera,
				// 	inputFramesDepth,
				// 	new THREE.MeshBasicMaterial({ map: inputFrameTexture })
				// );
				// this.scene.add(inputFramesPlane);

				// Render the scene
				this.render();

				window.addEventListener('resize', this.resize.bind(this));
			}

			resize() {
				this.width = window.innerWidth;
				this.height = window.innerHeight;
				this.camera.aspect = this.width / this.height;
				this.camera.updateProjectionMatrix();

				this.renderer.setSize(this.width, this.height);
				this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

				this.renderer.render(this.scene, this.camera);
			}

			render(time: number = this.lastTime): void {
				const delta = (time - this.lastTime) / 1000;
				this.lastTime = time;
				// Call all registered callbacks with deltaTime parameter
				for (const callback of this.callbacks) {
					callback(delta);
				}
				// Render the scene
				this.renderer.render(this.scene, this.camera);
				// Request next frame
				requestAnimationFrame((t) => this.render(t));
			}
		}

		interface MatrixRetargetOptions {
			decompose?: boolean;
			scale?: number;
		}

		class Avatar {
			scene: THREE.Scene;
			loader: GLTFLoader = new GLTFLoader();
			gltf?: GLTF;
			root?: THREE.Bone;
			morphTargetMeshes: THREE.Mesh[] = [];
			url: string;
			/* THREEJS WORLD SETUP */
			vrm?: VRM;
			// lookat target
			lookAtTarget = new THREE.Object3D();

			clock = new THREE.Clock();

			constructor(url: string, scene: THREE.Scene) {
				this.url = url;
				this.scene = scene;

				// Install GLTFLoader plugin
				this.loader.register((parser) => {
					return new VRMLoaderPlugin(parser);
				});

				this.loadModel(this.url);
			}

			loadModel(url: string) {
				this.url = url;
				this.loader.load(
					// URL of the model you want to load
					url,
					// Callback when the resource is loaded
					(gltf) => {
						if (this.gltf) {
							// Reset GLTF and morphTargetMeshes if a previous model was loaded.
							this.gltf.scene.remove();
							this.morphTargetMeshes = [];
						}
						this.gltf = gltf;

						this.vrm = gltf.userData.vrm;
						VRMUtils.removeUnnecessaryVertices(gltf.scene);
						VRMUtils.removeUnnecessaryJoints(gltf.scene);

						this.adjustArmsToAPose(this.vrm);
						this.vrm.update(0);

						// add the loaded vrm to the scene
						this.scene.add(this.vrm.scene);

						this.scene.add(this.lookAtTarget);
						this.lookAtTarget.position.z = 10;

						this.vrm.lookAt.target = this.lookAtTarget;

						// deal with vrm features
						console.log(this.vrm);

						// this.scene.add(gltf.scene);
						this.init(this.vrm);

						// const gridHelper = new THREE.GridHelper(10, 10);
						// this.scene.add(gridHelper);

						// const axesHelper = new THREE.AxesHelper(5);
						// this.scene.add(axesHelper);
					},

					// Called while loading is progressing
					(progress) =>
						console.log('Loading model...', 100.0 * (progress.loaded / progress.total), '%'),
					// Called when loading has errors
					(error) => console.error(error)
				);
			}

			// Adjust arms to A-pose function
			adjustArmsToAPose(vrm: VRM) {
				const leftUpperArm = vrm.humanoid.getNormalizedBoneNode('leftUpperArm');
				const rightUpperArm = vrm.humanoid.getNormalizedBoneNode('rightUpperArm');
				if (leftUpperArm) {
					leftUpperArm.rotation.z = -Math.PI / 2.5; // Rotate left arm down
				}
				if (rightUpperArm) {
					rightUpperArm.rotation.z = Math.PI / 2.5; // Rotate right arm down
				}
			}

			init(gltf: VRM) {
				gltf.scene.traverse((object) => {
					// Register first bone found as the root
					// if ((object as THREE.Bone).isBone && !this.root) {
					// 	this.root = object as THREE.Bone;
					// 	console.log(object);
					// }
					// Return early if no mesh is found.
					if (!(object as THREE.Mesh).isMesh) {
						// console.warn(`No mesh found`);
						return;
					}

					const mesh = object as THREE.Mesh;
					// Reduce clipping when model is close to camera.
					mesh.frustumCulled = false;

					// Return early if mesh doesn't include morphable targets
					if (!mesh.morphTargetDictionary || !mesh.morphTargetInfluences) {
						// console.warn(`Mesh ${mesh.name} does not have morphable targets`);
						return;
					}
					this.morphTargetMeshes.push(mesh);
				});
			}

			updateBlendshapes(blendshapes: Map<string, number>) {
				if (!this.vrm) {
					return;
				}
				const deltaTime = this.clock.getDelta();

				for (const mesh of this.morphTargetMeshes) {
					if (!mesh.morphTargetDictionary || !mesh.morphTargetInfluences) {
						// console.warn(`Mesh ${mesh.name} does not have morphable targets`);
						continue;
					}

					for (const [name, value] of blendshapes) {
						if (name === 'eyeBlinkLeft') {
							const blink = (value - 0.5) * 2;
							this.vrm.expressionManager.setValue('blinkLeft', blink);
							this.vrm.expressionManager.setValue('blinkRight', blink);
						}

						// if (name === 'eyeBlinkRight') {
						// 	const blink = value < 0.8 ? 0 : value;
						// 	this.vrm.expressionManager.setValue('blinkRight', blink);
						// }

						if (name === 'jawOpen') {
							const jawOpen = value * 1.8;
							if (jawOpen > 0.5) {
								this.vrm.expressionManager.setValue('ee', 0);
								this.vrm.expressionManager.setValue('aa', jawOpen);
							} else {
								this.vrm.expressionManager.setValue('aa', 0);
								this.vrm.expressionManager.setValue('ee', jawOpen);
							}
						}

						if (name === 'mouthPucker') {
							const ou = (value - 0.5) * 2;
							if (ou > 0) {
								this.vrm.expressionManager.setValue('ou', ou);
								this.vrm.expressionManager.setValue('ee', 0);
								this.vrm.expressionManager.setValue('aa', 0);
							} else {
								this.vrm.expressionManager.setValue('ou', 0);
							}
						}

						if (name === 'mouthFunnel') {
							this.vrm.expressionManager.setValue('oh', value);
						}

						if (name === 'browInnerUp') {
							if (value > 0.7) {
								this.vrm.expressionManager.setValue('surprised', value);
							} else {
								this.vrm.expressionManager.setValue('surprised', 0);
							}
						}

						if (!Object.keys(mesh.morphTargetDictionary).includes(name)) {
							// console.warn(`Model morphable target ${name} not found`);
							continue;
						}

						const idx = mesh.morphTargetDictionary[name];
						mesh.morphTargetInfluences[idx] = value;
					}

					const mouthSmileRight = blendshapes.get('mouthSmileRight');
					const mouthSmileLeft = blendshapes.get('mouthSmileLeft');
					if ((mouthSmileRight || 0) + (mouthSmileLeft || 0) > 0.5) {
						this.vrm.expressionManager.setValue('blinkLeft', 0);
						this.vrm.expressionManager.setValue('blinkRight', 0);
						this.vrm.expressionManager.setValue('happy', mouthSmileRight + mouthSmileLeft);
						this.vrm.expressionManager.setValue('surprised', 0);
					} else {
						this.vrm.expressionManager.setValue('happy', 0);
					}

					if (blendshapes) {
						const rightEyeLookRight =
							(blendshapes.get('eyeLookOutRight') || 0) - (blendshapes.get('eyeLookInRight') || 0);
						const leftEyeLookRight =
							(blendshapes.get('eyeLookInLeft') || 0) - (blendshapes.get('eyeLookOutLeft') || 0);

						const rightEyeLookUp =
							(blendshapes.get('eyeLookUpRight') || 0) - (blendshapes.get('eyeLookDownRight') || 0);
						const leftEyeLookUp =
							(blendshapes.get('eyeLookUpLeft') || 0) - (blendshapes.get('eyeLookDownLeft') || 0);

						this.lookAtTarget.position.x = (10 * (rightEyeLookRight + leftEyeLookRight)) / 2;
						this.lookAtTarget.position.y = 1.7 + (10 * (rightEyeLookUp + leftEyeLookUp)) / 2;
					}
				}

				this.vrm.update(deltaTime);
			}

			/**
			 * Apply a position, rotation, scale matrix to current GLTF.scene
			 * @param matrix
			 * @param matrixRetargetOptions
			 * @returns
			 */
			applyMatrix(matrix: THREE.Matrix4, matrixRetargetOptions?: MatrixRetargetOptions): void {
				const { decompose = false, scale = 1 } = matrixRetargetOptions || {};
				if (!this.gltf) {
					return;
				}
				// Three.js will update the object matrix when it render the page
				// according the object position, scale, rotation.
				// To manually set the object matrix, you have to set autoupdate to false.
				matrix.scale(new THREE.Vector3(scale, scale, scale));
				this.gltf.scene.matrixAutoUpdate = false;
				// Set new position and rotation from matrix
				this.gltf.scene.matrix.copy(matrix);
			}

			/**
			 * Takes the root object in the avatar and offsets its position for retargetting.
			 * @param offset
			 * @param rotation
			 */
			offsetRoot(offset: THREE.Vector3, rotation?: THREE.Vector3): void {
				if (this.root) {
					this.root.position.copy(offset);
					if (rotation) {
						let offsetQuat = new THREE.Quaternion().setFromEuler(
							new THREE.Euler(rotation.x, rotation.y, rotation.z)
						);
						this.root.quaternion.copy(offsetQuat);
					}
				}
			}
		}

		let faceLandmarker: FaceLandmarker;
		let video: HTMLVideoElement;

		const scene = new BasicScene();
		const avatar = new Avatar('/models/sample2.vrm', scene.scene);

		function detectFaceLandmarks(time: DOMHighResTimeStamp): void {
			if (!faceLandmarker) {
				return;
			}
			const landmarks = faceLandmarker.detectForVideo(video, time);

			// Apply transformation
			const transformationMatrices = landmarks.facialTransformationMatrixes;
			if (transformationMatrices && transformationMatrices.length > 0) {
				const matrix = new THREE.Matrix4().fromArray(transformationMatrices[0].data);
				const originalQuaternion = new THREE.Quaternion().setFromRotationMatrix(matrix);

				const reflectQuaternion = new THREE.Quaternion(
					originalQuaternion.x > 0 ? originalQuaternion.x : originalQuaternion.x * 0.3,
					-originalQuaternion.y * 0.5,
					-originalQuaternion.z,
					originalQuaternion.w
				);

				// Example of applying matrix directly to the avatar
				// avatar.applyMatrix(matrix, { scale: 40 });

				const LEFT_EYE_INDEX = 33;
				const RIGHT_EYE_INDEX = 263;
				const NOSE_INDEX = 4;

				// 左目と右目のランドマーク位置を取得
				const leftEye = landmarks.faceLandmarks[0][LEFT_EYE_INDEX];
				const rightEye = landmarks.faceLandmarks[0][RIGHT_EYE_INDEX];
				const nose = landmarks.faceLandmarks[0][NOSE_INDEX];

				// 顔の中心点を計算（左目と右目の中点）
				const centerX = (leftEye.x + rightEye.x) / 2;
				const centerY = (leftEye.y + rightEye.y) / 2;
				const centerZ = (leftEye.z + rightEye.z) / 2;

				// 顔の中心点から鼻先へのベクトルを計算
				const vectorX = centerX - nose.x;
				const vectorY = centerY - nose.y;
				const vectorZ = centerZ - nose.z;

				// 頭のY軸の回転角度（ヨー角）を計算
				yaw = Math.atan2(vectorX, vectorZ);

				// 頭のX軸の回転角度（ピッチ角）を計算
				pitch = Math.atan2(vectorY, vectorZ);

				// 頭のZ軸の回転角度（ロール角）を計算
				roll = Math.atan2(rightEye.y - leftEye.y, rightEye.x - leftEye.x);

				if (avatar.vrm) {
					const headBone = avatar.vrm.humanoid.getNormalizedBoneNode('head');
					const neckBone = avatar.vrm.humanoid.getNormalizedBoneNode('neck');
					const spineBone = avatar.vrm.humanoid.getNormalizedBoneNode('spine');

					spineBone.quaternion.slerp(adjustQuaternionAngleRatio(reflectQuaternion, 0.6), 0.3);
					neckBone.quaternion
						.copy(spineBone.quaternion)
						.multiply(adjustQuaternionAngleRatio(reflectQuaternion, 0.2));
					headBone.quaternion
						.copy(neckBone.quaternion)
						.multiply(adjustQuaternionAngleRatio(reflectQuaternion, 0));

					// // Spine rotation
					// const spineRotation = new THREE.Quaternion().setFromEuler(
					// 	new THREE.Euler(pitch, yaw * 0.5, roll * 0.5, 'YXZ')
					// );
					// spineBone.quaternion.slerp(spineRotation, 0.1);

					// // Neck rotation relative to spine
					// const neckRotation = new THREE.Quaternion().setFromEuler(
					// 	new THREE.Euler(0, yaw * 0.5, roll * 0.5, 'YXZ')
					// );
					// neckBone.quaternion.copy(spineBone.quaternion).multiply(neckRotation);

					// // Head rotation relative to neck
					// const headRotation = new THREE.Quaternion().setFromEuler(
					// 	new THREE.Euler(pitch * 0.2, -yaw * 0.2, -roll * 0.2, 'YXZ')
					// );
					// headBone.quaternion.copy(neckBone.quaternion).multiply(headRotation);

					// const headRotation = new THREE.Quaternion().setFromEuler(
					// 	new THREE.Euler(pitch * 0.5, -yaw * 0.5, -roll * 0.5, 'YXZ')
					// );
					// const neckRotation = new THREE.Quaternion().setFromEuler(
					// 	new THREE.Euler(pitch * 0.3, -yaw * 0.3, -roll * 0.3, 'YXZ')
					// );
					// const spineRotation = new THREE.Quaternion().setFromEuler(
					// 	new THREE.Euler(pitch * 0.2, -yaw * 0.2, -roll * 0.2, 'YXZ')
					// );

					// headBone.quaternion.slerp(headRotation, 0.1);
					// neckBone.quaternion.slerp(neckRotation, 0.1);
					// spineBone.quaternion.slerp(spineRotation, 0.1);

					// const headPitch = pitch * 0;
					// const headYaw = -yaw * 0;
					// const headRoll = roll * 0.5;

					// const neckPitch = pitch * 0.5 * 0;
					// const neckYaw = -yaw * 0.5 * 0;
					// const neckRoll = -roll * 0.5;

					// const spinePitch = pitch * 0.2 * 0;
					// const spineYaw = -yaw * 0.2 * 0;
					// const spineRoll = -roll * 0.2;

					// headBone.rotation.set(headPitch, headYaw, headRoll, 'YXZ');
					// neckBone.rotation.set(neckPitch, neckYaw, neckRoll, 'YXZ');
					// spineBone.rotation.set(spinePitch, spineYaw, spineRoll, 'YXZ');

					// // Three.jsのEulerオブジェクトを使用して回転を設定
					// neckBone.rotation.set(
					// 	-pitchAngle + Math.PI / 1.25,
					// 	yawAngle + Math.PI,
					// 	rollAngle + Math.PI,
					// 	'XYZ'
					// );

					// neckBone.rotation.set(-pitchAngle * 0.5, yawAngle * 0.5, rollAngle * 0.5, 'XYZ');

					// const chestBone = avatar.vrm.humanoid.getNormalizedBoneNode('spine');
					// chestBone.rotation.set(
					// 	-pitchAngle + Math.PI / 1.25,
					// 	yawAngle + Math.PI,
					// 	rollAngle + Math.PI,
					// 	'XYZ'
					// );
				}
			}

			// Apply Blendshapes
			const blendshapes = landmarks.faceBlendshapes;
			if (blendshapes && blendshapes.length > 0) {
				const coefsMap = retarget(blendshapes);
				avatar.updateBlendshapes(coefsMap);
			}
		}

		function adjustQuaternionAngleRatio(quaternion: THREE.Quaternion, ratio: number) {
			return new THREE.Quaternion(
				quaternion.x * ratio,
				quaternion.y * ratio,
				quaternion.z * ratio,
				quaternion.w
			);
		}

		function retarget(blendshapes: Classifications[]) {
			const categories = blendshapes[0].categories;
			let coefsMap = new Map<string, number>();
			for (let i = 0; i < categories.length; ++i) {
				const blendshape = categories[i];
				// Adjust certain blendshape values to be less prominent.
				switch (blendshape.categoryName) {
					case 'browOuterUpLeft':
						blendshape.score *= 1.2;
						break;
					case 'browOuterUpRight':
						blendshape.score *= 1.2;
						break;
					case 'eyeBlinkLeft':
						blendshape.score *= 1.2;
						break;
					case 'eyeBlinkRight':
						blendshape.score *= 1.2;
						break;
					default:
				}
				coefsMap.set(categories[i].categoryName, categories[i].score);
			}
			return coefsMap;
		}

		function onVideoFrame(time: DOMHighResTimeStamp): void {
			// Do something with the frame.
			detectFaceLandmarks(time);
			// Re-register the callback to be notified about the next frame.
			video.requestVideoFrameCallback(onVideoFrame);
		}

		// Stream webcam into landmarker loop (and also make video visible)
		async function streamWebcamThroughFaceLandmarker(): Promise<void> {
			video = document.getElementById('video') as HTMLVideoElement;

			function onAcquiredUserMedia(stream: MediaStream): void {
				video.srcObject = stream;
				video.onloadedmetadata = () => {
					video.play();
				};
			}

			try {
				const evt = await navigator.mediaDevices.getUserMedia({
					audio: false,
					video: {
						facingMode: 'user',
						width: 1280,
						height: 720
					}
				});
				onAcquiredUserMedia(evt);
				video.requestVideoFrameCallback(onVideoFrame);
			} catch (e: unknown) {
				console.error(`Failed to acquire camera feed: ${e}`);
			}
		}

		async function runDemo() {
			await streamWebcamThroughFaceLandmarker();
			const vision = await FilesetResolver.forVisionTasks(
				'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.14/wasm'
			);
			faceLandmarker = await FaceLandmarker.createFromModelPath(
				vision,
				'https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/latest/face_landmarker.task'
			);
			await faceLandmarker.setOptions({
				baseOptions: {
					delegate: 'GPU'
				},
				runningMode: 'VIDEO',
				outputFaceBlendshapes: true,
				outputFacialTransformationMatrixes: true
			});

			console.log('Finished Loading MediaPipe Model.');
		}

		runDemo();
	});
</script>

<div class="container">
	<video autoplay playsinline id="video"><track kind="captions" /></video>
</div>

<style>
</style>
