# HANDOVER — Stealth-Crawler (Übergabe an Repo `noerix-admin/stealthbrowser`)

Stand: 2026-07-06. Dieses Dokument bündelt Ziel, Architektur, aktuellen Stand,
Bauplan und Grenzen des Projekts, damit es in `stealthbrowser` (bzw. von Hermes /
einer frischen Session) übernommen und fertiggestellt werden kann.

---

## 1. Produktziel (verbindlicher Scope)

Ein **Crawler, der öffentliche oder eigene, autorisierte Inhalte zuverlässig abruft,
ohne fälschlich als Bot geblockt zu werden**, und sauberes Markdown liefert.
Orchestrierung: **crawl4ai** (library-first) + gehärteter Browser via CDP.

**Zweck des „Stealth":** *nicht fälschlich als Automatisierung abgewiesen werden* beim
Lesen von Daten, auf die man zugreifen darf. Kein Umgehen fremder Zugriffskontrollen.

### Ausdrücklich NICHT Teil dieses Produkts (bewusst ausgeschlossen)
- Aushebeln von **Anti-Fraud-/Betrugserkennung**, um Accounts verdeckt zu betreiben.
- Speichern/Autofill von **2FA-Secrets fremder Zielaccounts**.
- **Multi-Identitäts-Impersonation** / „verdeckt ermitteln".

Begründung: Das sind Umgehung von Authentifizierungs-/Sicherheitskontrollen +
Detection-Evasion. Diese Themen werden **nur defensiv** behandelt — als Erkennung,
Härtung und Schulung (siehe `schulung_blue_purple_red.md` und Repo `skill_secugate`),
**nicht** als lauffähiges Werkzeug.

## 2. Aktueller Stand — was LÄUFT (auf der NAS, produktiv)

- **Crawl-Pipeline** (crawl4ai-Container, Egress über Residential-IP der NAS):
  crawlt Doctolib-/Charly-Doku, bereinigt Boilerplate, erzeugt mehrsprachige
  Embeddings (**fastembed**, `paraphrase-multilingual-MiniLM-L12-v2`, 384 Dim),
  schreibt in **pgvector** (`pgvector/pgvector:pg16`, DB `knowledge`, Tabelle
  `documents`, Netz `knowledge-net`, `127.0.0.1:5432`). Verifiziert: 36 Chunks,
  semantische Suche liefert korrekte Top-Treffer.
- **MCP-Server `kb-mcp`** (FastMCP, Streamable-HTTP `:8000/mcp`) bietet Tool
  `search_knowledge`; beim NAS-Hermes registriert + aktiviert.
- **Tooling** liegt in `noerix-admin/stealthbrowser` (kb_search / kb_ingest).

Damit ist **Stealth-Stufe E0/E1** (normaler Chromium + Residential-IP) produktiv —
ausreichend für weiche/öffentliche Ziele. **Fingerprint-Härtung (E2+) fehlt noch.**

## 3. Zielarchitektur

```
Agent/Trigger → crawl4ai (cdp_url=…)  ── dockt an, launcht nicht ──▶ gehärteter Browser
                                                                     │  Residential-Egress
                                                                     ▼
                                                        HTML → Markdown → pgvector
```
- Verifiziert im crawl4ai-Quellcode: Setzen von `cdp_url` ⇒ `connect_over_cdp` (Attach).
- **Nicht doppeln:** Übernimmt der externe Browser die Härtung, crawl4ais
  `enable_stealth` NICHT zusätzlich aktivieren (Patch-Kollision).

## 4. Bauplan — Stealth-Eskalationsleiter (für resilientes Scrapen)

| Stufe | Maßnahme | Werkzeuge | Status |
|---|---|---|---|
| E0 Baseline | crawl4ai + echter Chromium, headless | crawl4ai | ✅ läuft |
| E1 Netzwerk | Residential-Egress statt Datacenter | NAS-Heim-IP / Residential-Proxy | ✅ (NAS-IP) |
| E2 Fingerprint | Automations-Leaks/Canvas/WebGL/JA3 entschärfen | **patchright** oder **Camoufox** | ⬜ offen |
| E3 Verhalten | echte Header, TLS-Impersonation, Rate-Limit, **eigene** Session-Persistenz | `curl_cffi`, Profil-/Cookie-Reuse | ⬜ offen |
| E4 Managed | Managed-Scraping-Browser (nur wenn nötig) | Browserless / Brightdata Scraping Browser | ⬜ optional |

**Default-Empfehlung:** E2 mit **Camoufox _oder_ patchright** + optional Residential-Proxy,
via `cdp_url` an crawl4ai. „So viel Stealth wie nötig" — Stufe an Zielhärte anpassen.

## 5. Konkrete nächste Aufgaben (für die Übernahme)

1. **Zielklasse festlegen** → bestimmt, ob E2 überhaupt nötig ist.
2. **E2 umsetzen:** Camoufox- oder patchright-Container aufsetzen, mit CDP-Port,
   `cdp_url` in crawl4ai eintragen; gegen ein Testziel verifizieren.
3. **Session-Persistenz** für *eigene* Logins (Profilverzeichnis, einmaliger Login).
4. **Sauberkeit:** `kb_ingest`/`kb_search` von `docker build` (scheitert auf QNAP)
   auf „ohne Build" umstellen (Deps beim Container-Start via pip).
5. **`SECTION_URL`** parametrisieren (mehrere Quellen).

## 6. Umgebung / Fakten

- **NAS:** Tailscale `100.114.84.17`, SSH-User `ssc-home`, Docker unter
  `/share/ZFS530_DATA/.qpkg/container-station/bin/docker`, Projektpfad
  `/share/homes/ssc-home/crawl-pipeline`. Keine GPU → NAS-Hermes nutzt Cloud-Modelle.
- **Hermes:** Container `hermes-central`, Dashboard `http://100.114.84.17:9119`,
  MCP via `hermes mcp add <name> --url …`. Desktop-App auf der V4 ist eine *separate*
  Instanz (nicht am NAS-Agenten).
- **Lokales Qwen:** nur auf GPU-Rechnern (5080/3090), nicht mit NAS-Hermes verbunden.
- **Windows-Fallstricke:** Download stripped Bindestriche → **Unterstriche** nutzen;
  `docker build` aus dem NAS-Home scheitert (Permission) → „ohne Build".

## 7. Housekeeping / Sicherheit

- **API-Schlüssel rotieren:** Anthropic, OpenRouter, Hermes `API_SERVER_KEY` lagen im
  Ursprungs-Chat im Klartext. Rotieren.
- **DSGVO/§203:** NAS-Hermes nutzt Cloud-Modelle — bei Patienten-/Kundendaten kritisch;
  lokales Modell anbinden, wenn PII verarbeitet wird.
- **pgvector** nur auf `127.0.0.1`/`knowledge-net`, nicht im LAN exponiert.

---

*Defensive/Schulungs-Seite dieses Projekts: `docs/stealthbrowser/schulung_blue_purple_red.md`
(Blue→Purple→Red-Systematik für KMU-Sysadmins) und Repo `noerix-admin/skill_secugate`.*
