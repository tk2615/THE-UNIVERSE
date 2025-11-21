<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Omni-Source Galaxy</title>
    
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Shippori+Mincho:wght=800&family=Roboto+Mono:wght=500&display=swap" rel="stylesheet">
    
    <style>
        /* (前回のスタイルとほぼ同じ。微修正のみ) */
        body { margin: 0; overflow: hidden; background-color: #000; color: white; font-family: 'Shippori Mincho', serif; transition: background-color 2s ease; }
        #canvas-container { width: 100vw; height: 100vh; position: fixed; top: 0; left: 0; z-index: 1; }
        
        /* HUD (mix-blend-mode: difference で色反転に対応) */
        #hud {
            position: absolute; top: 30px; left: 30px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 11px;
            line-height: 1.6; pointer-events: none; mix-blend-mode: difference;
            color: rgba(255,255,255,0.9);
        }
        .hud-val { font-weight: 700; margin-left: 5px; }

        /* Source List Display (右下: 白背景時も見えるように色調整) */
        #source-list {
            position: absolute; bottom: 30px; right: 30px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 9px; color: rgba(255,255,255,0.6);
            text-align: right; pointer-events: none; line-height: 1.4;
            transition: color 1s ease;
        }

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
        
        /* Log (左下) */
        #sys-log {
            position: absolute; bottom: 30px; left: 30px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 10px; color: #00ffaa;
            pointer-events: none; opacity: 0.7; width: 300px; transition: color 1s ease;
        }
        
        /* Control Button */
        #controls { position: absolute; top: 30px; right: 30px; z-index: 30; }
        .btn { background: rgba(128,128,128,0.2); border: 1px solid rgba(128,128,128,0.5); color: #fff; padding: 8px 16px; cursor: pointer; border-radius: 20px; backdrop-filter: blur(5px); font-size: 10px; mix-blend-mode: difference; }
    </style>
</head>
<body>

    <div id="overlay">
        <div class="title">THE UNIVERSE</div>
        <div class="subtitle">Omni-Source Real-time Stream</div>
        <button id="start-btn">CONNECT</button>
    </div>

    <div id="hud">
        <div>TIME <span id="clock-display" class="hud-val">--:--:--</span></div>
        <div>LOC  <span id="loc-display" class="hud-val">Scanning...</span></div>
        <div>ENV  <span id="env-display" class="hud-val">--</span></div>
    </div>

    <div id="sys-log">Initializing...</div>
    
    <div id="source-list">ACTIVE SOURCES WILL APPEAR HERE</div>

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
            spawnRate: 100,      
            wordLife: 40000,     
            baseSize: 60,        
            minSize: 40,         
            connectDist: 500,    
            particleCount: 4000,
            fogDensity: 0.0004   
        };

        // --- EXPANDED SOURCE LIST ---
        const ALL_FEEDS = [
            { name: "NHK NEWS", url: "https://www3.nhk.or.jp/rss/news/cat0.xml" },
            { name: "YAHOO! TOP", url: "https://news.yahoo.co.jp/rss/topics/top-picks.xml" },
            { name: "YAHOO! SCI", url: "https://news.yahoo.co.jp/rss/topics/science.xml" },
            { name: "BUSINESS INSIDER", url: "https://www.businessinsider.jp/feed/index.xml" },
            { name: "BBC WORLD", url: "https://feeds.bbci.co.uk/news/world/rss.xml" },
            { name: "REUTERS", url: "https://www.reutersagency.com/feed/?best-topics=political-general&post_type=best" },
            { name: "CNN", url: "http://rss.cnn.com/rss/edition.rss" },
            { name: "AL JAZEERA", url: "https://www.aljazeera.com/xml/rss/all.xml" },
            { name: "WIRED", url: "https://www.wired.com/feed/rss" },
            { name: "GIZMODO JP", url: "https://www.gizmodo.jp/index.xml" },
            { name: "TECHCRUNCH", url: "https://techcrunch.com/feed/" },
            { name: "NASA", url: "https://www.nasa.gov/rss/dyn/breaking_news.rss" },
            { name: "SPACE.COM", url: "https://www.space.com/feeds/all" },
            { name: "NAT GEO", url: "https://www.nationalgeographic.com/ngm/index.rss" },
            { name: "ART NEWS", url: "https://www.artnews.com/feed/" },
            { name: "DEZEEN", url: "https://www.dezeen.com/feed/" },
            { name: "HYPEBEAST", url: "https://hypebeast.com/feed" },
            { name: "VOGUE", url: "https://www.vogue.com/feed/rss" },
            { name: "FORBES", url: "https://www.forbes.com/most-popular/feed/" },
            { name: "FINANCIAL TIMES", url: "https://www.ft.com/?format=rss" }
        ];

        const PROXY = "https://api.rss2json.com/v1/api.json?rss_url=";

        // Dictionaries & Classifiers
        const TYPE_NEUTRAL=0, TYPE_HOPE=1, TYPE_CRISIS=2, TYPE_TECH=3, TYPE_NATURE=4, TYPE_ART=5, TYPE_MONEY=6;
        const CLASSIFIER = {
            crisis: ["WAR","CRISIS","DEATH","KILL","ATTACK","DANGER","WARNING","BOMB","VICTIM","戦争","危機","災害","殺害","事故","警報","死亡","脅威","爆発","炎上"],
            hope:   ["PEACE","WIN","HOPE","SUCCESS","JOY","BEST","HEAL","LOVE","SAVE","VICTORY","平和","希望","支援","回復","優勝","成功","愛","救助","勝利","祝福","金メダル"],
            tech:   ["AI","SPACE","DATA","FUTURE","CYBER","QUANTUM","ROBOT","APP","CODE","APPLE","GOOGLE","宇宙","未来","技術","開発","人工知能","量子","ロケット","電脳"],
            nature: ["EARTH","CLIMATE","OCEAN","FOREST","ANIMAL","PLANET","STORM","HEAT","GREEN","地球","環境","海","森","気候","動物","台風","猛暑","温暖化","自然"],
            art:    ["ART","SOUL","COLOR","MIND","DREAM","POEM","MUSIC","FILM","IDEA","DESIGN","芸術","魂","夢","思考","色","哲学","映画","音楽","言葉","感性","作品"],
            money:  ["MARKET","STOCK","MONEY","DOLLAR","PRICE","TRADE","ECONOMY","BITCOIN","株","円","ドル","経済","市場","投資","価格","値上げ","金融"]
        };
        const STOP_WORDS = ["THE","AND","FOR","WITH","NEWS","LIVE","FROM","THAT","REPORT","NEW","VIDEO","SAYS","YOUR","あ","い","う","え","お","の","に","は","を","が","で","て","と","も","し","た","る","こと","ます","です","さん","など","いる","ある","する","なる","ため"];
        const BACKUP = [{t:"System",s:"BACKUP",y:TYPE_TECH},{t:"Loading",s:"BACKUP",y:TYPE_NEUTRAL},{t:"Cosmos",s:"BACKUP",y:TYPE_NATURE}];

        let wordBuffer = [];
        let appState = { 
            mode: 'night',
            tension: 0, harmony: 0.5, noise: 0
        };
        let isAudioReady = false;

        // --- UTILS ---
        function log(msg) {
            const el = document.getElementById('sys-log');
            // 白背景時の色変更に対応
            el.style.color = (appState.mode === 'day') ? '#333333' : '#00ffaa';
            el.innerText = "> " + msg;
        }
        function toggleLog() {
            const el = document.getElementById('sys-log');
            el.style.display = (el.style.display === 'none') ? 'block' : 'none';
        }
        function updateTime() {
            const now = new Date();
            document.getElementById('clock-display').innerText = now.toLocaleTimeString();
            const h = now.getHours();
            let newMode = 'night';
            if(h>=4 && h<6) newMode='early';
            else if(h>=6 && h<10) newMode='morning';
            else if(h>=10 && h<16) newMode='day'; // 昼
            else if(h>=16 && h<19) newMode='evening';
            appState.mode = newMode;
            
            // 右下のソースリストの色を背景に合わせて切り替え
            const sourceListEl = document.getElementById('source-list');
            sourceListEl.style.color = (appState.mode === 'day') ? 'rgba(0,0,0,0.4)' : 'rgba(255,255,255,0.6)';
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

        // --- DATA ENGINE (FEED FETCHING IS AS BEFORE) ---
        const segmenter = new Intl.Segmenter("ja-JP", { granularity: "word" });
        function decodeEntity(str) { const t=document.createElement("textarea"); t.innerHTML=str; return t.value; }

        async function fetchFeeds() {
            const shuffled = ALL_FEEDS.sort(() => 0.5 - Math.random());
            const selectedFeeds = shuffled.slice(0, 8);
            
            const sourceListEl = document.getElementById('source-list');
            sourceListEl.innerHTML = "CURRENT SOURCES:<br>" + selectedFeeds.map(f => f.name).join("<br>");

            log(`Scanning ${selectedFeeds.length} active sources...`);
            
            const promises = selectedFeeds.map(f => fetch(PROXY + encodeURIComponent(f.url)).then(r=>r.ok?r.json():null).catch(e=>null));
            const results = await Promise.all(promises);
            
            let count = 0;
            results.forEach((data, idx) => {
                if(data && data.items) {
                    data.items.forEach(item => {
                        const srcName = selectedFeeds[idx].name;
                        processTitle(item.title, srcName);
                        count++;
                    });
                }
            });
            
            if(count === 0) BACKUP.forEach(d => wordBuffer.push({text:d.t, source:d.s, type:d.y}));
            log(`Stream updated: ${count} articles.`);
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
            else if(CLASSIFIER.money.some(k=>up.includes(k))) type = TYPE_MONEY;

            let disp = text;
            if(!text.match(/[亜-熙ぁ-んァ-ヶ]/)) disp = text.charAt(0).toUpperCase() + text.slice(1).toLowerCase();
            wordBuffer.push({ text: disp, type: type, source: source });
        }
        
        setInterval(fetchFeeds, 45000);

        // --- AUDIO ENGINE (UNCHANGED) ---
        const Audio = {
            drone: null, tensionSynth: null, hopeSynth: null, noise: null, filter: null,
            async init() {
                await Tone.start();
                const master = new Tone.Reverb({decay: 8, wet: 0.5}).toDestination();
                const limiter = new Tone.Limiter(-2).connect(master);
                this.filter = new Tone.AutoFilter(0.1).connect(limiter).start();
                this.drone = new Tone.PolySynth(Tone.Synth, { oscillator: { type: "fatsine", count: 3, spread: 20 }, envelope: { attack: 2, decay: 1, sustain: 1, release: 3 } }).connect(this.filter);
                this.drone.volume.value = -12;
                const dist = new Tone.Distortion(0.4).connect(limiter);
                this.tensionSynth = new Tone.MembraneSynth().connect(dist);
                this.tensionSynth.volume.value = -99; 
                const delay = new Tone.PingPongDelay("8n", 0.3).connect(limiter);
                this.hopeSynth = new Tone.PolySynth(Tone.FMSynth).connect(delay);
                this.hopeSynth.volume.value = -99;
                this.noise = new Tone.Noise("pink").connect(new Tone.Filter(500, "lowpass").connect(limiter));
                this.noise.start(); this.noise.volume.value = -99;
                this.startLoops();
                isAudioReady = true;
            },
            startLoops() {
                const chords = [["C3","G3","C4"], ["A2","E3","A3"], ["F2","C3","F3"], ["G2","D3","G3"]];
                new Tone.Loop(t => {
                    const c = chords[Math.floor(Math.random()*chords.length)];
                    this.drone.triggerAttackRelease(c, "8m", t);
                }, "10m").start(0);
                Tone.Transport.start();
            },
            updateMix() {
                if(!isAudioReady) return;
                const tensionVol = -30 + (appState.tension * 20);
                if(appState.tension < 0.1) this.tensionSynth.volume.rampTo(-99, 1);
                else this.tensionSynth.volume.rampTo(tensionVol, 0.5);
                const harmonyVol = -30 + (appState.harmony * 15); 
                this.hopeSynth.volume.rampTo(harmonyVol, 1);
                const noiseVol = -60 + (appState.noise * 40); 
                this.noise.volume.rampTo(noiseVol, 1);
            },
            triggerNote(type) {
                if(!isAudioReady) return;
                const now = Tone.now();
                if(type === TYPE_CRISIS) this.tensionSynth.triggerAttackRelease("A1", "8n", now);
                else if(type === TYPE_HOPE || type === TYPE_ART) this.hopeSynth.triggerAttackRelease(["C5","D5","E5","G5"][Math.floor(Math.random()*4)], "16n", now);
                else if(type === TYPE_TECH) this.hopeSynth.triggerAttackRelease("C6", "32n", now, 0.5);
            }
        };

        // --- VISUAL ENGINE ---
        const scene = new THREE.Scene();
        const fogCol = new THREE.Color(0x000000);
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

        const pGeo = new THREE.BufferGeometry();
        const pPos = [];
        for(let i=0; i<CONFIG.particleCount; i++) pPos.push((Math.random()-0.5)*5000, (Math.random()-0.5)*5000, (Math.random()-0.5)*5000);
        pGeo.setAttribute('position', new THREE.Float32BufferAttribute(pPos, 3));
        const pMat = new THREE.PointsMaterial({ color: 0x888888, size: 3, transparent: true, opacity: 0.5 });
        bgGroup.add(new THREE.Points(pGeo, pMat));

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
                case TYPE_MONEY:  return { hex:"#4488ff", r:0.2, g:0.5, b:1 };
                // 昼モード（白背景）の時のニュートラルカラーは黒系にする
                default:          return (appState.mode === 'day') ? { hex:"#333333", r:0.2, g:0.2, b:0.2 } : { hex:"#aaaaaa", r:0.6, g:0.6, b:0.6 };
            }
        }

        function createTexture(text, source, type) {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            const wSize = 80; 
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
            
            // 1. メインの単語
            // ニュートラル以外は常にジャンルカラー（鮮やかな色）を使う
            let mainColor = (type === TYPE_NEUTRAL) ? col.hex : col.hex;

            ctx.shadowColor = (appState.mode === 'day') ? "rgba(0,0,0,0.1)" : col.hex; // 影の色を調整
            ctx.shadowBlur = (appState.mode === 'day') ? 10 : 40;
            ctx.fillStyle = mainColor;
            ctx.font = `bold ${wSize}px "Shippori Mincho", serif`;
            ctx.fillText(text, width/2, 10);
            
            // 2. ソースラベル
            ctx.shadowBlur = 0; 
            // 白背景時は暗いグレー、それ以外は明るいグレー
            ctx.fillStyle = (appState.mode === 'day') ? "#666666" : "#888888";
            ctx.font = `500 ${sSize}px "Roboto Mono", monospace`;
            ctx.fillText(source, width/2, wSize + 25);

            return { tex: new THREE.CanvasTexture(canvas), ratio: width/height, col: col };
        }

        function spawnWord() {
            if(wordBuffer.length === 0) return;
            const data = wordBuffer.shift();
            if(wordBuffer.length < 50) wordBuffer.push(data);

            if(data.type === TYPE_CRISIS) appState.tension = Math.min(1, appState.tension + 0.2);
            if(data.type === TYPE_HOPE || data.type === TYPE_ART) appState.harmony = Math.min(1, appState.harmony + 0.1);
            if(data.type === TYPE_TECH) appState.noise = Math.min(1, appState.noise + 0.1);
            
            Audio.triggerNote(data.type);

            // テクスチャを再生成することで、最新の appState.mode に基づいた色になる
            const { tex, ratio, col } = createTexture(data.text, data.source, data.type);
            
            // THREE.jsのスプライト自体は白くしておき、テクスチャの色をそのまま反映させる
            const mat = new THREE.SpriteMaterial({ map: tex, transparent: true, opacity: 0, blending: THREE.NormalBlending, depthWrite: false, color: 0xffffff });
            const sprite = new THREE.Sprite(mat);
            
            const range = 2500;
            sprite.position.set((Math.random()-0.5)*range, (Math.random()-0.5)*range*0.6, (Math.random()-0.5)*2000);
            
            const size = Math.max(CONFIG.minSize, CONFIG.baseSize + Math.random() * 60);
            sprite.scale.set(size * ratio, size, 1);
            
            scene.add(sprite);
            activeSprites.push({
                mesh: sprite, age: 0, life: CONFIG.wordLife + Math.random()*5000, fadeIn: 1500, fadeOut: 1500,
                velocity: new THREE.Vector3((Math.random()-0.5)*0.2, (Math.random()-0.5)*0.2, (Math.random()-0.5)*0.1),
                type: data.type, col: col
            });
        }

        const simplex = new SimplexNoise();
        let time = 0;

        function animate() {
            requestAnimationFrame(animate);
            time += 0.001;

            appState.tension *= 0.995; appState.harmony *= 0.995; appState.noise *= 0.995;
            Audio.updateMix();
            
            // 背景色とフォグの更新
            let bgHex;
            if (appState.mode === 'early') bgHex = 0x101030;
            else if (appState.mode === 'morning') bgHex = 0x88ccff;
            else if (appState.mode === 'day') bgHex = 0xf2f2f2; // 白背景
            else if (appState.mode === 'evening') bgHex = 0x330511;
            else bgHex = 0x020205;

            let bgCol = new THREE.Color(bgHex);
            scene.background = scene.background ? scene.background.lerp(bgCol, 0.01) : bgCol;
            fogCol.lerp(bgCol, 0.01); scene.fog.color.copy(fogCol);
            
            if(appState.tension > 0.8 && Math.random() > 0.95) scene.background.setHex(0x220000);

            bgGroup.rotation.y = time * 0.02;

            for(let i = activeSprites.length - 1; i >= 0; i--) {
                const s = activeSprites[i]; s.age += 16;
                let op = 1;
                if(s.age < s.fadeIn) op = s.age / s.fadeIn;
                else if (s.age > s.life - s.fadeOut) op = (s.life - s.age) / s.fadeOut;
                s.mesh.material.opacity = op;
                s.mesh.position.add(s.velocity);
                const n = simplex.noise3D(s.mesh.position.x*0.001, s.mesh.position.y*0.001, time);
                s.mesh.position.y += n * 0.5;
                if(s.age > s.life) { scene.remove(s.mesh); s.mesh.material.map.dispose(); s.mesh.material.dispose(); activeSprites.splice(i, 1); }
            }

            // Connections
            let lineIdx = 0, colIdx = 0;
            const connectSq = CONFIG.connectDist * CONFIG.connectDist;
            const lineBaseOp = (appState.mode==='day') ? 0.6 : 0.3; // 昼間は線の視認性も高める

            for(let i=0; i<activeSprites.length; i++) {
                for(let k=1; k<8; k++) {
                    const j = (i + k*7) % activeSprites.length; if(i===j) continue;
                    const s1 = activeSprites[i]; const s2 = activeSprites[j];
                    if(s1.mesh.material.opacity < 0.2 || s2.mesh.material.opacity < 0.2) continue;
                    const d2 = s1.mesh.position.distanceToSquared(s2.mesh.position);
                    if(d2 < connectSq) {
                        let r, g, b, isStrong=false;
                        
                        // 接続線の色も背景に合わせて調整
                        if(s1.type === s2.type && s1.type !== TYPE_NEUTRAL) {
                            isStrong = true; r = s1.col.r; g = s1.col.g; b = s1.col.b; // ジャンルカラー
                        } else if(d2 < connectSq * 0.2) {
                            isStrong = true;
                            // ニュートラルな接続の色
                            if(appState.mode==='day') { r=0.1; g=0.1; b=0.1; } // 昼は黒線
                            else { r=0.3; g=0.3; b=0.3; } // 夜はグレー線
                        } else {
                            // 遠い線は薄い色
                            if(appState.mode==='day') { r=0.5; g=0.5; b=0.5; }
                            else { r=0.1; g=0.1; b=0.1; }
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
