# Gas Connect DIY

[Deutsch](#deutsch) | [English](#english)

## Deutsch

ESPHome-Firmware für den Bausatz **Gas Connect DIY** von [NG3D-Druck](https://ng3d-druck.de): einen Impulssensor für
Gaszähler mit Reedkontakt und Waveshare ESP32-C3-Zero zum Selbstaufbau, passend z. B. für Elster/Honeywell BK-G2,5–G25.

### Hardware

| Pin    | Funktion                          |
| ------ | --------------------------------- |
| GPIO4  | Reedkontakt (gegen GND, Pull-up)  |
| GPIO10 | WS2812-RGB-LED auf der Platine    |

Den Reedkontakt an GPIO4 und GND des ESP32-C3-Zero anlöten, die Polung ist egal.

Jede Umdrehung der letzten Rollenziffer erzeugt einen Impuls. Wie viel Gas das ist, steht auf dem Typenschild des Zählers
(bei den meisten Gaszählern 0,01 m³) und wird über **Pulse volume** eingestellt.

### Entitäten

- **Gas meter reading** (Zählerstand, m³, `total_increasing`, Geräteklasse `gas`) - für das Energie-Dashboard von Home Assistant
- **Gas flow** (Durchfluss, m³/h)
- **Gas today**, **Gas this month**, **Gas this year** (m³) - Verbrauch seit Mitternacht (Ortszeit) / dem 1. des Monats / dem 1. Januar
- **Gas yesterday** (m³) - Verbrauch des Vortags (unbekannt, wenn das Gerät einen ganzen Tag aus war)
- **Set meter reading** - einmal den Stand vom Zähler eintragen, er bleibt über Neustarts erhalten
- **Pulse volume** - Volumen pro Impuls (0,001 / 0,01 / 0,1 / 1 m³, Standard 0,01 m³); beim Umstellen bleiben Zählerstand und Verbrauch erhalten
- **Debounce time** - Entprellzeit des Reedkontakts (10-1000 ms, Standard 100 ms); erhöhen, wenn ein Impuls doppelt gezählt wird,
  verringern, wenn bei hohem Durchfluss Impulse fehlen
- **Reed contact** (Diagnose) - aktueller Zustand des Kontakts, hilfreich beim Ausrichten des Sensors
- **LED mode** - was die LED anzeigt: **Status + pulse** (Standard), **Status only** oder **Off**
- **Reset statistics** - setzt heute, Monat und Jahr auf 0 und gestern auf unbekannt, z. B. nach dem Test mit einem Magneten; der Zählerstand bleibt. Sicherheitsabfrage: Der erste Druck fragt nach (**Reset statistics status**: „Are you sure? …“), erst ein zweiter Druck innerhalb von 10 s setzt zurück
- **Restart**, **WiFi signal**, **Uptime**
- **Firmware**-Update (nur in der Werks-Firmware, entfällt nach der Übernahme im ESPHome Device Builder)

### LED

| LED                    | Bedeutung                                         |
| ---------------------- | ------------------------------------------------- |
| blau, blinkend         | kein WLAN (Hotspot bzw. Improv aktiv)             |
| gelb, blinkend         | WLAN verbunden, aber keine Verbindung zu Home Assistant (MQTT-Version: zum Broker) |
| kurz grün              | Impuls erkannt (nur bei **Status + pulse**)       |
| aus                    | alles in Ordnung                                  |

### Einrichtung

1. Das Gerät über USB-C mit Strom versorgen.
2. Mit dem WLAN verbinden: per Improv über Bluetooth (Home Assistant / ESPHome-App),
   per Improv über USB im [Web-Installer](https://ng3d-druck.github.io/Gas-Connect-DIY/) oder über den Fallback-Hotspot.
3. In Home Assistant übernehmen (es erscheint als `gas-connect-diy-xxxxxx`, die Endung sind die letzten Stellen der MAC-Adresse)
   und bei **Set meter reading** den aktuellen Stand des Zählers eintragen.
   Falls dein Zähler nicht 0,01 m³ pro Umdrehung hat, vorher **Pulse volume** anpassen.

**Einrichtung innerhalb von 15 Minuten:** Nach dem Einstecken ist das Gerät 15 Minuten lang einrichtbar. Danach
schließen sich Hotspot und Bluetooth-Einrichtung. Dann das Gerät kurz ab- und wieder anstecken.

**Verschlüsselung:** Die Verbindung zu Home Assistant ist verschlüsselt. Home Assistant richtet beim Hinzufügen
automatisch einen eigenen Schlüssel für das Gerät ein, du musst nichts eingeben. Danach nimmt das Gerät nur noch
verschlüsselte Verbindungen an. Wird das Gerät beim Neuinstallieren über den Web-Installer gelöscht, ist auch der
Schlüssel weg: Dann das Gerät in Home Assistant entfernen und neu hinzufügen.

**War das Gerät schon einmal in Home Assistant eingebunden?** Dann den alten Eintrag unter
*Einstellungen → Geräte & Dienste → ESPHome* löschen und Home Assistant neu starten. Sonst fragt Home Assistant beim
Hinzufügen nach einem Verschlüsselungsschlüssel.

Die Zeiträume wechseln um Mitternacht Ortszeit. Die Zeitzone übernimmt das Gerät automatisch aus den Einstellungen von
Home Assistant (ab Version 2026.3). Bis Home Assistant verbunden ist, oder bei älteren Versionen, gilt `Europe/Berlin`
(Substitution `timezone`).

### Energie und Kosten (optional)

Das Paket `gas-costs.yaml` ergänzt Energie, Kosten und das Abrechnungsjahr des Gasversorgers. Es kommt neben die
unveränderte `gas-connect.yaml` in die eigene Konfiguration, z. B. nach der Übernahme im ESPHome Device Builder:

```yaml
packages:
  core: !include gas-connect.yaml
  costs: !include gas-costs.yaml
```

Einstellungen (in Home Assistant unter **Konfiguration**, die Werte stehen auf der Gasrechnung), in dieser Reihenfolge eintragen:

- **Calorific value** (Brennwert, kWh/m³) und **Z-number** (Zustandszahl): kWh = m³ × Brennwert × Zustandszahl
- **Energy price** (Arbeitspreis, brutto, ct/kWh) - solange 0, bleiben die Kosten unbekannt
- **Base price** (Grundpreis, brutto, €/Jahr) und **Monthly installment** (monatlicher Abschlag, €)
- **Billing year start day** / **Billing year start month** - Beginn des Abrechnungsjahres, z. B. 1 und 10 für den 1. Oktober
- **Billing year start reading** - Zählerstand zu Beginn des Abrechnungsjahres (von der letzten Rechnung)

Neue Entitäten:

- **Gas energy** (kWh, `total_increasing`) - für das Energie-Dashboard, dazu **Gas energy today**, **yesterday**, **this week**, **this month**, **this year**
- **Gas power** (kW) sowie **Gas this week**, **Gas last week**, **Gas last month**, **Gas last year**, **Gas billing year** (m³)
- **Gas cost** (€): today, yesterday, this / last week, this / last month, this / last year und billing year; der Grundpreis ist für jeden begonnenen Tag enthalten
- **Installments paid** - bisher fällige Abschläge im Abrechnungsjahr
- **Billing balance** - Abschläge minus Kosten bisher; positiv heißt Guthaben, negativ Nachzahlung
- **Gas cost forecast this month**, **Gas cost forecast billing year**, **Billing balance forecast** - hochgerechnet mit der
  üblichen Verteilung über das Jahr (Gradtagzahlen nach VDI 2067, dazu 15 % gleichmäßig für Warmwasser), ein Wintertag zählt also mehr als ein Sommertag
- **Recalculate costs** - rechnet die laufenden Zeiträume mit den aktuellen Einstellungen neu, z. B. nach einem Tippfehler beim Preis

Jeder Impuls wird mit den Einstellungen umgerechnet, die in dem Moment gelten: Ein neuer Preis gilt wie auf der Gasrechnung
erst ab dann. Der erste Arbeitspreis gilt auch für den Verbrauch davor. **Reset statistics** setzt auch diese Woche,
Monat und Jahr auf 0, das Abrechnungsjahr bleibt. Die Rechnung in `gas-costs.h` lässt sich am PC testen:
`test/run-tests.ps1` (Visual Studio mit C++).

### MQTT-Version (ohne Home Assistant)

Für openHAB, ioBroker, Node-RED, FHEM und andere Systeme mit MQTT-Broker, installierbar über den
[Web-Installer](https://ng3d-druck.github.io/Gas-Connect-DIY/). Gleiche Entitäten wie oben, dazu:

| Entität                | Art    | Beschreibung                                               |
| ---------------------- | ------ | ---------------------------------------------------------- |
| MQTT broker            | Text   | Hostname oder IP-Adresse des Brokers                       |
| MQTT port              | Zahl   | Port des Brokers, Standard 1883                            |
| MQTT username          | Text   | Benutzername am Broker                                     |
| MQTT password          | Text   | Passwort am Broker, wird nie angezeigt oder per MQTT gesendet |
| Hostname               | Text   | Name im Router, des Hotspots und der `.local`-Adresse; leer = Standardname |
| Web page username      | Text   | Benutzername für die Webseite; leer = `admin`              |
| Web page password      | Text   | schützt die Webseite (Digest), mind. 8 Zeichen; Pflicht, ohne Passwort keine MQTT-Verbindung |
| Save and restart       | Button | speichert die Einstellungen und startet neu                |
| MQTT status            | Text-Sensor | Verbindung zum Broker bzw. Grund, warum sie fehlschlägt |

Die Einstellungen trägt man auf der Webseite des Geräts ein (`http://gas-connect-diy-mqtt-xxxxxx.local`). Sie lassen
sich nur in den ersten 15 Minuten nach dem Einstecken ändern (**Save and restart** lässt das Fenster offen, andere Neustarts öffnen es nicht); danach das Gerät
kurz ab- und wieder anstecken. Die MQTT-Topics behalten auch mit eigenem Hostnamen den Standardnamen. Das Gerät
meldet sich per MQTT-Discovery (Präfix `homeassistant`), die Topics beginnen mit `gas-connect-diy-mqtt-xxxxxx/`.
Firmware per WLAN (ESPHome-OTA, Port 3232) nimmt das Gerät nur mit dem Webseiten-Passwort an. Gelb blinkt die LED hier, solange keine Verbindung zum Broker besteht. Die Uhrzeit kommt per SNTP, die Zeitzone ist
`Europe/Berlin`.

### Dateien

- `gas-connect.yaml` - die Konfiguration, die man bei der Übernahme im ESPHome Device Builder erhält
- `gas-connect.factory.yaml` - die Werks-Firmware (ergänzt Improv, Projektinfos und OTA-Updates)
- `gas-connect-mqtt.yaml`, `gas-connect-mqtt.factory.yaml` - dasselbe für die MQTT-Version
- `gas-costs.yaml`, `gas-costs.h` - optionales Paket für Energie, Kosten und Abrechnungsjahr, Tests in `test/`

## English

ESPHome firmware for the **Gas Connect DIY** kit by [NG3D-Druck](https://ng3d-druck.de): a pulse sensor for gas meters
with a reed contact and a Waveshare ESP32-C3-Zero to build yourself, which fits e.g. Elster/Honeywell BK-G2.5–G25.

### Hardware

| Pin    | Function                          |
| ------ | --------------------------------- |
| GPIO4  | Reed contact (to GND, pull-up)    |
| GPIO10 | Onboard WS2812 RGB LED            |

Solder the reed contact to GPIO4 and GND of the ESP32-C3-Zero, the polarity does not matter.

Each revolution of the last drum digit creates one pulse. How much gas that is is printed on the meter
(0.01 m³ for most gas meters) and is set via **Pulse volume**.

### Entities

- **Gas meter reading** (m³, `total_increasing`, device class `gas`) - for the Home Assistant energy dashboard
- **Gas flow** (m³/h)
- **Gas today**, **Gas this month**, **Gas this year** (m³) - consumption since local midnight / the 1st of the month / January 1st
- **Gas yesterday** (m³) - consumption of the previous day (unknown if the device was off for a whole day)
- **Set meter reading** - enter the value shown on your meter once, it is persisted across reboots
- **Pulse volume** - volume per pulse (0.001 / 0.01 / 0.1 / 1 m³, default 0.01 m³); changing it keeps the meter reading and consumption
- **Debounce time** - debounce time of the reed contact (10-1000 ms, default 100 ms); raise it if a pulse is counted twice,
  lower it if pulses are missed at high flow
- **Reed contact** (diagnostic) - live contact state, useful when aligning the sensor
- **LED mode** - what the LED shows: **Status + pulse** (default), **Status only** or **Off**
- **Reset statistics** - sets today, this month and this year to 0 and yesterday to unknown, e.g. after testing with a magnet; the meter reading stays. Confirmation: the first press asks (**Reset statistics status**: "Are you sure? …"), only a second press within 10 s resets
- **Restart**, **WiFi signal**, **Uptime**
- **Firmware** update (factory firmware only, gone after taking control in the ESPHome Device Builder)

### LED

| LED                    | Meaning                                           |
| ---------------------- | ------------------------------------------------- |
| blue, blinking         | no Wi-Fi (hotspot or Improv active)               |
| yellow, blinking       | Wi-Fi connected, but no Home Assistant connection (MQTT version: to the broker) |
| short green flash      | pulse detected (only with **Status + pulse**)     |
| off                    | all fine                                          |

### Setup

1. Power the device via USB-C.
2. Connect it to your Wi-Fi with Improv via Bluetooth (Home Assistant / ESPHome app),
   Improv via USB at the [web installer](https://ng3d-druck.github.io/Gas-Connect-DIY/), or the fallback hotspot.
3. Adopt it in Home Assistant (it shows up as `gas-connect-diy-xxxxxx`, the suffix being the end of its MAC address)
   and set **Set meter reading** to the current value of your meter.
   If your meter does not count 0.01 m³ per revolution, set **Pulse volume** first.

**Set up within 15 minutes:** After plugging it in, the device can be set up for 15 minutes. Then the hotspot and
the Bluetooth setup close. Unplug the device briefly and plug it back in to set it up.

**Encryption:** The connection to Home Assistant is encrypted. Home Assistant sets up a key of its own for the device
when adding it, so there is nothing to enter. From then on the device only accepts encrypted connections. If the device
is erased while reinstalling it with the web installer, the key is gone too: remove the device from Home Assistant and
add it again.

**Was the device added to Home Assistant before?** Then delete the old entry under
*Settings → Devices & services → ESPHome* and restart Home Assistant. Otherwise Home Assistant asks for an encryption
key when adding it.

The periods roll over at local midnight. The device takes the timezone from the Home Assistant settings automatically
(Home Assistant 2026.3 and newer). Until Home Assistant is connected, or with older versions, `Europe/Berlin` applies
(`timezone` substitution).

### Energy and costs (optional)

The package `gas-costs.yaml` adds energy, costs and the billing year of the gas supplier. Include it next to the
unchanged `gas-connect.yaml` in your own config, e.g. after taking control in the ESPHome Device Builder:

```yaml
packages:
  core: !include gas-connect.yaml
  costs: !include gas-costs.yaml
```

Settings (in Home Assistant under **Configuration**, the values are on your gas bill), enter them in this order:

- **Calorific value** (kWh/m³) and **Z-number**: kWh = m³ × calorific value × z-number
- **Energy price** (gross, ct/kWh) - as long as it is 0, the costs stay unknown
- **Base price** (gross, EUR/year) and **Monthly installment** (EUR)
- **Billing year start day** / **Billing year start month** - start of the billing year, e.g. 1 and 10 for October 1st
- **Billing year start reading** - meter reading at the start of the billing year (from the last bill)

New entities:

- **Gas energy** (kWh, `total_increasing`) - for the energy dashboard, plus **Gas energy today**, **yesterday**, **this week**, **this month**, **this year**
- **Gas power** (kW) and **Gas this week**, **Gas last week**, **Gas last month**, **Gas last year**, **Gas billing year** (m³)
- **Gas cost** (EUR): today, yesterday, this / last week, this / last month, this / last year and billing year; the base price is included for each day begun
- **Installments paid** - installments due so far in the billing year
- **Billing balance** - installments minus costs so far; positive means credit, negative an additional payment
- **Gas cost forecast this month**, **Gas cost forecast billing year**, **Billing balance forecast** - projected with the
  usual distribution over the year (degree days per VDI 2067 plus 15 % evenly for hot water), so a winter day counts more than a summer day
- **Recalculate costs** - recalculates the running periods with the current settings, e.g. after a typo in the price

Each pulse is converted with the settings valid at that moment: like on the gas bill, a new price only applies from then
on. The first energy price also applies to the consumption before it. **Reset statistics** also resets this week,
month and year to 0, the billing year stays. The calculations in `gas-costs.h` can be tested on the PC:
`test/run-tests.ps1` (Visual Studio with C++).

### MQTT version (without Home Assistant)

For openHAB, ioBroker, Node-RED, FHEM and other systems with an MQTT broker, installable from the
[web installer](https://ng3d-druck.github.io/Gas-Connect-DIY/). Same entities as above, plus:

| Entity                 | Type   | Description                                                |
| ---------------------- | ------ | ---------------------------------------------------------- |
| MQTT broker            | text   | host name or IP address of the broker                      |
| MQTT port              | number | port of the broker, default 1883                           |
| MQTT username          | text   | user name on the broker                                    |
| MQTT password          | text   | password on the broker, never shown or sent via MQTT       |
| Hostname               | text   | name in the router, of the hotspot and the `.local` address; empty = default name |
| Web page username      | text   | user name for the web page; empty = `admin`                |
| Web page password      | text   | protects the web page (digest), at least 8 characters; required, no MQTT connection without it |
| Save and restart       | button | saves the settings and restarts                            |
| MQTT status            | text sensor | connection to the broker, or why it fails             |

Enter the settings on the web page of the device (`http://gas-connect-diy-mqtt-xxxxxx.local`). They can only be changed
within 15 minutes after plugging in (**Save and restart** keeps the window open, other restarts do not open it); after that, unplug the device briefly and plug it
back in. The MQTT topics keep the default name, also with a host name of your own. The device announces
itself via MQTT discovery (prefix `homeassistant`), its topics start with `gas-connect-diy-mqtt-xxxxxx/`. Firmware over Wi-Fi
(ESPHome OTA, port 3232) is only accepted with the web page password. Here the LED
blinks yellow while there is no connection to the broker. The time comes from SNTP, the timezone is `Europe/Berlin`.

### Files

- `gas-connect.yaml` - the configuration you get when taking control in the ESPHome Device Builder
- `gas-connect.factory.yaml` - the factory firmware (adds Improv, project info and OTA updates)
- `gas-connect-mqtt.yaml`, `gas-connect-mqtt.factory.yaml` - the same for the MQTT version
- `gas-costs.yaml`, `gas-costs.h` - optional package for energy, costs and the billing year, tests in `test/`
