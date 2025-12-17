<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Hyper-Realistic Black Hole</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; }
        canvas { display: block; width: 100vw; height: 100vh; }
    </style>
</head>
<body>
    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
            }
        }
    </script>
    <script type="module">
        import * as THREE from 'three';
        import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
        import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
        import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

        const fragmentShader = `
            uniform float u_time;
            uniform vec2 u_res;

            float hash(float n) { return fract(sin(n) * 43758.5453); }
            float noise(vec3 x) {
                vec3 p = floor(x); vec3 f = fract(x);
                f = f*f*(3.0-2.0*f);
                float n = p.x + p.y*57.0 + 113.0*p.z;
                return mix(mix(mix(hash(n+0.0),hash(n+1.0),f.x),mix(hash(n+57.0),hash(n+58.0),f.x),f.y),
                           mix(mix(hash(n+113.0),hash(n+114.0),f.x),mix(hash(n+170.0),hash(n+171.0),f.x),f.y),f.z);
            }

            void main() {
                vec2 uv = (gl_FragCoord.xy - 0.5 * u_res.xy) / min(u_res.y, u_res.x);
                vec3 ro = vec3(0, 2, -10); 
                vec3 rd = normalize(vec3(uv, 2.0));
                
                float a = 0.3; 
                rd.yz *= mat2(cos(a), -sin(a), sin(a), cos(a));
                ro.yz *= mat2(cos(a), -sin(a), sin(a), cos(a));

                vec3 col = vec3(0.0);
                float t = 0.0;
                float glow = 0.0;

                for(int i = 0; i < 80; i++) {
                    vec3 p = ro + rd * t;
                    float d = length(p);
                    vec3 force = -normalize(p) * (0.12 / (d * d + 0.01));
                    rd = normalize(rd + force);
                    if(d < 1.1) { col = vec3(0); break; }
                    float diskThickness = 0.12 + d * 0.03;
                    if(abs(p.y) < diskThickness && d > 1.2 && d < 6.5) {
                        float swirl = noise(p * 1.2 + vec3(u_time * 0.5, 0, 0));
                        glow += swirl * (1.0 - d/6.5) * 0.05;
                    }
                    t += 0.25;
                }
                col += vec3(1.0, 0.5, 0.1) * glow * 4.0;
                gl_FragColor = vec4(col, 1.0);
            }
        `;

        const scene = new THREE.Scene();
        const camera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        document.body.appendChild(renderer.domElement);

        const renderScene = new RenderPass(scene, camera);
        const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.6, 0.4, 0.1);
        const composer = new EffectComposer(renderer);
        composer.addPass(renderScene);
        composer.addPass(bloomPass);

        const material = new THREE.ShaderMaterial({
            uniforms: { 
                u_time: { value: 0 }, 
                u_res: { value: new THREE.Vector2(window.innerWidth, window.innerHeight) } 
            },
            fragmentShader
        });
        scene.add(new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material));

        function animate(time) {
            material.uniforms.u_time.value = time * 0.001;
            composer.render(); 
            requestAnimationFrame(animate);
        }
        animate(0);

        window.addEventListener('resize', () => {
            const w = window.innerWidth;
            const h = window.innerHeight;
            renderer.setSize(w, h);
            composer.setSize(w, h);
            material.uniforms.u_res.value.set(w, h);
        });
    </script>
</body>
</html>
