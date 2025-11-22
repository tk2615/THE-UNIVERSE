<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>THE UNIVERSE</title>
    
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Shippori+Mincho:wght@500&family=Roboto+Mono:wght@400&display=swap" rel="stylesheet">
    
    <style>
        /* --- 基底スタイル --- */
        body { 
            margin: 0; overflow: hidden; color: white; 
            font-family: 'Shippori Mincho', serif;
            --c1: #051020; --c2: #102035; --c3: #000510; --c4: #152540;
            background: linear-gradient(-45deg, var(--c1), var(--c2), var(--c3), var(--c4));
            background-size: 400% 400%;
            animation: gradient-shift 120s ease infinite;
            transition: color 3s ease;
        }
        @keyframes gradient-shift {
            0% { background-position: 0% 50%; }
            50% { background-position: 100% 50%; }
            100% { background-position: 0% 50%; }
        }
        #canvas-container { width: 100vw; height: 100vh; position: fixed; top: 0; left: 0; z-index: 1; }
        
        /* --- HUD (画面に馴染ませる: 背景なし・透過) --- */
        #hud {
            position: absolute; top: 40px; left: 40px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 11px;
            line-height: 1.8; 
            color: rgba(255, 255, 255, 0.5); /* 半透明で馴染ませる */
            letter-spacing: 2px;
            pointer-events: none;
            /* 背景色や枠線を削除 */
        }
        .hud-row { margin-bottom: 4px; }
        .hud-label { opacity: 0.5; margin-right: 10px; font-size: 9px; }
        .hud-val { color: rgba(255, 255, 255, 0.9); font-weight: 500; }

        /* --- Source List (右下: 極限まで目立たなく) --- */
        #source-list {
            position: absolute; bottom: 40px; right: 40px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 9px; 
            color: rgba(255, 255, 255, 0.3); /* かなり薄く */
            text-align: right; pointer-events: none; line-height: 1.6; 
            letter-spacing: 1px;
        }

        /* --- Overlay (以前のシンプルで美しいスタイルに復元) --- */
        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: #000510; z-index: 100; display: flex; flex-direction: column;
            align-items: center; justify-content: center; transition: opacity 3s ease;
        }
        .title { 
            font-size: 4vw; font-weight: 500; letter-spacing: 0.4em; margin-bottom: 20px; 
            color: #ffffff; text-shadow: 0 0 20px rgba(255,255,255,0.2);
            font-family: 'Shippori Mincho', serif;
        }
        .subtitle { 
            font-size: 11px; color: #667788; margin-bottom: 60px; 
            font-family: 'Roboto Mono'; letter-spacing: 6px; text-transform: uppercase;
        }
        #start-btn {
            background: transparent; border: 1px solid rgba(255,255,255,0.3); color: #cccccc;
            padding: 15px 60px; font-family: 'Shippori Mincho', serif; font-size: 14px; font-weight: 500;
            letter-spacing: 4px; cursor: pointer; transition: 0.6s; border-radius: 50px; /* 丸みのあるボタン */
        }
        #start-btn:hover { 
            background: rgba(255,255,255,0.1); 
            border-color: rgba(255,255,255,0.8); 
            color: white;
            box-shadow: 0 0 30px rgba(255,255,255,0.15); 
            transform: scale(1.02);
        }
        
        /* Log (左下: 控えめに) */
        #sys-log {
            position: absolute; bottom: 40px; left: 40px; z-index: 20;
            font-family: 'Shippori Mincho', serif; font-size: 11px; color: rgba(200, 220, 255, 0.5);
            pointer-events: none; width: 300px; 
        }
        
        #controls { position: absolute; top: 40px; right: 40px; z-index: 30; }
        .btn { 
            background: transparent; border: 1px solid rgba(255,255,255,0.1); 
            color: rgba(255,255,255,0.4); padding: 6px 12px; cursor: pointer; border-radius: 20px; 
            font-family: 'Roboto Mono'; font-size: 9px; letter-spacing: 1px; transition: 0.3s; 
        }
        .btn:hover { border-color: rgba(255,255,255,0.5); color: white; }
    </style>
</head>
<body>

    <div id="overlay">
        <div class="title">THE UNIVERSE</div>
        <div class="subtitle">Omni-Source Galaxy</div>
        <button id="start-btn">CONNECT</button>
    </div>

    <div id="hud">
        <div class="hud-row"><span class="hud-label">TIME</span> <span id="clock-display" class="hud-val">--:--</span></div>
        <div class="hud-row"><span class="hud-label">LOC</span> <span id="loc-display" class="hud-val">Scanning...</span></div>
        <div class="hud-row"><span class="hud-label">ENV</span> <span id="env-display" class="hud-val">--</span> <span id="temp-display" class="hud-val"></span></div>
    </div>

    <div id="sys-log">Initializing...</div>
    <div id="source-list">Waiting...</div>

    <div id="controls">
        <button class="btn" onclick="toggleLog()">LOG</button>
    </div>

    <div id="canvas-container"></div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/simplex-noise/2.4.0/simplex-noise.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.js"></script>

    <script>
        // --- CONFIGURATION ---
        const CONFIG = {
            spawnRate: 150,       // 漢字多めなので少し生成頻度を高めに
            wordLife: 65000,      
            baseSize: 65,         
            minSize: 40,          
            connectDist: 600,    
            particleCount: 3500, 
            fogDensity: 0.0003   
        };

        // --- SOURCES (日本語重視 + グローバル) ---
        const ALL_FEEDS = [
            // --- 日本語 (漢字を増やすため多めに配置) ---
            { name: "NHK", url: "https://www3.nhk.or.jp/rss/news/cat0.xml" },
            { name: "YAHOO", url: "https://news.yahoo.co.jp/rss/topics/top-picks.xml" },
            { name: "ASAHI", url: "https://www.asahi.com/rss/asahi/newsheadlines.rdf" },
            { name: "NIKKEI", url: "https://assets.wor.jp/rss/rdf/nikkei/news.rdf" },
            { name: "MAINICHI", url: "https://mainichi.jp/rss/etc/mainichi-flash.rss" },
            { name: "IT MEDIA", url: "https://rss.itmedia.co.jp/rss/2.0/itmedia_all.xml" },
            { name: "GIZMODO JP", url: "https://www.gizmodo.jp/index.xml" },
            { name: "WIRED JP", url: "https://wired.jp/rss/index.xml" },
            { name: "NAZOLOGY", url: "https://nazology.net/feed" },
            { name: "SORAE", url: "https://sorae.info/feed" },
            { name: "NAT GEO", url: "https://natgeo.nikkeibp.co.jp/rss/index.xml" },
            { name: "BIJUTSU", url: "https://bijutsutecho.com/rss" },
            { name: "VOGUE", url: "https://www.vogue.co.jp/rss/vogue" },
            { name: "CINRA", url: "https://www.cinra.net/feed" },
            { name: "PEN", url: "https://www.pen-online.jp/feed" },
            { name: "NOTE", url: "https://note.com/rss" },
            { name: "QIITA", url: "https://qiita.com/popular-items/feed" },

            // --- 英語 (アクセントとして混入) ---
            { name: "BBC", url: "https://feeds.bbci.co.uk/news/world/rss.xml" },
            { name: "CNN", url: "http://rss.cnn.com/rss/edition.rss" },
            { name: "NASA", url: "https://www.nasa.gov/rss/dyn/breaking_news.rss" },
            { name: "REUTERS", url: "https://www.reutersagency.com/feed/?best-topics=political-general&post_type=best" },
            { name: "ART NEWS", url: "https://www.artnews.com/feed/" }
        ];

        const PROXY = "https://api.rss2json.com/v1/api.json?rss_url=";

        // --- CLASSIFIERS ---
        const TYPE_NEUTRAL=0, TYPE_HOPE=1, TYPE_CRISIS=2, TYPE_TECH=3, TYPE_NATURE=4, TYPE_ART=5, TYPE_MONEY=6;
        
        const CLASSIFIER = {
            crisis: ["WAR","CRISIS","DEATH","KILL","ATTACK","DANGER","WARNING","VICTIM","BOMB","BLAST","戦争","危機","災害","殺害","事故","警報","死亡","脅威","爆発","炎上","地震","津波","被害","倒産","容疑","逮捕","衝突","ミサイル","闇","不倫","死去","罪"],
            hope:  ["PEACE","WIN","HOPE","SUCCESS","JOY","BEST","HEAL","LOVE","SAVE","VICTORY","AWARD","平和","希望","支援","回復","優勝","成功","愛","救助","勝利","祝福","金メダル","解決","誕生","新発見","ノーベル","結婚","推し","祝","最高","祈り","光"],
            tech:  ["AI","SPACE","DATA","FUTURE","CYBER","QUANTUM","ROBOT","APP","CODE","APPLE","GOOGLE","LAUNCH","宇宙","未来","技術","開発","人工知能","量子","ロケット","電脳","スマホ","アプリ","生成","半導体","コード","実験","科学","実装","発明"],
            nature: ["EARTH","CLIMATE","OCEAN","FOREST","ANIMAL","PLANET","STORM","HEAT","GREEN","WILD","地球","環境","海","森","気候","動物","台風","猛暑","温暖化","自然","生物","発見","星","細胞","猫","犬","鳥","雪","雨","月","花","風"],
            art:   ["ART","SOUL","COLOR","MIND","DREAM","POEM","MUSIC","FILM","IDEA","DESIGN","STYLE","芸術","魂","夢","思考","色","哲学","映画","音楽","言葉","感性","作品","展示","デザイン","建築","美","文学","文化","絵画","詩"],
            money: ["MARKET","STOCK","MONEY","DOLLAR","PRICE","TRADE","ECONOMY","BITCOIN","FINANCE","株","円","ドル","経済","市場","投資","価格","値上げ","金融","売上","利益","買収","上場","税","安","高","金"]
        };
        
        const STOP_WORDS = [
            "THE","AND","FOR","WITH","NEWS","LIVE","FROM","THAT","REPORT","NEW","VIDEO","SAYS","YOUR","UPDATE","TODAY","THIS","WHAT","WHICH","WHEN","WHERE","WHO","HOW","WHY","ARE","WAS","WERE","BEEN","HAVE","HAS","HAD","DOES","DID","CAN","COULD","WOULD","SHOULD","WILL","ABOUT","AFTER","BEFORE","UNDER","OVER","INTO","ONTO","JUST","ONLY","MORE","MOST","SOME","ANY","OTHER","THAN","THEN","NOW","HERE","THERE","OUT","OFF","DOWN","UP","LIKE","INTO","TIME","YEAR","OVER",
            "あ","い","う","え","お","の","に","は","を","が","で","て","と","も","し","た","る","こと","ます","です","さん","など","よう","その","この","あの","これ","それ","あれ","から","ので","もの","ない","って","いう","速報","情報","一覧","更新","記事","詳細","画像","写真","公開","開催","決定","発表","開始","終了","確認","問題","理由","方法","解説","結果","内容","対応","検討","速報"
        ];

        const BACKUP = [{t:"静寂",s:"...",y:TYPE_NATURE},{t:"待機",s:"...",y:TYPE_NEUTRAL},{t:"深呼吸",s:"...",y:TYPE_ART}];

        let wordBuffer = [];
        let appState = { 
            mode: 'night',
            tension: 0, harmony: 0.5, noise: 0,
            targetFogCol: new THREE.Color(0x051020),
            currentBgCols: [],
            targetPalette: null
        };
        let isAudioReady = false;

        // --- AUDIO ENGINE (INFINITE VARIATIONS) ---
        const SCALE = ["C4", "D4", "E4", "G4", "A4", "C5", "D5", "E5"];
        const BASS_NOTES = ["C2", "G2", "F2", "A1", "E2"];

        const Audio = {
            drone: null, harp: null, chime: null, water: null, ground: null, bird: null, wood: null, choir: null, reverb: null,
            
            async init() {
                await Tone.start();
                Tone.Context.lookAhead = 0.1;
                this.reverb = new Tone.Reverb({decay: 12, wet: 0.5}).toDestination();
                this.reverb.generate(); 
                const masterComp = new Tone.Compressor(-30, 3).connect(this.reverb);

                const filterLFO = new Tone.LFO(0.05, 200, 1000).start(); 
                const autoFilter = new Tone.Filter(400, "lowpass").connect(masterComp);
                filterLFO.connect(autoFilter.frequency);

                this.drone = new Tone.PolySynth(Tone.FMSynth, {
                    harmonicity: 0.5, modulationIndex: 1, oscillator: { type: "sine" }, modulation: { type: "triangle" },
                    envelope: { attack: 4, decay: 4, sustain: 1, release: 8 },
                }).connect(autoFilter);
                this.drone.volume.value = -22; 
                this.startDroneLoop();

                this.harp = new Tone.PolySynth(Tone.Synth, { oscillator: { type: "triangle" }, envelope: { attack: 0.01, decay: 0.4, sustain: 0, release: 1 } }).connect(masterComp);
                this.harp.volume.value = -14;

                const panner = new Tone.Panner3D().connect(masterComp);
                this.chime = new Tone.PolySynth(Tone.FMSynth, { harmonicity: 3, modulationIndex: 10, oscillator: { type: "sine" }, envelope: { attack: 0.01, decay: 2, sustain: 0, release: 2 }, modulation: { type: "square" }, modulationEnvelope: { attack: 0.01, decay: 0.5, sustain: 0, release: 0.5 } }).connect(panner);
                this.chime.volume.value = -16;

                this.water = new Tone.PolySynth(Tone.Synth, { oscillator: { type: "sine" }, envelope: { attack: 0.005, decay: 0.3, sustain: 0, release: 0.1 } }).connect(new Tone.PingPongDelay("8n", 0.3).connect(masterComp));
                this.water.volume.value = -14;

                this.ground = new Tone.MembraneSynth({ pitchDecay: 0.05, octaves: 2, oscillator: { type: "sine" }, envelope: { attack: 0.01, decay: 0.8, sustain: 0.01, release: 1.4 } }).connect(masterComp);
                this.ground.volume.value = -10;

                this.choir = new Tone.PolySynth(Tone.AMSynth, { harmonicity: 1.5, oscillator: { type: "sine" }, envelope: { attack: 0.5, decay: 1, sustain: 0.5, release: 3 }, modulation: { type: "sine" } }).connect(masterComp);
                this.choir.volume.value = -14;

                this.bird = new Tone.PolySynth(Tone.Synth, { oscillator: { type: "triangle" }, envelope: { attack: 0.02, decay: 0.2, sustain: 0.1, release: 1 }, }).connect(new Tone.FeedbackDelay("8n", 0.4).connect(masterComp));
                this.bird.volume.value = -18;
                this.wood = new Tone.PolySynth(Tone.MembraneSynth, { pitchDecay: 0.01, octaves: 4, oscillator: { type: "sine" }, envelope: { attack: 0.001, decay: 0.1, sustain: 0, release: 0.1 } }).connect(masterComp);
                this.wood.volume.value = -16;
                isAudioReady = true;
            },

            startDroneLoop() {
                new Tone.Loop(time => {
                    const note = BASS_NOTES[Math.floor(Math.random() * BASS_NOTES.length)];
                    this.drone.triggerAttackRelease(note, 16, time);
                    if(Math.random() > 0.6) {
                         const harmony = Tone.Frequency(note).transpose(7); 
                         this.drone.triggerAttackRelease(harmony, 12, time + 2);
                    }
                }, 15).start(0);
                Tone.Transport.start();
            },

            triggerEvent(type) {
                if(!isAudioReady) return;
                const now = Tone.now();
                const t = now + Math.random() * 0.1;
                const noteIdx = Math.floor(Math.random() * SCALE.length);
                const note = SCALE[noteIdx];

                switch(type) {
                    case TYPE_MONEY: 
                        this.chime.set({ harmonicity: 1 + Math.random() * 5 });
                        if(Math.random() > 0.7) { this.chime.triggerAttackRelease(note, "32n", t); this.chime.triggerAttackRelease(note, "32n", t + 0.05); } 
                        else { this.chime.triggerAttackRelease(Tone.Frequency(note).transpose(12), "2n", t); }
                        break;
                    case TYPE_TECH: 
                        this.water.set({ envelope: { decay: 0.1 + Math.random() * 0.4 } });
                        if(Math.random() > 0.8) { this.water.triggerAttackRelease(note, "64n", t); this.water.triggerAttackRelease(Tone.Frequency(note).transpose(12), "64n", t + 0.05); } 
                        else { this.water.triggerAttackRelease(note, "16n", t); }
                        break;
                    case TYPE_CRISIS: 
                        if(Math.random() > 0.5) this.ground.triggerAttackRelease("C1", "1n", t); else this.ground.triggerAttackRelease("C2", "4n", t);
                        break;
                    case TYPE_HOPE:
                    case TYPE_ART: 
                        this.choir.set({ harmonicity: 1 + Math.random() });
                        if(Math.random() > 0.6) { this.choir.triggerAttackRelease([SCALE[noteIdx % 5], SCALE[(noteIdx+2)%5], SCALE[(noteIdx+4)%5]], "1n", t); } 
                        else { this.choir.triggerAttackRelease(note, "2n", t); }
                        break;
                    case TYPE_NATURE: 
                        if(Math.random() > 0.5) this.bird.triggerAttackRelease(Tone.Frequency(note).transpose(24), "32n", t); else this.wood.triggerAttackRelease(note, "16n", t);
                        break;
                    default: 
                        this.harp.set({ envelope: { release: 0.2 + Math.random() * 2 } });
                        if(Math.random() > 0.8) { this.harp.triggerAttackRelease(note, "8n", t); this.harp.triggerAttackRelease(Tone.Frequency(note).transpose(4), "8n", t + 0.05); } 
                        else { this.harp.triggerAttackRelease(note, "4n", t); }
                        break;
                }
            }
        };

        // --- WEATHER & UTILS ---
        const WEATHER_CODES = {
            0: { text: "Clear", icon: "☀️" },
            1: { text: "Clear", icon: "🌤" },
            2: { text: "Cloudy", icon: "⛅️" },
            3: { text: "Overcast", icon: "☁️" },
            45: { text: "Fog", icon: "🌫" },
            48: { text: "Fog", icon: "🌫" },
            51: { text: "Drizzle", icon: "🌦" },
            53: { text: "Drizzle", icon: "🌦" },
            55: { text: "Drizzle", icon: "🌦" },
            61: { text: "Rain", icon: "☔️" },
            63: { text: "Rain", icon: "☔️" },
            65: { text: "Heavy Rain", icon: "⛈" },
            71: { text: "Snow", icon: "❄️" },
            73: { text: "Snow", icon: "❄️" },
            75: { text: "Heavy Snow", icon: "☃️" },
            95: { text: "Thunder", icon: "⚡️" },
        };

        async function fetchEnv() {
            try {
                const ip = await fetch('https://ipapi.co/json/').then(r=>r.json());
                const city = ip.city ? ip.city.toUpperCase() : 'LOC';
                document.getElementById('loc-display').innerText = city;
                
                if(ip.latitude){
                    const w = await fetch(`https://api.open-meteo.com/v1/forecast?latitude=${ip.latitude}&longitude=${ip.longitude}&current_weather=true`).then(r=>r.json());
                    const code = w.current_weather.weathercode;
                    const temp = w.current_weather.temperature;
                    
                    const wInfo = WEATHER_CODES[code] || { text: "--", icon: "" };
                    document.getElementById('env-display').innerText = `${wInfo.icon} ${wInfo.text}`;
                    document.getElementById('temp-display').innerText = `${temp}°C`;
                }
            } catch(e){
                document.getElementById('loc-display').innerText = "OFFLINE";
            }
        }

        function updateTime() {
            const now = new Date();
            const timeStr = now.toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'});
            document.getElementById('clock-display').innerText = timeStr;
        }
        setInterval(updateTime, 1000);

        // --- VISUALS ---
        const COLOR_PALETTES = {
            night: { hexes: ['#051020', '#102035', '#000510', '#152540'], fog: 0x051020, isLight: false },
            early: { hexes: ['#152030', '#2a3a50', '#101825', '#304055'], fog: 0x152030, isLight: false },
            morning: { hexes: ['#e0e5ec', '#f0f4f8', '#d0dbe5', '#eef2f5'], fog: 0xe0e5ec, isLight: true },
            day: { hexes: ['#f5f7fa', '#ffffff', '#edf1f5', '#f0f8ff'], fog: 0xf5f7fa, isLight: true },
            evening: { hexes: ['#2a1a20', '#402530', '#1a1015', '#50303a'], fog: 0x2a1a20, isLight: false }
        };

        function getGenreColor(type) {
            // 透過背景でも見えるように少し明るめに
            switch(type) {
                case TYPE_CRISIS: return { hex:"#ff5555", r:1, g:0.3, b:0.3 }; 
                case TYPE_HOPE:   return { hex:"#ffdd44", r:1, g:0.85, b:0.2 }; 
                case TYPE_TECH:   return { hex:"#00ffff", r:0, g:1, b:1 }; 
                case TYPE_NATURE: return { hex:"#66ff99", r:0.4, g:1, b:0.6 }; 
                case TYPE_ART:    return { hex:"#ee88ff", r:0.9, g:0.5, b:1 }; 
                case TYPE_MONEY:  return { hex:"#77bbff", r:0.4, g:0.7, b:1 };  
                default:          return { hex:"#dddddd", r:0.8, g:0.8, b:0.8 };
            }
        }

        function log(msg) { document.getElementById('sys-log').innerText = msg; }
        function toggleLog() {
            const el = document.getElementById('sys-log');
            el.style.display = (el.style.display === 'none') ? 'block' : 'none';
        }

        // --- THREE.JS ENGINE ---
        const scene = new THREE.Scene();
        const fogCol = new THREE.Color(0x051020);
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
        const pMat = new THREE.PointsMaterial({ color: 0x8899aa, size: 3, transparent: true, opacity: 0.3 }); 
        bgGroup.add(new THREE.Points(pGeo, pMat));

        const MAX_LINES = 1000;
        const lineGeo = new THREE.BufferGeometry();
        const linePos = new Float32Array(MAX_LINES * 6);
        const lineCols = new Float32Array(MAX_LINES * 6);
        lineGeo.setAttribute('position', new THREE.BufferAttribute(linePos, 3));
        lineGeo.setAttribute('color', new THREE.BufferAttribute(lineCols, 3));
        const lineMat = new THREE.LineBasicMaterial({ vertexColors: true, transparent: true, opacity: 0.15, blending: THREE.AdditiveBlending }); 
        const lines = new THREE.LineSegments(lineGeo, lineMat);
        lineGroup.add(lines);

        const activeSprites = [];
        const segmenter = new Intl.Segmenter("ja-JP", { granularity: "word" });

        function createTexture(text, source, type) {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            const wSize = 80; const sSize = 24;
            
            ctx.font = `600 ${wSize}px "Shippori Mincho", serif`; 
            const tWidth = ctx.measureText(text).width;
            ctx.font = `500 ${sSize}px "Roboto Mono", monospace`;
            const sWidth = ctx.measureText(source).width;
            const width = Math.max(tWidth, sWidth) + 80;
            const height = wSize + sSize + 80;
            
            canvas.width = width; canvas.height = height;
            ctx.textAlign = "center"; ctx.textBaseline = "top";
            
            const col = getGenreColor(type);
            
            // シンプルなグロー
            ctx.shadowColor = col.hex; 
            ctx.shadowBlur = 20; 
            ctx.fillStyle = col.hex;
            
            ctx.font = `600 ${wSize}px "Shippori Mincho", serif`;
            ctx.fillText(text, width/2, 20);
            
            ctx.shadowBlur = 0; 
            ctx.fillStyle = "rgba(255,255,255,0.5)";
            ctx.font = `500 ${sSize}px "Roboto Mono", monospace`;
            ctx.fillText(source, width/2, wSize + 35);

            return { tex: new THREE.CanvasTexture(canvas), ratio: width/height, col: col };
        }

        async function fetchFeeds() {
            const shuffled = ALL_FEEDS.sort(() => 0.5 - Math.random());
            const selectedFeeds = shuffled.slice(0, 8);
            const listEl = document.getElementById('source-list');
            // よりミニマルな表示
            listEl.innerHTML = selectedFeeds.map(f => f.name).join(" / ");

            log(`Scanning...`);
            const promises = selectedFeeds.map(f => fetch(PROXY + encodeURIComponent(f.url)).then(r=>r.ok?r.json():null).catch(e=>null));
            const results = await Promise.all(promises);
            
            let count = 0;
            results.forEach((data, idx) => {
                if(data && data.items) {
                    data.items.forEach(item => {
                        processTitle(item.title, selectedFeeds[idx].name);
                        count++;
                    });
                }
            });
            if(count === 0) BACKUP.forEach(d => wordBuffer.push({text:d.t, source:d.s, type:d.y}));
            log(`Received ${count}`);
        }

        function processTitle(title, source) {
            const tempDiv = document.createElement("div");
            tempDiv.innerHTML = title;
            const cleanTitle = tempDiv.textContent || tempDiv.innerText || "";

            if(cleanTitle.match(/[亜-熙ぁ-んァ-ヶ]/)) {
                for(const s of segmenter.segment(cleanTitle)) {
                    const word = s.segment;
                    if(!STOP_WORDS.includes(word)) {
                        const isKanji = /[\u4e00-\u9faf]/.test(word);
                        const isKatakana = /^[\u30a0-\u30ff]+$/.test(word);
                        const isHiraganaOnly = /^[\u3040-\u309f]+$/.test(word);
                        // 漢字1文字以上、またはカタカナ2文字以上
                        if ( (isKanji) || (isKatakana && word.length > 1) ) {
                            if(!isHiraganaOnly) addWord(word, source);
                        }
                    }
                }
            } else {
                cleanTitle.split(/\s+/).forEach(w => {
                    const c = w.replace(/[^a-zA-Z0-9]/g,'');
                    if(c.length > 2 && !STOP_WORDS.includes(c.toUpperCase())) {
                        if(!/^\d+$/.test(c)) addWord(c, source);
                    }
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

        function spawnWord() {
            if(wordBuffer.length === 0) return;
            const data = wordBuffer.shift();
            if(wordBuffer.length < 50) wordBuffer.push(data);

            Audio.triggerEvent(data.type);

            const { tex, ratio, col } = createTexture(data.text, data.source, data.type);
            const mat = new THREE.SpriteMaterial({ map: tex, transparent: true, opacity: 0, blending: THREE.AdditiveBlending, depthWrite: false, color: 0xffffff });
            const sprite = new THREE.Sprite(mat);
            
            const range = 2200;
            sprite.position.set((Math.random()-0.5)*range, (Math.random()-0.5)*range*0.7, (Math.random()-0.5)*2000);
            const size = Math.max(CONFIG.minSize, CONFIG.baseSize + Math.random() * 50);
            sprite.scale.set(size * ratio, size, 1);
            
            scene.add(sprite);
            activeSprites.push({
                mesh: sprite, age: 0, life: CONFIG.wordLife + Math.random()*10000, fadeIn: 3000, fadeOut: 3000,
                velocity: new THREE.Vector3((Math.random()-0.5)*0.05, (Math.random()-0.5)*0.05, (Math.random()-0.5)*0.02), 
                type: data.type, col: col
            });
        }

        const simplex = new SimplexNoise();
        let time = 0;

        function animate() {
            requestAnimationFrame(animate);
            time += 0.0005; 

            // Slow color drift
            const lerpFactor = 0.005; 
            if(appState.targetPalette && appState.currentBgCols) {
                for(let i = 0; i < appState.currentBgCols.length; i++) {
                    appState.currentBgCols[i].lerp(appState.targetPalette[i], lerpFactor);
                    document.body.style.setProperty(`--c${i+1}`, `#${appState.currentBgCols[i].getHexString()}`);
                }
                fogCol.lerp(appState.targetFogCol, lerpFactor);
                scene.fog.color.copy(fogCol);
            }
            scene.background = fogCol.clone();
            bgGroup.rotation.y = time * 0.01;

            for(let i = activeSprites.length - 1; i >= 0; i--) {
                const s = activeSprites[i]; s.age += 16;
                let op = 1;
                if(s.age < s.fadeIn) op = s.age / s.fadeIn;
                else if (s.age > s.life - s.fadeOut) op = (s.life - s.age) / s.fadeOut;
                s.mesh.material.opacity = op * (0.8 + Math.sin(time * 10 + i)*0.2); 
                s.mesh.position.add(s.velocity);
                s.mesh.position.y += simplex.noise3D(s.mesh.position.x*0.0005, s.mesh.position.y*0.0005, time) * 0.2;
                if(s.age > s.life) { scene.remove(s.mesh); s.mesh.material.map.dispose(); s.mesh.material.dispose(); activeSprites.splice(i, 1); }
            }

            let lineIdx = 0, colIdx = 0;
            const connectSq = CONFIG.connectDist * CONFIG.connectDist;
            const MAX_CONNECTIONS = Math.min(activeSprites.length * 3, MAX_LINES);

            for(let i=0; i<activeSprites.length; i++) {
                for(let k=1; k<6; k++) {
                    const j = (i + k*5) % activeSprites.length; if(i===j) continue;
                    const s1 = activeSprites[i]; const s2 = activeSprites[j];
                    if(s1.mesh.material.opacity < 0.1 || s2.mesh.material.opacity < 0.1) continue;
                    const d2 = s1.mesh.position.distanceToSquared(s2.mesh.position);
                    if(d2 < connectSq) {
                        if(lineIdx < MAX_CONNECTIONS * 6) {
                            const p1=s1.mesh.position; const p2=s2.mesh.position;
                            linePos[lineIdx++]=p1.x; linePos[lineIdx++]=p1.y; linePos[lineIdx++]=p1.z;
                            linePos[lineIdx++]=p2.x; linePos[lineIdx++]=p2.y; linePos[lineIdx++]=p2.z;
                            
                            const r = (s1.col.r + s2.col.r)*0.5;
                            const g = (s1.col.g + s2.col.g)*0.5;
                            const b = (s1.col.b + s2.col.b)*0.5;
                            
                            lineCols[colIdx++]=r; lineCols[colIdx++]=g; lineCols[colIdx++]=b;
                            lineCols[colIdx++]=r; lineCols[colIdx++]=g; lineCols[colIdx++]=b;
                        }
                    }
                }
            }
            lineGeo.setDrawRange(0, lineIdx/3);
            lineGeo.attributes.position.needsUpdate = true;
            lineGeo.attributes.color.needsUpdate = true;

            camera.rotation.y = Math.sin(time * 0.02) * 0.02;
            renderer.render(scene, camera);
        }

        function spawnLoop() {
            spawnWord();
            setTimeout(spawnLoop, CONFIG.spawnRate + Math.random() * 500);
        }

        document.getElementById('start-btn').addEventListener('click', async () => {
            document.getElementById('overlay').style.opacity = 0;
            setTimeout(() => document.getElementById('overlay').remove(), 3000);
            updateTime(); 
            setInitialCurrentBgCols(appState.mode); 
            setTargetPalette(appState.mode); 
            fetchEnv();
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
