## Članovi tima
- Dušan Đorđević SV1/2021
- Rade Pejanović SV10/2021
## Motivacija
Očuvanje vitalnosti pčelinjih društava i blagovremeno reagovanje (pregrevanje, rojenje, vlaga, manjak hrane, pad napona) smanjuju gubitke, podižu prinose i rasterećuju pčelara automatizacijom nadzora.
## Pregled problema
Sistem prikuplja senzorske podatke iz košnice i okoline, zaključuje na osnovu pravila i šalje preporuke/alarme ili automatski pokreće aktuatorske radnje (ventilacija, grejanje, sušenje, režim štednje).
## Metodologija rada
### Ulazi u sistem 
- **Temperatura**:
	- košnice - `hiveTemp` (°C)
	- legla - `broodTemp` (°C)
	- eksterna - `extTemp` (°C)
- **Vlaga**: 
	- košnice - `humidity` (%)
- **Kvalitet vazduha**: 
	- air-quality-index - `aqi` ($\mu$g/$m^3$)
- **Masa**: 
	- trenutna masa košnice - `hiveWeight` (kg)
	- razlika mase košnice u određenom intervalu vremena - `weightDelta` (kg)
	- stopa promene mase - `weightRate` (kg/h)
- **Zvučni/vibracioni profil**:
	- frekvencija zujanja - `buzzFreq` (Hz)
	- jačina zvučnog signala - `buzzAmp` (dB)
- **CO₂**: 
	- koncentracija CO₂ u košnici - `co2ppm` (ppm)
- **Aktivnost na poletnoj dasci**: 
	- broj pčela koje ulaze u košnicu - `entranceIn` (broj/min)
	- broj pčela koje napustaju košnicu - `entranceOut` (broj/min)
- **Matica**: 
	- prisustvo matice u košnici - `queenPresent` (boolean)


- **Vremenski uslovi**: 
	- kiša - `rain` (boolean)
	- brzina vetra - `windSpeed` (km/h)
- **Energija/bezbednost**: 
	- napon baterije - `batteryV` (V)
	- stanje vrata objekta - `doorOpen` (boolean)
	- akcelometar košnice - `vibrationG` (m/s²)
### Ulazi za dijagnostiku
**Senzori ispravni?** (boolean) 
- `tempOK` 
- `humOK` 
- `scaleOK` 
- `micOK`
- `co2OK`
- `entranceOK`
- `batteryOK`
- `aqiOK`
- `rfidOK`

**Aktuatori ispravni?** (boolean) 
- `fanOK`
- `heaterOK`
- `dehumidOK`
### Izlazi iz sistema
**Akcije:** 
- `fanOn(level)` - (low, high, medium)
- `heaterOn(level)` - (low, high, medium)
- `dehumidOn()`
- `batterySavingOn()`
- `lockHive()`

**Obaveštenja:**
- `coldAlert()`
- `heatAlert()`
- `humidityAlert()`
- `CO2Alert()`
- `swarmAlert()`
- `yieldAlert()`
- `harvestReady()`
- `feedSyrupAlert()`
- `moldAlert()`
- `stormAlert()`
- `lowPowerAlert()`
- `polutionAlert()`
### Konfiguracija
Sezonski pragovi i prozori: 
- minimalna temperatura legla - `minBroodTemp`
- maksimalna temperatura legla - `maxBroodTemp`
- maksimalna vlažnost - `maxHumidity`
- maksimalna koncentracija CO2 - `maxCO2ppm`
- granica jakog prinosa meda - `minWeightRate`
- minimalna težina košnice koja signalizira potrebu za prihranom u vidu sirupa - `minFeedWeight`
- minimalna eksterna temperatura na kojoj se sirup ne kristališe - `minFeedTemp`
- minimalna temperatura na kojoj je moguće rojenje - `minSwarmTemp`
- raspon frekvecija rojenja - `swarmFreqBand`
- minimalan pad težine koji signalizira rojenje - `swarmWeightDropMin`
- minimalan napon baterije - `batteryMinV`
- maksimalna bezbedna zagađenost vazduha - `maxAqi`
- maksimalan broj pčela koje se mogu žrtvovati - `maxSacrifice`
- temperatura pogodna za nastanak plesni - `moldTemp`
- vlažnost vazduha pogodna za nastanak plesni - `moldHumidity`
## Baza znanja
### Prosta pravila
**Temperatura, ventilacija, grejanje**
```java
when broodTemp < minBroodTemp 
then heaterOn(medium)

when broodTemp << minBroodTemp 
then heaterOn(high) and coldAlert()

when hiveTemp > maxBroodTemp 
then fanOn(medium)

when hiveTemp >> maxBroodTemp 
then fanOn(high) and heatAlert()

when extTemp < 0 and hiveTemp < minBroodTemp
then heaterOn(high)
```

**Vlaga**
```java
when humidity > maxHumidity
then dehumidOn()

when humidity >> maxHumidity 
then dehumidOn() and fanOn(low) and humidityAlert()
```

**CO₂, zagušljivost**
```java
when co2ppm > maxCO2ppm  
then fanOn(medium)

when co2ppm >> maxCO2ppm  
then fanOn(high) and CO2Alert()
```

**Masa, prinos, hranjenje**
```java
when weightRate > minWeightRate 
then yieldAlert()

when hiveWeight < minFeedWeight and extTemp > minFeedTemp
then feedSyrupAlert()
```

**Aktivnost ulaza**
```java
when entranceOut >> entranceIn and extTemp > minSwarmTemp 
then swarmAlert()

when rain and entranceIn ~ 0 and entranceOut ~ 0 
then stormAlert()
```

**Zvučni/vibracioni signali**
```java
when buzzFreq in swarmFreqBand and buzzAmp high
then swarmAlert()
```

**Energija/bezbednost**
```java
when batteryV < batteryMinV 
then batterySavingOn() and lowPowerAlert()

when aqi > maxAqi and entranceOut < minSacrifice
then lockHive() and polutionAlert()
```

**Upozorenje za pojavu plesni**
```java
when hiveTemp ~ moldTemp and humidity ~ moldHumidity  
then moldAlert()
```
### Forward chaining
#### Primer 1: **Rojenje**
```java
when queenPresent = false 
then trigger(QueenExit)

when buzzFreq in swarmFreqBand and buzzAmp high
then trigger(SwarmSound)
// -----
when QueenExit and SwarmSound and entranceOut > thresholdOut 
then trigger(Exodus)
// -----
when Exodus and weightDelta < swarmWeightDropMin over windowShort
then trigger(ConfirmedSwarm)
// -----
when ConfirmedSwarm
then swarmAlert()
```
#### Primer 2: **Prinos i priprema za vrcanje**
```java
when weightRate > minWeightRate
then trigger(StrongFlow)
// -----
when StrongFlow and extTemp in [18..30]
then trigger(StableFlow)
// -----
when StableFlow and weightDelta >= harvestWeightGainMinDay over windowLong
then trigger(HarvestWindow)
// -----
when HarvestWindow 
then harvestReady()
```
### CEP
CEP su izvedeni iz prethodno definisanih primera [[#Forward chaining]]-a, gde svaki radi agregaciju podataka u određenom vremenskom prozoru. 
#### Primer 1: Detekcija rojenja
Kada se uočava karakteristično zujanje (frekvencija i jačina), zatim nagli porast izlazaka pčela iz košnice, a potom i značajan pad težine u kratkom vremenu, sistem potvrđuje da je došlo do rojenja i šalje upozorenje.

#### Primer 2: Stabilan unos nektara
Ako košnica beleži stabilan rast mase (pozitivan weightRate), spoljna temperatura je u optimalnom opsegu, a u dužem periodu postoji kumulativni porast težine iznad praga, sistem označava da je počeo period jakog prinosa i da je košnica spremna za vrcanje.
#### Primer 3: Odsustvo matice
Sistem prati kada matica nije prisutna u košnici (`queenPresent = false`), ako odsustvo traje duže od zadatog praga, smatra se da je matica možda poginula ili izbačena od strane društva. U tom slučaju sistem šalje upozorenje pčelaru da proveri stanje pčelinjeg društva.
### Backward chaining 
Predstavljen kroz primer dijagnostikovanja ispravnosti sistema - sistem (košnica) se smatra ispravnim ako su svi senzori i aktuatori ispravni. 
![[Pasted image 20250824180654.png]]
### Template
Parametrizacija pravila pomoću tabela, gde se pragovi i opsezi za temperaturu, vlagu, prinos i energiju košnice učitavaju direktno iz odgovarajućeg reda tabele.

**Parametri u tabeli:**
- `season` – sezona (proleće, leto, jesen)
- `hiveType` – tip košnice (LR, Dadant, Warre, …)
- `minBroodTemp`, `maxBroodTemp` – minimalna i maksimalna temperatura legla
- `maxHumidity` – maksimalna dozvoljena vlaga
- `harvestWeightGainMinDay` – minimalni dnevni prirast meda
- `swarmFreqBandLow`, `swarmFreqBandHigh` – donja i gornja granica frekvencije zujanja za detekciju rojenja
- `batteryMinV` – minimalni napon baterije

**Primer:**
- red u tabeli `Proleće + LR košnica` 
- daje `(minBroodTemp=34.0°C, maxBroodTemp=36.0°C, maxHumidity=75%, harvestWeightGainMinDay=1.5kg, swarmFreqBand=[180–260]Hz, batteryMinV=11.5V)`
