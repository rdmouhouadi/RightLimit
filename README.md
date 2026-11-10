<div align="center">

# 🚦 Right Limit
### Context-Aware Speed-Limit Assistance

*The right limit, for your lane, right now.*

![Version](https://img.shields.io/badge/version-v0.0.1--design-lightgrey)
![License](https://img.shields.io/badge/license-AGPL--3.0-green)
![Python](https://img.shields.io/badge/python-3.11%2B-blue)
![Status](https://img.shields.io/badge/status-Phase%200%20Design-orange)

**Author:** Richie Mouhouadi

</div>

---

## 🧭 What is Right Limit?

Right Limit is an open-source **research prototype** that aims to cut false speed-limit alerts
by fusing **camera sign reading**, **map data** and **temporal context**.

It answers one question, continuously:

> *Which speed limit applies to **my lane**, **right now**?*

Right Limit is also a **benchmark**. It compares two ways of building the same system:

- **Case 1 — Specialist pipeline:** pretrained sign detector + pretrained audio classifier + a small fusion model trained from scratch
- **Case 2 — Multimodal model:** one small vision-language model given camera frames and the map limit

> ⚠️ Right Limit is a research prototype. It is **not** a driving aid and must never be used as one.

---

## 🎯 Vision

### The problem

Intelligent Speed Assistance (ISA) has been mandatory on new EU car models since July 2024.
Drivers don't trust it:

- **54%** of UK drivers switch off at least one driver-assistance feature; speed assist is the most disabled, at **46%** ([Which?, Jan 2025, n = 1,584](https://www.which.co.uk/news/article/is-technology-ruining-new-cars-aGVAU5Q3JD6u))
- Complaints: constant beeping, wrong limits, and it can't be permanently disabled
- Known failure causes: signs for parallel roads, unofficial signs, digital signage, outdated maps
- From 2026, Euro NCAP rates driver-assistance **usability**

### Why not just use Waze or Google Maps?

| Source | Strength | Failure modes |
|---|---|---|
| **Map only** (navigation apps) | Knows which road you're on | Stale data, roadworks, rain limits, wrong road matching on ramps and parallel roads |
| **Camera only** | Sees temporary and new signs | Reads signs meant for another lane, exit ramp or truck |
| **Right Limit** | Fuses both, with temporal context | Resolves the disagreements |

Example: *"The map says 90. The camera saw 70 two seconds ago — on the exit ramp I didn't take. Keep 90."*

### Why now?

Carmakers are moving to **software-defined vehicles** with over-the-air updates
(Renault: 90% of vehicle functions updatable OTA). A better perception model can now ship
to cars already on the road — no hardware change.

**Right Limit is also a documented learning journey**, built alongside the DSTI
*Artificial Neural Networks* and *Deep Learning* courses. Each phase records the concepts learned,
the mistakes made and the resources used.

---

## 🏗️ Architecture Overview

```sh
┌──────────────────────────────────────────────────────────────────┐
│                          Right Limit                             │
├──────────────────────────────────────────────────────────────────┤
│  Inputs:  camera frames · GPS + heading · map prior · audio      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CASE 1 — Specialist pipeline          CASE 2 — Multimodal       │
│  ┌──────────────────────────┐          ┌──────────────────────┐  │
│  │ Sign detector (pretrained)│          │ Small vision-language│  │
│  │ Audio: rain (pretrained)  │          │ model                │  │
│  │ Map prior (OSM maxspeed)  │          │ frames + map limit   │  │
│  └────────────┬─────────────┘          │ + heading as text    │  │
│               ▼                        └──────────┬───────────┘  │
│  ┌──────────────────────────┐                     │              │
│  │ Fusion model (trained)    │                     │              │
│  │ sequence over last ~5 s   │                     │              │
│  │ "does this sign apply to  │                     │              │
│  │  MY lane?"                │                     │              │
│  └────────────┬─────────────┘                     │              │
│               └──────────────┬────────────────────┘              │
│                              ▼                                   │
│                 ┌─────────────────────────┐                      │
│                 │  Evaluation harness     │                      │
│                 │  same drives · same     │                      │
│                 │  metrics · per scenario │                      │
│                 └─────────────────────────┘                      │
└──────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**
- The **fusion model is the core contribution** — off-the-shelf models read signs; none decides which sign applies to your lane
- **Map-only** and **camera-only** baselines run on the same data as both cases
- Rain detection (audio) applies **French rain speed limits** — something navigation apps don't do automatically
- Edge-first: models exported to **int8 ONNX**, latency and memory measured on phone / Raspberry Pi class hardware

---

## 📊 Data Strategy

There is no public dataset labelled for *"which limit applies to my lane"*, so Right Limit assembles one.

| Source | Role | Notes |
|---|---|---|
| **CARLA simulator** | Train the fusion model | Builds hard layouts (parallel roads, exit ramps) with perfect ground truth |
| **Mapillary (France)** | Main test set | Crowdsourced street-level image sequences with GPS, heading and traffic-sign detections |
| **OpenStreetMap** | Map prior | `maxspeed` tags |
| **Passenger recordings** | Real-video + audio test set | Phone on the dashboard, limits called out by voice as ground truth |
| **GTSRB / Mapillary Traffic Sign Dataset** | Sign classifier fine-tuning | European sign styles |

**Rules:**
- Test routes are **held out entirely** — the same road on another day counts as leakage
- Faces and plates are **blurred** in anything published
- Mapillary imagery is attributed as required; OSM data is © OpenStreetMap contributors

---

## 📏 Evaluation

| Metric | Why |
|---|---|
| **False alerts per 100 km** (headline) | The number drivers actually feel |
| Missed limit changes | Safety side |
| Seconds to correct limit after a sign | Responsiveness |
| % of distance with correct limit shown | Overall accuracy |
| Latency per decision · memory | Can it run in a car? |

**Hard-case suite** — results reported per scenario:
exit ramps · parallel roads · roadworks · variable digital signs · speed stickers on trucks ·
rain · tunnels (GPS loss) · night

---

## ✅ Current Status — v0.0.1 (Phase 0: Design)

| Component | Status | Description |
|-----------|--------|-------------|
| Problem framing | ✅ | ISA distrust, map vs camera failure modes |
| Benchmark design | ✅ | Case 1 vs Case 2, baselines, metrics |
| Data strategy | ✅ | CARLA · Mapillary · OSM · passenger recordings |
| Repository scaffold | 🔄 | Structure, environment, CI |
| Mapillary France loader | ⬜ | Sequences + detections + OSM join |
| Labelling tool | ⬜ | Applicable-limit timeline per sequence |
| Baselines | ⬜ | Map-only, camera-only |
| Case 1 / Case 2 | ⬜ | Phases 2–3 |

---

## 📚 A Documented Learning Journey

| Phase | Concepts Introduced |
|-------|-------------------|
| Phase 1 | Data pipelines, geospatial joins (GPS ↔ OSM), annotation, dataset splits and leakage |
| Phase 2 | CNNs, transfer learning, fine-tuning, initialisation, optimisers, regularisation |
| Phase 3 | Sequence models, sensor fusion, audio classification on spectrograms |
| Phase 4 | Vision-language models, prompting, hallucination measurement |
| Phase 5 | Quantisation, ONNX export, edge inference, latency profiling |

Every concept is backed by a curated reference in [`docs/learning-references.md`](docs/learning-references.md).
Errors and fixes are logged in [`docs/troubleshooting.md`](docs/troubleshooting.md).

---

## 🗺️ Roadmap

### Phase 0 — Design ✅ · `v0.0.1`
Problem framing → benchmark design → data strategy

### Phase 1 — Data ⬜ · `v0.1.0`
Mapillary France loader → OSM map prior → labelling tool → first hard-case test set

### Phase 2 — Baselines + Perception ⬜ · `v0.2.0`
Map-only and camera-only baselines → fine-tuned sign detector → first results table

### Phase 3 — Case 1: Specialist Pipeline ⬜ · `v0.3.0`
CARLA scenarios → fusion sequence model → audio rain detection → per-scenario evaluation

### Phase 4 — Case 2: Multimodal ⬜ · `v0.4.0`
Small vision-language model → same test set → hallucination and latency analysis

### Phase 5 — Edge + Demo ⬜ · `v1.0.0`
int8 ONNX → phone / Raspberry Pi benchmarks → side-by-side replay demo → write-up

> See [`docs/roadmap.md`](docs/roadmap.md) for the task-level breakdown.

---

## 🚀 Quick Start

> Right Limit is in the design phase. These steps set up the development environment.

### 1 — Clone the repository

```bash
git clone https://github.com/rdmouhouadi/RightLimit.git
cd RightLimit
```

### 2 — Create the environment

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3 — Configure data access

```bash
cp .env.example .env
```

Add your Mapillary access token to `.env` (`MAPILLARY_TOKEN=...`).

---

## 🗂️ Project Structure

```sh
RightLimit/
├── rightlimit/
│   ├── data/                 # Mapillary, OSM, CARLA and recording loaders
│   ├── labelling/            # Applicable-limit timeline annotation
│   ├── perception/           # Sign detection, audio rain detection
│   ├── fusion/               # Sequence fusion model (Case 1 core)
│   ├── multimodal/           # Vision-language model runner (Case 2)
│   ├── baselines/            # Map-only, camera-only
│   ├── evaluation/           # Metrics, hard-case suite, reports
│   └── export/               # ONNX / int8 quantisation
├── scenarios/                # CARLA scenario definitions
├── notebooks/                # Experiments
├── tests/
├── docs/
│   ├── architecture.md
│   ├── roadmap.md
│   ├── troubleshooting.md
│   └── learning-references.md
├── CHANGELOG.md
├── LICENSE
└── .env.example
```

---

## 🧪 Running Tests

```bash
pip install pytest
pytest -v
```

---

## 🙏 Data & Acknowledgements

- **Mapillary** — street-level imagery and traffic-sign detections
- **OpenStreetMap** — map data © OpenStreetMap contributors (ODbL)
- **CARLA** — open-source driving simulator
- **GTSRB** — German Traffic Sign Recognition Benchmark

Built as part of the DSTI *Artificial Neural Networks* and *Deep Learning* courses.

---

## 📄 License

This project is licensed under the **GNU Affero General Public License v3.0 (AGPL-3.0)**.

Datasets keep their own licenses (Mapillary, OSM/ODbL, GTSRB, CARLA) — check each before redistributing data.

See the [LICENSE](LICENSE) file for details.

---

## 🤝 Contributing

Contributions are welcome. Please open an issue before submitting a pull request
so we can discuss the proposed change.

---

<div align="center">
<sub>Built with Python · PyTorch · ONNX · CARLA · Mapillary · OpenStreetMap</sub>
</div>
