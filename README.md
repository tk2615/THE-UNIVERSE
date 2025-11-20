
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Evolving Wordscape</title>
    
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Shippori+Mincho:wght@800&family=Roboto+Mono:wght@500&display=swap" rel="stylesheet">
    
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; color: white; font-family: 'Shippori Mincho', serif; transition: background-color 2s ease; }
        #canvas-container { width: 100vw; height: 100vh; position: fixed; top: 0; left: 0; z-index: 1; }
        
        /* HUD */
        #hud {
            position: absolute; top: 30px; left: 30px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 11px;
            line-height: 1.6; pointer-events: none; mix-blend-mode: difference;
            color: rgba(255,255,255,0.9);
        }
        .hud-val { font-weight: 700; margin-left: 5px; }

        /* Sound Status Viz */
        #audio-viz {
            position: absolute; bottom: 30px; left: 30px; z-index: 20;
            width: 200px; height: 60px; pointer-events: none;
        }
        .bar { height: 2px; background: rgba(255,255,255,0.5); margin-bottom: 4px; transition: width 0.2s; }
        .label { font-size: 9px; font-family: 'Roboto Mono'; color: #888; margin-bottom: 2px; }

        /* Overlay */
        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: #000; z-index: 100; display: flex; flex-direction: column;
            align-items: center; justify-content: center; transition: opacity 1.5s ease;
        }
        .title { font-size: 5vw; font-weight: 800; letter-spacing: 0.3em; margin-bottom: 10px; }
        .subtitle { font-size: 12px; color: #888; margin-bottom: 40px; font-family: 'Roboto Mono'; letter-spacing: 2px; }
        #start-btn {
            background: transparent; border: 1px solid rgba(255,255,255,0.5); color: white;
            padding: 15px 60px; font-family: 'Shippori Mincho', serif; font-size: 16px; font-weight: 800;
            letter-spacing: 4px; cursor: pointer; transition: 0.4s; border-radius: 2px;
        }
        #start-btn:hover { background: white; color: black; box-shadow: 0 0 50px rgba(255,255,255,0.5); transform: scale(1.05); }
        
        /* Log Toggle */
        #controls { position: absolute; top: 30px; right: 30px; z-index: 30; }
        .btn { background: rgba(128,128,128,0.2); border: 1px solid rgba(128,128,128,0.5); color: #fff; padding: 8px 16px; cursor: pointer; border-radius: 20px; backdrop-filter: blur(5px); font-size: 10px; }
        
        /* Log */
        #sys-log {
            position: absolute; bottom: 30px; right: 30px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 10px; color: #00ffaa;
            pointer-events: none; text-align: right; opacity: 0.7;
        }
    </style>
</head>
<body>

    <div id="overlay">
        <div class="title">流動する言葉</div>
        <div class="subtitle">Fluid Audio-Visual News Stream</div>
        <button id="start-btn">接続 / CONNECT</button>
    </div>

    <div id="hud">
        <div>TIME <span id="clock-display" class="hud-val">--:--:--</span></div>
        <div>LOC  <span id="loc-display" class="hud-val">Scanning...</span></div>
        <div>ENV  <span id="env-display" class="hud-val">--</span></div>
    </div>

    <div id="audio-viz">
        <div class="label">HARMONY (HOPE/ART)</div>
        <div class="bar" id="bar-harmony" style="width: 50%;"></div>
        <div class="label">TENSION (CRISIS)</div>
        <div class="bar" id="bar-tension" style="width: 10%; background: #ff3333;"></div>
        <div class="label">NOISE (TECH/CHAOS)</div>
        <div class="bar" id="bar-noise" style="width: 20%; background: #00ffff;"></div>
    </div>

    <div id="sys-log">waiting for stream...</div>

    <div id="controls">
        <button class="btn" onclick="toggleLog()">LOG ON/OFF</button>
    </div>

    <div id="canvas-container"></div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/simplex-noise/2.4.0/simplex-noise.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.js"></script>

    <script>
        // --- CONFIGURATION ---
        const CONFIG = {
            spawnRate: 100,      // 高速出現
            wordLife: 40000,     // 長時間滞在
            baseSize: 60,        // 【変更点】文字サイズを大幅アップ (前回比2倍)
            minSize: 40,         // 【変更点】遠くてもこれ以上小さくしない
            connectDist: 500,    // 接続距離
            particleCount: 4000,
            fogDensity: 0.0004   // 【変更点】フォグを薄くして奥まで見えやすく
        };

        const FEEDS = [
            { name: "NHK", url: "https://www3.nhk.or.jp/rss/news/cat0.xml" },
            { name: "YAHOO", url: "https://news.yahoo.co.jp/rss/topics/top-picks.xml" },
            { name: "WIRED", url: "https://www.wired.com/feed/rss" },
            { name: "BBC", url: "https://feeds.bbci.co.uk/news/world/rss.xml" },
            { name: "REUTERS", url: "https://www.reutersagency.com/feed/?best-topics=political-general&post_type=best" }
        ];
        const PROXY = "https://api.rss2json.com/v1/api.json?rss_url=";

        const TYPE_NEUTRAL=0, TYPE_HOPE=1, TYPE_CRISIS=2, TYPE_TECH=3, TYPE_NATURE=4, TYPE_ART=5;
        const CLASSIFIER = {
            crisis: ["WAR","CRISIS","DEATH","KILL","ATTACK","DANGER","WARNING","BOMB","VICTIM","戦争","危機","災害","殺害","事故","警報","死亡","脅威","爆発","炎上"],
            hope:   ["PEACE","WIN","HOPE","SUCCESS","JOY","BEST","HEAL","LOVE","SAVE","VICTORY","平和","希望","支援","回復","優勝","成功","愛","救助","勝利","祝福","金メダル"],
            tech:   ["AI","SPACE","DATA","FUTURE","CYBER","QUANTUM","ROBOT","APP","CODE","APPLE","GOOGLE","宇宙","未来","技術","開発","人工知能","量子","ロケット","電脳"],
            nature: ["EARTH","CLIMATE","OCEAN","FOREST","ANIMAL","PLANET","STORM","GREEN","地球","環境","海","森","気候","動物","台風","猛暑","温暖化","自然"],
            art:    ["ART","SOUL","COLOR","MIND","DREAM","POEM","MUSIC","FILM","IDEA","DESIGN","芸術","魂","夢","思考","色","哲学","映画","音楽","言葉"]
        };
        const STOP_WORDS = ["THE","AND","FOR","WITH","NEWS","LIVE","FROM","THAT","REPORT","NEW","あ","い","う","え","お","の","に","は","を","が","で","て","と","も","し","た","る","こと","ます","です"];
        const BACKUP = [{t:"Vision",s:"SYS",y:TYPE_ART},{t:"Time",s:"SYS",y:TYPE_NEUTRAL},{t:"Echo",s:"SYS",y:TYPE_NATURE},{t:"Core",s:"SYS",y:TYPE_TECH}];

        let wordBuffer = [];
        let appState = { 
            mode: 'night',
            // Mood Values (0.0 - 1.0) for Audio Mixing
            tension: 0,
            harmony: 0.5,
            noise: 0
        };
        let isAudioReady = false;

        // --- UTILS ---
        function log(msg) {
            const el = document.getElementById('sys-log');
            if(el.style.display !== 'none') el.innerText = msg;
        }
        function toggleLog() {
            const el = document.getElementById('sys-log');
            el.style.display = (el.style.display === 'none') ? 'block' : 'none';
        }
        function updateTime() {
            const now = new Date();
            document.getElementById('clock-display').innerText = now.toLocaleTimeString();
            const h = now.getHours();
            if(h>=5 && h<8) appState.mode='early';
            else if(h>=8 && h<16) appState.mode='day';
            else if(h>=16 && h<19) appState.mode='evening';
            else appState.mode='night';
        }
        setInterval(updateTime, 1000);

        async function fetchEnv() {
            try {
                const ip = await fetch('https://ipapi.co/json/').then(r=>r.json());
                document.getElementById('loc-display').innerText = `${ip.city||'Unknown'}`;
                if(ip.latitude){
                    const w = await fetch(`https://api.open-meteo.com/v1/forecast?latitude=${ip.latitude}&longitude=${ip.longitude}&current_weather=true`).then(r=>r.json());
                    const temp = w.current_weather.temperature;
                    document.getElementById('env-display').innerText = `${temp}°C`;
                }
            } catch(e){}
        }

        // --- DATA ENGINE ---
        const segmenter = new Intl.Segmenter("ja-JP", { granularity: "word" });
        
        async function fetchFeeds() {
            log("Updating streams...");
            const promises = FEEDS.map(f => fetch(PROXY + encodeURIComponent(f.url)).then(r=>r.ok?r.json():null).catch(e=>null));
            const results = await Promise.all(promises);
            
            let count = 0;
            results.forEach((data, idx) => {
                if(data && data.items) {
                    data.items.forEach(item => {
                        const srcName = FEEDS[idx].name;
                        processTitle(item.title, srcName);
                        count++;
                    });
                }
            });
            if(count === 0) BACKUP.forEach(d => wordBuffer.push({text:d.t, source:d.s, type:d.y}));
            log(`Stream active: ${count} articles scanned`);
        }

        function processTitle(title, source) {
            const tempDiv = document.createElement("div");
            tempDiv.innerHTML = title;
            const cleanTitle = tempDiv.textContent || tempDiv.innerText || "";

            if(cleanTitle.match(/[亜-熙ぁ-んァ-ヶ]/)) {
                for(const s of segmenter.segment(cleanTitle)) {
                    if(s.isWordLike && s.segment.length>1 && !STOP_WORDS.includes(s.segment)) addWord(s.segment, source);
                }
            } else {
                cleanTitle.split(/\s+/).forEach(w => {
                    const c = w.replace(/[^a-zA-Z0-9]/g,'');
                    if(c.length>3 && !STOP_WORDS.includes(c.toUpperCase())) addWord(c, source);
                });
            }
        }

        function addWord(text, source) {
            let type = TYPE_NEUTRAL;
            const up = text.toUpperCase();
            if(CLASSIFIER.crisis.some(k=>up.includes(k))) type = TYPE_CRISIS;
            else if(CLASSIFIER.hope.some(k=>up.includes(k))) type = TYPE_HOPE;
            else if(CLASSIFIER.tech.some(k=>up.includes(k))) type = TYPE_TECH;
            else if(CLASSIFIER.nature.some(k=>up.includes(k))) type = TYPE_NATURE;
            else if(CLASSIFIER.art.some(k=>up.includes(k))) type = TYPE_ART;

            let disp = text;
            if(!text.match(/[亜-熙ぁ-んァ-ヶ]/)) disp = text.charAt(0).toUpperCase() + text.slice(1).toLowerCase();
            wordBuffer.push({ text: disp, type: type, source: source });
        }
        
        setInterval(fetchFeeds, 60000);

        // --- AUDIO ENGINE (FLUID MIXING) ---
        const Audio = {
            drone: null,
            tensionSynth: null,
            hopeSynth: null,
            noise: null,
            filter: null,
            
            async init() {
                await Tone.start();
                const master = new Tone.Reverb({decay: 8, wet: 0.5}).toDestination();
                const limiter = new Tone.Limiter(-2).connect(master);
                
                // 1. Fluid Drone (Base)
                this.filter = new Tone.AutoFilter(0.1).connect(limiter).start();
                this.drone = new Tone.PolySynth(Tone.Synth, {
                    oscillator: { type: "fatsine", count: 3, spread: 20 },
                    envelope: { attack: 2, decay: 1, sustain: 1, release: 3 }
                }).connect(this.filter);
                this.drone.volume.value = -12;

                // 2. Tension Layer (Distorted Lows)
                const dist = new Tone.Distortion(0.4).connect(limiter);
                this.tensionSynth = new Tone.MembraneSynth().connect(dist);
                this.tensionSynth.volume.value = -99; // Start silent

                // 3. Hope/Tech Layer (Bright Arps)
                const delay = new Tone.PingPongDelay("8n", 0.3).connect(limiter);
                this.hopeSynth = new Tone.PolySynth(Tone.FMSynth).connect(delay);
                this.hopeSynth.volume.value = -99;

                // 4. Noise Layer (Texture)
                this.noise = new Tone.Noise("pink").connect(new Tone.Filter(500, "lowpass").connect(limiter));
                this.noise.start();
                this.noise.volume.value = -99;

                // Start Background Chords
                this.startLoops();
                isAudioReady = true;
            },

            startLoops() {
                // Drone Chords
                const chords = [["C3","G3","C4"], ["A2","E3","A3"], ["F2","C3","F3"]];
                new Tone.Loop(t => {
                    const c = chords[Math.floor(Math.random()*chords.length)];
                    this.drone.triggerAttackRelease(c, "8m", t);
                }, "10m").start(0);
                Tone.Transport.start();
            },

            // 毎フレーム呼ばれて音響パラメータを更新
            updateMix() {
                if(!isAudioReady) return;

                // Smoothly ramp volumes based on global mood state
                // Tension -> Volume of MembraneSynth & Distortion amount
                const tensionVol = -30 + (appState.tension * 20); // -30 to -10
                if(appState.tension < 0.1) this.tensionSynth.volume.rampTo(-99, 1);
                else this.tensionSynth.volume.rampTo(tensionVol, 0.5);

                // Harmony -> Volume of FM Synth
                const harmonyVol = -30 + (appState.harmony * 15); 
                this.hopeSynth.volume.rampTo(harmonyVol, 1);

                // Noise -> Based on Tech/Chaos
                const noiseVol = -60 + (appState.noise * 40); // -60 to -20
                this.noise.volume.rampTo(noiseVol, 1);

                // Update UI Bars
                document.getElementById('bar-harmony').style.width = (appState.harmony * 100) + "%";
                document.getElementById('bar-tension').style.width = (appState.tension * 100) + "%";
                document.getElementById('bar-noise').style.width = (appState.noise * 100) + "%";
            },

            triggerNote(type) {
                if(!isAudioReady) return;
                const now = Tone.now();
                
                // Trigger melodic events based on type
                if(type === TYPE_CRISIS) {
                    this.tensionSynth.triggerAttackRelease("A1", "8n", now);
                } else if(type === TYPE_HOPE || type === TYPE_ART) {
                    const note = ["C5","D5","E5","G5"][Math.floor(Math.random()*4)];
                    this.hopeSynth.triggerAttackRelease(note, "16n", now);
                } else if(type === TYPE_TECH) {
                    this.hopeSynth.triggerAttackRelease("C6", "32n", now, 0.5);
                }
            }
        };

        // --- 3. VISUAL ENGINE ---
        const scene = new THREE.Scene();
        const fogCol = new THREE.Color(0x000000);
        // フォグ密度を下げて奥まで見えるように変更
        scene.fog = new THREE.FogExp2(fogCol, CONFIG.fogDensity);

        const camera = new THREE.PerspectiveCamera(60, window.innerWidth/window.innerHeight, 1, 4000);
        camera.position.z = 1200;
        const renderer = new THREE.WebGLRenderer({antialias: true, alpha: true});
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('canvas-container').appendChild(renderer.domElement);

        const wordGroup = new THREE.Group();
        const lineGroup = new THREE.Group();
        const bgGroup = new THREE.Group();
        scene.add(bgGroup); scene.add(lineGroup); scene.add(wordGroup);

        // Stars
        const pGeo = new THREE.BufferGeometry();
        const pPos = [];
        for(let i=0; i<CONFIG.particleCount; i++) pPos.push((Math.random()-0.5)*5000, (Math.random()-0.5)*5000, (Math.random()-0.5)*5000);
        pGeo.setAttribute('position', new THREE.Float32BufferAttribute(pPos, 3));
        const pMat = new THREE.PointsMaterial({ color: 0x888888, size: 3, transparent: true, opacity: 0.5 });
        bgGroup.add(new THREE.Points(pGeo, pMat));

        // Lines
        const lineGeo = new THREE.BufferGeometry();
        const linePos = new Float32Array(CONFIG.maxLines * 6);
        const lineCols = new Float32Array(CONFIG.maxLines * 6);
        lineGeo.setAttribute('position', new THREE.BufferAttribute(linePos, 3));
        lineGeo.setAttribute('color', new THREE.BufferAttribute(lineCols, 3));
        const lineMat = new THREE.LineBasicMaterial({ vertexColors: true, transparent: true, opacity: 0.3, blending: THREE.AdditiveBlending });
        const lines = new THREE.LineSegments(lineGeo, lineMat);
        lineGroup.add(lines);

        const activeSprites = [];

        function getGenreColor(type) {
            switch(type) {
                case TYPE_CRISIS: return { hex:"#ff2222", r:1, g:0.1, b:0.1 };
                case TYPE_HOPE:   return { hex:"#ffcc00", r:1, g:0.8, b:0 };
                case TYPE_TECH:   return { hex:"#00ffff", r:0, g:1, b:1 };
                case TYPE_NATURE: return { hex:"#00ff44", r:0, g:1, b:0.3 };
                case TYPE_ART:    return { hex:"#cc66ff", r:0.8, g:0.4, b:1 };
                default:          return { hex:"#aaaaaa", r:0.6, g:0.6, b:0.6 };
            }
        }

        function createTexture(text, source, type) {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            const wSize = 80; // Bigger font
            const sSize = 24;
            
            ctx.font = `bold ${wSize}px "Shippori Mincho", serif`;
            const tWidth = ctx.measureText(text).width;
            ctx.font = `500 ${sSize}px "Roboto Mono", monospace`;
            const sWidth = ctx.measureText(source).width;
            const width = Math.max(tWidth, sWidth) + 60;
            const height = wSize + sSize + 60;
            
            canvas.width = width; canvas.height = height;
            ctx.textAlign = "center"; ctx.textBaseline = "top";
            
            const col = getGenreColor(type);
            
            // Main Word
            ctx.shadowColor = col.hex; ctx.shadowBlur = 40; ctx.fillStyle = col.hex;
            ctx.font = `bold ${wSize}px "Shippori Mincho", serif`;
            ctx.fillText(text, width/2, 10);
            
            // Source Label
            ctx.shadowBlur = 0; ctx.fillStyle = (appState.mode === 'day') ? "#666666" : "#888888";
            ctx.font = `500 ${sSize}px "Roboto Mono", monospace`;
            ctx.fillText(source, width/2, wSize + 25);

            return { tex: new THREE.CanvasTexture(canvas), ratio: width/height, col: col };
        }

        function spawnWord() {
            if(wordBuffer.length === 0) return;
            const data = wordBuffer.shift();
            if(wordBuffer.length < 50) wordBuffer.push(data); // Keep looping

            // Update Mood State
            if(data.type === TYPE_CRISIS) appState.tension = Math.min(1, appState.tension + 0.2);
            if(data.type === TYPE_HOPE || data.type === TYPE_ART) appState.harmony = Math.min(1, appState.harmony + 0.1);
            if(data.type === TYPE_TECH) appState.noise = Math.min(1, appState.noise + 0.1);
            
            Audio.triggerNote(data.type);

            const { tex, ratio, col } = createTexture(data.text, data.source, data.type);
            const mat = new THREE.SpriteMaterial({ map: tex, transparent: true, opacity: 0, blending: THREE.AdditiveBlending, depthWrite: false });
            const sprite = new THREE.Sprite(mat);
            
            const range = 2500;
            sprite.position.set((Math.random()-0.5)*range, (Math.random()-0.5)*range*0.6, (Math.random()-0.5)*2000);
            
            // Base Size + Random Variation
            const size = Math.max(CONFIG.minSize, CONFIG.baseSize + Math.random() * 60);
            sprite.scale.set(size * ratio, size, 1);
            
            scene.add(sprite);
            activeSprites.push({
                mesh: sprite, age: 0, life: CONFIG.wordLife + Math.random()*5000, fadeIn: 1500, fadeOut: 1500,
                velocity: new THREE.Vector3((Math.random()-0.5)*0.2, (Math.random()-0.5)*0.2, (Math.random()-0.5)*0.1),
                type: data.type, col: col
            });
        }

        // --- ANIMATION LOOP ---
        const simplex = new SimplexNoise();
        let time = 0;

        function animate() {
            requestAnimationFrame(animate);
            time += 0.001;

            // Decay Moods
            appState.tension *= 0.995;
            appState.harmony *= 0.995;
            appState.noise *= 0.995;
            Audio.updateMix();

            // Colors
            let bgHex = (appState.mode === 'day') ? 0xf2f2f2 : 0x020205;
            let bgCol = new THREE.Color(bgHex);
            scene.background = scene.background ? scene.background.lerp(bgCol, 0.01) : bgCol;
            fogCol.lerp(bgCol, 0.01); scene.fog.color.copy(fogCol);
            
            // Flash background on high tension
            if(appState.tension > 0.8 && Math.random() > 0.95) {
                scene.background.setHex(0x220000);
            }

            bgGroup.rotation.y = time * 0.02;

            for(let i = activeSprites.length - 1; i >= 0; i--) {
                const s = activeSprites[i]; s.age += 16;
                
                let op = 1;
                if(s.age < s.fadeIn) op = s.age / s.fadeIn;
                else if (s.age > s.life - s.fadeOut) op = (s.life - s.age) / s.fadeOut;
                s.mesh.material.opacity = op;

                s.mesh.position.add(s.velocity);
                const n = simplex.noise3D(s.mesh.position.x*0.001, s.mesh.position.y*0.001, time);
                s.mesh.position.y += n * 0.4;

                // Day mode invert
                if(appState.mode === 'day' && s.type === TYPE_NEUTRAL) s.mesh.material.color.setHex(0x222222);
                else s.mesh.material.color.setHex(0xffffff);

                if(s.age > s.life) {
                    scene.remove(s.mesh); s.mesh.material.map.dispose(); s.mesh.material.dispose(); activeSprites.splice(i, 1);
                }
            }

            // Connections
            let lineIdx = 0, colIdx = 0;
            const connectSq = CONFIG.connectDist * CONFIG.connectDist;
            const lineBaseOp = (appState.mode==='day') ? 0.6 : 0.3;

            for(let i=0; i<activeSprites.length; i++) {
                for(let k=1; k<8; k++) {
                    const j = (i + k*7) % activeSprites.length; 
                    if(i===j) continue;
                    const s1 = activeSprites[i]; const s2 = activeSprites[j];
                    if(s1.mesh.material.opacity < 0.2 || s2.mesh.material.opacity < 0.2) continue;

                    const d2 = s1.mesh.position.distanceToSquared(s2.mesh.position);
                    if(d2 < connectSq) {
                        let r=0.5, g=0.5, b=0.5, isStrong=false;
                        if(s1.type === s2.type && s1.type !== TYPE_NEUTRAL) {
                            isStrong = true; r = s1.col.r; g = s1.col.g; b = s1.col.b;
                        } else if(d2 < connectSq * 0.2) {
                            isStrong = true;
                            if(appState.mode==='day') { r=0.7; g=0.7; b=0.7; }
                            else { r=0.3; g=0.3; b=0.3; }
                        }

                        if(isStrong && lineIdx < CONFIG.maxLines * 6) {
                            const p1=s1.mesh.position; const p2=s2.mesh.position;
                            linePos[lineIdx++]=p1.x; linePos[lineIdx++]=p1.y; linePos[lineIdx++]=p1.z;
                            linePos[lineIdx++]=p2.x; linePos[lineIdx++]=p2.y; linePos[lineIdx++]=p2.z;
                            lineCols[colIdx++]=r; lineCols[colIdx++]=g; lineCols[colIdx++]=b;
                            lineCols[colIdx++]=r; lineCols[colIdx++]=g; lineCols[colIdx++]=b;
                        }
                    }
                }
            }
            lineGeo.setDrawRange(0, lineIdx/3);
            lineGeo.attributes.position.needsUpdate = true;
            lineGeo.attributes.color.needsUpdate = true;
            lineMat.opacity = lineBaseOp;

            camera.rotation.y = Math.sin(time * 0.05) * 0.05;
            renderer.render(scene, camera);
        }

        function spawnLoop() {
            spawnWord();
            const rate = wordBuffer.length > 10 ? CONFIG.spawnRate : 500;
            setTimeout(spawnLoop, rate);
        }

        document.getElementById('start-btn').addEventListener('click', async () => {
            document.getElementById('overlay').style.opacity = 0;
            setTimeout(() => document.getElementById('overlay').remove(), 1000);
            updateTime(); fetchEnv();
            await Audio.init();
            await fetchFeeds();
            spawnLoop();
            animate();
        });

        window.toggleLog = toggleLog;
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth/window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
