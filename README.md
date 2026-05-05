# IoTMasterChef - Bezdrátový teploměr do grilu
Jedná se o projekt do kurzu BPC-IOT na FEKT VUT.

Programové řešení pro bezdrátový teploměr měřící teplotu uvnitř masa a v prostoru grilu, s přenosem dat přes Wi-Fi prostřednictvím protokolu MQTT. Součástí řešení je i aplikace, která ukládá údaje z teploměrů do formátu .csv a upozorňuje na dosaženou teplotu pomocí zvukové notifikace.

## Funkce mikrokontroléru
- Měření teploty masa a vnitřního prostoru grilu v reálném čase pomocí mikrokontroléru
- Pravidelné odesílání naměřených hodnot na broker server v různých intervalech adaptujících se na aktuální nastavenou teplotu v aplikaci vzhledem k úspoře baterie

## Funkce desktop aplikace
- Převzetí naměřených hodnot ze serveru a jejich zobrazení v reálném čase na zařízení
- Možnost nastavení cílové teploty masa a grilu pomocí slideru
- Odesílání cílové teploty na broker server pro použití mikrokontroléru
- Zvuková notifikace při dosažení cílové teploty i výkyvu z cílové teploty
- Průběžné ukládání přijaté hodnoty teploty masa a grilu do souboru `data_gril_export.csv` v aktuálním pracovním adresáři. Každý řádek obsahuje čas měření, teplotu masa a teplotu v grilu.
- Indikace online/offline stavu připojení na server

---
### Struktura projektu

```
.
├── README.md                                  # tento soubor
├── thermometer.py                             # firmware pro Pico W (MicroPython)
├── app.py                                     # desktop dashboard (Python 3 + Tkinter)
├── ahtx0.py                                   # aht20 driver teploměru
├── simple.py                                  # umqtt klient
└── docs/
    └── technicka_cast_hw_uspora_iotmasterchef.docx   # technická dokumentace
```

---
## Použité technologie
### HW:
- 2 teplotní senzory AHT20
- Mikrokontrolér Raspberry Pico W

### Komunikace
- Wi-Fi
- protokol MQTT pro přenos informací
- JSON formát zpráv
  
### Firmware teploměru
- programovací jazyk MicroPython
- MQTT klient

### Desktop aplikace
- Python 3
- Tkinter
- Knihovna `paho-mqtt`
- CSV

### Nástroje
- Thonny IDE pro vývoj a nahrávání MicroPython firmwaru do zařízení  
- Visual Studio Code pro vývoj desktopové aplikace v Pythonu
- open-source MQTT broker **Eclipse Mosquitto**, který zajišťuje směrování zpráv mezi teploměrem a uživatelskou aplikací (https://mosquitto.org/).

## Závislosti
**Závislosti pro Pico W:**

- `umqtt.simple` (modul `simple.py` z umqtt) — MQTT klient
- `ahtx0.py` — driver pro AHT20

**Závislosti pro desktop aplikaci:**

```bash
pip install paho-mqtt
# tkinter je součástí standardní Python distribuce
# pro Linux zvuk: sudo apt install sox  (kvůli příkazu `play`)
```


## Instalace a spuštění

### 1) MQTT broker

spuštění Mosquitto na portu **11883**:

```bash
# Linux (Arch/Debian/Ubuntu)
mosquitto -p 11883 -v
```

> Pro produkční nasazení nakonfigurujte `mosquitto.conf` s autentizací a TLS.

### 2) Konfigurace firmwaru

V souboru `thermometer.py` upravte:

```python
WIFI_SSID = "..."          # název Wi-Fi sítě
WIFI_PWD  = "..."          # heslo Wi-Fi
BROKER_IP = "..."          # IP adresa notebooku s brokerem
BROKER_PORT = 11883
GROUP_NUM = "9"            # identifikátor skupiny v MQTT topiku
```

### 3) Nahrání firmwaru na Pico W

1. Nainstalujte na Pico W MicroPython firmware (`.uf2` z [micropython.org](https://micropython.org/download/RPI_PICO_W/)).
2. Pomocí [Thonny](https://thonny.org) nebo `mpremote` nahrajte:
   - `thermometer.py` jako `main.py`
   - `simple.py` (umqtt klient)
   - `ahtx0.py` (driver senzoru)

```bash
mpremote cp thermometer.py :main.py
mpremote cp simple.py :
mpremote cp ahtx0.py :
mpremote reset
```

### 4) Spuštění desktop aplikace app.py

**Windows**

Pro spuštění aplikace je nutné mít nainstalovaný Python 3 a balíček `paho-mqtt`.

Jednodušší varianta bez virtuálního prostředí:
```bash
py -m pip install --upgrade pip
py -m pip install paho-mqtt
python app.py
```

Doporučená varianta s virtuálním prostředím:
```bash
py -m venv .venv
.venv\Scripts\activate
py -m pip install --upgrade pip
py -m pip install paho-mqtt
python app.py
```


**Linux**

Nainstalujte Python 3, Tkinter a SoX:
   ```bash
   sudo apt update
   sudo apt install python3 python3-venv python3-tk sox
   ```
V kořenové složce projektu vytvořte a aktivujte virtuální prostředí:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```
Nainstalujte potřebný Python balíček:
   ```bash
   python3 -m pip install --upgrade pip
   python3 -m pip install paho-mqtt
   ```
Spusťte aplikaci:
   ```bash
   python3 app.py
   ```

V aplikaci nastavte slidery na požadovanou teplotu masa a grilu a klikněte na **ODESLAŤ NASTAVENIA**. Aplikace začne přijímat naměřená data a zobrazovat je v reálném čase.

### MQTT topiky a formát zpráv

| Topik                              | Směr            | Payload (JSON)                       |
|------------------------------------|-----------------|--------------------------------------|
| `IoTProject/9/grill/teplota`       | Pico -> app     | `{"maso": 62.4, "gril": 187.3}`      |
| `IoTProject/9/grill/Tmaso`         | app -> Pico     | `{"masoTarget": 75.0}`               |
| `IoTProject/9/grill/Tgrill`        | app -> Pico     | `{"grillTarget": 200.0}`             |

Hodnoty teploty jsou v °C, zaokrouhleny na jedno desetinné místo.




----






## Reálné provedení

### Architektura řešení

```
┌────────────────────────────┐                ┌────────────────────────────┐
│  Raspberry Pi Pico W       │                │  Notebook                  │
│  (umístění u grilu)        │                │                            │
│                            │                │  ┌──────────────────────┐  │
│  ┌──────────────────────┐  │                │  │  Mosquitto broker    │  │
│  │  AHT20 (maso)   I2C1 │──┤                │  │  (port 11883)        │  │
│  └──────────────────────┘  │   Wi-Fi        │  └──────────┬───────────┘  │
│  ┌──────────────────────┐  │   ──────►      │             │              │
│  │  AHT20 (gril)   I2C0 │──┤   MQTT         │  ┌──────────▼───────────┐  │
│  └──────────────────────┘  │   ◄──────      │  │  Tkinter dashboard   │  │
│                            │                │  │  (app.py)            │  │
│  Napájení: Li-Ion 18650    │                │  │  + zvuková notif.    │  │
│           + TP4056         │                │  │  + CSV export        │  │
└────────────────────────────┘                │  └──────────────────────┘  │
                                              └────────────────────────────┘

  publish: IoTProject/9/grill/teplota   ──►
  subscribe: IoTProject/9/grill/Tmaso   ◄── (cílová teplota masa)
             IoTProject/9/grill/Tgrill  ◄── (cílová teplota grilu)
```

---


### Hardware

| Komponenta                       | Popis                                                  |
|----------------------------------|--------------------------------------------------------|
| Raspberry Pi Pico W              | Hlavní MCU s vestavěnou Wi-Fi                          |
| AHT20 (×2)                       | Teplotní senzory na I2C0 a I2C1                        |
| Li-Ion 18650 + TP4056            | Napájení s nabíjecím modulem                           |
| Mechanický vypínač               | Hlavní vypínač zařízení                                |

**Zapojení I2C:**

| Senzor              | I2C kanál | SCL pin | SDA pin |
|---------------------|-----------|---------|---------|
| AHT20 (maso)        | I2C1      | GP3     | GP2     |
| AHT20 (gril)        | I2C0      | GP5     | GP4     |

---

## Známá omezení

- **AHT20 jako teplotní senzor** - slouží pouze jako laboratorní prototyp, je nevhodný pro reálné měření teploty masa i grilu. Pro reálnnou apliakci viz `docs/Dokumentácia.pdf`.
- **Sleep režim** — v současné implementaci je použit `time.sleep()` v hlavní smyčce. Pro skutečné dosažení odhadované výdrže by bylo potřeba implementovat `machine.lightsleep()` mezi cykly měření a explicitně deaktivovat WLAN po publikaci. Aktuální kód funguje jako funkční prototyp; energetická optimalizace je rozpracována v technické dokumentaci.
- **MQTT bez autentizace** — broker je pro účely projektu spuštěn bez TLS a bez hesla. Pro reálné nasazení použijte `mosquitto.conf` s `password_file` a TLS certifikáty.

---
## Autoři
- Ema Ondrušková
- Adam Behúň
- Jakub Cíbik
- Jaroslav Cempírek
-----

## Odkazy

- Detailní technická dokumentace: `docs/Dokumentácia.pdf`
- [MicroPython na Raspberry Pi Pico W](https://docs.micropython.org/en/latest/rp2/quickref.html)
- [Mosquitto MQTT broker](https://mosquitto.org/)
- [paho-mqtt (Python client)](https://www.eclipse.org/paho/index.php?page=clients/python/index.php)
- [umqtt.simple (MicroPython)](https://github.com/micropython/micropython-lib/tree/master/micropython/umqtt.simple)
