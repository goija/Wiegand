# Architectuur van de Vortex Engine: Peer-to-Peer IoT met WebRTC

De **Vortex Engine** is ontworpen om razendsnelle, lokale M2M (Machine-to-Machine) communicatie op te zetten zonder continue afhankelijkheid van een centrale server. Door gebruik te maken van WebRTC wordt data—zoals gesimuleerde Wiegand 26-bit RFID-scans—direct tussen de zender en ontvanger uitgewisseld.

Dit document beschrijft de technische architectuur en de datastroom van het systeem.

---

## 🏗️ De Drie Pijlers van de Architectuur

Het systeem leunt op drie hoofdcomponenten die naadloos samenwerken om de peer-to-peer (P2P) tunnel op te zetten:

1. **De Signaling Server (Node-RED)**
   * **Rol:** De "Matchmaker" of bemiddelaar.
   * **Techniek:** WebSockets (WSS).
   * **Functie:** Apparaten in een netwerk weten elkaars IP-adres niet direct te vinden. Node-RED fungeert als een centraal doorgeefluik waarlangs de apparaten tijdelijk hun netwerkgegevens (SDP-offers en ICE-kandidaten) uitwisselen. Nadat de verbinding is gelegd, stapt Node-RED opzij.

2. **De STUN-Server (Google)**
   * **Rol:** De "Spiegel".
   * **Techniek:** STUN-protocol (`stun:stun.l.google.com:19302`).
   * **Functie:** Apparaten gebruiken deze server om erachter te komen hoe ze van buitenaf bereikbaar zijn (hun publieke IP en poort), zodat ze deze informatie met elkaar kunnen delen.

3. **De Clients (Zender & Ontvanger)**
   * **Rol:** De "Peers" (gelijkwaardige knooppunten).
   * **Techniek:** WebRTC DataChannel (HTML/JS).
   * **Functie:** De daadwerkelijke eindpunten van de applicatie. Dit is waar de data (bijvoorbeeld een pasnummer) wordt gegenereerd, verzonden, ontvangen en gedecodeerd.

---

## 🔄 Sequence Diagram: De Datastroom

Hieronder is visueel weergegeven hoe de 'handshake' verloopt en hoe de verbinding uiteindelijk overgaat in een directe offline P2P-tunnel.

```mermaid
sequenceDiagram
    autonumber
    participant Z as Zender (Scanner)
    participant S as STUN Server (Google)
    participant N as Node-RED (Signaling)
    participant O as Ontvanger (Scherm)

    Note over Z, O: Fase 1: Initialisatie & WebSockets
    Z->>N: Verbinden met WSS (ws://127.0.0.1:1880)
    O->>N: Verbinden met WSS (ws://127.0.0.1:1880)

    Note over Z, O: Fase 2: IP & Route Ontdekking
    Z->>S: Verzoek IP & Poort configuratie
    S-->>Z: STUN Response 
    O->>S: Verzoek IP & Poort configuratie
    S-->>O: STUN Response 

    Note over Z, O: Fase 3: De Handshake (via Node-RED)
    Z->>N: Stuur SDP Offer
    N->>O: Stuur Offer door
    O->>N: Stuur SDP Answer
    N->>Z: Stuur Answer door
    
    Z->>N: Stuur ICE Kandidaten (Netwerkroutes)
    N->>O: Stuur ICE door
    O->>N: Stuur ICE Kandidaten (Netwerkroutes)
    N->>Z: Stuur ICE door

    Note over Z, O: Fase 4: De P2P Tunnel (Node-RED stapt opzij)
    Z--)O: 🟢 WebRTC DataChannel geopend (Directe P2P verbinding)

    Note over Z, O: Fase 5: Apparaat Communicatie (Offline modus)
    Z--)O: Verzenden ruwe Wiegand 26-bit data
    Note right of O: Decodeer bitreeks & Toon Resultaat
ebRTC stelt twee browsers (je Zender en je Ontvanger) in staat om direct met elkaar te praten (Peer-to-Peer), maar ze hebben eerst een "ontmoetingsplek" nodig om elkaars IP-adressen te vinden. Die ontmoetingsplek is je Node-RED server (de Signaling Server).

Hier is het stappenplan om de P2P-tunnel te graven.
Stap 1: De Node-RED "Switchboard" (Signaling Server)

Node-RED hoeft de 90-bit seeds niet te begrijpen; het hoeft alleen maar de "Offers", "Answers" en "ICE-candidates" (de adresgegevens) door te geven tussen de Zender en Ontvanger.

    Open je Node-RED interface.

    Sleep een websocket in node naar je flow.

        Stel het pad in op: /ws/vortex

        Zet inkomende data op: Listen on

    Sleep een websocket out node naar je flow.

        Koppel deze aan hetzelfde pad: /ws/vortex

    Verbind de uitgang van de websocket in direct met de ingang van de websocket out.

    Klik op Deploy. Je Signaling Server draait nu!

Stap 2: De VORTEX WebRTC Engine (JavaScript)

Nu voegen we de echte P2P-logica toe aan je interface. Dit script vervangt je huidige (gesimuleerde) netwerkcode. Het verbindt eerst met Node-RED via WebSockets en bouwt daarna de directe P2P-tunnel op.

Voeg deze code toe aan je JavaScript-sectie in zowel de Zender als de Ontvanger:
// 1. Configuratie
const WS_URL = "ws://127.0.0.1:1880/ws/vortex"; // Zorg dat dit overeenkomt met Node-RED
let signalingSocket;
let peerConnection;
let dataChannel;

// Standaard Google STUN servers om ICE (IP/Poort) te vinden
const rtcConfig = {
    iceServers: [{ urls: "stun:stun.l.google.com:19302" }]
};

// 2. Initialiseer Netwerk (Verbind met Node-RED)
function initNetwork() {
    addLog("Verbinden met Node-RED...", "#00d2ff");
    signalingSocket = new WebSocket(WS_URL);

    signalingSocket.onopen = () => {
        addLog("✅ Verbonden met Node-RED (WSS ONLINE)", "var(--neon-green)");
        document.querySelector('.status-header span').style.color = "var(--neon-green)";
    };

    signalingSocket.onmessage = async (message) => {
        const data = JSON.parse(message.data);
        
        if (data.type === 'offer') {
            addLog("Signaal ontvangen: offer", "#ff9d00");
            await peerConnection.setRemoteDescription(new RTCSessionDescription(data));
            const answer = await peerConnection.createAnswer();
            await peerConnection.setLocalDescription(answer);
            signalingSocket.send(JSON.stringify(peerConnection.localDescription));
            addLog("Signaal verzonden: answer", "var(--neon-blue)");
        } 
        else if (data.type === 'answer') {
            addLog("Signaal ontvangen: answer", "var(--neon-blue)");
            await peerConnection.setRemoteDescription(new RTCSessionDescription(data));
        } 
        else if (data.type === 'candidate') {
            addLog("Signaal ontvangen: ICE", "var(--neon-blue)");
            await peerConnection.addIceCandidate(new RTCIceCandidate(data.candidate));
        }
    };
}

// 3. Bouw de P2P Tunnel (WebRTC)
function openP2PTunnel(isSender) {
    addLog("Tunnel graven...", "#ff9d00");
    peerConnection = new RTCPeerConnection(rtcConfig);

    // Als we ICE candidates vinden, stuur ze via Node-RED naar de ander
    peerConnection.onicecandidate = (event) => {
        if (event.candidate) {
            addLog("Signaal verzonden: ICE", "var(--neon-green)");
            signalingSocket.send(JSON.stringify({ type: 'candidate', candidate: event.candidate }));
        }
    };

    if (isSender) {
        // Zender creëert het DataKanaal
        dataChannel = peerConnection.createDataChannel("vortex_90bit_stream");
        setupDataChannel(dataChannel);

        // Zender maakt het Offer
        peerConnection.createOffer().then(offer => {
            peerConnection.setLocalDescription(offer);
            signalingSocket.send(JSON.stringify(offer));
            addLog("Signaal verzonden: offer", "#ff00ff");
        });
    } else {
        // Ontvanger wacht op het DataKanaal
        peerConnection.ondatachannel = (event) => {
            dataChannel = event.channel;
            setupDataChannel(dataChannel);
        };
    }
}

// 4. Beheer het P2P Kanaal
function setupDataChannel(channel) {
    channel.onopen = () => {
        addLog("✅ P2P TUNNEL OPEN", "var(--neon-green)");
        document.querySelectorAll('.status-header span')[1].style.color = "var(--neon-green)";
    };

    channel.onclose = () => {
        addLog("❌ Tunnel gesloten!", "var(--error-red)");
    };

    channel.onmessage = (event) => {
        // Hier komt de 26-bit data binnen bij de Ontvanger!
        addLog(`INKOMEND: ${event.data}`, "#ff00ff");
        // Decodeer logica komt hier...
    };
}

// 5. Verzend Data Functie
function sendP2PData(seedData) {
    if (dataChannel && dataChannel.readyState === 'open') {
        dataChannel.send(seedData);
        addLog(`VERZONDEN: ${seedData}`, "#ff00ff");
    } else {
        addLog("Fout: Tunnel niet open!", "var(--error-red)");
    }
}

tap 3: Koppel de knoppen uit je interface

Om dit te laten werken, koppel je de knoppen uit je linkermenu (uit de HTML die we eerder maakten) aan deze nieuwe functies:

    Koppel "1. INITIALISEER NETWERK" aan initNetwork().

    Koppel "2. OPEN P2P TUNNEL" aan openP2PTunnel(true) op de Zender, en openP2PTunnel(false) op de Ontvanger.

Wat gebeurt er nu als je op de knoppen drukt?

    Je klikt op Initialiseer Netwerk: Je VORTEX dashboard verbindt via WebSockets met je Node-RED server. De eerste groene status (WSS ONLINE) gaat aan.

    Je klikt op Open P2P Tunnel: De Zender verstuurt een digitaal handdruk-verzoek (Offer) via Node-RED naar de Ontvanger. Ze wisselen ICE-candidates (IP-adressen) uit.

    Zodra ze elkaar gevonden hebben, wordt de P2P tunnel geopend. De "Tunnel niet open!" fout verdwijnt definitief, en je P2P ONLINE status wordt groen. Vanaf dat moment vloeit de hexadecimale data direct van Zender naar Ontvanger, zónder nog via Node-RED te gaan.
