<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Escáner QR Pro</title>
    <script src="https://unpkg.com/html5-qrcode" type="text/javascript"></script>
    
    <style>
        :root {
            --primary: #4f46e5;
            --primary-hover: #4338ca;
            --danger: #ef4444;
            --bg: #f9fafb;
            --card: #ffffff;
            --text: #111827;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background-color: var(--bg);
            color: var(--text);
            margin: 0;
            padding: 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
        }

        .app-container {
            max-width: 480px;
            width: 100%;
            background: var(--card);
            padding: 24px;
            border-radius: 16px;
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
            box-sizing: border-box;
        }

        h1 {
            font-size: 22px;
            text-align: center;
            margin-top: 0;
            margin-bottom: 20px;
        }

        .btn {
            display: block;
            width: 100%;
            padding: 14px;
            background-color: var(--primary);
            color: white;
            border: none;
            border-radius: 10px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: background-color 0.2s, transform 0.1s;
            margin-bottom: 20px;
        }

        .btn:active {
            transform: scale(0.98);
        }

        /* Contenedor del Visor de la Cámara */
        #camera-wrapper {
            width: 100%;
            border-radius: 12px;
            overflow: hidden;
            background: #000;
            margin-bottom: 20px;
            display: none; /* Oculto hasta que se presione el botón */
        }

        #reader {
            width: 100%;
            border: none !important; /* Elimina bordes internos de la librería */
        }

        /* Historial de Escaneos */
        .history-section {
            margin-top: 10px;
        }

        .history-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 12px;
            border-bottom: 2px solid #f3f4f6;
            padding-bottom: 8px;
        }

        .history-header h2 {
            font-size: 16px;
            margin: 0;
            font-weight: 600;
        }

        .btn-clear {
            background: none;
            border: none;
            color: var(--danger);
            cursor: pointer;
            font-size: 13px;
            font-weight: 500;
        }

        .scan-list {
            list-style: none;
            padding: 0;
            margin: 0;
            max-height: 280px;
            overflow-y: auto;
        }

        .scan-item {
            background: #f3f4f6;
            padding: 12px;
            border-radius: 8px;
            margin-bottom: 8px;
            font-size: 14px;
            word-break: break-all;
            animation: fadeIn 0.3s ease;
        }

        .scan-meta {
            display: block;
            font-size: 11px;
            color: #6b7280;
            margin-top: 6px;
        }

        .empty-state {
            text-align: center;
            color: #9ca3af;
            font-size: 14px;
            padding: 30px 0;
        }

        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(-5px); }
            to { opacity: 1; transform: translateY(0); }
        }
    </style>
</head>
<body>

    <div class="app-container">
        <h1>Escáner de Códigos QR</h1>
        
        <button id="btn-action" class="btn">Iniciar Cámara</button>
        
        <div id="camera-wrapper">
            <div id="reader"></div>
        </div>

        <div class="history-section">
            <div class="history-header">
                <h2>Registros Guardados</h2>
                <button id="btn-clear" class="btn-clear" style="display: none;">Borrar todo</button>
            </div>
            <ul id="scan-list" class="scan-list">
                </ul>
        </div>
    </div>

    <script>
        const btnAction = document.getElementById('btn-action');
        const btnClear = document.getElementById('btn-clear');
        const scanListUI = document.getElementById('scan-list');
        const cameraWrapper = document.getElementById('camera-wrapper');
        
        let html5QrCode;
        let isScanning = false;
        
        // Cargar registros previos de LocalStorage o empezar con array vacío
        let registroQR = JSON.parse(localStorage.getItem('historial_qr')) || [];

        // Dibujar el historial inicial al cargar la página
        renderHistorial();

        // Manejador del botón principal (Encender / Apagar)
        btnAction.addEventListener('click', () => {
            if (!isScanning) {
                startScanning();
            } else {
                stopScanning();
            }
        });

        // Manejador para limpiar el historial
        btnClear.addEventListener('click', () => {
            if (confirm('¿Estás seguro de que deseas eliminar todos los registros guardados?')) {
                registroQR = [];
                localStorage.removeItem('historial_qr');
                renderHistorial();
            }
        });

        function startScanning() {
            cameraWrapper.style.display = 'block';
            html5QrCode = new Html5Qrcode("reader");
            
            const config = { 
                fps: 10,                 // Cuadros por segundo (óptimo para no saturar el móvil)
                qrbox: { width: 230, height: 230 } // Cuadro guía de enfoque
            };

            // "environment" prioriza de forma automática la cámara trasera del smartphone
            html5QrCode.start(
                { facingMode: "environment" }, 
                config,
                onScanSuccess
            ).then(() => {
                isScanning = true;
                btnAction.innerText = "Detener Cámara";
                btnAction.style.backgroundColor = "#6b7280"; // Cambia a gris mientras escanea
            }).catch((err) => {
                console.error("Error al acceder a la cámara:", err);
                alert("Error: Asegúrate de otorgar permisos de cámara y usar una conexión segura (HTTPS).");
                cameraWrapper.style.display = 'none';
            });
        }

        function stopScanning() {
            if (html5QrCode) {
                html5QrCode.stop().then(() => {
                    isScanning = false;
                    btnAction.innerText = "Iniciar Cámara";
                    btnAction.style.backgroundColor = "var(--primary)";
                    cameraWrapper.style.display = 'none';
                }).catch((err) => console.error("Error al detener la cámara:", err));
            }
        }

        // Acción que se ejecuta al detectar exitosamente un QR
        function onScanSuccess(decodedText, decodedResult) {
            const ahora = new Date();
            // Formato de fecha local amigable (DD/MM/AAAA, HH:MM:SS)
            const fechaHora = ahora.toLocaleString(); 

            // Estructura del nuevo dato
            const nuevoRegistro = {
                contenido: decodedText,
                fecha: fechaHora
            };

            // Evitar que el escáner registre repetidas veces seguidas el mismo código en el mismo segundo
            if (registroQR.length > 0 && registroQR[0].contenido === decodedText) {
                return; 
            }

            // Guardar al inicio del array para mostrar lo más nuevo primero
            registroQR.unshift(nuevoRegistro);
            localStorage.setItem('historial_qr', JSON.stringify(registroQR));
            
            renderHistorial();
            triggerBeep(); // Feedback auditivo
        }

        function renderHistorial() {
            scanListUI.innerHTML = '';
            
            if (registroQR.length === 0) {
                scanListUI.innerHTML = '<li class="empty-state">No hay datos registrados aún.</li>';
                btnClear.style.display = 'none';
                return;
            }

            btnClear.style.display = 'block';

            registroQR.forEach(item => {
                const li = document.createElement('li');
                li.className = 'scan-item';
                
                // Si el texto del QR es un enlace web (URL), lo transformamos en un link interactivo
                if (item.contenido.startsWith('http://') || item.contenido.startsWith('https://')) {
                    li.innerHTML = `<a href="${item.contenido}" target="_blank" style="color: var(--primary); text-decoration: none; font-weight: 500;">${item.contenido}</a>`;
                } else {
                    li.innerText = item.contenido;
                }

                // Añadir fecha y hora
                const meta = document.createElement('span');
                meta.className = 'scan-meta';
                meta.innerText = `📅 ${item.fecha}`;
                
                li.appendChild(meta);
                scanListUI.appendChild(li);
            });
        }

        // Función de audio nativo para hacer un "Beep" al escanear
        function triggerBeep() {
            try {
                const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
                const oscillator = audioCtx.createOscillator();
                oscillator.type = 'sine';
                oscillator.frequency.setValueAtTime(600, audioCtx.currentTime); // Frecuencia del sonido
                oscillator.connect(audioCtx.destination);
                oscillator.start();
                oscillator.stop(audioCtx.currentTime + 0.15); // Duración de 150 milisegundos
            } catch (e) {
                console.log("AudioContext no permitido hasta interacción del usuario.");
            }
        }
    </script>
</body>
</html>
