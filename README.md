# PZPE — PZP Economic v Home Assistant

Tahle věc Vám umožní připojit tepelné čerpadlo PZP Economic (Kermi) k Home Assistant skrze TCP/IP Modbus. Síťovým rozhraním je vybaveno každé tepelné čerpadlo z řady PZP Economic. Úmyslně nejsou vytaženy všechny atributy z dokumentace (XLS soubor též v repozitáři), protože neodbornou manipulací s parametry si lze akorát zkomplikovat život. Pointou je spíše možnost regulovat teplotu podlahového topení, případně teplotu teplé vody například s ohledem na výkon fotovoltaického systému či situaci na spotovém trhu.

## Na co se soustředit

Doporučuji se soustředit především na tyto entity:

| Entita | Typ | K čemu slouží |
|--------|-----|---------------|
| `pzpe_climate_rw_regbod_topeni` | climate | Regulační bod topení (15–35 °C) |
| `pzpe_climate_rw_regbod_tv` | climate | Regulační bod teplé vody (20–70 °C) |
| `pzpe_ib_wr50005_bit2_regbod_topeni` | input_boolean | Povolení ext. regulačního bodu topení |
| `pzpe_ib_wr50015_bit0_regbod_tv` | input_boolean | Povolení ext. regulačního bodu teplé vody |

Vše ostatní vypínejte / zapínejte a štelujte pouze v případě, že máte představu, co to může způsobit :-)

## Jak to funguje

Zapnutím příslušného input_boolean (např. `pzpe_ib_wr50005_bit2_regbod_topeni`) přebíráte nad teplotou topení kontrolu. Přes climate entitu ji pak lze měnit. Takže například v případě vysokých cen na spotu není nic jednoduššího, než cílovou teplotu podlahy snížit třeba na 20 °C. Pokud je dům dobře zaizolován, lze s tím bez problémů překlenout cenový peak po několik hodin a to bez snížení komfortu žití. Totéž platí pro teplou vodu (`pzpe_ib_wr50015_bit0_regbod_tv` + `pzpe_climate_rw_regbod_tv`).

Aby došlo k přebrání kontroly nad TČ, je potřeba nechat běžet automatizaci „PZPE, ohlášení nadřazeného řízení", která dává TČ každou minutu vědět, že Home Assistant kontrolu přebírá. Pokud TČ tuto informaci pravidelně nedostává, přebere po nějaké době řízení opět na sebe. Stav lze sledovat přes entitu `binary_sensor.pzpe_coil_ro_nadrazene_rizeni_aktivni`.

### Přepínače bitů a synchronizace s TČ

Bitové registry (WR50005, WR50015, WR50050, WR50055) mají v HA jeden input_boolean na každý bit. Synchronizace běží oběma směry a zdrojem pravdy je TČ:

1. Přepnete input_boolean → automatizace „PZPE, zápis do registru …“ po 1 s složí hodnotu celého registru ze všech jeho input_boolean a zapíše ji do TČ.
2. Při dalším čtení Modbusu se registr načte zpět z TČ a automatizace „PZPE, sync … z TČ do input_boolean“ podle něj nastaví všechny input_boolean. Pokud TČ změnu nepřijme, přepínač se vrátí do skutečného stavu; změna provedená přímo na TČ se tak objeví i v HA.

Do dalšího čtení (desítky sekund) tedy přepínač ukazuje požadovaný, ještě nepotvrzený stav. Bity, které nemají input_boolean, se při zápisu zapisují jako 0.

## Instalace

### 1. Nastavte IP adresu

V souboru `pzpe_modbus.yaml` na řádku `host:` doplňte IP adresu Vašeho PZP Economic:

```yaml
host: 192.168.x.x    # <-- IP adresa Vašeho TČ
```

### 2. Přidejte includes do `configuration.yaml`

```yaml
modbus:
  - !include pzpe_modbus.yaml

automation pzpe: !include pzpe_automations.yaml

homeassistant:
  packages:
    pzpe_helpers: !include pzpe_helpers.yaml

```

Pokud už některé z těchto sekcí (`modbus:`, `homeassistant:`) v konfiguraci máte, přidejte příslušné řádky k existujícím položkám.

### 3. Restartujte Home Assistant

Po restartu se objeví všechny entity s prefixem `pzpe_`.

### 4. Nechte běžet všechny PZPE automatizace

V provozu je potřeba nechat **všechny automatizace** s prefixem PZPE:

| Automatizace | Co dělá |
|-------------|---------|
| PZPE, ohlášení nadřazeného řízení | Watchdog — TČ ví, že HA je aktivní |
| PZPE, zápis do registru WR50005 | Zápis bitů řízení topení do TČ |
| PZPE, sync WR50005 z TČ do input_boolean | Čtení bitů řízení topení z TČ |
| PZPE, zápis do registru WR50015 | Zápis bitů řízení teplé vody do TČ |
| PZPE, sync WR50015 z TČ do input_boolean | Čtení bitů řízení teplé vody z TČ |

## Soubory

| Soubor | Obsah |
|--------|-------|
| `pzpe_modbus.yaml` | Modbus TCP hub + sensory, binary sensory, switche, climaty |
| `pzpe_automations.yaml` | Synchronizace bitových registrů (WR50005, WR50015, WR50050, WR50055) a dalších zapisovatelných registrů + watchdog nadřazeného řízení |
| `pzpe_helpers.yaml` | Input boolean pro bity WR50005 (bit 0–8), WR50015 (bit 0–3), WR50050 a WR50055 a další pomocníci |

## Bonus: senzor stavu TČ

Na zobrazení stavu tepelného čerpadla si lze vytvořit template senzor. V `configuration.yaml` do sekce `template:` přidejte:

```yaml
- sensor:
    - name: "PZPE Stav TČ"
      unique_id: pzpe_stav_tc_text
      state: >
        {% set val = states('sensor.pzpe_hreg_ro_stav_tc') | int(0) %}
        {% set text = {
          0: 'Vypnuto / pohotovostní režim',
          1: 'Běží',
          2: 'Topí',
          3: 'Ohřívá TV',
          4: 'Odtává',
          5: 'HDO',
          6: 'Porucha',
          7: 'Chladí',
          8: 'Blokování'} %}
        {{ text[val] if val in text else '?' }}
```
