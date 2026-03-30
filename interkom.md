```html
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Antigravity Interkom PTT</title>
    <!-- Tautan Manifest PWA -->
    <link rel="manifest" href="manifest.json">
    <meta name="theme-color" content="#121212">
    <link rel="apple-touch-icon" href="https://cdn-icons-png.flaticon.com/512/3246/3246830.png">
    <!-- PeerJS Library -->
    <script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
    <style>
        :root {
            --color-standby: #FF3B30; /* Merah */
            --color-talking: #34C759; /* Hijau */
            --color-searching: #FF9500; /* Orange */
            --bg-color: #121212;
            --text-color: #f2f2f7;
        }

        body {
            margin: 0;
            padding: 0;
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            overflow: hidden;
            user-select: none;
            -webkit-user-select: none;
        }

        /* Layar Pemilihan Ruang */
        #selection-screen {
            display: flex;
            flex-direction: column;
            gap: 20px;
            text-align: center;
        }

        .btn-room {
            background-color: #2c2c2e;
            color: white;
            border: 2px solid #48484a;
            padding: 20px 40px;
            font-size: 24px;
            border-radius: 12px;
            cursor: pointer;
            transition: all 0.2s ease;
            text-transform: uppercase;
            font-weight: bold;
        }

        .btn-room:hover {
            background-color: #3a3a3c;
            transform: scale(1.05);
        }

        /* Layar PTT */
        #intercom-screen {
            display: none;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            width: 100%;
            height: 100%;
        }

        .status-text {
            font-size: 18px;
            margin-bottom: 40px;
            font-weight: bold;
            text-transform: uppercase;
            letter-spacing: 1px;
            transition: color 0.3s ease;
        }

        /* Tombol PTT Utama */
        #ptt-btn {
            width: 200px;
            height: 200px;
            border-radius: 50%;
            border: none;
            color: white;
            font-size: 20px;
            font-weight: bold;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            cursor: pointer;
            transition: transform 0.1s ease-in-out, background-color 0.3s ease;
            user-select: none;
            -webkit-user-select: none;
            touch-action: none; /* Cegah zoom/scroll di hp */
            outline: none;
            display: flex;
            align-items: center;
            justify-content: center;
            text-align: center;
            padding: 20px;
        }

        #ptt-btn:active, .ptt-active {
            transform: scale(0.95);
        }

        /* Warna Status */
        .status-searching {
            background-color: var(--color-searching) !important;
            box-shadow: 0 0 30px rgba(255, 149, 0, 0.6) !important;
        }
        .status-standby {
            background-color: var(--color-standby) !important;
            box-shadow: 0 0 30px rgba(255, 59, 48, 0.6) !important;
        }
        .status-talking {
            background-color: var(--color-talking) !important;
            box-shadow: 0 0 50px rgba(52, 199, 89, 0.8) !important;
        }

        #room-info {
            position: absolute;
            top: 30px;
            font-size: 14px;
            color: #8e8e93;
            letter-spacing: 2px;
            font-weight: 600;
        }
    </style>
</head>
<body>

    <div id="selection-screen">
        <h2 style="margin-bottom: 10px;">ANTIGRAVITY INTERKOM</h2>
        <p style="color: #8e8e93; margin-top: 0; margin-bottom: 30px;">Pilih identitas perangkat ini:</p>
        <button class="btn-room" onclick="initIntercom('ANTIGRAVITY-RUANG-A', 'ANTIGRAVITY-RUANG-B')">RUANG A</button>
        <button class="btn-room" onclick="initIntercom('ANTIGRAVITY-RUANG-B', 'ANTIGRAVITY-RUANG-A')">RUANG B</button>
    </div>

    <div id="intercom-screen">
        <div id="room-info"></div>
        <div class="status-text" id="status-display" style="color: var(--color-searching);">MENUNGGU PILIHAN...</div>
        <button id="ptt-btn" class="status-searching">MENCARI...</button>
        <audio id="remote-audio" autoplay></audio>
    </div>

    <script>
        let peer;
        let myId, targetId;
        let localStream;
        
        const selectionScreen = document.getElementById('selection-screen');
        const intercomScreen = document.getElementById('intercom-screen');
        const pttBtn = document.getElementById('ptt-btn');
        const statusDisplay = document.getElementById('status-display');
        const remoteAudio = document.getElementById('remote-audio');
        const roomInfo = document.getElementById('room-info');

        let isConnected = false;
        let isTalking = false;
        let autoReconnectInterval = null;

        async function initIntercom(selectedId, peerTargetId) {
            myId = selectedId;
            targetId = peerTargetId;

            selectionScreen.style.display = 'none';
            intercomScreen.style.display = 'flex';
            
            let displayName = myId.includes("A") ? "RUANG A" : "RUANG B";
            roomInfo.innerText = "SESI AKTIF: " + displayName;
            updateStatus('MENCARI KONEKSI...', 'searching', 'MENCARI<br>LAWAN...');

            try {
                // Konfigurasi audio high-quality intercom
                localStream = await navigator.mediaDevices.getUserMedia({
                    audio: {
                        echoCancellation: true,
                        noiseSuppression: true,
                        autoGainControl: true,
                        channelCount: 1, // Mono cukup untuk suara
                        sampleRate: 48000
                    },
                    video: false
                });

                // Set mic ke mute secara default (Mode PTT)
                muteMic();

                setupPeer();
                setupPTT();

            } catch (err) {
                alert("Gagal mengakses mikrofon. Pastikan izin diberikan! Error: " + err.message);
                console.error(err);
                updateStatus('GAGAL AKSES MIC', 'standby', 'ERROR');
            }
        }

        function muteMic() {
            if (localStream) {
                localStream.getAudioTracks().forEach(track => track.enabled = false);
            }
        }

        function unmuteMic() {
            if (localStream) {
                localStream.getAudioTracks().forEach(track => track.enabled = true);
            }
        }

        function setupPeer() {
            peer = new Peer(myId, {
                debug: 2 // Log ke console
            });

            peer.on('open', (id) => {
                console.log('Terdaftar di jaringan dengan ID: ' + id);
                startAutoDiscovery();
            });

            // Menerima panggilan dari lawan
            peer.on('call', (call) => {
                console.log("Menerima koneksi masuk...");
                call.answer(localStream); // Jawab dengan mic (yg posisinya mute)
                handleCall(call);
            });

            peer.on('error', (err) => {
                console.error("PeerJS error:", err);
                if (err.type === 'unavailable-id') {
                    alert('ID Identitas ini sudah aktif di perangkat lain!');
                }
            });
        }

        function startAutoDiscovery() {
            if (autoReconnectInterval) clearInterval(autoReconnectInterval);
            
            // Re-run setiap 3 detik jika tidak terhubung
            attemptConnection();
            autoReconnectInterval = setInterval(() => {
                if (!isConnected) {
                    attemptConnection();
                }
            }, 3000);
        }

        function attemptConnection() {
            if (isConnected) return;
            console.log("Ping: Mencoba memanggil " + targetId + "...");
            const call = peer.call(targetId, localStream);
            if (call) {
                handleCall(call);
            }
        }

        function handleCall(call) {
            call.on('stream', (remoteStream) => {
                console.log("Koneksi audio P2P berhasil dibuat!");
                remoteAudio.srcObject = remoteStream;
                
                // Atur agar audio auto play (safari butuh catch promise)
                remoteAudio.play().catch(e => console.warn("Autoplay ter-block:", e));

                isConnected = true;
                updateStatus('TERHUBUNG - STANDBY', 'standby', 'TAHAN UNTUK<br>BICARA');
            });

            call.on('close', () => {
                console.log("Koneksi terputus.");
                handleDisconnection();
            });

            call.on('error', (err) => {
                console.error("Koneksi gagal:", err);
                handleDisconnection();
            });
        }

        function handleDisconnection() {
            isConnected = false;
            updateStatus('TERPUTUS - MENCARI ULANG...', 'searching', 'MENCARI<br>LAWAN...');
            // Loop koneksi ulang akan otomatis menangani ini
        }

        function setupPTT() {
            const startTalking = (e) => {
                if (e.type !== 'mousedown') e.preventDefault(); // Cegah klik ganda di sentuhan layer
                if (!isConnected) return;
                
                isTalking = true;
                unmuteMic();
                updateStatus('MENGIRIM SUARA...', 'talking', 'BICARA<br>SEKARANG');
                pttBtn.classList.add('ptt-active');
            };

            const stopTalking = (e) => {
                if (e.type !== 'mouseup') e.preventDefault();
                if (!isConnected) return;

                isTalking = false;
                muteMic();
                updateStatus('TERHUBUNG - STANDBY', 'standby', 'TAHAN UNTUK<br>BICARA');
                pttBtn.classList.remove('ptt-active');
            };

            // Event untuk Desktop (Mouse)
            pttBtn.addEventListener('mousedown', startTalking);
            window.addEventListener('mouseup', stopTalking); // Deteksi lepas diluar tombol

            // Event untuk Mobile (Touch / Sentuh)
            pttBtn.addEventListener('touchstart', startTalking, {passive: false});
            window.addEventListener('touchend', stopTalking);
            window.addEventListener('touchcancel', stopTalking); // Jika scroll dll memutus touch
        }

        function updateStatus(text, stateClass, btnHTML) {
            statusDisplay.innerText = text;
            statusDisplay.style.color = `var(--color-${stateClass})`;
            
            pttBtn.innerHTML = btnHTML;
            pttBtn.className = ''; 
            pttBtn.classList.add(`status-${stateClass}`);
            if(isTalking) pttBtn.classList.add('ptt-active');
        }

        // Pendaftaran Service Worker untuk PWA
        if ('serviceWorker' in navigator) {
            window.addEventListener('load', () => {
                navigator.serviceWorker.register('sw.js').then((reg) => {
                    console.log('Service Worker PWA terdaftar secara sukses!', reg.scope);
                }).catch((err) => {
                    console.log('Service Worker gagal terdaftar:', err);
                });
            });
        }
    </script>
</body>
</html>
```
