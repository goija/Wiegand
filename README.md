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

Hieronder is visueel weergegeven hoe de 'handshake' verloopt en hoe de verbinding uiteindelijk over
