<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Slow-Mo Heart Explosion</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; color: #00ffcc; font-family: 'Courier New', Courier, monospace; }
        #overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; display: flex; flex-direction: column; 
                   justify-content: center; align-items: center; background: rgba(0,0,0,0.9); z-index: 100; text-align: center;}
        button { padding: 15px 30px; font-size: 18px; background: transparent; border: 2px solid #00ffcc; 
                 color: #00ffcc; cursor: pointer; border-radius: 5px; text-transform: uppercase; margin-top: 20px;}
        button:hover { background: #00ffcc; color: #000; }
        #ui-status { position: absolute; top: 20px; left: 20px; z-index: 10; font-size: 12px; pointer-events: none; }
        #video-preview { position: absolute; bottom: 20px; right: 20px; width: 180px; height: 135px; 
                         border: 1px solid #00ffcc; border-radius: 8px; transform: scaleX(-1); background: #111; opacity: 0.4; }
        #message { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); 
                   pointer-events: none; font-size: 45px; font-weight: bold; text-shadow: 0 0 20px #00ffcc; 
                   display: none; text-align: center; width: 100%; z-index: 5; }
        canvas { display: block; }
    </style>
</head>
<body>

    <div id="overlay">
        <h1>HEART PROTOCOL READY</h1>
        <p>OPEN HAND = SLOW EXPLOSION<br>CLOSED HAND = SLOW COLLAPSE</p>
        <button onclick="init()">Launch Program</button>
    </div>

    <div id="ui-status">
        <div id="status">STATUS: STANDBY</div>
        <div id="input-mode">INPUT: SCANNING...</div>
    </div>

    <div id="message">ILOVEYOUBABYYAHHH</div>

    <video id="video-preview" autoplay playsinline muted></video>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

    <script>
        let scene, camera, renderer, points, particlesCount = 15000;
        let heartPositions, velocities, currentPositions;
        let isFist = true; 
        let lastFistState = true;

        // PHYSICS CONFIG (The "Slow-Mo" Sliders)
        const EXPLOSION_STRENGTH = 0.5; // Lower = gentler burst
        const GRAVITY_PULL = 0.005;     // Lower = slower formation
        const FRICTION = 0.94;          // Closer to 1.0 = more "drifting" (less air resistance)

        const previewElement = document.getElementById('video-preview');
        const messageElement = document.getElementById('message');
        const statusText = document.getElementById('status');
        const inputModeText = document.getElementById('input-mode');

        function setupThree() {
            scene = new THREE.Scene();
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            const geometry = new THREE.BufferGeometry();
            currentPositions = new Float32Array(particlesCount * 3);
            heartPositions = new Float32Array(particlesCount * 3);
            velocities = new Float32Array(particlesCount * 3);

            for (let i = 0; i < particlesCount; i++) {
                const i3 = i * 3;
                const t = Math.random() * Math.PI * 2;
                
                const x = 16 * Math.pow(Math.sin(t), 3);
                const y = 13 * Math.cos(t) - 5 * Math.cos(2*t) - 2 * Math.cos(3*t) - Math.cos(4*t);
                
                heartPositions[i3] = x / 12;
                heartPositions[i3+1] = y / 12;
                heartPositions[i3+2] = (Math.random() - 0.5) * 1.5;

                currentPositions[i3] = 0;
                currentPositions[i3+1] = 0;
                currentPositions[i3+2] = 0;
            }

            geometry.setAttribute('position', new THREE.BufferAttribute(currentPositions, 3));
            const material = new THREE.PointsMaterial({ 
                size: 0.04, 
                color: 0x00ffcc, 
                transparent: true, 
                opacity: 0.7,
                blending: THREE.AdditiveBlending 
            });
            points = new THREE.Points(geometry, material);
            scene.add(points);
            camera.position.z = 6;
        }

        async function init() {
            document.getElementById('overlay').style.display = 'none';
            messageElement.style.display = 'block'; 
            setupThree();
            animate();
            
            setTimeout(() => { isFist = false; lastFistState = false; triggerExplosion(); }, 200);

            try {
                const hands = new Hands({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`});
                hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.6, minTrackingConfidence: 0.6 });
                hands.onResults(onResults);

                const cam = new Camera(previewElement, {
                    onFrame: async () => { await hands.send({image: previewElement}); },
                    width: 640, height: 480
                });
                await cam.start();
                statusText.innerText = "STATUS: SYSTEM ACTIVE";
            } catch (e) {
                statusText.innerText = "STATUS: MOUSE MODE";
                enableMouseControl();
            }
        }

        function triggerExplosion() {
            for (let i = 0; i < particlesCount * 3; i++) {
                // Slower random burst
                velocities[i] = (Math.random() - 0.5) * EXPLOSION_STRENGTH; 
            }
        }

        function enableMouseControl() {
            window.addEventListener('mousemove', (e) => {
                points.rotation.y = (e.clientX / window.innerWidth - 0.5) * 2;
                points.rotation.x = (e.clientY / window.innerHeight - 0.5) * 2;
            });
            window.addEventListener('mousedown', () => updateGestureState(true));
            window.addEventListener('mouseup', () => updateGestureState(false));
        }

        function onResults(results) {
            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                const landmarks = results.multiHandLandmarks[0];
                points.rotation.y = (landmarks[8].x - 0.5) * 2.2;
                points.rotation.x = (landmarks[8].y - 0.5) * 2.2;

                const dx = landmarks[12].x - landmarks[0].x;
                const dy = landmarks[12].y - landmarks[0].y;
                const distance = Math.sqrt(dx*dx + dy*dy);
                updateGestureState(distance < 0.28);
            }
        }

        function updateGestureState(fist) {
            isFist = fist;
            if (lastFistState === true && isFist === false) {
                triggerExplosion(); 
            }
            lastFistState = isFist;
            inputModeText.innerText = isFist ? "INPUT: COLLAPSING" : "INPUT: EXPLODING";
        }

        function animate() {
            requestAnimationFrame(animate);
            const pos = points.geometry.attributes.position;
            
            for (let i = 0; i < particlesCount; i++) {
                const i3 = i * 3;
                
                if (isFist) {
                    // Slow collapse to center
                    velocities[i3] += (0 - pos.array[i3]) * (GRAVITY_PULL * 2);
                    velocities[i3+1] += (0 - pos.array[i3+1]) * (GRAVITY_PULL * 2);
                    velocities[i3+2] += (0 - pos.array[i3+2]) * (GRAVITY_PULL * 2);
                } else {
                    // Slow pull to heart shape
                    const tx = heartPositions[i3];
                    const ty = heartPositions[i3+1];
                    const tz = heartPositions[i3+2];
                    
                    velocities[i3] += (tx - pos.array[i3]) * GRAVITY_PULL;
                    velocities[i3+1] += (ty - pos.array[i3+1]) * GRAVITY_PULL;
                    velocities[i3+2] += (tz - pos.array[i3+2]) * GRAVITY_PULL;
                }

                // Apply Friction (Damping)
                velocities[i3] *= FRICTION;
                velocities[i3+1] *= FRICTION;
                velocities[i3+2] *= FRICTION;

                // Move position
                pos.array[i3] += velocities[i3];
                pos.array[i3+1] += velocities[i3+1];
                pos.array[i3+2] += velocities[i3+2];
            }
            pos.needsUpdate = true;

            // Pulsing the heart scale
            const pulse = 1 + Math.sin(Date.now() * 0.002) * 0.04;
            points.scale.set(pulse, pulse, pulse);

            renderer.render(scene, camera);
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
