# LOB Garage Door – Produksjonskontroller

Styrer produksjonen av garasjeporter på fabrikkgulvet: plukk/lagring, løfteanlegg, fresecelle og to robotceller.  
Systemet integrerer mot Dynamics 365 F&O (produksjonsordre, BOM, lager), samt roboter og fres via signaler og CSV-filer.  
Kjøringen orkestreres av en ASP.NET Core-basert bakgrunnstjeneste, støttet av et HTTP API for operatørstyring.

**Status:** Utkast – oppdatert med verifiserte kilder fra denne gjennomgangen.  
Bekreftet grunnlag (filer gjennomgått):

Application/Services/ProductionExecutionService.cs  
Application/Services/ProductionSchedulingService.cs  
LOBGarageDoorProductionControllerService/Controllers/OperationsController.cs  
LOBGarageDoorProductionControllerService/Controllers/SettingsController.cs  
LOBGarageDoorProductionControllerService/Controllers/SimulationController.cs  
Domain/Entities/Lift/LiftLoadingInput.cs, LiftUnloadingInput.cs  
Domain/Entities/Production/ManualProductionItem.cs, ProductionStorageTransfer.cs  
Application/Interfaces/ISettingsService.cs  
Application/Interfaces/ID365DataProcessingService.cs, ID365ActionProcessingService.cs  
Application/General/TestSignalsList.cs  
Application/BinPacking/BinPackingElements.cs, BinPackingSolver.cs  
Program.cs (test/debug-støtte)  
Program.cs, MainService.cs, WebAPIService.cs, AppEnvironmentConfig.cs, ILoggingService.cs  
+ Flere tidligere analyserte kjernetjenester og domeneobjekter


## Innhold

- Tjenestestruktur og arkitektur
- Teknologier og avhengigheter
- Kjerne domenemodell
- Hovedflyter
- Planlegging og køgenerering
- API – Endepunkter
- Installasjon & kjøring
- Konfigurasjon
- Persistens og datakilder
- Logging & overvåkning
- Tester & CI/CD
- Feilsøking (kjente fallgruver)
- Veikart / mangler
- Bilag: Signaler som overvåkes (eksempler)
- Bilag: Forretningsregler (utdrag)
- Eksempelflyt (test)


## Tjenestestruktur og arkitektur

Systemet består av to hovedkomponenter:

1. **Windows Service Host**
   - Kjører som en systemtjeneste på fabrikkens produksjonsserver
   - Initialiseres via `Program.cs` og starter `MainService` og `WebAPIService`
   - CLI-baserte kommandoer støttes: `/Install`, `/Uninstall`

2. **HTTP API (WebAPIService)**
   - Eksponerer kontrollere på port `4000`
   - Tillater operatørstyring, test og konfigurasjonsstyring
   - Kontrollere:
     - `OperationsController`
     - `SettingsController`
     - `SimulationController`

---

### 🧵 Parallelt kjørende bakgrunnstråder

`MainService` starter flere tråder, hver med sitt ansvar:

- `ProductionExecutionThreadJob`  
  → Kjører `ProductionExecutionService.ProcessUntilCanceled(...)` i 2 sek intervall  
  → Koordinerer hele produksjonsflyten: robot1, robot2, frese, lager, signaler

- `ProductionSchedulingThreadJob`  
  → Kjører `ProductionSchedulingService.RunScheduling()` hvert 5. minutt  
  → Lager produksjonskø via `BinPackingElements.CreateQueue(...)`

- `ProductionStorageSupervisionThreadJob`  
  → Overvåker tray-posisjoner og initierer replenishment

- `RobotToCloudMessageThreadJob`  
  → Leser signaler fra robotene og sender statuser til sky (i prod)

- `TestSignalsThreadJob`  
  → Leser test-signalverdier fra `TestSignalsList` (kun når `Testing == true`)

---

### 🔧 Applikasjonstjenester (via Dependency Injection)

| Tjeneste | Ansvar |
|---------|--------|
| `ProductionExecutionService` | Kjøringsmotor for produksjon |
| `ProductionSchedulingService` | Lager produksjonskø |
| `RobotFileProcessingService` | Genererer og sender CSV-filer til Robot1 og Robot2 |
| `MillingMachineFileProcessingService` | Genererer `.lbs` freseprogrammer |
| `WarehouseManagementService` | Lagerflytting, transfer journal, replenishment |
| `ID365DataProcessingService` | Leser data fra D365 (ordre, BOM, lager) |
| `ID365ActionProcessingService` | Skriver til D365 (start produksjon, post journaler) |
| `ISettingsService` | Holder runtime state (in-memory key-value) |
| `ILoggingService` | Logger driftshendelser (output avhenger av implementasjon) |
| `AppEnvironmentConfig` | Inneholder konfigurasjon fra `appsettings.json` |

---

### 🔗 Eksterne integrasjoner

- **Dynamics 365 F&O**  
  → Produksjonsordrer, BOM, journalføring  
  → Kommunikasjon via `ID365DataProcessingService` og `ID365ActionProcessingService`

- **WMS**  
  → Lagerposisjoner, beholdning, flytting  
  → Brukes av `WarehouseManagementService`

- **Robot 1 (10.5.15.21)**  
  → Mottar CSV-filer og digitale signaler (mastership kreves)

- **Robot 2 (10.5.15.73)**  
  → Krever `DOF_OkToSendNewCsvFilesRob2 == true` før filoverføring

- **Fresemaskin**  
  → Leser `.lbs` filer med makroer (`O:CUT`, `O:MACRO`, `C:` osv.)  
  → Filer genereres av `MillingMachineFileProcessingService`

---

### 📊 Arkitekturoversikt

```mermaid
flowchart LR
    subgraph Host ["Windows Service"]
        Main["MainService\n(thread-basert produksjonsmotor)"]
        API["WebAPIService\n(HTTP API på port 4000)"]
    end

    subgraph API_Lag ["HTTP API – WebAPIService"]
        Ops["OperationsController"]
        Sim["SimulationController"]
        Set["SettingsController"]
    end

    subgraph AppLag ["Applikasjonstjenester"]
        Exec["ProductionExecutionService"]
        Sched["ProductionSchedulingService"]
        Mill["MillingMachineFileProcessingService"]
        Rob["RobotFileProcessingService"]
        WMS["WarehouseManagementService"]
        D365Read["ID365DataProcessingService"]
        D365Write["ID365ActionProcessingService"]
        Log["ILoggingService"]
        Env["AppEnvironmentConfig"]
        Settings["ISettingsService"]
    end

    subgraph Integrasjoner
        D365Sys["Dynamics 365 F&O"]
        WmsSys["WMS"]
        Robot1["Robot 1 (10.5.15.21)"]
        Robot2["Robot 2 (10.5.15.73)"]
        Milling["Fresemaskin (.lbs)"]
    end

    Main --> Exec
    Main --> Sched
    Main --> Log
    API --> API_Lag
    API_Lag --> Ops
    API_Lag --> Sim
    API_Lag --> Set
    Exec --> AppLag
    Sched --> AppLag
    AppLag --> Integrasjoner
    Rob --> Robot1 & Robot2
    Mill --> Milling


### Roller og ansvar (utdrag)

Dette avsnittet oppsummerer ansvarsområder for sentrale tjenester, slik de faktisk opptrer i kildekoden.

---

#### `ProductionExecutionService`

Sentralt orkestreringsloop (2s intervall).  
- Leser state fra `ISettingsService`
- Henter produksjonsordre:
  - Manuell: via `ManualProductionItems`-køen
  - Automatisk: fra `GetProductionListAsync(...)` (D365)
- Genererer CSV-filer for Robot1 og Robot2, samt `.lbs` for fresemaskin
- Overvåker signaler (`DOF_OkToSendNewCsvFilesRob1/Rob2`)
- Koordinerer replenishment og tray-håndtering

➡️ Avhengigheter (DI):
- `WarehouseManagementService`  
- `RobotFileProcessingService`  
- `MillingMachineFileProcessingService`  
- `ID365ActionProcessingService`, `ID365DataProcessingService`

---

#### `ProductionSchedulingService`

Planleggingsmotor som kjører hvert 5. minutt.  
- Henter tilgjengelig lager (I, K, N) og produksjonsordre
- Bruker `BinPackingElements.CreateQueue(...)` til å generere produksjonskø
- Lagrer resultatet i `schedulingResult`, som leses av `ProductionExecutionService`

---

#### `BinPackingElements` og `BinPackingSolver`

Står for optimering av materialbruk ved hjelp av bin-packing-algoritme:
- Matcher ordrer med tilgjengelige profil-lengder
- Minimerer kapp og antall profiler brukt
- Bruker Google OR-Tools (`SCIP`) som underliggende løsningsmotor
- Tar hensyn til elementlengde, resthåndtering og fordeling

---

#### `HTTP API`

Eksponerer **`Operations`**, **`Settings`** og **`Simulation`** kontrollere.  
- Støtter operatørkommandoer (lasting, lossing, batchlegging)  
- Statusflagg og runtime-styring (`/StartStop`, `/LiftInactive`)  
- Manuell data via JSON  
- Testverdier og signaler via `SimulationController`  
- Swagger aktivert  
- Kjøres via `WebAPIService` på port `4000`

---

#### `Runtime state (Settings)`

Lages i minne via `ISettingsService`, tilgjengelig via nøkkelbasert API.  
⚠️ Listebaserte nøkler må initieres ved oppstart, ellers vil systemet kaste `NullReferenceException`

- **Boolean flagg:**  
  - `StartAutomaticExecution`  
  - `DoorProductionInactive`  
  - `LiftInactive`  
  - `CheckReturnFeeder`  
  - `ProductionStarted`

- **Lister:**  
  - `ManualProductionItems`  
  - `LiftLoadingKassettList`  
  - `ManualLiftUnloadingList`  
  - `ProductionStorageTransfers`  
  - `signalsToMonitor`

➡️ Runtime state kan resettes/testes via `SimulationController`

---

#### `Testsignaler`

`TestSignalsList` gir simulerte signaler for utvikling og test  
- Styrt via `SimulationController` (GET/POST-endepunkter)
- Aktiveres når `AppEnvironmentConfig.Testing == true`

**Typiske signaler:**  
- `DOF_OkToSendNewCsvFilesRob1`  
- `DOF_OrderDone`  
- `strLiftCommand`  
- `numElementLength`

📌 Disse signalene brukes også i produksjon – testmiljøet overstyrer ekte IO

---

#### `WarehouseManagementService`

Styrer produksjonslageret, tray-/heislogikk og D365-integrasjoner for overføringsjournaler.  
- Velger plukklokasjoner via `GetPickLocationsAsync(...)`  
- Validerer beholdning i `I*` og `K*` lokasjoner  
- Initierer replenishment når beholdning < 5 stk  
- Lager og poster transfer journaler i D365  
- Synkroniserer `nSourceStorage`, `nTargetStorage` med Robot1

---

#### `RobotFileProcessingService`

Genererer CSV-filer for **Robot1** og **Robot2** basert på produksjonsdata og plukk-informasjon.  
- CSV-format:
  - Robot1: 11 kolonner, `;`-separert
  - Robot2: 40+ kolonner, opp til 9 `ElementPart`
- Skriver til konfigurerte filbaner (`AppEnvironmentConfig.FilePaths`)  
- Laster opp via FTP  
- Håndterer også opprydding og parsing av kasserte CSV-filer  
⚠️ Forutsetter synkronisert tilgang ved samtidige oppgaver

---

#### `MillingMachineFileProcessingService`

Skriver `.lbs`-filer for fresemaskin.  
- Bygger kommandosekvenser med `O:CIRCLE`, `O:CUT`, `O:MACRO`, `C:` osv.  
- Basert på `MillingItems` og D365-data  
- Støtter høydekompensasjon og korrekt klemmeplassering  
- Kutter i henhold til resthåndteringsregler (754mm+, 2400mm osv.)

---

#### `D365-integrasjon`

Delt i to tjenester:

- **`ID365DataProcessingService`**  
  → Leser: ordrer, BOM, lager, varianter, journaler  
  → Brukes av: Scheduling, WarehouseManagement, Milling

- **`ID365ActionProcessingService`**  
  → Skriver: Start/finish produksjon, transfer journaler, forbruk  
  → Brukes av: ExecutionService, WarehouseManagementService

---

#### `Robot-integrasjon`

- **CSV-generering:** via `RobotFileProcessingService`  
- **Direkte signal- og variabelskriving:** via `IRobotOutboundMessageProcessingService`  
- Setter signaler som `DOF_Port1Start`, `DOF_ConfirmFeederReturnInPos`  
- Krever *mastership per IP* for å kunne skrive

---

#### `ILoggingService`

- Gir asynkron logging via `LogAsync(...)` og `ReadLogAsync()`  
- Logger bl.a. produksjonsstart, signalstatus, feil  
- Install-logg skrives til `InstallLog.txt` i `AppContext.BaseDirectory`  
- Sink-type ikke spesifisert (kan være File, Console, EventLog)  
- Loggnivå konfigureres via `AppEnvironmentConfig.Logging.Level`

---

#### `AppEnvironmentConfig`

- Laster inn statisk konfigurasjon fra `appsettings.json`
- Felter:
  - `Testing` (true/false)
  - `FilePaths` (CSV/LBS-stier)
  - `Logging.Level`
  - `D365.BaseUrl`, `ClientId`, `Tenant`, osv.
- Injiseres via `IOptions<AppEnvironmentConfig>` til relevante tjenester

---



## Teknologier og avhengigheter

Denne seksjonen beskriver de teknologiske rammene systemet er bygget på, inkludert tredjepartsbiblioteker, integrasjoner og nøkkelkomponenter.

---

### .NET 7 og ASP.NET Core

Systemet er utviklet med .NET 7 og ASP.NET Core, og benytter:

- **Windows Service-hosting** via `IHostedService` (`MainService`)
- **Parallelt kjørende bakgrunnsjobber** (`ProductionExecutionService`, `ProductionSchedulingService`)
- **HTTP API** på port 4000 (`WebAPIService` → `OperationsController`, `SettingsController`, `SimulationController`)
- **CLI-støtte for installasjon** via `CliWrap` i `Program.cs`

---

### Dynamics 365 F&O (ERP-integrasjon)

Systemet samhandler tett med D365 Finance & Operations, via dedikerte tjenester:

- `ID365DataProcessingService` (lesing):
  - Henter produksjonsordrer, BOM, lagerbeholdning, transaksjoner
- `ID365ActionProcessingService` (skriving):
  - Starter og fullfører produksjon
  - Lager og poster transfer journals
  - Rapporterer forbruk

Integrasjonen skjer trolig via OData/REST og autentiseres via verdier definert i `AppEnvironmentConfig.D365`.

---

### WMS (lagerstyring)

Systemet kommuniserer med eksternt WMS-system for:

- Lokasjonsdata (`I*`, `K*`, `N*`)
- Tray- og beholdningsstyring
- Returhåndtering og replenishment

Brukes av `WarehouseManagementService` og `ProductionSchedulingService`.

---

### Filbasert robot- og maskinintegrasjon

#### CSV til Robot1 og Robot2

- **Filformat:** `.csv` – genereres av `RobotFileProcessingService`
- **Robot1:** 11 kolonner, enkle batcher
- **Robot2:** Opptil 9 `ElementPart`, 40+ kolonner
- **Overføring:** FTP til:
  - `10.5.15.21` (Robot1)
  - `10.5.15.73` (Robot2)
- **Konfigurasjon:** Filbaner og legitimasjon settes i `AppEnvironmentConfig.FilePaths`

#### LBS til fresemaskin

- **Filformat:** `.lbs` – genereres av `MillingMachineFileProcessingService`
- **Kommandospråk:** `O:CUT`, `O:MACRO`, `C:`, osv.
- **Sti:** `\\lob-file01\Produksjon\515\FomCam\`
- **Bruk:** Styrer makroprogrammering og precut for paneler

---

### Digital IO og signalstyring

- Roboter og frese kommuniserer via boolske signaler (DOF)
- Systemet leser og skriver signaler via `IRobotOutboundMessageProcessingService`
- Signalene er definert i `signalsToMonitor` ved oppstart

📌 I testmodus byttes ekte signaler ut med verdier fra `TestSignalsList`.

---

### Testmiljø og simuleringsstøtte

- Når `AppEnvironmentConfig.Testing == true`:
  - **Ekte signaler deaktiveres**
  - `SimulationController` eksponeres for å styre testverdier
  - `TestSignalsList` brukes til å lagre bool, string og numeriske testdata

Simulering dekker:
- IO-signalstatus (`/Simulation/SignalValue/{navn}`)
- Variabler som lengder og kommandoer (`/VariableValue`)
- Klar-signal for CSV-overføring (`DOF_OkToSendNewCsvFilesRob1`/Rob2)

---

### Bin-packing og optimering

Systemet bruker en avansert bin-packing-algoritme for å gruppere produksjonslinjer i materialrester:

- Implementert i `BinPackingElements` og `BinPackingSolver`
- Bruker **Google OR-Tools (SCIP)** som optimeringsmotor
- Tar hensyn til profiltyper, lengder og tilgjengelighet
- Sørger for høy materialutnyttelse og lavt kapp

📦 Output brukes som grunnlag for `schedulingResult` i `ProductionExecutionService`

---

### Tredjepartsbiblioteker

| Bibliotek                 | Bruksområde                                |
|---------------------------|---------------------------------------------|
| `CliWrap`                 | Kommandohåndtering for serviceinstallasjon |
| `Google OR-Tools (SCIP)` | Bin-packing-optimalisering                  |
| `Microsoft.Extensions.Hosting` | Hosting av bakgrunnsjobber             |
| `Microsoft.Dynamics.DataEntities` | Tilgang til D365-data via OData     |
| `FTP-klient` (custom)     | Overføring av `.csv` og `.lbs`             |

---


### Egendefinerte applikasjonstjenester

| Interface                                 | Rolle                                      |
|-------------------------------------------|---------------------------------------------|
| `IRobotOutboundMessageProcessingService`  | Signalutveksling med roboter                |
| `IRobotFileProcessingService`             | CSV-generering for Robot1/2                 |
| `IMillingMachineFileProcessingService`    | `.lbs`-filer for fres                       |
| `IWarehouseManagementService`             | Lagerlogikk og journalhåndtering            |
| `IProductionFunctionsService`             | Statusoppdatering i D365                    |
| `ID365DataProcessingService`              | Lese operasjoner fra D365                   |
| `ID365ActionProcessingService`            | Skrive operasjoner til D365                 |
| `ISettingsService`                        | Runtime state-lagring                       |
| `ILoggingService`                         | Logging av hendelser (sink ukjent)          |


## Kjerne domenemodell

```mermaid
classDiagram

class CurrentProductionOrder {
  string ProdId
  int QtyScheduled
  int QtyStarted
  bool Reproduction
  List~PickLocation~ PickLocations
  List~ProductionOrderBillOfMaterialLine~ PickList
}

class PickLocation {
  string PickLocationID
  string SizeID
  string BatchID
  double RawMaterial
  double PreCut
  double Endspacing
  double InventQty
  List~ProductionOrderBillOfMaterialLine~ BillOfMaterialLines
}

class ProductionOrderBillOfMaterialLine {
  string ProductionOrderNumber
  double EstimatedInventoryQuantity
  string ItemNumber
  string EGBarCodeId
  string EGProfileCodeId
  string EGDoorThicknessId
  string EGDoorTypeID
  string EGSectionId
  string EGCutTemplateId
  string EGDoorWidthId
  string ProductSizeId
  string InventoryLotId
  string EGRalColorId
  double LineNumber
}

class ElementPart {
  double FinishLength
  LengthType LengthType  // Finished | Return | Scrap
  // Enum: beskriver type kutt – brukt i fres og etterbearbeiding
  string Barcode
  int ClampsRequired
  double InventQty
  bool IsDoor
  bool ToPainting
  string PartData
}

class Robot1Input
class Robot2Input
class MillingMachineInput

class LiftLoadingInput {
  string Location
  string ItemId
  string Length
  string Profile
  string BatchId
  double InventQty
  string SizeId
}

class LiftUnloadingInput {
  string Location
  string ItemId
  string Length
  bool Scrap
  string BatchId
  double InventQty
  string SizeId
}

class ManualProductionItem {
  string ProdId
  string Rawlength
  string Precut
  string Endspacing
  double QtyScheduled
  bool Reproduction
}

class ProductionStorageTransfer {
  string JournalId
  string Barcode
  string Total
  string Usable
}

class WarehouseLocationOnHand {
  string InventoryLocationId
  string ItemNumber
  string ProductSizeId
  double OnHandQuantity
  string InventoryBatchId
  string InventoryLicensePlateId
}

CurrentProductionOrder "1" o-- "many" PickLocation
PickLocation "1" o-- "many" ProductionOrderBillOfMaterialLine
Robot1Input ..> CurrentProductionOrder
Robot2Input ..> CurrentProductionOrder
MillingMachineInput ..> CurrentProductionOrder
ElementPart ..> Robot2Input

LiftLoadingInput ..> CurrentProductionOrder
LiftUnloadingInput ..> CurrentProductionOrder
ManualProductionItem ..> CurrentProductionOrder
ProductionStorageTransfer ..> CurrentProductionOrder
WarehouseLocationOnHand ..> PickLocation

```
Denne modellen representerer de viktigste entitetsstrukturene og dataobjektene i produksjonsflyten.
Knyttede relasjoner viser hvordan ordre, lager, plukk og maskinkommunikasjon henger sammen.

## Hovedflyter

Systemet støtter to produksjonsmodi: *manuell* og *automatisk*.  
En sentral bakgrunnstjeneste (`ProductionExecutionService`) orkestrerer prosessen i tre hovedfaser, med 2 sekunders intervall.

---

### A) Valg av produksjon

#### Robotklar-sjekk

- Systemet sjekker signalet `DOF_OkToSendNewCsvFilesRob1` (fra R1: `10.5.15.21`)
- I testmiljø leses verdien fra `TestSignalsList.DOF_OkToSendNewCsvFilesRob1`
- Uten "klar"-signal utsettes produksjonen og logges

#### Manuell produksjon

- Dersom `Settings["ManualProductionItems"]` inneholder batcher →  
  `GetManualProduction()` henter `ManualProductionItem`-data fra D365 og WMS
- Batchene behandles i rekkefølge (FIFO)
- Robot1Input genereres basert på plukkdata og elementlengde

#### Automatisk produksjon

- Hvis `Settings["StartAutomaticExecution"] == true` og roboten er klar →  
  `GetProduction()` henter neste tilgjengelige ordre via `GetProductionListAsync(...)`
- Lagerbeholdning sjekkes → ved mangel:
  - logges: *"Not enough sections... Please refill storage."*
  - `skipCount` økes og lagres i `ISettingsService`

- Resultatet av planleggingen hentes fra `schedulingResult`, generert av `ProductionSchedulingService`

---

### B) Kjøre en ordre (`ProcessProduction`)

- Status på produksjonsordren oppdateres i D365:  
  `StartProduction(...)`, `UpdateProductionStatus(...)`
- For hver `PickLocation`:
  - Råmateriale velges via `GetPickLocationsAsync(...)`
  - Løftekommando genereres via `LiftService.CreateLiftString(...)`
  - `Robot1Input` genereres og skrives til CSV:
    `RobotFileProcessingService.CreateRobot1File(...)`
- CSV skrives til delt mappe og lastes opp via FTP til Robot1

---

### C) Fresecelle og Robot2 (`ProcessMillingCellData`)

#### Signal-synkronisering

- Filskriving for Robot2 og fres **blokkeres** inntil `DOF_OkToSendNewCsvFilesRob2 == true`
- I testmodus brukes `TestSignalsList.DOF_OkToSendNewCsvFilesRob2`

#### Panel-deling

- Råmateriale splittes i `ElementPart` basert på:
  - `PreCut`, `FinishLength`, `Endspacing`
  - Klassifiseres som `Finished`, `Return`, eller `Scrap`

#### Klem og kasettregler

- Egen logikk for `KASSETT`-paneler (klemmer og makroer utelates)
- `CalculateClampsUsed()`, `CalculatePreCutKassett()` brukes

#### Artikkelrester (resthåndtering)

| Lengde (mm)     | Handling                 |
|-----------------|--------------------------|
| ≥ 2400          | Returneres til lager     |
| 1200–2399       | Return manuelt eller automatisk |
| < 1200          | Skrotes                  |

#### Dør fra rest (opsjonelt)

- Hvis `Settings["DoorProductionInactive"] == false` og restlengde er 754–2399 mm:  
  - Ny produksjonsordre opprettes automatisk i D365
  - Brukes i dørproduksjon fra restpaneler (754mm-modulbasert)

#### Filskriving

- `.lbs`-fil genereres via `MillingMachineFileProcessingService.CreateMillingFile(...)`
- `Robot2Input` skrives til CSV via `CreateRobot2File(...)`
- Filene FTP-overføres til Robot2 og fresemaskin

---

### D) Støtteflyter og vedlikehold

#### Løfteoperasjoner

- Manuell lasting/lossing mottas via API:
  - `LiftLoadingInput`, `LiftUnloadingInput`
- Verdiene legges inn i `ISettingsService` og prosesseres i neste loop
- Heisens reelle posisjon valideres via signaler

#### Produksjonslagerpåfylling

- `CheckProductionStorageInventoryAsync()` ser etter lokasjoner med < 5 stk
- `ReplenishStockAsync(...)` genererer og poster transfer journal i D365
- Flytting skjer via Robot1 og heis

#### Replenish-scenarier (styrt via runtime state)

| Scenario             | Inputliste                       | Output                      |
|----------------------|----------------------------------|-----------------------------|
| loading              | `ManualLiftLoadingList`          | Robot1 CSV (lasting)        |
| unloading            | `ManualLiftUnloadingList`        | Robot1 CSV (lossing)        |
| kassett-loading      | `LiftLoadingKassettList`         | Robot1 CSV (standardisert)  |
| productionStorage    | `ProductionStorageTransfers`     | D365 journal, Robot1 CSV    |

---

#### Robotvariabler og synkronisering

- `UpdateStoragePickPositionAsync()` leser verdier som `nSourceStorage`, `nTargetPosition`
- Disse variablene speiles mot faktisk beholdning i D365
- Krever mastership før skriving → skjer via `IRobotOutboundMessageProcessingService`

---

#### Runtime state

- Alle operasjoner bruker `ISettingsService` som key-value store
- Liste-nøkler må initieres eksplisitt (ellers `NullReferenceException`)
- Flagg:
  - `StartAutomaticExecution`
  - `DoorProductionInactive`
  - `LiftInactive`
  - `CheckReturnFeeder`

## API – Endepunkter

Systemet eksponerer tre hovedkontrollere: `Operations`, `Settings`, og `Simulation`.  
Alle endepunkter er HTTP-baserte (JSON inn/ut) og benytter `ISettingsService` til runtime state-håndtering.

---

### 🧭 OperationsController (`/Operations`)

---

## 🛠️ OperationsController (`/Operations`)

| Metode | Rute                    | Body                                | Retur  | Beskrivelse |
|--------|--------------------------|-------------------------------------|--------|-------------|
| **POST** | `/CheckReturnFeeder`     | `bool`                              | `bool` | Setter `CheckReturnFeeder`. Tjenesten prosesserer retur-feeder ved neste poll. |
| **POST** | `/ManualLoadingLift`     | `List<LiftLoadingInput>`            | –      | Legger manuelle løfte-innlastinger i state. Prosesseres av `WarehouseManagementService.ReplenishStockAsync(...)`. |
| **POST** | `/LoadingLiftKassett`    | `List<LiftLoadingInput>`            | –      | Legger kassett-innlastinger. Brukes til standardisert produksjon. |
| **POST** | `/ManualUnloadingLift`   | `List<LiftUnloadingInput>`          | –      | Legger manuelle utlastinger. Håndteres i bakgrunn via `RobotInput`. |
| **GET**  | `/GetFeedback`           | –                                   | `string` | Returnerer konsolidert loggtekst. |
| **POST** | `/ManualProduction`      | `List<List<ManualProductionItem>>` | –      | Legger manuell produksjon i kø (`ManualProductionItems`). |
| **POST** | `/ProductionStorageTransfer` | `ProductionStorageTransfer`      | –      | Brukes til overføring mellom lager og produksjonslokasjon. Prosesseres av `ReplenishStockAsync(..., productionStorage: true)`. |

---

Eksempel: `/Operations/ManualProduction`

```json
[
  [
    {
      "ProdId": "116024",
      "Rawlength": "3508",
      "Precut": "2850",
      "Endspacing": "100",
      "LineNumber": "2",
      "Location": "K01AA01",
      "Port": "Port1",
      "Reproduction": false,
      "QtyScheduled": 3,
      "QtyStarted": 0
    }
  ]
]
```

Eksempel: /Operations/ManualLoadingLift
```json
[
  {
    "Location": "LIFT1",
    "ItemId": "200050",
    "ItemName": "Profil A",
    "Length": "2600",
    "Endspacing": "50",
    "Profile": "ELEGANT",
    "Select": true,
    "BatchId": "B-2025-0910",
    "InventQty": 5.0,
    "SizeId": "6000"
  }
]

```

Eksempel: /Operations/ManualUnloadingLift
```json
[
  {
    "Location": "LIFT2",
    "ItemId": "113798",
    "ItemName": "Sprosse B",
    "Length": "1800",
    "Profile": "MODERN",
    "Select": true,
    "Scrap": false,
    "BatchId": "B-2025-0905",
    "InventQty": 1.0,
    "SizeId": "5240"
  }
]
```

Eksempel: /Operations/ProductionStorageTransfer
```json
{
  "JournalId": "TRX-142076",
  "Barcode": "113798-5240",
  "Total": "5",
  "Usable": "3"
}
```

### ⚙️ SettingsController (`/Settings`)

| Metode | Rute                     | Body | Retur | Effekt |
|--------|---------------------------|------|-------|--------|
| **POST** | `/StartStop`              | `bool` | `bool` | Starter eller stopper automatisk produksjon. |
| **GET**  | `/StartStop`              | –    | `bool` | Leser verdien for `StartAutomaticExecution`. |
| **POST** | `/LiftInactive`           | `bool` | `bool` | Setter `LiftInactive` (ikke i aktiv bruk). |
| **GET**  | `/LiftInactive`           | –    | `bool` | Leser `LiftInactive`. |
| **POST** | `/DoorProductionInactive` | `bool` | `bool` | Slår dør fra restlengde-produksjon på/av. |
| **GET**  | `/DoorProductionInactive` | –    | `bool` | Leser status. |

---


### 🧪 SimulationController (`/Simulation`) – kun test

| Metode | Rute                              | Body    | Retur   | Beskrivelse              |
|--------|------------------------------------|---------|---------|--------------------------|
| **GET**  | `/VariableValue/{variable}`        | –       | `double` | Leser numerisk testvariabel. |
| **POST** | `/VariableValue/{variable}`        | `double` | –       | Setter numerisk testvariabel. |
| **GET**  | `/StringVariableValue/{variable}`  | –       | `string` | Leser strengvariabel. |
| **POST** | `/StringVariableValue/{variable}`  | `string` | –       | Setter strengvariabel. |
| **GET**  | `/SignalValue/{signal}`            | –       | `bool`   | Leser boolsk testsignal. |
| **POST** | `/SignalValue/{signal}`            | `bool`   | –       | Setter boolsk testsignal. |

---


Eksempel: Sett DOF_OkToSendNewCsvFilesRob1 til `true`:

```bash
curl -X POST http://<host>/Simulation/SignalValue/DOF_OkToSendNewCsvFilesRob1 \
     -H "Content-Type: application/json" \
     -d true
```

📌 Merk:

Alle test- og operatør-APIer bruker ISettingsService til å lagre og hente runtime state.

Listebaserte nøkler (ManualProductionItems, LiftLoadingKassettList, ...) må være eksplisitt initiert som tomme lister ved oppstart.
Ellers vil systemet kaste NullReferenceException.

WarehouseManagementService.ReplenishStockAsync(...) og
CheckProductionStorageInventoryAsync(...) kalles automatisk basert på lagersituasjon i produksjon.
De trigger lagerpåfylling og heisstyring via interne bakgrunnsprosesser.


## Installasjon & kjøring

> 🛠 **Status:** Delvis kartlagt – `AppEnvironmentConfig`, `Program.cs` og CLI-installasjon er verifisert.  
> Systemet starter som en **Windows-tjeneste** med navn: `LOBAS Garage Door Production Service`.  
> CLI-basert installasjon støttes via `CliWrap`.

---

### 📦 Krav

- [.NET SDK 7.0+](https://dotnet.microsoft.com/download)
- Tilgang til produksjonsnettverket og IP-er:
  - Robot1: `10.5.15.21`
  - Robot2: `10.5.15.73`
- `appsettings.json` må inneholde gyldige verdier for:
  - `D365.BaseUrl`, `Tenant`, `ClientId`, `ClientSecret`
  - `WMS.BaseUrl`
  - `FilePaths.*` (CSV og .lbs)
  - `Testing`, `Logging.Level`, osv.

---

### 🧪 Lokalt testmiljø

- Sett `AppEnvironmentConfig.Testing = true` i `appsettings.json`
- Dette gjør at:
  - **Reelle signaler ikke sendes** til roboter eller fres
  - `SimulationController` åpnes i API (`/Simulation/SignalValue/...`)
  - `TestSignalsList` aktiveres for å styre digitale signaler
- Bruk API-kallet:

```bash
curl -X POST http://localhost:4000/Simulation/SignalValue/DOF_OkToSendNewCsvFilesRob1 -H "Content-Type: application/json" -d true
```

### 🚀 Kjøre lokalt

```bash
dotnet restore
dotnet build
dotnet run --project LOBGarageDoorProductionControllerService

```
### 🖥️ Service-installasjon (Windows)

`Program.cs` støtter CLI-basert installasjon via `sc.exe` og `CliWrap`.  
💡 Spør ChatGPT: sjekke om tjenesten finnes, stoppe og slette eksisterende instans, og installere på nytt.

#### 🔵 For installasjon:

```bash
dotnet run --project LOBGarageDoorProductionControllerService -- /Install
```
#### 🔴 For avinstallasjon:

```bash
dotnet run --project LOBGarageDoorProductionControllerService -- /Uninstall
```
> 📌 Merk: Installasjonen skriver logg til `InstallLog.txt` i `AppContext.BaseDirectory`

---

### 📁 Filplassering og miljø

- CSV-filer og .lbs-filer genereres til mapper angitt i `AppEnvironmentConfig.FilePaths`
- For eksempel:
\\lob-file01\Produksjon\515\csvRob1\
\\lob-file01\Produksjon\515\csvRob2\
\\lob-file01\Produksjon\515\FomCam\

### 🔐 Avhengigheter

AppEnvironmentConfig injiseres via IOptions<AppEnvironmentConfig>
(brukes i f.eks. RobotFileProcessingService, MillingMachineFileProcessingService)

ISettingsService benytter initialiserte nøkler fra appsettings.json for runtime state

Listebaserte nøkler som ikke er initiert → kaster NullReferenceException

Sørg for at ManualProductionItems, LiftLoadingKassettList og lignende er satt til [] ved første oppstart

## ⚙️ Konfigurasjon

Systemet benytter en kombinasjon av statisk konfigurasjon (`AppEnvironmentConfig`) og runtime-state (`ISettingsService`) for å styre kjøringen.

---

### 🛠️ AppEnvironmentConfig

Konfigureres via `appsettings.json` og injiseres via `IOptions<AppEnvironmentConfig>`.  
Brukes av flere sentrale tjenester som:

- `RobotFileProcessingService`
- `MillingMachineFileProcessingService`
- `ProductionExecutionService`

#### Eksempel: `appsettings.json`

```json
{
  "AppEnvironmentConfig": {
    "Testing": true,
    "D365": {
      "BaseUrl": "<ukjent>",
      "Tenant": "<ukjent>",
      "ClientId": "<ukjent>",
      "ClientSecret": "<ukjent>"
    },
    "WMS": {
      "BaseUrl": "<ukjent>"
    },
    "FilePaths": {
      "Robot1CsvOut": "<ukjent>",
      "Robot2CsvOut": "<ukjent>",
      "MillingCsvOut": "<ukjent>"
    },
    "Logging": {
      "Level": "Information"
    }
  }
}
```
📌 Fyll ut <ukjent> med miljøspesifikke verdier (f.eks. URLer, hemmeligheter, filstier)

### 📄 Kjente felt

| Felt                      | Type   | Beskrivelse                                                  |
|---------------------------|--------|---------------------------------------------------------------|
| `Testing`                | bool   | Aktiverer testmodus – signaler simuleres via `TestsignalList` |
| `FilePaths.Robot1CsvOut` | string | Output-mappe for Robot1 CSV-filer                             |
| `FilePaths.Robot2CsvOut` | string | Output-mappe for Robot2 CSV-filer                             |
| `FilePaths.MillingCsvOut`| string | Output-mappe for fres `.lbs`-filer                            |
| `D365.BaseUrl`           | string | Base-URL for D365 API                                         |
| `D365.Tenant`            | string | Tenant-ID for D365 miljøet                                    |
| `D365.ClientId`          | string | App-registrering ID                                           |
| `D365.ClientSecret`      | string | Tilgangsnøkkel for klienten                                   |
| `WMS.BaseUrl`            | string | Endepunkt for WMS-integrasjonen                               |
| `Logging.Level`          | string | Loggnivå (`Information`, `Warning`, `Error`, …)               |

> 📁 Typiske utdata-mapper: `\\lob-file01\Produksjon\515\` eller `515-Test\` avhengig av `Testing == true`

### 🧠 ISettingsService – runtime state

En nøkkelbasert in-memory key-value store brukt til å lagre produksjonsnær tilstand.  
API-endepunkter og `ProductionExecutionService` leser/skriver her kontinuerlig.



### 🔐 Kjente nøkler og datatyper

#### ⚪ Boolean-flagg

| Nøkkel                   | Beskrivelse                                    |
|--------------------------|------------------------------------------------|
| `StartAutomaticExecution`| Aktiverer automatisk kjøring                   |
| `LiftInactive`           | Slår av heis midlertidig                       |
| `DoorProductionInactive` | Slår av dør fra rest-funksjon                  |
| `CheckReturnFeeder`      | Behandler returfeeder ved neste poll           |
| `ProductionStarted`      | Internt flagg for pågående kjøring             |

---

#### ⏱️ Tidsstempler og tellere

| Nøkkel              | Type      | Beskrivelse                                 |
|---------------------|-----------|---------------------------------------------|
| `skipCount`         | int       | Antall hoppede ordrer pga. lagermangel      |
| `skipCountUpdated`  | DateTime  | Tidspunkt for siste `skipCount`-endring     |
| `ProductionCycleStart` | DateTime | Starttid for nåværende produksjonssyklus    |
| `ProductionStartTime`  | DateTime | Når produksjonen faktisk startet            |
| `NumberOfDoors`     | int       | Antall produserte dører                     |


#### 📦 Listbaserte nøkler (må initieres!)

🔥 Må settes til `new List<>()` ved oppstart – ellers `NullReferenceException`

| Nøkkel                      | Type                            |
|-----------------------------|----------------------------------|
| `ManualProductionItems`     | `List<ManualProductionItem>`     |
| `ManualLiftLoadingList`     | `List<LiftLoadingInput>`         |
| `ManualLiftUnloadingList`   | `List<LiftUnloadingInput>`       |
| `ProductionStorageTransfers`| `List<ProductionStorageTransfer>`|
| `signalsToMonitor`          | `List<RobotSignal>`              |

---

#### Eksempel (initiering i oppstart – pseudo-C#)

```csharp
_settingsService.SetSetting("ManualProductionItems", new List<ManualProductionItem>());
_settingsService.SetSetting("signalsToMonitor", new List<RobotSignal>());
```
⚠️ LiftInactive og DoorProductionInactive kan være misvisende hvis de ikke brukes i praksis.

### 📡 Standard signaler (fra `ProductionExecutionService`)

| Robot | IP         | Signaler                                                                 |
|-------|------------|--------------------------------------------------------------------------|
| R1    | 10.5.15.21 | `DOF_ConfirmFeederReturnInPos`, `DOF_ActiveMessages`, `DOF_Port1Start`, `DOF_Port3Start`, osv. |
| R2    | 10.5.15.73 | `DOF_ConfirmLeaveFeederOut`, `DOF_PrintLabel`, `DOF_OrderStarted`, `DOF_OrderDone`, osv.     |

> 🔄 Disse signalene overvåkes kontinuerlig av `ProductionExecutionService` og brukes til synkronisering og filoverføring.


## 📦 Persistens og datakilder

Systemet bruker ingen tradisjonell database.  
All dataflyt skjer enten via:

- 🔁 Runtime state i minne (`ISettingsService`)
- 🔗 Eksterne API-er (D365, WMS)
- 📁 Filgenerering og FTP
- 🧾 Konfigurasjonsfiler (`appsettings.json`)

---

### 🧠 Runtime-state (ISettingsService)

- Brukes som nøkkelbasert in-memory key-value store.
- Inneholder dynamiske lister, flagg og telleverdier for kjørende produksjon.
- Leses og oppdateres kontinuerlig av `ProductionExecutionService`, API-er og bakgrunnsprosesser.
- Data går tapt ved restart – må initialiseres via API eller `appsettings.json`.

**Eksempler på nøkler:**

| Nøkkel                  | Type                     | Beskrivelse |
|-------------------------|--------------------------|-------------|
| `ManualProductionItems` | `List<ManualProductionItem>` | Manuell produksjon |
| `signalsToMonitor`      | `List<string>`           | Signalovervåking |
| `skipCount`             | `int`                    | Hopper ordrer ved lagerfeil |
| `LiftInactive`          | `bool`                   | Deaktiverer heisstyring |
| `DoorProductionInactive`| `bool`                   | Slår av restpanel-produksjon |

> ⚠️ Uinitialiserte lister kaster `NullReferenceException`. Alle liste-nøkler må eksplisitt initieres før bruk.

---

### 🔗 Eksterne systemer

#### Microsoft Dynamics 365 (D365)

- Hovedkilde for produksjonsordre og materialbehov.
- Tjenester:
  - `ID365DataProcessingService` (lese)
  - `ID365ActionProcessingService` (skrive)

**Typiske kall:**
- `GetProductionListAsync(...)`
- `GetPickList(...)`
- `StartProduction(...)`
- `CreateTransferJournal(...)`
- `PostTransferJournal(...)`

#### WMS-system (lager og tray)

- Brukes for beholdning, tray-lokasjoner og replenishment.
- Kommunikasjon via `IWarehouseManagementService`
- Understøtter både manuell og automatisk lasting/lossing.

---

### 📁 Filbasert persistens (.csv og .lbs)

| Type     | Format | Generert av                       | Brukes av   | Sti (eksempel)                           |
|----------|--------|-----------------------------------|-------------|------------------------------------------|
| Robotfil | `.csv` | `RobotFileProcessingService`      | Robot1/2    | `\\lob-file01\Produksjon\515\csvRob1\`   |
| Fresefil | `.lbs` | `MillingMachineFileProcessingService` | Fresemaskin | `\\lob-file01\Produksjon\515\FomCam\`    |

- Filer sendes via FTP til IP-adresser definert i `AppEnvironmentConfig.FilePaths`
- Filnavn genereres dynamisk per ordre og timestamp

---

### 🗂️ Konfigurasjonsfiler

- `appsettings.json` gir statisk konfigurasjon via `AppEnvironmentConfig`.
- Inneholder: D365/WMS credentials, logging-nivå, og filstier for utdata.
- Eksempel:
  - `FilePaths.Robot1CsvOut` → mappe for `.csv`-filer
  - `FilePaths.MillingCsvOut` → mappe for `.lbs`-filer

> 📁 Utdata skrives som `.csv` og `.lbs`-filer til mapper konfigurert i `appsettings.json`

---

### 🧾 Lokale filer (ikke relasjonsdatabase)

- Ingen relasjonsdatabase brukes i dag.
- Persistens skjer via:
  - `.csv`-filer generert av `RobotFileProcessingService`
  - `.lbs`-filer generert av `MillingMachineFileProcessingService`
  - Loggfil (`InstallLog.txt`) under installasjon (`AppContext.BaseDirectory`)

---

### ❗ Mangler og vurderinger

- `ISettingsService` holder ikke data over restarts → kan skape uforutsigbarhet.
- Enkelte signalverdier bør kunne persists mellom oppstarter for robusthet.
- Vurdering: En lettvekts database (f.eks. LiteDB eller SQLite) kan være aktuell.




## 📊 Logging & overvåkning

Systemet benytter Microsoft.Extensions.Logging for applikasjonslogging, og støtter både konsoll og filbasert loggføring.

---

### 🧾 Loggutdata

| Logger                        | Type     | Beskrivelse                                         |
|------------------------------|----------|-----------------------------------------------------|
| `ILogger<T>`                 | Console  | Standard logging til terminal ved utvikling         |
| `EventLogLoggerProvider`     | Windows EventLog | Brukes for Windows-tjeneste i produksjon         |
| `InstallLog.txt`             | Fil      | CLI-installasjonslogg (`AppContext.BaseDirectory`) |

> 🛠️ Loggnivå (f.eks. `Information`, `Warning`, `Error`) settes i `appsettings.json` via `AppEnvironmentConfig.Logging.Level`

---

### 📡 Eksempel – logging i tjeneste

```csharp
_logger.LogInformation("Windows Service is starting.");
_logger.LogError("Feil under produksjonsstart: {message}", ex.Message);
```
### 🔭 Overvåkning (potensiale)

Per dags dato finnes det ikke innebygd overvåkning eller health checks, men følgende er mulig:

- 🩺 **Health endpoints** kan legges til i WebAPI (via `MapHealthChecks()`).
- 📊 **Integrasjon mot Prometheus / Grafana** via middleware.
- 📬 **Varsling via e-post / Teams** ved kritiske feil (loggnivå: `Error` / `Critical`).

### 💡 Anbefalinger

- Logg til fil i produksjon for sporbarhet og support.
- Vurder å introdusere strukturert logging (f.eks. Serilog + JSON).
- Vurder å visualisere drift via dashboard.

## 🧪 Tester & CI/CD

Systemet støtter lokal test og simulering via flagg og API-endepunkter – men har foreløpig **ingen integrert CI/CD-pipeline**.

---

### 🧪 Lokal testing

- Aktiveres ved å sette `AppEnvironmentConfig.Testing = true` i `appsettings.json`
- Ruter alle digitale signaler til `TestSignalsList`
- Muliggjør manuell verdiendring via `SimulationController`
- Fysiske robot-/heiskommunikasjon deaktiveres fullstendig

**Eksempel på signalsetting via curl:**

```bash
curl -X POST http://localhost:4000/Simulation/SignalValue/DOF_ConfirmFeederReturnInPos \
  -H "Content-Type: application/json" \
  -d true

```
### 🧪 Testsignaler og testdata

| Type              | Controller / tjeneste      | Beskrivelse                              |
|-------------------|----------------------------|------------------------------------------|
| Simulerte signaler| `SimulationController`      | Leser og setter testverdier              |
| Testsignal-mapping| `TestSignalsList`           | Holder signalverdi per nøkkel            |
| Testdata          | `ManualProductionItems`, `LiftLoadingInput` | Simulerer ordrer, lasting/lossing        |

> ⚠️ Testdata må initialiseres manuelt via API eller `ISettingsService` ved oppstart.

---

### ⚙️ CI/CD-status (per nå)

- Ingen pipeline for bygg, test eller deployment (f.eks. GitHub Actions, Azure DevOps)
- Produksjonsdeploy skjer manuelt:
  - `dotnet publish`
  - CLI-installasjon av Windows Service (`/Install`)
- Ingen enhetstester oppdaget i repo
- Ingen automatisert validering eller linting

---

### 💡 Anbefalinger

- ✅ Legg til enhetstester for nøkkeltjenester (`RobotFileProcessingService`, `WarehouseManagementService`, osv.)
- ✅ Bruk **GitHub Actions** for:
  - Automatisk bygg ved push
  - Testkjøring og feilrapportering
- ✅ Automatiser **deploy til testmiljø** via PowerShell eller CLI-skript
- ⏳ Vurder mocking/abstraksjon for eksterne systemer (D365, WMS) ved test

- Automatiser deployment av Windows-tjeneste i testmiljø

## Feilsøking (kjente fallgruver)

Denne seksjonen beskriver vanlige problemer og hvordan de kan identifiseres og løses.

### 🚫 Automatisk kjøring starter ikke

**Symptom:** Produksjonen starter ikke automatisk selv om batch er tilgjengelig.

✅ Sjekk:

- `Settings["StartAutomaticExecution"] == true`?
- Signal `DOF_OkToSendNewCsvFilesRob1` == true?  
  → IP: `10.5.15.21`  
  → I testmiljø: `TestSignalsList.DOF_OkToSendNewCsvFilesRob1`
- Logger viser:  
  _"Waiting for robot1 ready signal..."_

### 📭 Robot2 eller fresefil ikke generert

**Symptom:** Ingen CSV/LBS-filer blir skrevet for frese og/eller Robot2.

✅ Sjekk:

- `DOF_OkToSendNewCsvFilesRob2 == true`? (R2: `10.5.15.73`)
- `TestSignalsList.DOF_OkToSendNewCsvFilesRob2` satt til `true` i test?
- `ProcessMillingCellData` ble kalt etter Robot1?
- Logger viser:  
  _"Waiting for Robot2 ready signal..."_

### 📉 Ikke nok seksjoner i lager

**Symptom:** Automatisk produksjon hopper over batcher.

✅ Sjekk:

- Logger viser: _"Not enough sections… Please refill storage."_
- `skipCount` øker kontinuerlig
- Sjekk `WarehouseManagementService.CheckCapacityLiftAsync(...)`


### 💥 Null-lister i runtime state

**Symptom:** Systemet feiler med `NullReferenceException` ved første kall.

✅ Sjekk:

- Har `ISettingsService` blitt initialisert ved oppstart?
- Alle listebaserte nøkler må settes:
```csharp
  _settingsService.SetSetting("ManualProductionItems", new List<List<ManualProductionItem>>());
```

### ✅ Testing vs produksjon forvirring
**Symptom:** Systemet sender ikke signaler eller filer i produksjon.  

🔍 **Sjekk:**  
- `AppEnvironmentConfig.Testing == true` ?  
- Da er `SimulationController` aktiv og signaler går ikke ut til roboter  
- Endre `Testing = false` for produksjon  


### ✅ Tray-posisjon ikke oppdatert
**Symptom:** Robot prøver å hente/laste til feil posisjon.  

🔍 **Sjekk:**  
- `UpdateStoragePickPositionAsync()` ble kalt?  
- Robotvariablene `nSourceStorage`, `nSourcePosition` stemmer?  
- Sjekk logg for mismatch mellom `OnHandQuantity` og robotposisjon  

### ✅ TransferJournal oppdateres ikke
**Symptom:** D365 transfer feiler, eller journalposter mangler.  

🔍 **Sjekk:**  
- `WarehouseManagementService.ReplenishStockAsync(...)` feilet?  
- Var batchnummer feil/utgått?  
- Logger viser:  
  *“Error in CreateTransferJournalEntriesAsync...”*  

💡 Tips Bruk `/Operations/GetFeedback` for å hente siste loggutdrag når feilsøking pågår.  

## 🧭 Veikart / mangler

### 🚧 Kjente hull og forbedringsområder

| Område                | Status         | Kommentar |
|-----------------------|----------------|-----------|
| 🧪 Testing            | ❌ Mangler      | Ingen testprosjekter, teststrategi eller mockingrammeverk er implementert |
| ⚙️ CI/CD              | ❌ Ufullstendig | Ingen GitHub Actions eller automatisert deploy er satt opp |
| 🔍 Observabilitet    | 🔸 Delvis       | Logger finnes, men det mangler strukturert overvåkning og alarmering |
| 📊 Dashboard         | ❌ Mangler      | Ingen visuell overvåkning (dashboard) er tilgjengelig |
| 📄 Dokumentasjon     | 🔸 Delvis       | Teknisk arkitektur og domene er dokumentert, men miljøoppsett og feilsøking er lite dekket i kodebasen |
| 🧩 Komponentdeling   | ✅ Fullført     | Tjenestelag er godt separert fra domenelag og hostmiljø |
| ⚠️ Robusthet         | 🔸 Begrenset    | Runtime state er sårbar for restarts; mangler vedvarende lagring og validering |

---

### 🗺️ Anbefalt veikart (neste steg)

1. ✅ Fullfør README med domenemodell og driftsdetaljer *(📌 du er her nå!)*
2. 🧪 Introduser enhetstester (`xUnit`) for sentrale komponenter (f.eks. `RobotFileProcessingService`)
3. ⚙️ Sett opp GitHub Actions for CI/CD og testkjøring
4. 📤 Automatiser Windows-tjeneste-deploy via PowerShell eller CLI-skript
5. 🔭 Implementer observabilitet med Serilog, HealthChecks og eventuelt Prometheus/Grafana

---

### 📌 Merk

Systemet er godt strukturert og lett å utvide, men **mangler flere grunnleggende driftskomponenter** som:
- Automatisert testing og deploy  
- Vedvarende state-lagring  
- Observasjon og innsikt i kjørende system  

Disse bør prioriteres før full produksjonssetting.



## Bilag: Signaler som overvåkes (eksempler)

`signalsToMonitor` er en liste over digitale signaler systemet kontinuerlig overvåker.  
Disse signalene brukes til å koordinere flyt mellom backend og roboter (R1 og R2), samt fres.


### 🚦 Typiske DOF-signaler

| Robot | IP         | Signaler |
|-------|------------|----------|
| **R1** | 10.5.15.21 | `DOF_ConfirmFeederReturnInPos`, `DOF_ActiveMessages`, `DOF_Port1Start`, `DOF_Port2Start`, `DOF_Port3Start`, `DOF_UpdatePositionData`, `DOF_ConfirmLeaveElement`, `DOF_SendLift1Command`, `DOF_SendLift2Command` |
| **R2** | 10.5.15.73 | `DOF_ConfirmLeaveFeederOut`, `DOF_PrintLabel`, `DOF_ActiveMessages`, `DOF_OrderStarted`, `DOF_OrderDone`, `DOF_MeasurementsConfirmed`, `DOF_ConfirmLeaveScrap` |

> Disse signalene blir definert og satt i `signalsToMonitor`-listen i `ProductionExecutionService`.  
> Systemet leser disse hver 2. sekund i hovedløkken.


### 🧪 I testmiljø

Når `AppEnvironmentConfig.Testing == true`, brukes `TestSignalsList` i stedet for ekte IO.

Disse verdiene kan endres via `SimulationController`:

```bash
curl -X POST http://<host>/Simulation/SignalValue/DOF_OkToSendNewCsvFilesRob1 \
  -H "Content-Type: application/json" -d true
```
🛠 Eksempler på testverdier:
DOF_OkToSendNewCsvFilesRob1, DOF_OrderDone, strLiftCommand, numElementLength

📌 Anbefaling

Ved innføring av nye roboter eller signaler:

- Oppdater signalsToMonitor i oppstarten (eks: Startup.cs)

- Dokumenter nye signalnavn, type (bool/string/double) og funksjon
- 
## Bilag: Forretningsregler (utdrag)


Systemet inneholder flere forretningsregler som styrer hvordan paneler håndteres i produksjon, spesielt ved splitting, resthåndtering og kassettlogikk.

### ✂️ Precut-regel

- Hvis `Precut < 754 mm` → klassifiseres som `Scrap` (skrot)
- Hvis `Precut >= 754 mm` → klassifiseres som `Finished` (ferdig dørpanel)

### 🪵 Håndtering av rester (restlengde)

Etter fresing vurderes gjenværende lengde (rest) slik:

| Lengde (mm)     | Handling |
|-----------------|----------|
| `≥ 2400 mm`     | Returneres til lager |
| `1200 – 2399 mm`| Returneres manuelt eller automatisk |
| `< 1200 mm`     | Skrotes |


### 🏗️ Dør fra rest

Dersom `Settings["DoorProductionInactive"] == false`, vil systemet:

- Automatisk opprette en ny produksjonsordre i D365
- Bruke restlengder mellom `754 mm` og `2399 mm`
- Gjelder kun hvis rest er nøyaktig 754-modulbasert (f.eks. 1508, 2262, ...)


### 🧰 KASSETT-dører

Ved produksjon av kassettdører gjelder spesielle beregninger:

- **Precut-beregning** skjer med egen metode: `CalculatePreCutKassett()`
- **Klemplassering** beregnes med: `CalculateClampsUsed()`
- Enkelte makroer utelates i fresing, f.eks. `HaO`, `HaF`, `T40BLV`, `I40BLH`, ...


### ♻️ Lagerpåfylling

Hvis beholdningen i produksjonslager (`I*`) faller under `5 stk`:

- `WarehouseManagementService.CheckProductionStorageInventoryAsync()` oppretter automatisk en transferjournal
- Systemet forsøker å fylle lokasjonen med maks `18 stk`, eller så mye som er tilgjengelig

### 🔀 Plukklokasjonsvalg (PickLocation)

Når `GetPickLocationsAsync()` kalles:

- Systemet vurderer `EGDoorWidthId`, `ProductSizeId` og `EGCutTemplateId`
- Prøver å finne et sted med nok råmateriale til å dekke flere BOM-linjer (opp til 6)
- Bruker `Fres`, `ReturnLaneLocationID` eller `Kxx` som fallback


📌 Merk: Disse reglene er hardkodet i logikken og bør synkroniseres med operasjonelle prosedyrer.

## 🧪 Eksempelflyt (test)

Denne seksjonen viser hvordan systemet kan testes lokalt i **simuleringsmodus**, uten fysisk tilkobling til roboter, fres eller D365.

---

### 🔧 Forutsetninger

- `AppEnvironmentConfig.Testing = true` i `appsettings.json`
- `SimulationController` er aktivert
- Signaler og data går **ikke fysisk** til maskiner eller roboter
- Systemet kjøres med:

```bash
dotnet run --project LOBGarageDoorProductionControllerService

 ```
### 🧪 Eksempel på testscenario

#### 1. Start automatisk kjøring

```bash
curl -X POST http://<host>/Settings/StartStop \
  -H "Content-Type: application/json" -d true

```
#### 2. Simuler at roboter er klare
```bash
curl -X POST http://<host>/Simulation/SignalValue/DOF_OkToSendNewCsvFilesRob1 \
  -H "Content-Type: application/json" -d true
curl -X POST http://<host>/Simulation/SignalValue/DOF_OkToSendNewCsvFilesRob2 \
  -H "Content-Type: application/json" -d true
```
#### 3. Legg inn en manuell produksjonsbatch
```bash
curl -X POST http://<host>/Operations/ManualProduction \
  -H "Content-Type: application/json" \
  -d '[
        [
          {
            "ProdId": "116024",
            "Rawlength": "3050",
            "Precut": "2600",
            "Endspacing": "100",
            "LineNumber": "1",
            "Location": "K01AA01",
            "Port": "Port1",
            "Reproduction": false,
            "QtyScheduled": 2,
            "QtyStarted": 0
          }
        ]
      ]'

```
#### 4. Simuler signaler underveis (valgfritt)
```bash
curl -X POST http://<host>/Simulation/SignalValue/DOF_OrderDone \
  -H "Content-Type: application/json" -d true
```
#### 5. Hent logg / status
```bash
curl http://<host>/Operations/GetFeedback
```
> 📌 **Tips**  
> Bruk `SimulationController` til å sette testverdier:

| Type           | Endpoint                                  |
|----------------|--------------------------------------------|
| Boolsk signal  | `/Simulation/SignalValue/{navn}`           |
| Numerisk verdi | `/Simulation/VariableValue/{navn}`         |
| Tekstverdi     | `/Simulation/StringVariableValue/{navn}`   |

> ℹ️ Hvis ingenting skjer: sjekk loggen via `/Operations/GetFeedback`

---

🍃 **Med dette kan du teste hele produksjonsløpet uten faktisk utstyr – ideelt for utvikling og feilsøking!**
