# Tamagotcha — Custom Tamagotchi (Raspberry Pi Zero) Implementation Plan

> **For Hermes:** use `subagent-driven-development` to implement task-by-task. One issue per PR; **branch → PR → Sourcery AI review → merge**; never push to `main`.

**Goal:** Build a pocketable, battery-powered, V3-style virtual pet on an **original Raspberry Pi Zero v1.3** (BCM2835, single-core ARM11 @1 GHz, 512 MB, **no WiFi/BT**), written in **Python 3**.

**Proposed hardware baseline (NOT yet frozen — decided by ADR-0001 / issue #10).** Alternatives below stay open until #10 is accepted:
- Display (choose in #10): **Waveshare 2.13" e-Paper HAT** (250×122, SPI, partial 0.3 s / full 2 s) *or* ST7735 1.8" colour TFT.
- Power (choose in #10): **Pimoroni LiPo SHIM** (1.5 A boost **with a built-in 3.0 V cutoff** + 3.4 V low-battery flag) *or* **Adafruit PowerBoost 1000C** (**low-battery *signal* only — it does NOT cut the cell off; requires additional cell-protection/cutoff hardware**, see #25). Soft power-button on both.
- Cell: 18650 Li-ion or LiPo pouch, **2000–2500 mAh**.
- Time: **DS3231 RTC** (I²C) — required because the base Zero has no network.
- OS: **Raspberry Pi OS Lite, armhf (32-bit)** — mandatory (ARMv6).

**Architecture (hexagonal / ports & adapters):**

```
        domain/  (pure Python — no I/O, fully unit-tested; Clock lives in ports/)
        ├─ state.py      PetState, Stage, invariants
        ├─ tick.py       advance(state, seconds)  ← offline catch-up
        ├─ actions.py    feed/discipline/clean/sleep
        ├─ evolution.py  care-group → character
        ├─ mortality.py  death / generations / family history
        └─ shop.py, games/  content

        ports/   Protocol interfaces: Display | Input | Clock | Storage | Battery | Power | Sound
        adapters/  concrete: epd_display | st7735 | gpio_buttons | rtc_ds3231 |
                   ina219 | lipopi_power | buzzer | json_storage | dummy_* (tests)
        app.py   wiring + main loop     __main__.py
```

**Core design rules**
1. `domain/` never imports `gpiozero`, `RPi.GPIO`, `luma`, `waveshare`, `smbus2`, or does file/network I/O.
2. The renderer talks only to the `Display` **port**; the board picks the adapter.
3. Persistence is **atomic** (write temp → `fsync` the file → `os.replace` → **`fsync` the containing directory**) and **versioned** (schema migrations).
4. Time always comes from a `Clock` port; on boot, **elapsed-time catch-up** makes the pet feel alive.
5. Everything must run headless in CI using fakes (`DummyDisplay`, `FakeButtons`, `FakeClock`).

**Runtime budget (modelled, 85 % boost efficiency, on FULL nameplate capacity — de-rate ~75–85 % for the 3.0 V cutoff). NOT measured; to be validated by #38:**

| Battery | idle (82 mA) | app (~140 mA) | heavy (220 mA) |
|---|---|---|---|
| 1000 mAh | 7.7 h | 4.5 h | 2.9 h |
| 2000 mAh | 15.3 h | **9.0 h** | 5.7 h |
| 2500 mAh | 19.2 h | 11.2 h | 7.1 h |

---

## Roadmap — epics and issues

| Epic | Issue | Tasks |
|---|---|---|
| Hardware platform & enclosure | #1 | #10, #21, #22, #23, #24, #25, #36 |
| Simulation core & architecture | #2 | #7, #11, #12, #13, #14, #15, #16 |
| Rendering, UI & input UX | #3 | #17, #18, #19, #20, #26 |
| Device operations & power mgmt | #4 | #27, #28, #29, #30 |
| Gameplay content & characters | #5 | #31, #32, #33, #34, #35 |
| Quality, testing & CI | #6 | #8, #9, #37, #38 |

---

## Phase 0 — Scaffolding (do first)
- **#7** Bootstrap Python project (`uv`, `src/` layout, `pyproject.toml`).
- **#8** Dev tooling (pytest, coverage, ruff, mypy, pre-commit).
- **#9** CI + **Sourcery AI review gate** (repo requirement). Branch protection on `main`.
- **#10** ADR-0001 freezing display/battery/OS/RTC choices.

**Verify:** `uv sync`, `uv run pytest -q`, `uv run ruff check .`, `uv run mypy src` all green; CI red on a broken test.

## Phase 1 — Simulation core (pure, TDD)
- **#11** Domain model & lifecycle state machine.
- **#12** Tick engine + offline catch-up (injected `Clock`; deterministic, step-size independent).
- **#13** Care actions (V3 semantics: discipline only when called; snacks vs meals; overfeed → sickness).
- **#14** Evolution engine from V3 care-group tables (5 groups → 5 teens / 15 adults).
- **#15** Death, aging, generational reset, family history.
- **#16** Atomic, versioned persistence + boot catch-up.

**Verify:** full unit coverage on `domain/`; import-scan test proves `domain/` has no hardware deps.

## Phase 2 — UI (ports & adapters, testable headless)
- **#17** `Display` port + `DummyDisplay`.
- **#18** Sprite/palette pipeline (Pillow; deterministic packer).
- **#19** Renderer: HUD, screens, idle animations.
- **#20** Input state machine (A/B/C).

**Verify:** snapshot tests through `DummyDisplay`; navigation tests with synthetic button events.

## Phase 3 — Hardware integration
- **#21** GPIO buttons · **#22** buzzer (via the `Sound` port) · **#23** DS3231 RTC · **#24** battery monitor · **#25** soft power button + graceful low-battery shutdown.
- **#26** Real display bring-up on the target.

**Verify:** bench checklist per issue; `tamagotcha.selftest` (see #37).

## Phase 4 — Device ops
- **#27** Provisioning script (config.txt power tweaks, disable HDMI/LED, governor).
- **#28** Read-only rootfs + writable data partition.
- **#29** systemd autostart service.
- **#30** Boot-time optimisation & measurement.

**Verify:** boots straight into the pet; pull-power test leaves a consistent state; idle draw near the 63 mA class.

## Phase 5 — Gameplay content
- **#31** Get & Sprint · **#32** Bump/Flag/Heading/Memory · **#33** Shop & GP economy · **#34** (stretch) TamaTown codes · **#35** full V3 sprite set.

**Verify:** deterministic game-scoring tests; schema test that every growth-chart id has sprites.

## Phase 6 — Enclosure & polish
- **#36** 3D-print case + assembly guide (or retrofit a real shell).

## Phase 7 — QA
- **#37** Hardware-in-the-loop smoke tests · **#38** battery soak test & power profiling.

---

## Files likely to change / be created
```
pyproject.toml, uv.lock, .pre-commit-config.yaml, CONTRIBUTING.md
.github/workflows/ci.yml
src/tamagotcha/{__init__,__main__,app,selftest}.py
src/tamagotcha/domain/{state,tick,actions,evolution,mortality,shop,clock}.py
src/tamagotcha/domain/games/{get,sprint,bump,flag,heading,memory}.py
src/tamagotcha/ports/{display,input,clock,storage,battery,power,sound}.py
src/tamagotcha/adapters/{epd_display,st7735_display,gpio_buttons,rtc_ds3231,
                        ina219,lipopi_power,buzzer,json_storage,dummy_display}.py
src/tamagotcha/render/{screens,hud}.py
data/{v3_growth.yaml,items.yaml}
assets/sprites/**, tools/pack_sprites.py
deploy/{provision.sh,config.txt.append,tamagotcha.service}
docs/adr/0001-hardware-baseline.md, docs/{assets,hardware/*,ops/*}.md
tests/**
hardware/case/**.stl
```

## Testing & validation strategy
- **Unit (CI, every PR):** domain rules, tick/catch-up determinism, evolution tables, persistence atomicity, renderer snapshots, navigation event sequences.
- **Contract tests:** each port adapter tested against the port's Protocol with a fake.
- **Hardware-in-the-loop (on device, tagged/manual):** buttons, display partial/full refresh, buzzer, RTC power-cycle, battery read (#37).
- **Soak:** full discharge run to record real runtime vs the model (#38).
- **CI gate:** `ruff` + `mypy` + `pytest --cov` + **Sourcery** all required.

## Risks, tradeoffs, open questions
1. **Zero vs MCU.** A Linux SBC can't deep-sleep; ~8–9 h per charge and ~20–45 s boot are inherent. If runtime/instant-on dominate, an RP2040/ESP32 is superior — flagged, not chosen.
2. **No network on the base Zero** → RTC mandatory; no NTP, no remote features without a USB WiFi dongle (the single OTG port then needs a hub). Alternatively migrate to Zero W.
3. **ARMv6 package gaps** — some modern wheels lack armv6; stay on Raspberry Pi OS armhf packages.
4. **E-paper animation limits** — 0.3 s partial refresh ⇒ ~3 fps; ghosting needs periodic full refresh. Design animations slow/deliberate (matches original LCD charm).
5. **e-paper vs colour TFT** — battery vs liveliness; ADR-0001 (#10) decides.
6. **Cell capacity honesty** — LiPo pouches overstate mAh; prefer 18650 or reputable vendors. Low-battery cutoff at 3.0 V means ~75–85 % usable capacity.
7. **"V3.5" unverified** — no official Bandai V3.5 SKU found; assumed V3 (Feb 2006). Confirm the reference unit before finalising character tables (#14/#35).
8. **Power strategy undecided** — always-on (simplest, ~8 h) vs power-off-between-uses (needs RTC + boot optimisation + robust catch-up). Resolve in epic #4.

## Open questions for the owner
- Which exact unit are we cloning (confirm V3 vs "V3.5" variant)?
- e-paper (battery, authentic mono) vs colour TFT (livelier)?
- Always-on or power-off-between-uses?
- Do we want any connectivity (Zero W upgrade / USB dongle), or keep it offline?
