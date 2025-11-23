<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Omni-Source: RHYTHMIC AMBIENT</title>
    
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Shippori+Mincho:wght@500&family=Roboto+Mono:wght@400&display=swap" rel="stylesheet">
    
    <style>
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
        
        #hud {
            position: absolute; top: 30px; left: 30px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 10px;
            line-height: 1.8; pointer-events: none; mix-blend-mode: normal;
            color: rgba(255,255,255,0.8); letter-spacing: 1px;
            transition: color 1s;
        }
        .hud-val { font-weight: 400; margin-left: 10px; }

        #source-list {
            position: absolute; bottom: 30px; right: 30px; z-index: 20;
            font-family: 'Roboto Mono', monospace; font-size: 9px; color: rgba(255,255,255,0.4);
            text-align: right; pointer-events: none; line-height: 1.6; letter-spacing: 1px;
            transition: color 2s ease;
        }

        #overlay {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: #000510; z-index: 100; display: flex; flex-direction: column;
            align-items: center; justify-content: center; transition: opacity 3s ease;
        }
        .title { font-size: 5vw; font-weight: 500; letter-spacing: 0.3em; margin-bottom: 15px; color: #e0f0ff; text-shadow: 0 0 20px rgba(200,230,255,0.3); }
        .subtitle { font-size: 11px; color: #8899aa; margin-bottom: 50px; font-family: 'Roboto Mono'; letter-spacing: 4px; }
        #start-btn {
            background: transparent; border: 1px solid rgba(255,255,255,0.3); color: #e0e0e0;
            padding: 15px 50px; font-family: 'Shippori Mincho', serif; font-size: 14px; font-weight: 500;
            letter-spacing: 4px; cursor: pointer; transition: 0.8s; border-radius: 50px;
        }
        #start-btn:hover { background: rgba(255,255,255,0.1); border-color: rgba(255,255,255,0.8); box-shadow: 0 0 30px rgba(255,255,255,0.2); transform: scale(1.05); color: white; }
        
        #sys-log {
            position: absolute; bottom: 30px; left: 30px; z-index: 20;
            font-family: 'Shippori Mincho', serif; font-size: 11px; color: #a0c0ff;
            pointer-events: none; opacity: 0.6; width: 300px; transition: color 2s ease;
        }
        
        #controls { position: absolute; top: 30px; right: 30px; z-index: 30; }
        .btn { background: rgba(255,255,255,0.05); border: none; color: #aaa; padding: 8px 16px; cursor: pointer; border-radius: 20px; backdrop-filter: blur(5px); font-size: 10px; letter-spacing: 1px; transition: 0.3s; }
        .btn:hover { color: white; background: rgba(255,255,255,0.1); }
    </style>
</head>
<body>

    <div id="overlay">
        <div class="title">THE UNIVERSE</div>
        <div class="subtitle">Rhythmic Ambient Stream</div>
        <button id="start-btn">CONNECT</button>
    </div>

    <div id="hud">
        <div>TIME <span id="clock-display" class="hud-val">--:--</span></div>
        <div>LOC <span id="loc-display" class="hud-val">Scanning...</span></div>
        <div>ENV <span id="env-display" class="hud-val">--</span></div>
    </div>

    <div id="sys-log">Initializing System...</div>
    <div id="source-list">Waiting for connection...</div>

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
            spawnRate: 200,      // 少し間隔を広げてリズムを聞かせる
            wordLife: 60000,     
            baseSize: 60,        
            minSize: 35,         
            connectDist: 600,    
            particleCount: 3000, 
            fogDensity: 0.0003   
        };

        // --- MIXED SOURCES (JP & EN) ---
        const ALL_FEEDS = [
            // English Global
            { name: "BBC WORLD", url: "https://feeds.bbci.co.uk/news/world/rss.xml" },
            { name: "CNN TOP", url: "http://rss.cnn.com/rss/edition.rss" },
            { name: "REUTERS", url: "https://www.reutersagency.com/feed/?best-topics=political-general&post_type=best" },
            { name: "NASA", url: "https://www.nasa.gov/rss/dyn/breaking_news.rss" },
            { name: "TECHCRUNCH", url: "https://techcrunch.com/feed/" },
            { name: "WIRED US", url: "https://www.wired.com/feed/rss" },
            { name: "ART NEWS", url: "https://www.artnews.com/feed/" },
            
            // Japanese
            { name: "NHK NEWS", url: "https://www3.nhk.or.jp/rss/news/cat0.xml" },
            { name: "YAHOO! TOP", url: "https://news.yahoo.co.jp/rss/topics/top-picks.xml" },
            { name: "ASAHI", url: "https://www.asahi.com/rss/asahi/newsheadlines.rdf" },
            { name: "GIZMODO JP", url: "https://www.gizmodo.jp/index.xml" },
            { name: "VOGUE JP", url: "https://www.vogue.co.jp/rss/vogue" }
        ];

        const PROXY = "https://api.rss2json.com/v1/api.json?rss_url=";

        // --- CLASSIFIERS ---
        const TYPE_NEUTRAL=0, TYPE_HOPE=1, TYPE_CRISIS=2, TYPE_TECH=3, TYPE_NATURE=4, TYPE_ART=5, TYPE_MONEY=6;
        
        const CLASSIFIER = {
            crisis: ["WAR","CRISIS","DEATH","KILL","ATTACK","DANGER","WARNING","VICTIM","BOMB","BLAST","CRASH","DISASTER","戦争","危機","災害","殺害","事故","警報","死亡","脅威","爆発","炎上","地震","津波","被害","倒産","容疑","逮捕","衝突","ミサイル","闇","不倫","死去"],
            hope:  ["PEACE","WIN","HOPE","SUCCESS","JOY","BEST","HEAL","LOVE","SAVE","VICTORY","AWARD","PRIZE","PEACE","平和","希望","支援","回復","優勝","成功","愛","救助","勝利","祝福","金メダル","解決","誕生","新発見","ノーベル","結婚","推し","祝","最高","祈り"],
            tech:  ["AI","SPACE","DATA","FUTURE","CYBER","QUANTUM","ROBOT","APP","CODE","APPLE","GOOGLE","MICROSOFT","META","LAUNCH","ROCKET","宇宙","未来","技術","開発","人工知能","量子","ロケット","電脳","スマホ","アプリ","生成","半導体","コード","実験","科学"],
            nature: ["EARTH","CLIMATE","OCEAN","FOREST","ANIMAL","PLANET","STORM","HEAT","GREEN","WILD","NATURE","ENVIRONMENT","地球","環境","海","森","気候","動物","台風","猛暑","温暖化","自然","生物","発見","星","細胞","猫","犬","鳥","雪","雨","月","花"],
            art:   ["ART","SOUL","COLOR","MIND","DREAM","POEM","MUSIC","FILM","IDEA","DESIGN","STYLE","FASHION","CULTURE","MOVIE","芸術","魂","夢","思考","色","哲学","映画","音楽","言葉","感性","作品","展示","デザイン","建築","美","文学","文化","絵画"],
            money: ["MARKET","STOCK","MONEY","DOLLAR","PRICE","TRADE","ECONOMY","BITCOIN","FINANCE","BUSINESS","BANK","株","円","ドル","経済","市場","投資","価格","値上げ","金融","売上","利益","買収","上場","税","安","高"]
        };
        
        const STOP_WORDS = [
            "THE","AND","FOR","WITH","NEWS","LIVE","FROM","THAT","REPORT","NEW","VIDEO","SAYS","YOUR","UPDATE","TODAY","THIS","WHAT","WHICH","WHEN","WHERE","WHO","HOW","WHY","ARE","WAS","WERE","BEEN","HAVE","HAS","HAD","DOES","DID","CAN","COULD","WOULD","SHOULD","WILL","ABOUT","AFTER","BEFORE","UNDER","OVER","INTO","ONTO","JUST","ONLY","MORE","MOST","SOME","ANY","OTHER","THAN","THEN","NOW","HERE","THERE","OUT","OFF","DOWN","UP","LIKE","INTO","TIME","YEAR","OVER",
            "あ","い","う","え","お","の","に","は","を","が","で","て","と","も","し","た","る","こと","ます","です","さん","など","よう","その","この","あの","これ","それ","あれ","から","ので","もの","ない","って","いう","速報","情報","一覧","更新","記事","詳細","画像","写真","公開","開催","決定","発表","開始","終了","確認","問題","理由","方法","解説","結果","内容","対応","検討"
        ];

        const BACKUP = [{t:"Silence",s:"...",y:TYPE_NATURE},{t:"Waiting",s:"...",y:TYPE_NEUTRAL},{t:"Breath",s:"...",y:TYPE_ART}];

        let wordBuffer = [];
        let appState = { 
            mode: 'night',
            tension: 0, harmony: 0.5, noise: 0,
            targetFogCol: new THREE.Color(0x051020),
            currentBgCols: [],
            targetPalette: null
        };
        let isAudioReady = false;

        // --- RHYTHM & SCALES ---
        // ペンタトニック系を中心に、心地よく響くスケール構成
        const SCALES = {
            night:   ["C3", "G3", "Bb3", "C4", "Eb4", "G4", "Bb4"], 
            early:   ["D3", "A3", "D4", "E4", "F#4", "A4"],   
            morning: ["E3", "B3", "E4", "G#4", "B4", "C#5", "E5"], 
            day:     ["C3", "G3", "C4", "D4", "E4", "G4", "A4"],    
            evening: ["Db3", "Ab3", "Db4", "Eb4", "F4", "Ab4"]  
        };
        
        // ベースとなるリズムループ用の音
        const PULSE_NOTES = {
            night:   ["C2", "C3"],
            early:   ["D2", "A2"],
            morning: ["E2", "B2"],
            day:     ["C2", "G2"],
            evening: ["Db2", "Ab2"]
        };

        const Audio = {
            // Instruments
            drone: null, pulseSynth: null, mainSynth: null, bassSynth: null, 
            reverb: null, delay: null, 
            
            currentScale: SCALES.night,
            currentPulse: PULSE_NOTES.night,

            async init() {
                await Tone.start();
                Tone.Context.lookAhead = 0.1;
                
                // BPM設定（ゆったりとした心拍数程度）
                Tone.Transport.bpm.value = 66; 

                // Master FX (深いリバーブとディレイでアンビエント感を出す)
                this.reverb = new Tone.Reverb({decay: 8, wet: 0.5}).toDestination();
                await this.reverb.generate();
                
                // PingPongDelayで広がりを
                this.delay = new Tone.PingPongDelay("8n.", 0.3).connect(this.reverb);
                const masterComp = new Tone.Compressor(-20, 3).connect(this.delay);

                // 1. DRONE (背景の持続音) - フィルターが開閉して呼吸する
                const autoFilter = new Tone.AutoFilter({ frequency: 0.1, baseFrequency: 200, octaves: 2.6 }).connect(masterComp).start();
                this.drone = new Tone.PolySynth(Tone.Synth, {
                    oscillator: { type: "fatsine", count: 3, spread: 30 },
                    envelope: { attack: 2, decay: 1, sustain: 1, release: 5 }
                }).connect(autoFilter);
                this.drone.volume.value = -20;

                // 2. PULSE (リズム隊) - 柔らかいプラック音
                // コトコトという心地よいリズム
                this.pulseSynth = new Tone.PolySynth(Tone.Synth, {
                    oscillator: { type: "triangle" },
                    envelope: { attack: 0.005, decay: 0.1, sustain: 0, release: 0.1 }
                }).connect(this.delay);
                this.pulseSynth.volume.value = -18;

                // 3. MAIN SYNTH (単語出現時の音) - ベルっぽい透明感
                this.mainSynth = new Tone.PolySynth(Tone.FMSynth, {
                    harmonicity: 2, modulationIndex: 2,
                    oscillator: { type: "sine" },
                    envelope: { attack: 0.01, decay: 0.5, sustain: 0, release: 2 },
                    modulation: { type: "sine" },
                    modulationEnvelope: { attack: 0.5, decay: 0, sustain: 1, release: 0.5 }
                }).connect(this.delay);
                this.mainSynth.volume.value = -12;

                // 4. BASS (低音の支え)
                this.bassSynth = new Tone.MonoSynth({
                    oscillator: { type: "square" },
                    filter: { Q: 2, type: "lowpass", rollover: -12 },
                    envelope: { attack: 0.1, decay: 0.3, sustain: 0.5, release: 2 },
                    filterEnvelope: { attack: 0.001, decay: 0.1, sustain: 0.5, release: 1, baseFrequency: 100, octaves: 2 }
                }).connect(masterComp);
                this.bassSynth.volume.value = -14;

                this.startLoops();
                isAudioReady = true;
                Tone.Transport.start();
            },

            startLoops() {
                // 背景リズム：8分音符で淡々と刻む（ミニマルミュージック的）
                let step = 0;
                Tone.Transport.scheduleRepeat((time) => {
                    const notes = this.currentPulse;
                    // 4拍に1回ルート音、裏拍で5度などを混ぜる
                    if (step % 4 === 0) {
                        this.pulseSynth.triggerAttackRelease(notes[0], "16n", time);
                    } else if (step % 2 !== 0 && Math.random() > 0.4) {
                        // ランダムな裏打ち
                        this.pulseSynth.triggerAttackRelease(notes[1] || notes[0], "16n", time, 0.5);
                    }
                    step++;
                }, "8n");

                // ドローン：小節の頭でコードが変わるイメージ
                Tone.Transport.scheduleRepeat((time) => {
                    if (Math.random() > 0.3) {
                        const root = this.currentPulse[0];
                        // ルート + 5度
                        this.drone.triggerAttackRelease([root, Tone.Frequency(root).transpose(7)], "4m", time);
                    }
                }, "4m"); // 4小節ごと
            },

            setMode(mode) {
                if(!isAudioReady) return;
                this.currentScale = SCALES[mode] || SCALES.night;
                this.currentPulse = PULSE_NOTES[mode] || PULSE_NOTES.night;

                switch(mode) {
                    case 'night':
                        Tone.Transport.bpm.rampTo(60, 10);
                        this.reverb.wet.rampTo(0.6, 10);
                        break;
                    case 'morning':
                        Tone.Transport.bpm.rampTo(72, 10);
                        this.reverb.wet.rampTo(0.3, 10);
                        break;
                    case 'day':
                        Tone.Transport.bpm.rampTo(76, 10);
                        this.reverb.wet.rampTo(0.2, 10);
                        break;
                    default:
                        Tone.Transport.bpm.rampTo(66, 10);
                        this.reverb.wet.rampTo(0.5, 10);
                        break;
                }
            },

            // 重要な変更：音を即座に鳴らさず、次の「16分音符」のタイミングに予約する（Quantize）
            triggerEvent(type) {
                if(!isAudioReady) return;

                // ランダムにノートを選ぶ
                const scale = this.currentScale;
                const note = scale[Math.floor(Math.random() * scale.length)];
                
                // 次の16分音符のグリッド
                const nextSubdivision = Tone.Transport.nextSubdivision("16n");

                // 音色のバリエーション
                switch(type) {
                    case TYPE_CRISIS:
                        // 低音で重く
                        this.bassSynth.triggerAttackRelease(Tone.Frequency(note).transpose(-12), "8n", nextSubdivision);
                        break;
                    case TYPE_TECH:
                        // 短く高い音（アルペジオ的）
                        this.mainSynth.triggerAttackRelease(Tone.Frequency(note).transpose(12), "32n", nextSubdivision);
                        this.mainSynth.triggerAttackRelease(Tone.Frequency(note).transpose(19), "32n", nextSubdivision + Tone.Time("32n"));
                        break;
                    case TYPE_MONEY:
                         // 金属的
                        this.mainSynth.set({ harmonicity: 3 });
                        this.mainSynth.triggerAttackRelease(note, "16n", nextSubdivision);
                        setTimeout(() => this.mainSynth.set({ harmonicity: 2 }), 200);
                        break;
                    default:
                        // 通常：和音っぽく鳴らすことも
                        if(Math.random() > 0.7) {
                            const n2 = scale[(scale.indexOf(note) + 2) % scale.length];
                            this.mainSynth.triggerAttackRelease([note, n2], "8n", nextSubdivision);
                        } else {
                            this.mainSynth.triggerAttackRelease(note, "8n", nextSubdivision);
                        }
                        break;
                }
            }
        };

        // --- COLOR PALETTES ---
        const COLOR_PALETTES = {
            night: { hexes: ['#051020', '#102035', '#000510', '#152540'], fog: 0x051020, isLight: false },
            early: { hexes: ['#152030', '#2a3a50', '#101825', '#304055'], fog: 0x152030, isLight: false },
            morning: { hexes: ['#e0e5ec', '#f0f4f8', '#d0dbe5', '#eef2f5'], fog: 0xe0e5ec, isLight: true },
            day: { hexes: ['#f5f7fa', '#ffffff', '#edf1f5', '#f0f8ff'], fog: 0xf5f7fa, isLight: true },
            evening: { hexes: ['#2a1a20', '#402530', '#1a1015', '#50303a'], fog: 0x2a1a20, isLight: false }
        };

        let lastPaletteMinute = -1; 

        function log(msg) { document.getElementById('sys-log').innerText = msg; }
        function toggleLog() {
            const el = document.getElementById('sys-log');
            el.style.display = (el.style.display === 'none') ? 'block' : 'none';
        }

        function setInitialCurrentBgCols(mode) {
            const palette = COLOR_PALETTES[mode];
            appState.currentBgCols = palette.hexes.map(hex => new THREE.Color(hex));
        }

        function setTargetPalette(mode) {
            const palette = COLOR_PALETTES[mode];
            appState.targetPalette = palette.hexes.map(hex => new THREE.Color(hex));
            appState.targetFogCol.setHex(palette.fog);
        }
        
        function updateHUDColors(mode) {
            const palette = COLOR_PALETTES[mode];
            const isLight = palette.isLight;
            document.getElementById('source-list').style.color = isLight ? 'rgba(0,0,0,0.8)' : 'rgba(255,255,255,0.4)';
            document.getElementById('hud').style.color = isLight ? 'rgba(0,0,0,0.8)' : 'rgba(255,255,255,0.8)';
            document.body.style.color = isLight ? '#112' : '#eef';
            const startBtn = document.getElementById('start-btn');
            if (startBtn) {
                startBtn.style.borderColor = isLight ? 'rgba(0,0,0,0.4)' : 'rgba(255,255,255,0.3)';
                startBtn.style.color = isLight ? '#223' : '#e0e0e0';
            }
            const ctrls = document.querySelectorAll('.btn');
            ctrls.forEach(btn => {
                if(isLight) { btn.style.color = '#333'; btn.style.background = 'rgba(0,0,0,0.05)'; }
                else { btn.style.color = '#aaa'; btn.style.background = 'rgba(255,255,255,0.05)'; }
            });
        }

        function updateTime() {
            const now = new Date();
            const timeStr = now.toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'});
            document.getElementById('clock-display').innerText = timeStr;

            const h = now.getHours();
            const m = now.getMinutes();
            let newMode = 'night';
            if(h>=5 && h<7) newMode='early';
            else if(h>=7 && h<10) newMode='morning';
            else if(h>=10 && h<16) newMode='day';
            else if(h>=16 && h<19) newMode='evening';
            
            if(appState.mode !== newMode || m !== lastPaletteMinute) {
                appState.mode = newMode;
                lastPaletteMinute = m;
                setTargetPalette(newMode);
                updateHUDColors(newMode);
                if(isAudioReady) Audio.setMode(newMode);
            }
        }
        setInterval(updateTime, 1000);

        async function fetchEnv() {
            try {
                const ip = await fetch('https://ipapi.co/json/').then(r=>r.json());
                document.getElementById('loc-display').innerText = `${ip.city||'Earth'}`;
                if(ip.latitude){
                    const w = await fetch(`https://api.open-meteo.com/v1/forecast?latitude=${ip.latitude}&longitude=${ip.longitude}&current_weather=true`).then(r=>r.json());
                    document.getElementById('env-display').innerText = `${w.current_weather.temperature}°C`;
                }
            } catch(e){}
        }

        const segmenter = new Intl.Segmenter("ja-JP", { granularity: "word" });

        async function fetchFeeds() {
            const shuffled = ALL_FEEDS.sort(() => 0.5 - Math.random());
            const selectedFeeds = shuffled.slice(0, 8);
            document.getElementById('source-list').innerHTML = "Connected to:<br>" + selectedFeeds.map(f => f.name).join("<br>");
            log(`Scanning frequencies...`);
            const promises = selectedFeeds.map(f => fetch(PROXY + encodeURIComponent(f.url)).then(r=>r.ok?r.json():null).catch(e=>null));
            const results = await Promise.all(promises);
            let count = 0;
            results.forEach((data, idx) => {
                if(data && data.items) {
                    data.items.forEach(item => { processTitle(item.title, selectedFeeds[idx].name); count++; });
                }
            });
            if(count === 0) BACKUP.forEach(d => wordBuffer.push({text:d.t, source:d.s, type:d.y}));
            log(`Detected ${count} entities.`);
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
                        if ( (isKanji) || (isKatakana && word.length > 1) ) {
                            if(!/^[\u3040-\u309f]+$/.test(word)) addWord(word, source);
                        }
                    }
                }
            } else {
                cleanTitle.split(/\s+/).forEach(w => {
                    const c = w.replace(/[^a-zA-Z0-9]/g,'');
                    if(c.length > 2 && !STOP_WORDS.includes(c.toUpperCase()) && !/^\d+$/.test(c)) addWord(c, source);
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

        // --- VISUAL ---
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

        function getGenreColor(type) {
            const isLight = (appState.mode === 'day' || appState.mode === 'morning');
            if (isLight) {
                switch(type) {
                    case TYPE_CRISIS: return { hex:"#b71c1c", r:0.7, g:0.1, b:0.1 };
                    case TYPE_HOPE:   return { hex:"#e65100", r:0.9, g:0.3, b:0 };
                    case TYPE_TECH:   return { hex:"#01579b", r:0, g:0.35, b:0.6 };
                    case TYPE_NATURE: return { hex:"#1b5e20", r:0.1, g:0.37, b:0.12 };
                    case TYPE_ART:    return { hex:"#4a148c", r:0.29, g:0.08, b:0.55 };
                    case TYPE_MONEY:  return { hex:"#0d47a1", r:0.05, g:0.28, b:0.63 };
                    default:          return { hex:"#37474f", r:0.2, g:0.28, b:0.31 };
                }
            } else {
                switch(type) {
                    case TYPE_CRISIS: return { hex:"#ff8888", r:1, g:0.5, b:0.5 }; 
                    case TYPE_HOPE:   return { hex:"#ffddaa", r:1, g:0.85, b:0.6 }; 
                    case TYPE_TECH:   return { hex:"#aaddff", r:0.6, g:0.85, b:1 }; 
                    case TYPE_NATURE: return { hex:"#99ffaa", r:0.6, g:1, b:0.65 }; 
                    case TYPE_ART:    return { hex:"#ddbbff", r:0.85, g:0.7, b:1 }; 
                    case TYPE_MONEY:  return { hex:"#88aaff", r:0.5, g:0.6, b:1 };  
                    default:          return { hex:"#99aabb", r:0.6, g:0.7, b:0.75 };
                }
            }
        }

        function createTexture(text, source, type) {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            const wSize = 80; const sSize = 24;
            ctx.font = `500 ${wSize}px "Shippori Mincho", serif`; 
            const tWidth = ctx.measureText(text).width;
            ctx.font = `400 ${sSize}px "Roboto Mono", monospace`;
            const sWidth = ctx.measureText(source).width;
            const width = Math.max(tWidth, sWidth) + 80;
            const height = wSize + sSize + 80;
            
            canvas.width = width; canvas.height = height;
            ctx.textAlign = "center"; ctx.textBaseline = "top";
            
            const col = getGenreColor(type);
            const isLightBg = (appState.mode === 'day' || appState.mode === 'morning');
            
            ctx.shadowColor = isLightBg ? "rgba(0,0,0,0)" : col.hex; 
            ctx.shadowBlur = isLightBg ? 0 : 30; 
            ctx.fillStyle = col.hex;
            ctx.font = `500 ${wSize}px "Shippori Mincho", serif`;
            ctx.fillText(text, width/2, 20);
            
            ctx.shadowBlur = 0; 
            ctx.fillStyle = isLightBg ? "#555555" : "#999999";
            ctx.font = `400 ${sSize}px "Roboto Mono", monospace`;
            ctx.fillText(source, width/2, wSize + 35);

            return { tex: new THREE.CanvasTexture(canvas), ratio: width/height, col: col };
        }

        function spawnWord() {
            if(wordBuffer.length === 0) return;
            const data = wordBuffer.shift();
            if(wordBuffer.length < 50) wordBuffer.push(data);

            Audio.triggerEvent(data.type);

            const { tex, ratio, col } = createTexture(data.text, data.source, data.type);
            const mat = new THREE.SpriteMaterial({ map: tex, transparent: true, opacity: 0, blending: THREE.AdditiveBlending, depthWrite: false, color: 0xffffff });
            
            if (appState.mode === 'day' || appState.mode === 'morning') mat.blending = THREE.NormalBlending;

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
            Audio.setMode(appState.mode);
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
