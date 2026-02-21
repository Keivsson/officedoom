# Office Doom

> Ein Wolfenstein 3D-inspirierter Raycasting-Shooter im Comic-Büro-Setting.  
> Vollständig neu implementiert in C++ / SDL2. Läuft auf Windows und im Browser (WebAssembly).

---

## ☕ Spielprinzip

Es ist Montagmorgen. Die Büro-Zombies sind aufgewacht.  
Du hast nur deinen Kaffeebecher. Überlebe.

---

## Architektur-Übersicht

```
┌─────────────────────────────────────────────────────┐
│                    Game Loop                        │
│   handleEvents() → update() → render()             │
└──────────┬──────────┬────────────┬─────────────────┘
           │          │            │
      ┌────▼──┐  ┌────▼───┐  ┌────▼────────────────┐
      │ Player│  │  Map   │  │  Raycaster          │
      │ Input │  │ Tiles  │  │  DDA per column     │
      │ Move  │  │ Doors  │  │  Wall/Floor/Ceiling │
      │ Shoot │  │ Items  │  │  Sprite Billboard   │
      └───────┘  └────────┘  └─────────────────────┘
                                        │
                 ┌──────────────────────▼───────────┐
                 │           EnemySystem            │
                 │   Idle→Alert→Chase→Attack→Dead   │
                 │   LoS Raycast + HitScan          │
                 └──────────────────────────────────┘
                                        │
                 ┌──────────────────────▼───────────┐
                 │              HUD                 │
                 │   Health Bar, Ammo, Score        │
                 │   Crosshair, Weapon Sprite       │
                 └──────────────────────────────────┘
```

### Kern-Konzepte

**Raycasting (DDA):**  
Pro Bildschirmspalte wird ein Ray von der Spielerposition aus gesendet.  
Der DDA-Algorithmus berechnet schnell den nächsten Wand-Treffer.  
Wandhöhe = `renderHeight / perpDist`. Texture-Mapping via normiertem Trefferort.

**Z-Buffer:**  
Jede Spalte speichert die Wanddistanz. Sprite-Pixel werden nur gezeichnet, wenn ihre Distanz kleiner ist.

**Türen:**  
Tür-Tiles werden bei halbem Tile (Mittelposition) getroffen.  
`openAmount` (0..1) verschiebt die Textur und damit die Kollisionsgrenze.

**Sprites (Billboard):**  
Sprites werden nach Distanz sortiert (weit→nah) und via invertierter Kameramatrix projiziert.

**Gegner-KI:**  
State Machine: IDLE → ALERT (Sichtlinie) → CHASE → ATTACK → DEAD.  
Sichtlinie per Mini-Raycast (Schritte zum Spieler, Wand-Check).

---

## Projektstruktur

```
office-doom/
├── CMakeLists.txt          Build-Definition (Desktop + Web)
├── src/
│   ├── main.cpp            Entry Point
│   ├── game.h/cpp          Haupt-Loop, Level-Verwaltung
│   ├── map.h/cpp           Tile-Grid, Türen, Level-Loader
│   ├── player.h/cpp        Bewegung, Kollision, Input
│   ├── raycaster.h/cpp     DDA Renderer (Wände, Boden, Sprites)
│   ├── assets.h/cpp        PNG-Loader, Fallback-Texturen
│   ├── enemy.h/cpp         Gegner-KI, HitScan
│   └── hud.h/cpp           UI, Crosshair, Waffe, Debug
├── assets/
│   ├── walls/              64×64 Wandtexturen
│   ├── floor_ceiling/      64×64 Boden/Decke
│   ├── sprites/            64×96 Figuren + Deko
│   ├── weapons/            128×96 Waffen-Sprites
│   ├── ui/                 HUD, Crosshair, Font
│   └── levels/
│       └── level01.txt     20×20 ASCII-Level
├── tools/
│   └── asset_prep.py       Asset-Verarbeitungs-Tool
├── web/
│   ├── shell.html          Emscripten HTML-Shell
│   ├── index.html          Standalone Web-Seite
│   └── build/              ← Emscripten Output hier
└── docs/
    ├── README_desktop.md   Windows Build-Anleitung
    ├── README_web.md        Emscripten Build
    ├── wordpress_guide.md  WordPress Integration
    ├── asset_guide.md      Asset-Einbindung
    └── performance_guide.md Optimierung
```

---

## Quick Start

### Desktop (Windows)
```bat
# vcpkg SDL2 vorausgesetzt (s. docs/README_desktop.md)
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake
cmake --build . --config Release
cd ../desktop && officedoom.exe
```

### Web (Emscripten)
```bash
source emsdk/emsdk_env.sh
mkdir build-web && cd build-web
emcmake cmake .. -DCMAKE_BUILD_TYPE=Release
emmake cmake --build .
# Spielen: web/build/index.html (via HTTP-Server!)
python3 -m http.server 8080 --directory web/build
```

### Assets einbinden
```bash
pip install Pillow
python tools/asset_prep.py --input DEINE_BILDER/ --output assets/
# Spiel neu starten – fertig
```

---

## Level Format

ASCII-Datei, Zahlen durch Leerzeichen getrennt:

| Wert | Bedeutung |
|------|-----------|
| 0 | Leer |
| 1 | Ziegelwand |
| 2 | Glaswand |
| 3 | Bürowand |
| 4 | Tür |
| 5 | Spieler-Spawn |
| 6 | Zombie (Krawatte) |
| 7 | Zombie (Minirock) |
| 8 | Zombie (Blonde Hose) |
| 9 | Medkit |
| 10 | Munition |
| 11 | Pflanze |
| 12 | Schreibtisch |

---

## Test-Checkliste Meilenstein 1

- [ ] Startet ohne Crash
- [ ] Raycaster zeigt Wände mit Texturen
- [ ] Boden und Decke sichtbar
- [ ] Spieler bewegt sich (WASD)
- [ ] Kollision mit Wänden funktioniert
- [ ] Tür öffnet sich mit F
- [ ] Tür-Animation (langsam öffnen/schließen)
- [ ] Sprites (Pflanzen, Schreibtische) sichtbar
- [ ] Sprites korrekt durch Wände verdeckt
- [ ] HUD zeigt Health und Ammo
- [ ] Crosshair in der Mitte
- [ ] Waffe unten rechts sichtbar
- [ ] FPS ≥ 30 (TAB für Debug)
- [ ] Gegner idle sichtbar
- [ ] Gegner verfolgen Spieler
- [ ] Schuss trifft Gegner (SPACE)
- [ ] Gegner stirbt, Score steigt
