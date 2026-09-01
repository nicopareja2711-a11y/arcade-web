# arcade-web
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Snake Arcade - Top 10 e Historial Completo</title>
    <style>
        body {
            background-color: #0d1117;
            color: #ffffff;
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            overflow: hidden;
        }
        .main-container {
            text-align: center;
            position: relative;
        }
        canvas {
            border: 4px solid #30363d;
            border-radius: 8px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
            background-color: #141820;
        }
        #info {
            margin-top: 10px;
            font-size: 14px;
            color: #8b949e;
            text-align: center;
        }
        /* Modal flotante para introducir iniciales al perder */
        #firebase-modal {
            display: none;
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background-color: rgba(22, 27, 34, 0.95);
            border: 2px solid #58a6ff;
            border-radius: 8px;
            padding: 20px;
            text-align: center;
            z-index: 10;
            box-shadow: 0 10px 25px rgba(0,0,0,0.8);
        }
        #firebase-modal input {
            background-color: #0d1117;
            border: 1px solid #30363d;
            color: #fff;
            font-size: 18px;
            text-align: center;
            width: 80px;
            padding: 5px;
            border-radius: 4px;
            text-transform: uppercase;
            margin: 10px 0;
        }
        #firebase-modal button {
            background-color: #238636;
            color: white;
            border: none;
            padding: 8px 15px;
            border-radius: 4px;
            cursor: pointer;
            font-weight: bold;
        }
        #firebase-modal button:hover {
            background-color: #2ea043;
        }
    </style>
</head>
<body>

    <div class="main-container">
        <canvas id="juego" width="540" height="540"></canvas>
        <div id="info">Usa las <b>Flechas</b> o <b>W, A, S, D</b> para moverte | Pulsa <b>TAB</b> para cambiar pantallas | <b>R</b> para reiniciar.</div>

        <div id="firebase-modal">
            <h3 style="margin:0 0 10px 0; color:#58a6ff;">¡NUEVA PUNTUACIÓN!</h3>
            <p style="font-size:13px; margin:5px 0;">Introduce tus iniciales (máx 3):</p>
            <input type="text" id="input-iniciales" maxlength="3" autocomplete="off" />
            <br>
            <button id="btn-guardar-score">Guardar Puntuación</button>
        </div>
    </div>

    <script type="module">
        // Importación del SDK Modular de Firebase v12 vía ESM (CDN)
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.0.1/app.js";
        import { getFirestore, collection, addDoc, getDocs, query, orderBy, limit, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.0.1/firestore.js";

        // CONFIGURACIÓN DE FIREBASE (Reemplaza con tus credenciales reales)
        const firebaseConfig = {
            apiKey: "AIzaSyC-dctgL8ugJg6bAYzZUUcN_QvIUdlNEjI",
            authDomain: "mi-arcade.firebaseapp.com",
            projectId: "mi-arcade",
            storageBucket: "mi-arcade.firebasestorage.app",
            messagingSenderId: "483727672543",
            appId: "1:483727672543:web:ff23857fc21d5adfb0d56a",
            measurementId: "G-B5MSMGVMBM"

        };

        // Inicializar Firebase y Firestore
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);

        // Referencias al DOM del modal de Firebase
        const modalFirebase = document.getElementById("firebase-modal");
        const inputIniciales = document.getElementById("input-iniciales");
        const btnGuardarScore = document.getElementById("btn-guardar-score");

        let scoreYaGuardadoEnPartida = false;
        let topRankingGlobal = [];      // Almacena el Top 10 de Firestore
        let historialGlobal = [];       // Almacena el Historial Completo (últimas 100 partidas)

        // Función para consultar el Top 10 (ordenado de mayor a menor puntuación)
        async function cargarTopRanking() {
            try {
                const q = query(
                    collection(db, "puntuaciones_snake"), 
                    orderBy("puntos", "desc"), 
                    limit(10)
                );
                const querySnapshot = await getDocs(q);
                
                topRankingGlobal = [];
                querySnapshot.forEach((doc) => {
                    topRankingGlobal.push(doc.data());
                });
            } catch (e) {
                console.error("Error al cargar el Top 10 desde Firestore: ", e);
                topRankingGlobal = [{ iniciales: "ERR", puntos: 0 }];
            }
        }

        // Función para consultar el Historial Completo (ordenado de más reciente a más antiguo, límite 100)
        async function cargarHistorialCompleto() {
            try {
                const q = query(
                    collection(db, "puntuaciones_snake"), 
                    orderBy("fecha", "desc"), 
                    limit(100)
                );
                const querySnapshot = await getDocs(q);
                
                historialGlobal = [];
                querySnapshot.forEach((doc) => {
                    historialGlobal.push(doc.data());
                });
            } catch (e) {
                console.error("Error al cargar el historial desde Firestore: ", e);
                historialGlobal = [];
            }
        }

        // Cargar ambas colecciones al iniciar la aplicación
        async function actualizarDatosFirestore() {
            await Promise.all([cargarTopRanking(), cargarHistorialCompleto()]);
        }
        actualizarDatosFirestore();

        // Evento click del botón guardar iniciales en Firestore
        btnGuardarScore.addEventListener("click", async () => {
            let iniciales = inputIniciales.value.trim().toUpperCase();
            if (!iniciales) iniciales = "AAA";
            
            try {
                await addDoc(collection(db, "puntuaciones_snake"), {
                    iniciales: iniciales,
                    puntos: puntos,
                    fecha: serverTimestamp()
                });

                modalFirebase.style.display = "none";
                inputIniciales.value = "";
                await actualizarDatosFirestore(); // Actualizar las listas locales tras guardar
            } catch (e) {
                console.error("Error al guardar en Firestore: ", e);
                alert("Hubo un error al guardar tu puntuación.");
            }
        });

        // Código del Juego Snake
        const canvas = document.getElementById("juego");
        const ctx = canvas.getContext("2d");

        const ANCHO = 540;
        const ALTO = 540;
        const TAM_CASILLA = 30;

        let serpiente = [];
        let serpienteAnterior = [];
        let direccion = { x: TAM_CASILLA, y: 0 };
        let bufferGiros = [];
        let manzanaPrincipal = { x: 0, y: 0, escurridiza: false };
        let puntos = 0;
        let record = parseInt(localStorage.getItem("snake_record")) || 0;
        let enEspera = true;
        let gameOver = false;
        
        // Control de pantallas de menús: 0 = Juego, 1 = Top 10, 2 = Historial Completo
        let estadoPantalla = 0; 
        let ultimoPasoTiempo = 0;

        let listaDoradas = [];
        let listaAzules = [];
        let listaCuchillos = [];
        let listaArcoiris = [];
        let listaChillis = [];
        let listaTotems = [];
        let listaCalaveras = [];
        let listaBombas = [];
        let listaLasers = [];
        let listaEventosActivos = [];

        let tiempoFinAzul = 0;
        let tiempoFinArcoiris = 0;
        let modoChilliActivo = false;
        let tiempoFinTotem = 0;
        let tiempoFinInvulnerabilidad = 0;
        let tiempoAnimacionResurreccion = 0;
        let bonusVelocidadDoradas = 0;
        let manzanasComidas = 0;
        let ultimoSpawnCaos = 0;
        let tiempoProximoEvento = 0;
        let tiempoProximoLaser = 0;

        let particulasExplosion = [];
        let particulasFuego = [];
        let particulasResurreccion = [];
        let piezasCortadas = [];

        const COLORES_PORTALES = ["#ff3232", "#32ff32", "#3296ff", "#ffdc00", "#ff64ff"];

        let audioCtx = null;
        function playSound(frecuencia, duracion, volumen = 0.3) {
            try {
                if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                let osc = audioCtx.createOscillator();
                let gain = audioCtx.createGain();
                osc.connect(gain);
                gain.connect(audioCtx.destination);
                let now = audioCtx.currentTime;
                osc.frequency.setValueAtTime(frecuencia, now);
                gain.gain.setValueAtTime(volumen, now);
                gain.gain.exponentialRampToValueAtTime(0.001, now + duracion);
                osc.start(now);
                osc.stop(now + duracion);
            } catch (e) {}
        }

        function reiniciarJuego() {
            modalFirebase.style.display = "none";
            scoreYaGuardadoEnPartida = false;
            serpiente = [{ x: 270, y: 270 }, { x: 240, y: 270 }, { x: 210, y: 270 }];
            serpienteAnterior = serpiente.map(s => ({ ...s }));
            direccion = { x: TAM_CASILLA, y: 0 };
            bufferGiros = [];
            puntos = 0;
            manzanasComidas = 0;
            bonusVelocidadDoradas = 0;
            listaDoradas = [];
            listaAzules = [];
            listaCuchillos = [];
            listaArcoiris = [];
            listaChillis = [];
            listaTotems = [];
            listaCalaveras = [];
            listaBombas = [];
            listaLasers = [];
            listaEventosActivos = [];
            particulasExplosion = [];
            particulasFuego = [];
            particulasResurreccion = [];
            piezasCortadas = [];
            tiempoFinAzul = 0;
            tiempoFinArcoiris = 0;
            modoChilliActivo = false;
            tiempoFinTotem = 0;
            tiempoFinInvulnerabilidad = 0;
            tiempoAnimacionResurreccion = 0;
            enEspera = true;
            gameOver = false;
            generarManzanaPrincipal();
            ultimoPasoTiempo = performance.now();
            tiempoProximoEvento = performance.now() + 15000;
            tiempoProximoLaser = performance.now() + 10000;
            actualizarDatosFirestore();
        }

        function obtenerPosicionesOcupadasGlobales() {
            let ocup = [{ x: Math.round(manzanaPrincipal.x), y: Math.round(manzanaPrincipal.y) }];
            serpiente.forEach(s => ocup.push({ x: s.x, y: s.y }));
            let coleccionables = [...listaDoradas, ...listaAzules, ...listaCuchillos, ...listaArcoiris, ...listaChillis, ...listaTotems, ...listaBombas];
            coleccionables.forEach(c => ocup.push({ x: Math.round(c.x), y: Math.round(c.y) }));
            return ocup;
        }

        function generarPosicionLibre(excepciones = []) {
            let columnas = ANCHO / TAM_CASILLA;
            let filas = ALTO / TAM_CASILLA;
            let ocupadasGlobales = obtenerPosicionesOcupadasGlobales().concat(excepciones);
            let pos;
            let intentos = 0;
            do {
                pos = {
                    x: Math.floor(Math.random() * columnas) * TAM_CASILLA,
                    y: Math.floor(Math.random() * filas) * TAM_CASILLA
                };
                intentos++;
                let ocupada = ocupadasGlobales.some(o => o.x === pos.x && o.y === pos.y);
                if (!ocupada || intentos > 300) break;
            } while (true);
            return pos;
        }

        function generarSpawnCalaveraSeguro(cabeza) {
            let esquinas = [{x: 30, y: 30}, {x: ANCHO - 30, y: 30}, {x: 30, y: ALTO - 30}, {x: ANCHO - 30, y: ALTO - 30}];
            let cabCx = cabeza.x + TAM_CASILLA / 2;
            let cabCy = cabeza.y + TAM_CASILLA / 2;
            esquinas.sort((a, b) => Math.hypot(b.x - cabCx, b.y - cabCy) - Math.hypot(a.x - cabCx, a.y - cabCy));
            return { x: esquinas[0].x, y: esquinas[0].y, t_aparicion: performance.now() };
        }

        function generarManzanaPrincipal() {
            let pos = generarPosicionLibre();
            let probEscurridiza = Math.min(0.75, puntos * 0.025);
            manzanaPrincipal = {
                x: pos.x,
                y: pos.y,
                escurridiza: Math.random() < probEscurridiza
            };
        }

        window.addEventListener("keydown", e => {
            if (audioCtx && audioCtx.state === 'suspended') audioCtx.resume();
            
            if (modalFirebase.style.display === "block") {
                if (e.key === "Enter") {
                    btnGuardarScore.click();
                }
                return;
            }

            // Alternar pantallas de menús con la tecla TAB: Juego (0) -> Top 10 (1) -> Historial (2) -> Juego (0)
            if (e.key === "Tab") {
                e.preventDefault();
                estadoPantalla = (estadoPantalla + 1) % 3;
                return;
            }

            // Si estamos en cualquier pantalla de menú y pulsamos otra tecla que no sea TAB, volvemos al juego (0) si no estamos escribiendo
            if (estadoPantalla !== 0) {
                estadoPantalla = 0;
            }

            let nuevaDir = null;
            if (e.key === "ArrowUp" || e.key === "w" || e.key === "W") nuevaDir = { x: 0, y: -TAM_CASILLA };
            if (e.key === "ArrowDown" || e.key === "s" || e.key === "S") nuevaDir = { x: 0, y: TAM_CASILLA };
            if (e.key === "ArrowLeft" || e.key === "a" || e.key === "A") nuevaDir = { x: -TAM_CASILLA, y: 0 };
            if (e.key === "ArrowRight" || e.key === "d" || e.key === "D") nuevaDir = { x: TAM_CASILLA, y: 0 };

            if ((e.key === "r" || e.key === "R") && gameOver) {
                reiniciarJuego();
                return;
            }

            if (enEspera && nuevaDir && estadoPantalla === 0) {
                if (!(nuevaDir.x === -direccion.x && nuevaDir.y === -direccion.y)) {
                    direccion = nuevaDir;
                    enEspera = false;
                    ultimoPasoTiempo = performance.now();
                }
                return;
            }

            if (!gameOver && !modoChilliActivo && nuevaDir && estadoPantalla === 0) {
                let refDir = bufferGiros.length > 0 ? bufferGiros[bufferGiros.length - 1] : direccion;
                if (!(nuevaDir.x === -refDir.x && nuevaDir.y === -refDir.y) && !(nuevaDir.x === refDir.x && nuevaDir.y === refDir.y)) {
                    if (bufferGiros.length < 2) bufferGiros.push(nuevaDir);
                }
            }
        });

        function actualizar(tiempoActual) {
            let dt = tiempoActual - ultimoPasoTiempo;

            let efectoArcoiris = tiempoActual < tiempoFinArcoiris;
            let efectoCongelado = tiempoActual < tiempoFinAzul && !efectoArcoiris && !modoChilliActivo;
            let efectoInvulnerable = (tiempoActual < tiempoFinInvulnerabilidad) || efectoArcoiris;
            let tieneTotem = tiempoActual < tiempoFinTotem;

            let segmentosExtra = Math.max(0, serpiente.length - 3);
            let pasosBase = Math.min(6.5 + (segmentosExtra * 0.10) + bonusVelocidadDoradas, 18.0);
            let pasosPorSegundo = modoChilliActivo ? 32.0 : (efectoArcoiris ? Math.min(pasosBase * 1.85, 24.0) : (efectoCongelado ? pasosBase * 0.60 : pasosBase));
            let intervaloPasoMs = 1000.0 / pasosPorSegundo;

            if (!gameOver && !enEspera && estadoPantalla === 0) {
                if (tiempoActual >= tiempoProximoEvento && listaEventosActivos.length === 0) {
                    let tipoEv = ["LAVA", "NIEBLA", "LLUVIA_DORADA", "PORTAL_CUANTICO", "TORMENTA_MAGNETICA"][Math.floor(Math.random() * 5)];
                    let ev = {
                        tipo: tipoEv,
                        t_inicio: tiempoActual,
                        t_activo: tiempoActual + 3000,
                        t_fin: tiempoActual + 9000,
                        casillas_lava: new Set(),
                        parejas_portales: [],
                        lluviaGenerada: false,
                        portalesGenerados: false,
                        magnetismoFinalizado: false
                    };

                    if (tipoEv === "LAVA") {
                        let colFoco = Math.floor(Math.random() * (ANCHO / TAM_CASILLA));
                        let filFoco = Math.floor(Math.random() * (ALTO / TAM_CASILLA));
                        for (let c = 0; c < ANCHO / TAM_CASILLA; c++) {
                            for (let f = 0; f < ALTO / TAM_CASILLA; f++) {
                                if (Math.hypot(c - colFoco, f - filFoco) <= 4.0) {
                                    ev.casillas_lava.add(`${c * TAM_CASILLA},${f * TAM_CASILLA}`);
                                }
                            }
                        }
                    } else if (tipoEv === "PORTAL_CUANTICO") {
                        ev.portalesGenerados = true;
                        let ocupadosPortales = [];
                        for (let idxP = 0; idxP < 4; idxP++) {
                            let pA = generarPosicionLibre(ocupadosPortales);
                            ocupadosPortales.push(pA);
                            let pB = generarPosicionLibre(ocupadosPortales);
                            ocupadosPortales.push(pB);
                            let colorPareja = COLORES_PORTALES[idxP % COLORES_PORTALES.length];
                            ev.parejas_portales.push({ A: [pA.x, pA.y], B: [pB.x, pB.y], color: colorPareja });
                        }
                    }

                    listaEventosActivos.push(ev);
                    tiempoProximoEvento = tiempoActual + 20000;
                    playSound(440, 0.20, 0.35);
                }

                if (tiempoActual >= tiempoProximoLaser) {
                    playSound(1200, 0.4, 0.25);
                    let tipoL = Math.random() < 0.5 ? "H" : "V";
                    let coord = Math.floor(Math.random() * 16) * TAM_CASILLA;
                    listaLasers.push({
                        tipo: tipoL,
                        coord: coord,
                        t_inicio: tiempoActual,
                        t_aviso_fin: tiempoActual + 1400,
                        t_disparo_fin: tiempoActual + 2000
                    });
                    playSound(220, 0.3, 0.4);
                    tiempoProximoLaser = tiempoActual + 12000;
                }

                listaEventosActivos = listaEventosActivos.filter(ev => {
                    if (ev.tipo === "LLUVIA_DORADA" && tiempoActual >= ev.t_activo && !ev.lluviaGenerada) {
                        ev.lluviaGenerada = true;
                        for(let i = 0; i < 6; i++) {
                            let p = generarPosicionLibre();
                            listaDoradas.push({ x: p.x, y: p.y, t_aparicion: tiempoActual });
                        }
                    }
                    if (ev.tipo === "TORMENTA_MAGNETICA" && tiempoActual >= ev.t_activo) {
                        let cab = serpiente[0];
                        let cxCab = cab.x + TAM_CASILLA / 2;
                        let cyCab = cab.y + TAM_CASILLA / 2;
                        let coleccionables = [manzanaPrincipal, ...listaDoradas, ...listaAzules, ...listaCuchillos, ...listaArcoiris, ...listaChillis, ...listaTotems, ...listaBombas];
                        coleccionables.forEach(col => {
                            let colx = col.x + TAM_CASILLA / 2;
                            let coly = col.y + TAM_CASILLA / 2;
                            let dist = Math.hypot(cxCab - colx, cyCab - coly);
                            if (dist > 30 && dist < 180) {
                                col.x += (cxCab - colx) / dist * 0.7;
                                col.y += (cyCab - coly) / dist * 0.7;
                            }
                        });

                        if (tiempoActual >= ev.t_fin - 50 && !ev.magnetismoFinalizado) {
                            ev.magnetismoFinalizado = true;
                            coleccionables.forEach(col => {
                                let cxRed = Math.round(col.x / TAM_CASILLA) * TAM_CASILLA;
                                let cyRed = Math.round(col.y / TAM_CASILLA) * TAM_CASILLA;
                                if (cxRed < 0 || cxRed >= ANCHO || cyRed < 0 || cyRed >= ALTO || serpiente.some(s => s.x === cxRed && s.y === cyRed)) {
                                    let pSegura = generarPosicionLibre();
                                    col.x = pSegura.x; col.y = pSegura.y;
                                } else {
                                    col.x = cxRed; col.y = cyRed;
                                }
                            });
                        }
                    }
                    return tiempoActual < ev.t_fin;
                });

                listaLasers = listaLasers.filter(laser => {
                    if (tiempoActual >= laser.t_aviso_fin && tiempoActual < laser.t_disparo_fin && !efectoInvulnerable) {
                        let cab = serpiente[0];
                        let impact = laser.tipo === "H" ? Math.abs(cab.y - laser.coord) < TAM_CASILLA : Math.abs(cab.x - laser.coord) < TAM_CASILLA;
                        if (impact) {
                            if (tieneTotem) {
                                tiempoFinTotem = 0;
                                tiempoFinInvulnerabilidad = tiempoActual + 2600;
                                tiempoAnimacionResurreccion = tiempoActual;
                                playSound(1400, 0.45, 0.45);
                            } else { gameOver = true; playSound(150, 0.35, 0.4); }
                        }
                    }
                    return tiempoActual < laser.t_disparo_fin;
                });

                if (tiempoActual - ultimoSpawnCaos > 4000) {
                    ultimoSpawnCaos = tiempoActual;
                    if (Math.random() < 0.3 && listaAzules.length < 3) { let p = generarPosicionLibre(); listaAzules.push({ x: p.x, y: p.y, t_aparicion: tiempoActual }); }
                    if (Math.random() < 0.25 && listaChillis.length < 2) { let p = generarPosicionLibre(); listaChillis.push({ x: p.x, y: p.y, t_aparicion: tiempoActual }); }
                    if (serpiente.length >= 4 && Math.random() < 0.2 && listaCuchillos.length < 2) { let p = generarPosicionLibre(); listaCuchillos.push({ x: p.x, y: p.y, t_aparicion: tiempoActual }); }
                    if (Math.random() < 0.08 && listaArcoiris.length < 1) { let p = generarPosicionLibre(); listaArcoiris.push({ x: p.x, y: p.y, t_aparicion: tiempoActual }); }
                    if (!tieneTotem && Math.random() < 0.05 && listaTotems.length < 1) { let p = generarPosicionLibre(); listaTotems.push({ x: p.x, y: p.y, t_aparicion: tiempoActual }); }
                    if (Math.random() < 0.15 && listaBombas.length < 2) { let p = generarPosicionLibre(); listaBombas.push({ x: p.x, y: p.y, t_aparicion: tiempoActual }); }
                    if (puntos >= 3 && Math.random() < 0.25 && listaCalaveras.length < 3) {
                        listaCalaveras.push(generarSpawnCalaveraSeguro(serpiente[0]));
                    }
                }

                if (manzanaPrincipal.escurridiza) {
                    let cab = serpiente[0];
                    let dx = (manzanaPrincipal.x + TAM_CASILLA/2) - (cab.x + TAM_CASILLA/2);
                    let dy = (manzanaPrincipal.y + TAM_CASILLA/2) - (cab.y + TAM_CASILLA/2);
                    let dist = Math.hypot(dx, dy);
                    if (dist < 160 && dist > 0) {
                        manzanaPrincipal.x += (dx / dist) * 1.05;
                        manzanaPrincipal.y += (dy / dist) * 1.05;
                    }
                }

                let calaverasVivas = [];
                listaCalaveras.forEach(cal => {
                    if (tiempoActual - cal.t_aparicion < 12000) {
                        let cab = serpiente[0];
                        let dx = (cab.x + TAM_CASILLA / 2) - cal.x;
                        let dy = (cab.y + TAM_CASILLA / 2) - cal.y;
                        let dist = Math.hypot(dx, dy);
                        let vel = 0.75;

                        if (dist > 0) {
                            cal.x += (dx / dist) * vel;
                            cal.y += (dy / dist) * vel;
                        }

                        let tocaCuerpo = serpiente.some(seg => Math.hypot(cal.x - (seg.x + TAM_CASILLA/2), cal.y - (seg.y + TAM_CASILLA/2)) < 16);

                        if (efectoArcoiris || modoChilliActivo) {
                            if (dist < 20 || tocaCuerpo) {
                                playSound(950, 0.25, 0.4);
                                for (let i = 0; i < 20; i++) {
                                    let ang = Math.random() * Math.PI * 2;
                                    let v = Math.random() * 4 + 2;
                                    particulasExplosion.push({ x: cal.x, y: cal.y, vx: Math.cos(ang)*v, vy: Math.sin(ang)*v, color: "#ff6414", vida: 25 });
                                }
                                return;
                            }
                        } else if (!efectoInvulnerable) {
                            if (dist < 20) {
                                if (tieneTotem) {
                                    tiempoFinTotem = 0;
                                    tiempoFinInvulnerabilidad = tiempoActual + 2600;
                                    tiempoAnimacionResurreccion = tiempoActual;
                                    playSound(1400, 0.45, 0.45);
                                    for (let i = 0; i < 35; i++) {
                                        let ang = Math.random() * Math.PI * 2;
                                        let v = Math.random() * 4.5 + 1.5;
                                        particulasResurreccion.push({
                                            x: cab.x + TAM_CASILLA / 2, y: cab.y + TAM_CASILLA / 2,
                                            vx: Math.cos(ang) * v, vy: Math.sin(ang) * v,
                                            radio: Math.random() * 4 + 3, color: "#00ffb4", vida: 35
                                        });
                                    }
                                    return;
                                } else {
                                    gameOver = true;
                                    playSound(150, 0.35, 0.4);
                                }
                            }
                        }
                        calaverasVivas.push(cal);
                    }
                });
                listaCalaveras = calaverasVivas;

                if (modoChilliActivo) {
                    let cxCab = serpiente[0].x + TAM_CASILLA / 2;
                    let cyCab = serpiente[0].y + TAM_CASILLA / 2;
                    for (let i = 0; i < 3; i++) {
                        particulasFuego.push({
                            x: cxCab + (Math.random() - 0.5) * 12, y: cyCab + (Math.random() - 0.5) * 12,
                            vx: (Math.random() - 0.5) * 3, vy: (Math.random() - 0.5) * 3,
                            color: ["#ff3200", "#ff8c00", "#ffea14"][Math.floor(Math.random() * 3)],
                            radio: Math.random() * 4 + 3, vida: 18
                        });
                    }
                }
            }

            particulasFuego.forEach(pf => { pf.x += pf.vx; pf.y += pf.vy; pf.radio = Math.max(0.5, pf.radio - 0.25); pf.vida--; });
            particulasFuego = particulasFuego.filter(pf => pf.vida > 0);

            particulasExplosion.forEach(part => { part.x += part.vx; part.y += part.vy; part.vida--; });
            particulasExplosion = particulasExplosion.filter(part => part.vida > 0);

            particulasResurreccion.forEach(pr => { pr.x += pr.vx; pr.y += pr.vy; pr.radio = Math.max(0.5, pr.radio - 0.15); pr.vida--; });
            particulasResurreccion = particulasResurreccion.filter(pr => pr.vida > 0);

            piezasCortadas.forEach(p => {
                p.x1 += p.vx1; p.y1 += p.vy1;
                p.x2 += p.vx2; p.y2 += p.vy2;
                p.vy1 += 0.22; p.vy2 += 0.22;
                p.rot += p.vrot;
            });
            piezasCortadas = piezasCortadas.filter(p => tiempoActual - p.t_inicio < 700);

            listaBombas = listaBombas.filter(bom => {
                if (tiempoActual - bom.t_aparicion >= 2000) {
                    playSound(80, 0.45, 0.5);
                    let bxCentro = bom.x + TAM_CASILLA / 2;
                    let byCentro = bom.y + TAM_CASILLA / 2;

                    for (let i = 0; i < 35; i++) {
                        let ang = Math.random() * Math.PI * 2;
                        let v = Math.random() * 5.0 + 1.5;
                        particulasExplosion.push({
                            x: bxCentro, y: byCentro,
                            vx: Math.cos(ang) * v, vy: Math.sin(ang) * v,
                            color: ["#ff5000", "#ffc800", "#3c3c46", "#ff1e3c"][Math.floor(Math.random() * 4)],
                            vida: 30
                        });
                    }

                    let cab = serpiente[0];
                    let dx = Math.abs(cab.x - bom.x);
                    let dy = Math.abs(cab.y - bom.y);
                    let impacto3x3 = dx <= TAM_CASILLA && dy <= TAM_CASILLA;

                    if (impacto3x3 && !efectoInvulnerable) {
                        if (tieneTotem) {
                            tiempoFinTotem = 0;
                            tiempoFinInvulnerabilidad = tiempoActual + 2600;
                            tiempoAnimacionResurreccion = tiempoActual;
                            playSound(1400, 0.45, 0.45);
                            for (let i = 0; i < 35; i++) {
                                let ang = Math.random() * Math.PI * 2;
                                let v = Math.random() * 4.5 + 1.5;
                                particulasResurreccion.push({
                                    x: cab.x + TAM_CASILLA / 2, y: cab.y + TAM_CASILLA / 2,
                                    vx: Math.cos(ang) * v, vy: Math.sin(ang) * v,
                                    radio: Math.random() * 4 + 3, color: "#00ffb4", vida: 35
                                });
                            }
                        } else { gameOver = true; playSound(150, 0.35, 0.4); }
                    }
                    return false;
                }
                return true;
            });

            if (!gameOver && !enEspera && estadoPantalla === 0 && dt >= intervaloPasoMs) {
                ultimoPasoTiempo = tiempoActual;
                if (bufferGiros.length > 0 && !modoChilliActivo) direccion = bufferGiros.shift();

                serpienteAnterior = serpiente.map(s => ({ ...s }));

                let nx = serpiente[0].x + direccion.x;
                let ny = serpiente[0].y + direccion.y;

                if (modoChilliActivo) {
                    if (nx < 0 || nx >= ANCHO || ny < 0 || ny >= ALTO) {
                        modoChilliActivo = false;
                        if (direccion.x !== 0) {
                            let arriba = serpiente[0].y / TAM_CASILLA;
                            let abajo = (ALTO - serpiente[0].y - TAM_CASILLA) / TAM_CASILLA;
                            direccion = arriba >= abajo ? { x: 0, y: -TAM_CASILLA } : { x: 0, y: TAM_CASILLA };
                        } else {
                            let izq = serpiente[0].x / TAM_CASILLA;
                            let der = (ANCHO - serpiente[0].x - TAM_CASILLA) / TAM_CASILLA;
                            direccion = izq >= der ? { x: -TAM_CASILLA, y: 0 } : { x: TAM_CASILLA, y: 0 };
                        }
                        nx = serpiente[0].x + direccion.x; ny = serpiente[0].y + direccion.y;
                    }
                }

                if (efectoArcoiris) {
                    if (nx < 0) nx = ANCHO - TAM_CASILLA;
                    else if (nx >= ANCHO) nx = 0;
                    if (ny < 0) ny = ALTO - TAM_CASILLA;
                    else if (ny >= ALTO) ny = 0;
                }

                let nuevaCabeza = { x: nx, y: ny };
                let choqueBorde = nx < 0 || nx >= ANCHO || ny < 0 || ny >= ALTO;
                let choqueCuerpo = serpiente.slice(1).some(s => s.x === nx && s.y === ny);

                let choqueLava = false;
                listaEventosActivos.forEach(ev => {
                    if (ev.tipo === "LAVA" && tiempoActual >= ev.t_activo) {
                        if (ev.casillas_lava.has(`${nx},${ny}`)) choqueLava = true;
                    }
                });

                if ((choqueBorde || choqueCuerpo || choqueLava) && !efectoInvulnerable) {
                    if (tieneTotem) {
                        tiempoFinTotem = 0;
                        tiempoFinInvulnerabilidad = tiempoActual + 2600;
                        tiempoAnimacionResurreccion = tiempoActual;
                        playSound(1400, 0.45, 0.45);
                        if (choqueBorde) {
                            nuevaCabeza.x = Math.max(0, Math.min(ANCHO - TAM_CASILLA, serpiente[0].x));
                            nuevaCabeza.y = Math.max(0, Math.min(ALTO - TAM_CASILLA, serpiente[0].y));
                            direccion = { x: -direccion.x, y: -direccion.y };
                        }
                        for (let i = 0; i < 35; i++) {
                            let ang = Math.random() * Math.PI * 2;
                            let v = Math.random() * 4.5 + 1.5;
                            particulasResurreccion.push({
                                x: nuevaCabeza.x + TAM_CASILLA / 2, y: nuevaCabeza.y + TAM_CASILLA / 2,
                                vx: Math.cos(ang) * v, vy: Math.sin(ang) * v,
                                radio: Math.random() * 4 + 3, color: "#00ffb4", vida: 35
                            });
                        }
                    } else {
                        gameOver = true;
                        playSound(150, 0.35, 0.4);
                    }
                }

                if (!gameOver) {
                    serpiente.unshift(nuevaCabeza);
                    let crecio = false;

                    let distManz = Math.hypot((nuevaCabeza.x + TAM_CASILLA/2) - (manzanaPrincipal.x + TAM_CASILLA/2), (nuevaCabeza.y + TAM_CASILLA/2) - (manzanaPrincipal.y + TAM_CASILLA/2));
                    if (distManz <= 20) {
                        puntos += 1; manzanasComidas++; crecio = true;
                        playSound(880, 0.08, 0.25);
                        if (puntos > record) { record = puntos; localStorage.setItem("snake_record", record); }
                        if (manzanasComidas % 3 === 0) {
                            let p = generarPosicionLibre();
                            listaDoradas.push({ x: p.x, y: p.y, t_aparicion: tiempoActual });
                        }
                        generarManzanaPrincipal();
                    }

                    listaDoradas = listaDoradas.filter(dor => {
                        if (nuevaCabeza.x === dor.x && nuevaCabeza.y === dor.y) {
                            puntos += 3; bonusVelocidadDoradas += 0.10; crecio = true;
                            playSound(1320, 0.15, 0.35);
                            if (puntos > record) { record = puntos; localStorage.setItem("snake_record", record); }
                            return false;
                        }
                        return tiempoActual - dor.t_aparicion < 5000;
                    });

                    listaAzules = listaAzules.filter(az => {
                        if (nuevaCabeza.x === az.x && nuevaCabeza.y === az.y) {
                            tiempoFinAzul = tiempoActual + 7000;
                            playSound(520, 0.18, 0.3);
                            return false;
                        }
                        return tiempoActual - az.t_aparicion < 6000;
                    });

                    listaCuchillos = listaCuchillos.filter(cuch => {
                        if (nuevaCabeza.x === cuch.x && nuevaCabeza.y === cuch.y) {
                            playSound(350, 0.10, 0.3);
                            for(let i = 0; i < 2 && serpiente.length > 2; i++) {
                                let piezaEliminada = serpiente.pop();
                                piezasCortadas.push({
                                    x1: piezaEliminada.x, y1: piezaEliminada.y,
                                    x2: piezaEliminada.x + TAM_CASILLA / 2, y2: piezaEliminada.y,
                                    vx1: (Math.random() * 1.6 + 1.2) * (Math.random() < 0.5 ? 1 : -1),
                                    vy1: Math.random() * -2.0 - 1.5,
                                    vx2: (Math.random() * 1.6 + 1.2) * (Math.random() < 0.5 ? 1 : -1),
                                    vy2: Math.random() * -2.0 - 1.5,
                                    rot: 0.0,
                                    vrot: Math.random() * 5 + 4,
                                    t_inicio: tiempoActual
                                });
                            }
                            serpienteAnterior = serpiente.map(s => ({ ...s }));
                            return false;
                        }
                        return tiempoActual - cuch.t_aparicion < 6000;
                    });

                    listaArcoiris = listaArcoiris.filter(arc => {
                        if (nuevaCabeza.x === arc.x && nuevaCabeza.y === arc.y) {
                            puntos += 5; tiempoFinArcoiris = tiempoActual + 5000;
                            playSound(1760, 0.22, 0.35);
                            if (puntos > record) { record = puntos; localStorage.setItem("snake_record", record); }
                            return false;
                        }
                        return tiempoActual - arc.t_aparicion < 5000;
                    });

                    listaChillis = listaChillis.filter(chi => {
                        if (nuevaCabeza.x === chi.x && nuevaCabeza.y === chi.y) {
                            puntos += 2; modoChilliActivo = true;
                            playSound(600, 0.20, 0.35);
                            if (puntos > record) { record = puntos; localStorage.setItem("snake_record", record); }
                            return false;
                        }
                        return tiempoActual - chi.t_aparicion < 5000;
                    });

                    listaTotems = listaTotems.filter(tot => {
                        if (nuevaCabeza.x === tot.x && nuevaCabeza.y === tot.y) {
                            tiempoFinTotem = tiempoActual + 15000;
                            playSound(1050, 0.30, 0.35);
                            return false;
                        }
                        return tiempoActual - tot.t_aparicion < 6000;
                    });

                    listaEventosActivos.forEach(ev => {
                        if (ev.tipo === "PORTAL_CUANTICO" && tiempoActual >= ev.t_activo) {
                            ev.parejas_portales.forEach(p => {
                                if (Math.abs(nuevaCabeza.x - p.A[0]) < 5 && Math.abs(nuevaCabeza.y - p.A[1]) < 5) {
                                    serpiente[0] = { x: p.B[0], y: p.B[1] };
                                    playSound(1850, 0.2, 0.4);
                                } else if (Math.abs(nuevaCabeza.x - p.B[0]) < 5 && Math.abs(nuevaCabeza.y - p.B[1]) < 5) {
                                    serpiente[0] = { x: p.A[0], y: p.A[1] };
                                    playSound(1850, 0.2, 0.4);
                                }
                            });
                        }
                    });

                    if (!crecio) serpiente.pop();
                    else serpienteAnterior.push({...serpienteAnterior[serpienteAnterior.length - 1]});
                }
            }

            dibujar(tiempoActual, efectoCongelado, efectoArcoiris, modoChilliActivo);
            
            if (gameOver && !scoreYaGuardadoEnPartida) {
                scoreYaGuardadoEnPartida = true;
                modalFirebase.style.display = "block";
                inputIniciales.focus();
            }

            requestAnimationFrame(actualizar);
        }

        function dibujarManzanaPrincipal(x, y, escurridiza, tiempoMs) {
            let rx = x, ry = y;
            if (escurridiza) {
                let aleteo = Math.sin(tiempoMs / 45) * 6;
                ctx.fillStyle = "rgba(255, 255, 255, 0.75)";
                ctx.beginPath(); ctx.moveTo(rx + 4, ry + 12); ctx.lineTo(rx - 9, ry + 1 + aleteo); ctx.lineTo(rx - 3, ry + 17 + aleteo); ctx.closePath(); ctx.fill();
                ctx.beginPath(); ctx.moveTo(rx + TAM_CASILLA - 4, ry + 12); ctx.lineTo(rx + TAM_CASILLA + 9, ry + 1 + aleteo); ctx.lineTo(rx + TAM_CASILLA + 3, ry + 17 + aleteo); ctx.closePath(); ctx.fill();
            }
            ctx.fillStyle = "#ff1e3c";
            ctx.beginPath(); ctx.roundRect(rx + 3, ry + 4, TAM_CASILLA - 6, TAM_CASILLA - 6, 10); ctx.fill();

            ctx.fillStyle = "rgba(255, 255, 255, 0.45)";
            ctx.beginPath(); ctx.ellipse(rx + 10, ry + 10, 4, 2, Math.PI / 4, 0, Math.PI * 2); ctx.fill();

            ctx.strokeStyle = "#5a3214"; ctx.lineWidth = 3;
            ctx.beginPath(); ctx.moveTo(rx + TAM_CASILLA/2, ry + 4); ctx.quadraticCurveTo(rx + TAM_CASILLA/2 + 4, ry - 2, rx + TAM_CASILLA/2 + 6, ry - 3); ctx.stroke();
            ctx.fillStyle = "#2eb43c";
            ctx.beginPath(); ctx.ellipse(rx + TAM_CASILLA/2 + 8, ry, 4, 2, Math.PI / 6, 0, Math.PI * 2); ctx.fill();
        }

        function dibujarCuchillo(x, y) {
            ctx.fillStyle = "#dcdeeb";
            ctx.beginPath();
            ctx.moveTo(x + 10, y + 20);
            ctx.lineTo(x + 14, y + 8);
            ctx.lineTo(x + 24, y + 6);
            ctx.lineTo(x + 22, y + 16);
            ctx.lineTo(x + 12, y + 22);
            ctx.closePath();
            ctx.fill();
            ctx.strokeStyle = "#828c9b"; ctx.lineWidth = 1; ctx.stroke();

            ctx.fillStyle = "#783c14";
            ctx.beginPath();
            ctx.moveTo(x + 9, y + 21);
            ctx.lineTo(x + 5, y + 25);
            ctx.lineTo(x + 7, y + 27);
            ctx.lineTo(x + 11, y + 23);
            ctx.closePath();
            ctx.fill();
        }

        function dibujarCopoNieve(x, y) {
            let cx = x + TAM_CASILLA / 2;
            let cy = y + TAM_CASILLA / 2;
            ctx.fillStyle = "#143c64";
            ctx.beginPath(); ctx.arc(cx, cy, 14, 0, Math.PI * 2); ctx.fill();

            for (let i = 0; i < 6; i++) {
                let angulo = i * (Math.PI / 3);
                let bx = cx + 11 * Math.cos(angulo);
                let by = cy + 11 * Math.sin(angulo);
                ctx.strokeStyle = "#b4f0ff"; ctx.lineWidth = 2;
                ctx.beginPath(); ctx.moveTo(cx, cy); ctx.lineTo(bx, by); ctx.stroke();
            }
            ctx.fillStyle = "#ffffff";
            ctx.beginPath(); ctx.arc(cx, cy, 3, 0, Math.PI * 2); ctx.fill();
        }

        function dibujarChilli(x, y) {
            ctx.fillStyle = "#e61919";
            ctx.beginPath();
            ctx.moveTo(x + 7, y + 9);
            ctx.lineTo(x + 17, y + 7);
            ctx.lineTo(x + 23, y + 14);
            ctx.lineTo(x + 20, y + 23);
            ctx.lineTo(x + 12, y + 24);
            ctx.lineTo(x + 7, y + 18);
            ctx.closePath();
            ctx.fill();
            ctx.strokeStyle = "#ff6432"; ctx.lineWidth = 1; ctx.stroke();
            ctx.strokeStyle = "#28b428"; ctx.lineWidth = 3;
            ctx.beginPath(); ctx.moveTo(x + 10, y + 8); ctx.lineTo(x + 7, y + 4); ctx.stroke();
        }

        function dibujarTotem(x, y, tiempoMs) {
            let cx = x + TAM_CASILLA/2, cy = y + TAM_CASILLA/2;
            let puls = (Math.sin(tiempoMs / 120) + 1.0) * 0.5;
            ctx.fillStyle = `rgba(0, 230, 150, ${0.3 + 0.3 * puls})`;
            ctx.beginPath(); ctx.arc(cx, cy, 14 + puls*3, 0, Math.PI*2); ctx.fill();

            ctx.fillStyle = "#00e696";
            ctx.beginPath(); ctx.roundRect(x + 2, y + 2, TAM_CASILLA - 4, TAM_CASILLA - 4, 8); ctx.fill();
            ctx.strokeStyle = "#ffffff"; ctx.lineWidth = 2; ctx.stroke();
            ctx.fillStyle = "#ffffff";
            ctx.fillRect(cx - 1, cy - 6, 2, 12);
            ctx.fillRect(cx - 5, cy - 2, 10, 2);
        }

        function dibujarBomba(x, y, tiempoMs) {
            let cx = x + TAM_CASILLA/2, cy = y + TAM_CASILLA/2;
            ctx.fillStyle = "#141419"; ctx.beginPath(); ctx.arc(cx, cy + 3, 11, 0, Math.PI * 2); ctx.fill();
            ctx.fillStyle = "#32323a"; ctx.beginPath(); ctx.arc(cx, cy + 2, 11, 0, Math.PI * 2); ctx.fill();
            ctx.fillStyle = "#5f5f6e"; ctx.beginPath(); ctx.arc(cx - 3, cy - 1, 4, 0, Math.PI * 2); ctx.fill();

            ctx.strokeStyle = "#b48c50"; ctx.lineWidth = 2;
            ctx.beginPath(); ctx.moveTo(cx, cy - 9); ctx.lineTo(cx + 5, cy - 15); ctx.stroke();
            if (Math.floor(tiempoMs / 80) % 2 === 0) {
                ctx.fillStyle = "#ffc800";
                ctx.beginPath(); ctx.arc(cx + 5, cy - 15, 3, 0, Math.PI * 2); ctx.fill();
            }
        }

        function dibujarCalavera(cx, cy, tiempoMs) {
            let puls = (Math.sin(tiempoMs / 100) + 1.0) * 0.5;
            let rAura = 16 + puls * 3;
            ctx.fillStyle = `rgba(180, 0, 40, ${0.3 + 0.2 * puls})`;
            ctx.beginPath(); ctx.arc(cx, cy, rAura, 0, Math.PI * 2); ctx.fill();

            ctx.fillStyle = "#e6e6eb"; ctx.beginPath(); ctx.arc(cx, cy - 2, 11, 0, Math.PI * 2); ctx.fill();
            ctx.fillStyle = "#d7d7dc"; ctx.fillRect(cx - 6, cy + 2, 12, 9);
            ctx.fillStyle = "#140a0f";
            ctx.beginPath(); ctx.arc(cx - 4, cy - 2, 3, 0, Math.PI * 2); ctx.fill();
            ctx.beginPath(); ctx.arc(cx + 4, cy - 2, 3, 0, Math.PI * 2); ctx.fill();
            ctx.fillStyle = "#ff1e1e";
            ctx.fillRect(cx - 5, cy - 3, 1.5, 1.5);
            ctx.fillRect(cx + 3, cy - 3, 1.5, 1.5);
        }

        function dibujarCabezaSerpiente(x, y, dir, chilli, congelada, arcoiris) {
            let cx = x + TAM_CASILLA / 2;
            let cy = y + TAM_CASILLA / 2;

            ctx.fillStyle = chilli ? "#ff3c00" : (arcoiris ? `hsl(${performance.now() / 8}, 90%, 50%)` : (congelada ? "#64dcff" : "#50eb46"));
            ctx.beginPath();
            ctx.roundRect(x + 1, y + 1, TAM_CASILLA - 2, TAM_CASILLA - 2, 8);
            ctx.fill();
            ctx.strokeStyle = chilli ? "#ffd232" : (arcoiris ? "#ffffff" : (congelada ? "#b4f0ff" : "#b4ffaa"));
            ctx.lineWidth = 2;
            ctx.stroke();

            let ox1 = cx, oy1 = cy, ox2 = cx, oy2 = cy;
            let px1 = cx, py1 = cy, px2 = cx, py2 = cy;
            let puntosLengua = [];

            if (dir.x > 0) {
                ox1 = cx + 3; oy1 = cy - 6; ox2 = cx + 3; oy2 = cy + 6;
                px1 = cx + 5; py1 = cy - 6; px2 = cx + 5; py2 = cy + 6;
                puntosLengua = [[cx + 14, cy - 2], [cx + 19, cy - 4], [cx + 17, cy], [cx + 19, cy + 4], [cx + 14, cy + 2]];
            } else if (dir.x < 0) {
                ox1 = cx - 3; oy1 = cy - 6; ox2 = cx - 3; oy2 = cy + 6;
                px1 = cx - 5; py1 = cy - 6; px2 = cx - 5; py2 = cy + 6;
                puntosLengua = [[cx - 14, cy - 2], [cx - 19, cy - 4], [cx - 17, cy], [cx - 19, cy + 4], [cx - 14, cy + 2]];
            } else if (dir.y < 0) {
                ox1 = cx - 6; oy1 = cy - 3; ox2 = cx + 6; oy2 = cy - 3;
                px1 = cx - 6; py1 = cy - 5; px2 = cx + 6; py2 = cy - 5;
                puntosLengua = [[cx - 2, cy - 14], [cx - 4, cy - 19], [cx, cy - 17], [cx + 4, cy - 19], [cx + 2, cy - 14]];
            } else {
                ox1 = cx - 6; oy1 = cy + 3; ox2 = cx + 6; oy2 = cy + 3;
                px1 = cx - 6; py1 = cy + 5; px2 = cx + 6; py2 = cy + 5;
                puntosLengua = [[cx - 2, cy + 14], [cx - 4, cy + 19], [cx, cy + 17], [cx + 4, cy + 19], [cx + 2, cy + 14]];
            }

            ctx.fillStyle = "#ff3c3c";
            ctx.beginPath();
            ctx.moveTo(puntosLengua[0][0], puntosLengua[0][1]);
            for(let p of puntosLengua) ctx.lineTo(p[0], p[1]);
            ctx.closePath();
            ctx.fill();

            ctx.fillStyle = "#ffffff";
            ctx.beginPath(); ctx.arc(ox1, oy1, 4, 0, Math.PI * 2); ctx.fill();
            ctx.beginPath(); ctx.arc(ox2, oy2, 4, 0, Math.PI * 2); ctx.fill();

            ctx.fillStyle = "#141414";
            ctx.beginPath(); ctx.arc(px1, py1, 2, 0, Math.PI * 2); ctx.fill();
            ctx.beginPath(); ctx.arc(px2, py2, 2, 0, Math.PI * 2); ctx.fill();
        }

        // Pantalla 1: Mejores Puntuaciones (Top 10)
        function dibujarTopRanking() {
            ctx.fillStyle = "rgba(13, 17, 23, 0.96)";
            ctx.fillRect(0, 0, ANCHO, ALTO);

            ctx.fillStyle = "#58a6ff";
            ctx.font = "bold 26px Arial";
            ctx.textAlign = "center";
            ctx.fillText("🏆 TOP 10 MEJORES PUNTUACIONES", ANCHO / 2, 50);

            ctx.fillStyle = "#8b949e";
            ctx.font = "13px Arial";
            ctx.fillText("[TAB] Cambiar a Historial Completo  |  [Cualquier tecla] Volver al juego", ANCHO / 2, 78);

            ctx.fillStyle = "#30363d";
            ctx.fillRect(50, 105, ANCHO - 100, 30);
            ctx.fillStyle = "#ffffff";
            ctx.font = "bold 13px Arial";
            ctx.textAlign = "left";
            ctx.fillText("POS", 70, 125);
            ctx.fillText("INICIALES", 160, 125);
            ctx.textAlign = "right";
            ctx.fillText("PUNTOS", ANCHO - 70, 125);

            if (topRankingGlobal.length === 0) {
                ctx.textAlign = "center";
                ctx.fillStyle = "#8b949e";
                ctx.font = "15px Arial";
                ctx.fillText("Cargando registros...", ANCHO / 2, ALTO / 2);
            } else {
                topRankingGlobal.forEach((item, index) => {
                    let yPos = 160 + (index * 33);
                    if (index % 2 === 0) {
                        ctx.fillStyle = "rgba(255, 255, 255, 0.03)";
                        ctx.fillRect(50, yPos - 20, ANCHO - 100, 28);
                    }

                    ctx.fillStyle = index === 0 ? "#ffd700" : (index === 1 ? "#c0c0c0" : (index === 2 ? "#cd7f32" : "#ffffff"));
                    ctx.font = "bold 14px Arial";
                    ctx.textAlign = "left";
                    ctx.fillText(`${index + 1}.`, 70, yPos);
                    ctx.fillText(item.iniciales || "AAA", 160, yPos);
                    
                    ctx.textAlign = "right";
                    ctx.fillStyle = "#58a6ff";
                    ctx.fillText(`${item.puntos} pts`, ANCHO - 70, yPos);
                });
            }
            ctx.textAlign = "left";
        }

        // Pantalla 2: Historial Completo (Últimas 100 partidas ordenadas de más reciente a más antigua)
        function dibujarHistorialCompleto() {
            ctx.fillStyle = "rgba(13, 17, 23, 0.96)";
            ctx.fillRect(0, 0, ANCHO, ALTO);

            ctx.fillStyle = "#3fb950";
            ctx.font = "bold 26px Arial";
            ctx.textAlign = "center";
            ctx.fillText("📜 HISTORIAL COMPLETO", ANCHO / 2, 50);

            ctx.fillStyle = "#8b949e";
            ctx.font = "13px Arial";
            ctx.fillText("[TAB] Cambiar a Top 10  |  [Cualquier tecla] Volver al juego", ANCHO / 2, 78);

            ctx.fillStyle = "#30363d";
            ctx.fillRect(40, 105, ANCHO - 80, 30);
            ctx.fillStyle = "#ffffff";
            ctx.font = "bold 12px Arial";
            ctx.textAlign = "left";
            ctx.fillText("FECHA Y HORA", 55, 125);
            ctx.fillText("INICIALES", 270, 125);
            ctx.textAlign = "right";
            ctx.fillText("PUNTOS", ANCHO - 55, 125);

            if (historialGlobal.length === 0) {
                ctx.textAlign = "center";
                ctx.fillStyle = "#8b949e";
                ctx.font = "15px Arial";
                ctx.fillText("Cargando historial...", ANCHO / 2, ALTO / 2);
            } else {
                historialGlobal.slice(0, 11).forEach((item, index) => {
                    let yPos = 160 + (index * 33);
                    if (index % 2 === 0) {
                        ctx.fillStyle = "rgba(255, 255, 255, 0.03)";
                        ctx.fillRect(40, yPos - 20, ANCHO - 80, 28);
                    }

                    let fechaStr = "Reciente";
                    if (item.fecha && item.fecha.toDate) {
                        fechaStr = item.fecha.toDate().toLocaleString("es-ES", {
                            day: '2-digit', month: '2-digit', year: 'numeric',
                            hour: '2-digit', minute: '2-digit'
                        });
                    }

                    ctx.fillStyle = "#ffffff";
                    ctx.font = "13px Arial";
                    ctx.textAlign = "left";
                    ctx.fillText(fechaStr, 55, yPos);
                    ctx.fillText(item.iniciales || "AAA", 280, yPos);
                    
                    ctx.textAlign = "right";
                    ctx.fillStyle = "#3fb950";
                    ctx.fillText(`${item.puntos} pts`, ANCHO - 55, yPos);
                });

                if (historialGlobal.length > 11) {
                    ctx.textAlign = "center";
                    ctx.fillStyle = "#8b949e";
                    ctx.font = "11px Arial";
                    ctx.fillText(`(Mostrando las últimas 11 de ${historialGlobal.length} partidas registradas)`, ANCHO / 2, ALTO - 20);
                }
            }
            ctx.textAlign = "left";
        }

        function dibujar(tiempoActual, congelado, arcoiris, chilli) {
            ctx.clearRect(0, 0, ANCHO, ALTO);

            if (estadoPantalla === 1) {
                dibujarTopRanking();
                return;
            } else if (estadoPantalla === 2) {
                dibujarHistorialCompleto();
                return;
            }

            let cols = ANCHO / TAM_CASILLA;
            let filas = ALTO / TAM_CASILLA;
            for (let f = 0; f < filas; f++) {
                for (let c = 0; c < cols; c++) {
                    ctx.fillStyle = (f + c) % 2 === 0 ? "#141820" : "#1c212d";
                    ctx.fillRect(c * TAM_CASILLA, f * TAM_CASILLA, TAM_CASILLA, TAM_CASILLA);
                }
            }

            if (arcoiris) {
                ctx.fillStyle = `hsla(${(tiempoActual/8)%360}, 80%, 50%, 0.12)`;
                ctx.fillRect(0, 0, ANCHO, ALTO);
            }

            listaEventosActivos.forEach(ev => {
                if (ev.tipo === "LAVA" && tiempoActual >= ev.t_activo) {
                    ev.casillas_lava.forEach(coord => {
                        let [lx, ly] = coord.split(',').map(Number);
                        ctx.fillStyle = (Math.floor(tiempoActual / 100) + lx) % 2 === 0 ? "#f03c00" : "#ff7800";
                        ctx.fillRect(lx, ly, TAM_CASILLA, TAM_CASILLA);
                    });
                } else if (ev.tipo === "LAVA" && tiempoActual < ev.t_activo) {
                    ev.casillas_lava.forEach(coord => {
                        let [lx, ly] = coord.split(',').map(Number);
                        ctx.fillStyle = "rgba(255, 30, 30, 0.4)";
                        ctx.fillRect(lx, ly, TAM_CASILLA, TAM_CASILLA);
                    });
                } else if (ev.tipo === "PORTAL_CUANTICO" && tiempoActual >= ev.t_activo) {
                    ev.parejas_portales.forEach(p => {
                        let puls = (Math.sin(tiempoActual / 90) + 1.0) * 0.5;
                        let r1 = 12 + 3 * puls;
                        let r2 = 7 - 2 * puls;
                        ctx.strokeStyle = p.color; ctx.lineWidth = 3;
                        ctx.beginPath(); ctx.arc(p.A[0] + TAM_CASILLA/2, p.A[1] + TAM_CASILLA/2, r1, 0, Math.PI*2); ctx.stroke();
                        ctx.beginPath(); ctx.arc(p.B[0] + TAM_CASILLA/2, p.B[1] + TAM_CASILLA/2, r1, 0, Math.PI*2); ctx.stroke();
                        ctx.fillStyle = "#ffffff";
                        ctx.beginPath(); ctx.arc(p.A[0] + TAM_CASILLA/2, p.A[1] + TAM_CASILLA/2, r2, 0, Math.PI*2); ctx.fill();
                        ctx.beginPath(); ctx.arc(p.B[0] + TAM_CASILLA/2, p.B[1] + TAM_CASILLA/2, r2, 0, Math.PI*2); ctx.fill();
                    });
                } else if (ev.tipo === "TORMENTA_MAGNETICA" && tiempoActual >= ev.t_activo) {
                    ctx.fillStyle = "rgba(0, 255, 200, 0.12)";
                    ctx.fillRect(0, 0, ANCHO, ALTO);
                }
            });

            listaLasers.forEach(laser => {
                let enAviso = tiempoActual < laser.t_aviso_fin;
                ctx.strokeStyle = enAviso ? "#ff3232" : "#ffffff";
                ctx.lineWidth = enAviso ? 2 : 6;
                ctx.beginPath();
                if (laser.tipo === "H") {
                    ctx.moveTo(0, laser.coord + TAM_CASILLA/2); ctx.lineTo(ANCHO, laser.coord + TAM_CASILLA/2);
                } else {
                    ctx.moveTo(laser.coord + TAM_CASILLA/2, 0); ctx.lineTo(laser.coord + TAM_CASILLA/2, ALTO);
                }
                ctx.stroke();
            });

            particulasFuego.forEach(pf => {
                ctx.fillStyle = pf.color;
                ctx.beginPath(); ctx.arc(pf.x, pf.y, pf.radio, 0, Math.PI * 2); ctx.fill();
            });

            particulasExplosion.forEach(part => {
                ctx.fillStyle = part.color;
                ctx.beginPath(); ctx.arc(part.x, part.y, 3, 0, Math.PI * 2); ctx.fill();
            });

            particulasResurreccion.forEach(pr => {
                ctx.fillStyle = pr.color;
                ctx.beginPath(); ctx.arc(pr.x, pr.y, pr.radio, 0, Math.PI * 2); ctx.fill();
            });

            piezasCortadas.forEach(p => {
                let progresoCorte = (tiempoActual - p.t_inicio) / 700.0;
                let alfaCorte = Math.max(0, 255 * (1.0 - progresoCorte));
                ctx.save();
                ctx.translate(p.x1, p.y1);
                ctx.rotate(p.rot * Math.PI / 180);
                ctx.fillStyle = `rgba(18, 75, 28, ${alfaCorte / 255})`;
                ctx.fillRect(-TAM_CASILLA / 4, -TAM_CASILLA / 2, TAM_CASILLA / 2, TAM_CASILLA - 4);
                ctx.restore();
            });

            if (tiempoActual - tiempoAnimacionResurreccion < 800 && serpiente.length > 0) {
                let prog = (tiempoActual - tiempoAnimacionResurreccion) / 800.0;
                let rOnda = prog * 180;
                let alfaOnda = Math.max(0, 255 * (1.0 - prog));
                let cab = serpiente[0];
                ctx.strokeStyle = `rgba(0, 255, 180, ${alfaOnda / 255})`;
                ctx.lineWidth = 4;
                ctx.beginPath();
                ctx.arc(cab.x + TAM_CASILLA / 2, cab.y + TAM_CASILLA / 2, rOnda, 0, Math.PI * 2);
                ctx.stroke();
            }

            dibujarManzanaPrincipal(manzanaPrincipal.x, manzanaPrincipal.y, manzanaPrincipal.escurridiza, tiempoActual);

            listaDoradas.forEach(dor => {
                if (!(5000 - (tiempoActual - dor.t_aparicion) < 1200 && Math.floor(tiempoActual / 120) % 2 === 0)) {
                    ctx.fillStyle = "#ffd700";
                    ctx.beginPath(); ctx.roundRect(dor.x + 2, dor.y + 2, TAM_CASILLA - 4, TAM_CASILLA - 4, 8); ctx.fill();
                }
            });
            listaAzules.forEach(az => {
                if (!(6000 - (tiempoActual - az.t_aparicion) < 1200 && Math.floor(tiempoActual / 120) % 2 === 0)) dibujarCopoNieve(az.x, az.y);
            });
            listaChillis.forEach(chi => {
                if (!(5000 - (tiempoActual - chi.t_aparicion) < 1200 && Math.floor(tiempoActual / 120) % 2 === 0)) dibujarChilli(chi.x, chi.y);
            });
            listaCuchillos.forEach(cuch => {
                if (!(6000 - (tiempoActual - cuch.t_aparicion) < 1200 && Math.floor(tiempoActual / 120) % 2 === 0)) dibujarCuchillo(cuch.x, cuch.y);
            });
            listaArcoiris.forEach(arc => {
                if (!(5000 - (tiempoActual - arc.t_aparicion) < 1200 && Math.floor(tiempoActual / 120) % 2 === 0)) {
                    ctx.fillStyle = `hsl(${(tiempoActual/8)%360}, 90%, 50%)`;
                    ctx.beginPath(); ctx.roundRect(arc.x + 2, arc.y + 2, TAM_CASILLA - 4, TAM_CASILLA - 4, 8); ctx.fill();
                }
            });
            listaTotems.forEach(tot => {
                if (!(6000 - (tiempoActual - tot.t_aparicion) < 1200 && Math.floor(tiempoActual / 120) % 2 === 0)) dibujarTotem(tot.x, tot.y, tiempoActual);
            });
            listaBombas.forEach(bom => {
                if (!(2000 - (tiempoActual - bom.t_aparicion) < 600 && Math.floor(tiempoActual / 80) % 2 === 0)) dibujarBomba(bom.x, bom.y, tiempoActual);
            });
            listaCalaveras.forEach(cal => {
                if (!(12000 - (tiempoActual - cal.t_aparicion) < 1500 && Math.floor(tiempoActual / 100) % 2 === 0)) dibujarCalavera(cal.x, cal.y, tiempoActual);
            });

            let tLin = Math.min(1.0, Math.max(0.0, (tiempoActual - ultimoPasoTiempo) / (1000.0 / 6.5)));
            let fInter = 1.0 - Math.pow(1.0 - tLin, 3);

            for (let i = serpiente.length - 1; i > 0; i--) {
                let act = serpiente[i];
                let ant = serpienteAnterior[i] || act;
                let px = ant.x + (act.x - ant.x) * fInter;
                let py = ant.y + (act.y - ant.y) * fInter;

                ctx.fillStyle = chilli ? "#ff3c00" : (arcoiris ? `hsl(${(tiempoActual/8 + i*22)%360}, 90%, 50%)` : (congelado ? "#64dcff" : "#2eb43c"));
                ctx.beginPath();
                ctx.roundRect(px + 1, py + 1, TAM_CASILLA - 2, TAM_CASILLA - 2, 6);
                ctx.fill();
            }

            if (serpiente.length > 0) {
                let act = serpiente[0];
                let ant = serpienteAnterior[0] || act;
                let cx = ant.x + (act.x - ant.x) * fInter;
                let cy = ant.y + (act.y - ant.y) * fInter;
                dibujarCabezaSerpiente(cx, cy, direccion, chilli, congelado, arcoiris);
            }

            let eventoNiebla = listaEventosActivos.find(ev => ev.tipo === "NIEBLA");
            if (eventoNiebla && serpiente.length > 0) {
                let cab = serpiente[0];
                let cxCab = cab.x + TAM_CASILLA / 2;
                let cyCab = cab.y + TAM_CASILLA / 2;

                let radioVision = 95;
                if (tiempoActual < eventoNiebla.t_activo) {
                    let progLlegada = (tiempoActual - eventoNiebla.t_inicio) / (eventoNiebla.t_activo - eventoNiebla.t_inicio);
                    radioVision = 320 - (225 * progLlegada);
                }

                ctx.save();
                ctx.fillStyle = "rgba(10, 12, 18, 0.94)";
                ctx.fillRect(0, 0, ANCHO, ALTO);

                ctx.globalCompositeOperation = "destination-out";
                ctx.beginPath();
                ctx.arc(cxCab, cyCab, radioVision, 0, Math.PI * 2);
                ctx.fill();
                ctx.restore();

                ctx.strokeStyle = "rgba(200, 220, 255, 0.4)";
                ctx.lineWidth = 4;
                ctx.beginPath();
                ctx.arc(cxCab, cyCab, radioVision, 0, Math.PI * 2);
                ctx.stroke();
            }

            if (congelado) {
                let tRestante = Math.max(0, tiempoFinAzul - tiempoActual);
                let factor = tRestante / 7000;
                ctx.strokeStyle = `rgba(0, 191, 255, ${0.6 * factor})`;
                ctx.lineWidth = 8;
                ctx.strokeRect(4, 4, ANCHO - 8, ALTO - 8);
            }

            if (listaEventosActivos.length > 0) {
                listaEventosActivos.forEach((ev, idx) => {
                    let activo = tiempoActual >= ev.t_activo;
                    let segs = Math.max(1, Math.round(((activo ? ev.t_fin : ev.t_activo) - tiempoActual) / 1000));
                    let textoBanner = "";
                    let colorBanner = "";

                    if (ev.tipo === "LAVA") {
                        textoBanner = activo ? `🔥 ¡LAVA ACTIVA EN SECTOR! (${segs}s)` : `⚠️ ¡CHARCO DE LAVA ACERCÁNDOSE! (${segs}s)`;
                        colorBanner = activo ? "#ff3214" : "#ff8c00";
                    } else if (ev.tipo === "NIEBLA") {
                        textoBanner = activo ? `👁️ VISIÓN REDUCIDA POR NIEBLA (${segs}s)` : `🌫️ ¡SE ACERCA NIEBLA ESPESA! (${segs}s)`;
                        colorBanner = "#c8dcff";
                    } else if (ev.tipo === "LLUVIA_DORADA") {
                        textoBanner = activo ? `✨ ¡LLUVIA DORADA! BOTÍN (${segs}s)` : `⭐ LLUVIA DORADA EN CAMINO (${segs}s)`;
                        colorBanner = "#ffdc3c";
                    } else if (ev.tipo === "PORTAL_CUANTICO") {
                        textoBanner = activo ? `🌀 ¡PORTALES CUÁNTICOS ACTIVOS! (${segs}s)` : `⚡ AVISO DE PORTALES CUÁNTICOS (${segs}s)`;
                        colorBanner = "#b43cb4";
                    } else if (ev.tipo === "TORMENTA_MAGNETICA") {
                        textoBanner = activo ? `🧲 ¡TORMENTA MAGNÉTICA! (¡ATRAYENDO OBJETOS!) (${segs}s)` : `⚡ AVISO DE TORMENTA MAGNÉTICA (${segs}s)`;
                        colorBanner = "#00e6c8";
                    }

                    ctx.fillStyle = "rgba(10, 10, 15, 0.85)";
                    ctx.fillRect(0, 38 + idx * 26, ANCHO, 24);
                    ctx.fillStyle = colorBanner;
                    ctx.font = "bold 13px Arial";
                    ctx.textAlign = "center";
                    ctx.fillText(textoBanner, ANCHO / 2, 55 + idx * 26);
                    ctx.textAlign = "left";
                });
            } else if (!enEspera && !gameOver && tiempoProximoEvento) {
                let segsProx = Math.max(1, Math.round((tiempoProximoEvento - tiempoActual) / 1000));
                ctx.fillStyle = "#8b949e";
                ctx.font = "13px Arial";
                ctx.fillText(`Próximo Evento en: ${segsProx}s`, 15, 45);
            }

            ctx.fillStyle = "#ffffff";
            ctx.font = "bold 16px Arial";
            ctx.fillText(`Puntos: ${puntos} | Récord: ${record}`, 15, 25);
            
            ctx.fillStyle = "#58a6ff";
            ctx.font = "12px Arial";
            ctx.fillText("Pulse [TAB] Menús", ANCHO - 130, 25);

            if (enEspera) {
                ctx.fillStyle = "#ffd700";
                ctx.font = "bold 20px Arial";
                ctx.textAlign = "center";
                ctx.fillText("PULSA UNA FLECHA PARA EMPEZAR", ANCHO / 2, ALTO / 2 + 70);
                ctx.textAlign = "left";
            }

            if (gameOver) {
                ctx.fillStyle = "rgba(0, 0, 0, 0.75)";
                ctx.fillRect(0, 0, ANCHO, ALTO);
                ctx.fillStyle = "#dc143c";
                ctx.font = "bold 42px Arial";
                ctx.textAlign = "center";
                ctx.fillText("GAME OVER", ANCHO / 2, ALTO / 2 - 35);
                ctx.fillStyle = "#ffffff";
                ctx.font = "bold 20px Arial";
                ctx.fillText(`Puntuación Final: ${puntos} pts`, ANCHO / 2, ALTO / 2 + 5);
                ctx.fillText("Pulsa 'R' para reiniciar", ANCHO / 2, ALTO / 2 + 40);
                ctx.textAlign = "left";
            }
        }

        reiniciarJuego();
        requestAnimationFrame(actualizar);
    </script>
</body>
</html>