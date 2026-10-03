# 🛰️ VectorFind

> High-performance Tactical Air Traffic Control (ATC) Plan Position Indicator & Transponder Search in Vue 3.

![Vue.js](https://img.shields.io/badge/Vue.js-41B883?style=for-the-badge&logo=vue.js&logoColor=white)
![Canvas API](https://img.shields.io/badge/Render-60FPS_HiDPI-00ff78?style=for-the-badge)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg?style=for-the-badge)

📡 **[Live Demo](https://twojnick.github.io/vectorfind/)**

**VectorFind** to symulator zielonego ekranu radaru kontroli zbliżania (TRACON / APP) zoptymalizowany pod maksymalną liczbę klatek na sekundę (60-144 FPS).

## ✈️ Co wyróżnia VectorFind?
- ⚡ **Zero Reactive Overhead:** Fizyka wektorów działa na niemutowalnych obiektach w natywnym JS, eliminując zbędne pętle reaktywności Vue.
- 🎯 **VectorFind Filter:** Wyszukiwanie celów w czasie rzeczywistym po znaku wywoławczym (Callsign), kodzie transpondera (Squawk) lub pułapie lotu.
- 📺 **CRT Phosphor Decay:** Realistyczny efekt powolnego wygaszania luminoforu za obracającą się anteną radaru.
- 🔊 **Zero-Asset Web Audio:** Syntezator pingu radaru i alarmu TCAS generowany bezpośrednio z częstotliwości matematycznych (brak zewnętrznych plików mp3).
- 🖥️ **HiDPI / Retina Ready:** Automatyczna adaptacja do gęstości pikseli ekranu bez rozmywania grafiki wektorowej.

## 🚀 Uruchomienie lokalne

```bash
git clone [https://github.com/TwojNick/vectorfind.git](https://github.com/TwojNick/vectorfind.git)
cd vectorfind
npm install
npm run dev
