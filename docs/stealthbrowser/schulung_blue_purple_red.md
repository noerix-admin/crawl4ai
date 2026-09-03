# Schulungsreihe: KMU-Sysadmins gegen KI-getriebene Bedrohungen
### Systematik Blue → Purple → Red (Awareness-Level, nicht-operativ)

> **Didaktisches Prinzip:** Red-Techniken werden **theoretisch/konzeptuell** beschrieben
> (was, warum, welche Spuren) — **kein** lauffähiger Angriffscode, keine Waffen-Tools.
> Der Lernwert liegt in **Erkennen (Blue)** und der **Lernschleife (Purple)**.
> Praktische Übungen laufen ausschließlich **autorisiert, gescoped, auf eigenen Systemen**
> mit etablierten Emulations-Frameworks (MITRE ATT&CK, Atomic Red Team, Caldera).

---

## 0. Aufbau jedes Moduls (wiederverwendbare Vorlage)

1. **Lernziele** — was der Admin danach erkennen/tun kann.
2. **Bedrohung im Klartext** — 2–3 Sätze, was der Angreifer erreichen will.
3. **RED (theoretisch)** — die Technik konzeptuell: *Mechanismus* + *warum sie wirkt* +
   *welche Ebenen sie manipuliert*. Keine Schritt-für-Schritt-Anleitung.
4. **Spuren & Indikatoren** — welche Signale die Technik hinterlässt.
5. **BLUE — Erkennung** — konkrete Detektionen/Telemetrie.
6. **BLUE — Härtung** — präventive Controls.
7. **PURPLE — Lernschleife** — wie Blue die Detektion validiert & verbessert
   (Emulation → Messen → Nachschärfen).
8. **Hands-on (safe lab)** — autorisierte Übung mit Standard-Tooling.
9. **Merksätze / Checkliste** — für den Admin-Alltag.

---

## Modul 1 — KI-Agent + Anti-Detect-Browser: Account-Übernahme & verdeckte Automatisierung

### 1. Lernziele
Der Admin kann erklären, **wie** automatisierte Agenten Bot-/Betrugserkennung zu umgehen
versuchen, **welche Spuren** das hinterlässt, und **welche Controls** es wirksam stoppen.

### 2. Bedrohung im Klartext
Ein KI-Agent steuert einen getarnten Browser, um sich als legitimer Nutzer auszugeben,
Konten automatisiert zu bedienen und dabei für die Plattform „menschlich" auszusehen.

### 3. RED (theoretisch — drei Manipulationsebenen)
- **Fingerprint:** Der Browser fälscht Merkmale (Canvas/WebGL/Audio, `navigator.webdriver`,
  Fonts, UA-Client-Hints), damit Automatisierung nicht auffällt. *Warum es wirkt:* viele
  Erkennungen prüfen nur einzelne Merkmale statt deren Konsistenz.
- **Netzwerk:** Ausgehender Verkehr über Residential-/Mobile-IPs + angepasster
  TLS/JA3-Fingerprint. *Warum es wirkt:* IP-Reputation und TLS wirken „echt".
- **Verhalten:** Simuliertes menschliches Tippen/Timing (auch bei Code-Eingaben).
  *Warum es wirkt:* rein statische Regeln sehen kein „Roboter-Timing".

*(Bewusst ohne Bauanleitung — für die Schulung genügt das Wirkprinzip.)*

### 4. Spuren & Indikatoren
- Widersprüchliche Fingerprints (UA sagt „Chrome/Windows", TLS/JA3 oder WebGL-Vendor passen nicht).
- „Zu saubere"/atypisch konsistente Merkmale, fehlende Gerätehistorie.
- Impossible Travel / Velocity: ein Konto aus wechselnden Residential-IPs in kurzer Zeit.
- Session-/Cookie-Reuse über fremde Geräte; Header-Reihenfolge untypisch.
- Auffällig gleichmäßiges Eingabe-Timing, fehlende Idle-Phasen.

### 5. BLUE — Erkennung
- **Fingerprint-Konsistenz-Checks** serverseitig (UA ↔ JA3 ↔ Client-Hints korrelieren).
- **Auth-Anomalien**: Velocity-, Impossible-Travel-, neue-Geräte-Regeln auf den Login-Logs.
- **Session-Integrität**: Bindung an Gerät/IP-Bereich, Alarm bei Bruch.
- **Bot-Management** (z. B. Cloudflare/DataDome) aus *Verteidiger*-Sicht konfigurieren + Logs auswerten.

### 6. BLUE — Härtung
- **Phishing-resistente MFA (FIDO2/Passkeys) statt TOTP** — wichtigste Einzelmaßnahme:
  an den Origin gebunden, nicht kopier-/autofill-bar, entwertet „2FA-Autofill"-Angriffe.
- Kurze Session-Lebensdauer + Re-Auth bei Anomalie; Device-Binding.
- Least-Privilege für Automatisierungs-/Service-Konten; getrennte Konten für Automatisierung.
- Rate-/Velocity-Limits, Geo-/ASN-Policies wo sinnvoll.

### 7. PURPLE — Lernschleife
1. **Hypothese**: „Wir erkennen widersprüchliche Fingerprints / Impossible Travel."
2. **Emulation** (autorisiert, eigenes System): TTP mit ATT&CK-Mapping nachstellen
   (Atomic Red Team / Caldera) — z. B. Login aus wechselnden Netzen, geänderter UA.
3. **Messen**: Hat die Detektion ausgelöst? Zeit bis Alarm? False Positives?
4. **Nachschärfen**: Regel/Schwellen anpassen, erneut testen. Ergebnis dokumentieren.
5. **Wissenstransfer**: Runbook + Alarm-Playbook aktualisieren.

### 8. Hands-on (safe lab)
- Nur auf eigener Test-Instanz, klarer Scope, schriftliche Freigabe.
- Standard-Tooling zur **Emulation** (keine Eigenbau-Waffe): ATT&CK-Testfälle,
  Log-Generierung, dann in SIEM/Reverse-Proxy die Detektion prüfen.
- Ziel der Übung ist die **Detektion**, nicht der erfolgreiche Angriff.

### 9. Merksätze / Checkliste
- [ ] Passkeys/FIDO2 statt TOTP, wo möglich.
- [ ] Fingerprint-Konsistenz + Velocity-Regeln aktiv und getestet.
- [ ] Automatisierungs-Konten getrennt, least privilege, überwacht.
- [ ] Purple-Test dokumentiert (Detektion ausgelöst? Zeit? FP-Rate?).
- [ ] Alarm-Playbook aktuell.

---

## Weitere geplante Module (gleiche Vorlage)
- **Modul 2:** Prompt-Injection / Tool-Missbrauch bei KI-Agenten (→ passt zu `skill_secugate`).
- **Modul 3:** Datenexfiltration über Agenten-Tools; Secret-Zugriff erkennen.
- **Modul 4:** Credential-Stuffing / Session-Hijacking gegen KMU-Portale.
- **Modul 5:** DSGVO/§203-Sicht: KI + Patienten-/Kundendaten, Zonen-Konzept.

## Nutzungs-/Ethik-Hinweis für die Schulung
Dieses Material ist **defensiv & theoretisch**. Es enthält bewusst **keine** einsatzfähigen
Umgehungs-Tools oder gespeicherten Fremd-Credentials/2FA. Praktische Tests nur autorisiert,
gescoped, auf eigenen Systemen.
