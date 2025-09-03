# LOB Garage Door – Produksjonskontroller

Styrer produksjonen av garasjeporter på fabrikkgulvet: plukk/lagring, løfteanlegg, fresecelle og to robotceller.  
Systemet integrerer mot Dynamics 365 F&O (produksjonsordre, BOM, lager), samt roboter og fres via signaler og CSV-filer.  
Kjøringen orkestreres av en ASP.NET Core-basert bakgrunnstjeneste, støttet av et HTTP API for operatørstyring.

**Status:** Utkast – oppdatert med verifiserte kilder fra denne gjennomgangen.  
**Bekreftet grunnlag (filer gjennomgått):**

Application/Services/ProductionExecutionService.cs

LOBGarageDoorProductionControllerService/Controllers/OperationsController.cs

LOBGarageDoorProductionControllerService/Controllers/SettingsController.cs

LOBGarageDoorProductionControllerService/Controllers/SimulationController.cs

Domain/Entities/Lift/LiftLoadingInput.cs, LiftUnloadingInput.cs

Domain/Entities/Production/ManualProductionItem.cs, ProductionStorageTransfer.cs

Application/Interfaces/ISettingsService.cs

Application/General/TestSignalsList.cs

Program.cs (test/debug-støtte)

Program.cs, MainService.cs, WebAPIService.cs, AppEnvironmentConfig.cs, ILoggingService.cs, ProductionSchedulingService.cs  
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
        Exec["ProductionExecutionService\n(orkestreringsloop)"]
        Sched["ProductionSchedulingService\n(kø-generering)"]
        Mill["MillingMachineFileProcessingService"]
        Rob["RobotFileProcessingService"]
        WMS["WarehouseManagementService"]
        D365["D365Data/Action Services"]
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
```

### Oversikt

Systemet består av to hovedelementer:

1. **Windows-tjeneste (ServiceHost)**  
   - Starter `MainService` og `WebAPIService`  
   - Kjøres via `Program.cs` og CLI-wrap installasjonslogikk

2. **HTTP API (port 4000)**  
   - Kontrollere: `OperationsController`, `SettingsController`, `SimulationController`  
   - Swagger aktivert  
   - CORS-policy tillater lokal utvikling

---

### Bakgrunnsprosesser

`MainService` starter flere tråder parallelt:

- `ProductionExecutionThreadJob` → Starter `ProductionExecutionService.ProcessUntilCanceled(...)`
- `ProductionSchedulingThreadJob` → Kjører `RunScheduling()` hvert 5. minutt
- `ProductionStorageSupervisionThreadJob` → Overvåker lagerstatus og replenishment
- `RobotToCloudMessageThreadJob` → Leser robot-signalstatus (i prod)
- `TestSignalsThreadJob` → Leser simulerte signaler (i testmodus)

📌 Aktiveres betinget basert på `AppEnvironmentConfig.Testing`


### Applikasjonslag og domene

Applikasjonslaget inneholder spesialiserte tjenester for:

- CSV-generering: `RobotFileProcessingService`
- Freseprogrammer: `MillingMachineFileProcessingService`
- Lagerstyring: `WarehouseManagementService`
- Runtime state: `ISettingsService`
- Logging: `ILoggingService`


### Integrasjoner

- **D365**: Produksjonsordre, lager, journalføring
- **WMS**: Tray-posisjoner og beholdning
- **Robot 1 / 2**: CSV og digitale signaler (IP: 10.5.15.21 / 10.5.15.73)
- **Fres**: `.lbs`-filer med makroer


🧠 Arkitekturen er modulær, testvennlig og tydelig adskilt mellom host, applikasjon og integrasjoner.


### Roller og ansvar (utdrag)


### Roller og ansvar (utdrag)

#### `ProductionExecutionService`

Sentralt orkestreringsloop (2s intervall).  
- Leser state fra `ISettingsService`
- Henter produksjonsordre (manuell/automatisk)
- Genererer CSV-filer for Robot1/Robot2 og fresecelle
- Håndterer el-signal-synkronisering via `IRobotOutboundMessageProcessingService`

➡️ Koordinerer hele produksjonsflyten via DI – inkludert:  
- `WarehouseManagementService`  
- `RobotFileProcessingService`  
- `MillingMachineFileProcessingService`

#### `HTTP API`

Eksponerer **`Operations`**, **`Settings`** og **`Simulation`** kontrollere.  
- Støtter operatørkommandoer  
- Statusflagg  
- Manuell lasting/lossing  
- Testverdier i utviklingsmodus  
- Swagger aktivert  
- Kjøres via `WebAPIService` på port `4000`



#### `Runtime state (Settings)`

Lages i minne via `ISettingsService`, tilgjengelig via nøkkelbasert API.  
⚠️ Listebaserte nøkler må initieres ved oppstart, ellers vil systemet kaste `NullReferenceException`

- **Boolean flagg:**  
  - `StartAutomaticExecution`  
  - `DoorProductionInactive`  
  - m.fl.

- **Lister:**  
  - `ManualProductionItems`  
  - `LiftLoadingKassettList`  
  - `ProductionStorageTransferList`  
  - m.fl.


#### `Testsignaler`

`TestSignalsList` gir simulerte signaler for utvikling og test  
- Styrt via `SimulationController`

**Typiske signaler:**  
- `DOF_OkToSendNewCsvFilesRob1`  
- `DOF_OrderDone`  
- `strLiftCommand`  
- `numElementLength`  
*(Brukes både i test og prod – styres via TestSignalsList i testmodus)*



#### `WarehouseManagementService`

Styrer produksjonslageret, tray-/heislogikk og D365-integrasjoner for overføringsjournaler.  
- Velger plukklokasjoner  
- Validerer lagerbeholdning  
- Genererer `Robot1Input` for lasting og lossing

Bruker `GetPickLocationsAsync(...)` for optimal råvareutnyttelse og segmentering.


#### `RobotFileProcessingService`

Genererer CSV-filer for **Robot1** og **Robot2** basert på produksjonsdata og plukk-informasjon.  
- Skriver til konfigurerte filbaner (via `AppEnvironmentConfig`)  
- Laster opp via FTP  
- Håndterer også opprydding og parsing av kasserte CSV-filer  
⚠️ Forutsetter synkronisert tilgang ved samtidige oppgaver


#### `MillingMachineFileProcessingService`

Skriver `.lbs`-filer for fresecelle.  
- Basert på data fra `D365DataProcessingService`  
- Genereres detaljert fresedata: `O:CIRCLE`, `O:CUT`, `O:MACRO`, `C:` (klemmer), osv.  
- Støtter høydekompensasjon (`HeightCompensationMillingFile`)  
- Konfliktfri klemmeplassering


#### `D365-integrasjon`

- **Ordrehåndtering:**  
  - `StartProduction(...)`  
  - `CreatePurchaseOrder(...)`

- **Lagerstyring:**  
  - `UpdateWMSLocation(...)`  
  - `GetPickList(...)`  
  - `AdjustOnHandQty(...)`

- **Journalføring:**  
  - `CreateTransferJournal(...)` (brukes ved lagerpåfylling fra WMS til produksjonslokasjon)  
  - `UpdateTransferJournal(...)`  
  - `PostTransferJournal(...)`


#### `Robot-integrasjon`

- **CSV-generering:** via `RobotFileProcessingService`  
- **Direkte signal-/variable-skriving:** via `IRobotOutboundMessageProcessingService`  
- Krever *mastership per IP* for å sette verdier



## Teknologier og avhengigheter

### .NET / ASP.NET Core

Systemet er bygget på .NET 7 med ASP.NET Core.  
Brukes både til:

- Hosted Service (`ProductionExecutionService`, `MainService`)
- HTTP API (`OperationsController`, `SettingsController`, `SimulationController`)
- CLI/Windows Service-installasjon via `Program.cs` og `CliWrap`

---

### Dynamics 365 F&O

Brukes til:

- **Ordrehåndtering:**
  - `StartProduction(...)`
  - `CreatePurchaseOrder(...)`

- **Lagerstyring:**
  - `UpdateWMSLocation(...)`
  - `GetPickList(...)`
  - `AdjustOnHandQty(...)`

- **Journalføring:**
  - `CreateTransferJournal(...)`
  - `UpdateTransferJournal(...)`
  - `PostTransferJournal(...)`

Integrasjonen gjøres via `ID365DataProcessingService` og `ID365ActionProcessingService`.

---

### WMS (Warehouse Management System)

Systemet kommuniserer med WMS for:

- Tray-posisjoner
- Beholdningskontroll
- Retur-lokasjoner
- Lagerflytting via heis og robot

---

### CSV / filutveksling (Robot)

- Robot1 og Robot2 mottar `.csv`-filer
- Filene genereres av `RobotFileProcessingService`
- Eksempelbaner: `\\lob-file01\Produksjon\515\csvRob1`, `csvRob2`
- Filene overføres via **FTP** til IP-adressene:
  - **Robot1 →** `10.5.15.21`
  - **Robot2 →** `10.5.15.73`
- FTP-legitimasjon og filbaner settes i `AppEnvironmentConfig.FilePaths`

---

### Fresemaskin – .lbs-filer

- Fresecellen bruker et proprietært tekstformat (`.lbs`)
- Filene inneholder kommandokoder som `O:CIRCLE`, `O:CUT`, `O:MACRO`, `C:` osv.
- Genereres via `MillingMachineFileProcessingService`
- Format følger engelsk lokaliseringsstandard (`en-GB`) for desimaler
- Lagringsbane: `\\lob-file01\Produksjon\515\FomCam\`

---

### Digitale IO-signaler

- Roboter og fres samhandler via digitale signaler (boolske)
- Håndteres gjennom `IRobotOutboundMessageProcessingService`

Eksempler:
- `DOF_OkToSendNewCsvFilesRob1`
- `DOF_ConfirmFeederReturnInPos`
- `DOF_ConfirmLeaveScrap`
- `DOF_Port1Start`, `DOF_Port2Start`

⚠️ Verdiene rutes til `TestSignalsList` dersom testmodus er aktivert.

---

### Runtime-konfigurasjon og state

Systemet benytter to lag med konfigurasjon:

#### `AppEnvironmentConfig` (via `IOptions<>`)

- Inneholder statiske verdier som:
  - `Testing` → aktiverer simulasjonsmodus
  - `FilePaths` → filbaner til CSV og LBS
  - `Logging.Level` → loggnivå
- Konfigureres i `appsettings.json`

#### `ISettingsService`

- Holder runtime-state i minne
- Tilgjengelig via nøkkel-streng (eks: `"ManualProductionItems"`)
- Brukes til både lister og flagg (`bool`, `int`, `DateTime`, `List<T>`)
- ⚠️ Alle liste-nøkler må initieres ved oppstart

---

### Testmiljø og simulering

- Når `AppEnvironmentConfig.Testing == true`:
  - Signaler rutes til `TestSignalsList`
  - `SimulationController` åpnes for manuell verdi-endring
  - Ingen fysisk IO sendes til roboter eller fres
- Brukes aktivt under utvikling og integrasjonstesting

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

- Utføres ved å lese signalet `DOF_OkToSendNewCsvFilesRob1` (R1: `10.5.15.21`)
- I testmiljø leses verdien fra `TestSignalsList.DOF_OkToSendNewCsvFilesRob1`

#### Manuell produksjon

- Dersom `Settings["ManualProductionItems"]` inneholder batcher →  
  `GetManualProduction()` henter data fra D365 og WMS  
- DTO: `ManualProductionItem`

#### Automatisk produksjon

- Hvis `Settings["StartAutomaticExecution"] == true` og robot er klar →  
  `GetProduction()` henter neste tilgjengelige ordre
- Ved manglende lagerkvantum logges:  
  *"Not enough sections… Please refill storage."*  
  - `skipCount` økes (lagres i runtime state)

---

### B) Kjøre en ordre (`ProcessProduction`)

- Ordrestatus oppdateres i D365 (`ProductionFunctionsService.UpdateProductionStatus(...)`)
- For hver `PickLocation`:
  - Lagerlokasjon identifiseres (`WarehouseManagementService.GetPickLocationsAsync(...)`)
  - `LiftService.CreateLiftString(...)` genererer løftekommando
  - `Robot1Input` bygges og skrives til CSV via `RobotFileProcessingService.CreateRobot1File(...)`

---

### C) Fresecelle og Robot2 (`ProcessMillingCellData`)

#### Signal-synkronisering

- Krever `DOF_OkToSendNewCsvFilesRob2 == true` (eller testsignal)

#### Panel-deling

- Råmaterialet splittes i `ElementPart` med regler for **Finished | Return | Scrap**

#### Kasettregler

- Egne metoder for precut og klemmer

#### Artikkelrester

- ≥ 2400 mm → returneres til lager  
- 1200–2399 mm → return manuelt eller automatisk  
- < 1200 mm → skrotes

#### Dør fra rest (opsjonelt)

- Hvis `Settings["DoorProductionInactive"] == false` og restlengde er 754–2399 mm →  
  ny dørordre opprettes i D365

#### Filskriving

- Fres: `MillingMachineFileProcessingService.CreateMillingFile(...)` → `.lbs`  
- Robot2: `RobotFileProcessingService.CreateRobot2File(...)` → `.csv`

---

### D) Støtteflyter og vedlikehold

#### Løfteoperasjoner

- Manuell lasting/lossing (`LiftLoadingInput`, `LiftUnloadingInput`) håndteres via API og state
- Etterpå genereres `RobotFile` for heis

#### Produksjonslagerpåfylling

- `WarehouseManagementService.CheckProductionStorageInventoryAsync()` identifiserer lokasjoner < 5 stk
- Oppretter overføringsjournal i D365 og genererer flyttinger

#### ReplenishStock (lagerflytting via heis/palleplass)

**Scenarier:**

- `loading` : Flytting til lager via heis → Robot1 CSV  
- `unloading` : Utlasting fra heis → Robot1 CSV  
- `LiftLoadingKassettList` : Kassett lasting → Robot1 CSV  
- `productionStorage` : Fullfører journal og poster i D365

→ Bruker `ProductionStorageTransfer` fra state

---

#### Runtime state

- Alle operasjoner lagres midlertidig i `ISettingsService`  
- Initielt må alle lister være tomme for å unngå `NullReferenceException`

#### Robotvariabler

- `WarehouseManagementService.UpdateStoragePickPositionAsync()` leser `nSourceStorage`, `nTargetStorage` osv. fra robot (R1)
- Synkroniserer med faktiske lagerkvanta i D365
- Krever `MastershipRequest` for å skrive verdier tilbake


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


Eksempel: Sett DOF_OkToSendNewCsvFilesRob1 til true
curl -X POST http://<host>/Simulation/SignalValue/DOF_OkToSendNewCsvFilesRob1 \
-H "Content-Type: application/json" -d true

📌 Merk:

- Alle API-operasjoner interagerer med `ISettingsService`.  
- Listebaserte nøkler må initieres med tomme lister ved oppstart.  
- `WarehouseManagementService.ReplenishStockAsync(...)` og `CheckProductionStorageInventoryAsync()`  
  er kjerneprosesser bak mange av disse API-endepunktene.  


## Installasjon & kjøring

> 🛠 Status: Delvis kjent – `AppEnvironmentConfig` og `Program.cs` er nå delvis kartlagt.  
> Vi har verifisert at systemet starter som Windows-tjeneste og bruker `CliWrap` for service-installasjon.

Her er realistiske steg for både lokal testing og installasjon som tjeneste.

---

### 📦 Krav

- [.NET SDK 7.0+](https://dotnet.microsoft.com/download)
- Tilgang til produksjonsnettverket og IP-er:
  - Robot1: `10.5.15.21`
  - Robot2: `10.5.15.73`
- Tilkobling mot D365 og WMS må være konfigurert i `appsettings.json`
- Konfigurasjonsfil må inneholde gyldige verdier for:
  - `D365.BaseUrl`, `Tenant`, `ClientId`, `ClientSecret`
  - `WMS.BaseUrl`
  - `FilePaths.*`

---

### 🧪 Lokalt testmiljø

- Sett `AppEnvironmentConfig.Testing = true` i `appsettings.json`
- Bruk `SimulationController` til å sette testverdier og signaler manuelt
- Ingen faktiske signaler sendes til roboter eller fres

---

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
\\lob-file01\Produksjon\515\FomCam\

### 🔐 Avhengigheter

- Konfigurasjonen må settes **før første oppstart** – ellers feiler `ISettingsService` og signalinitiering
- Sørg for at `appsettings` ikke inneholder tomme eller manglende `List<>`-nøkler


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


## Database

## 📦 Persistens og datakilder

Systemet lagrer og henter data fra følgende kilder:

---

### 🧠 Runtime-state (ISettingsService)

- Brukes som en nøkkelbasert in-memory key-value store.
- Inneholder dynamiske lister, flagg og telleverdier for kjørende produksjon.
- Leses og skrives kontinuerlig av `ProductionExecutionService`, API-endepunkter og bakgrunnstråder.
- Data går tapt ved restart – må initialiseres på nytt via API eller testmiljø.

Eksempler på nøkler:

- `ManualProductionItems`, `signalsToMonitor`
- `ProductionStartTime`, `skipCount`, `LiftInactive`, osv.

---

### 🏭 Eksterne systemer

#### 🔗 Microsoft Dynamics 365 (D365)

- Primærkilde for produksjonsordre og materialbehov.
- Tjenester: `ID365DataProcessingService`, `ID365ActionProcessingService`
- Brukes til:
  - `GetProductionListAsync(...)`
  - `GetPickList(...)`
  - `UpdateProductionStatus(...)`
  - `CreateTransferJournal(...)`, osv.

#### 📦 WMS-system (lager og tray)

- Brukes for tray-posisjoner, lagerbeholdning og replenishment.
- API-kommunikasjon via `IWarehouseManagementService`
- Støtter manuell lasting/lossing og robotlogikk for tray-heis.

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

Systemet støtter lokal test og simulering via flagg og API-endepunkter – men har foreløpig ikke CI/CD-pipeline integrert.

---

### 🧪 Lokal testing

Testing aktiveres ved å sette `AppEnvironmentConfig.Testing = true`.

Dette muliggjør:

- Signalruting via `TestSignalsList`
- Manuell verdiendring via `SimulationController`
- Frakoblet kjøring uten fysiske roboter eller heiser

Eksempel:  
```bash
curl -X POST http://localhost:4000/Simulation/SignalValue/DOF_ConfirmFeederReturnInPos -H "Content-Type: application/json" -d true
```
### 🧪 Testsignaler og testdata

| Type              | Verktøy/controller        | Formål                                 |
|-------------------|---------------------------|-----------------------------------------|
| Simulerte signaler| `SimulationController`     | Setter og leser testverdier             |
| Testlister        | `TestSignalsList`          | Nøkkelbasert signalmapping              |
| Testdata          | `ManualProductionItems`, `LiftInput`-lister | Simulerer ordrer, lasting og lossing |

> ⚠️ Testdata må settes manuelt ved oppstart via `ISettingsService` eller API.

### ⚙️ CI/CD-status (per nå)

- Ingen automatisert byggeverktøy (f.eks. GitHub Actions eller Azure Pipelines) er konfigurert.
- Distribusjon skjer manuelt – f.eks. via `dotnet publish` og Windows Service-installasjon.
- Logging er tilgjengelig, men det finnes ingen automatisk validering/test.

### 💡 Anbefalinger

- Legg til enhetstester for sentrale tjenester (f.eks. `RobotFileProcessingService`)
- Vurder GitHub Actions for bygg og test
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

## Veikart / mangler

## 🧭 Veikart / mangler

### 🚧 Kjente hull og forbedringsområder

| Område                 | Status        | Kommentar |
|------------------------|---------------|-----------|
| 🧪 Testing              | ❌ Mangler     | Ingen testprosjekter, teststrategi eller mocks observert |
| ⚙️ CI/CD                | ❌ Ufullstendig| Ingen GitHub Actions eller automatisert deployment |
| 🔍 Observabilitet      | 🔸 Delvis       | Logging finnes, men ingen strukturert observasjon eller varsling |
| 📊 Dashboard           | ❌ Mangler     | Ingen visuell overvåkning av produksjonssystemet |
| 📄 Dokumentasjon       | 🔸 Delvis       | Intern arkitektur og domenelogikk er godt forklart, men konfigurasjon og feilsøking mangler detaljer i kode |
| 🧩 Komponentdeling     | ✅ Fullført     | God separasjon mellom host, applikasjon og domene |
| ⚠️ Robusthet           | 🔸 God, men sårbar | Systemet er testvennlig, men avhenger av korrekt initiering av runtime-state |

---

### 🗺️ Anbefalt veikart (kort sikt)

1. ✅ Fullfør README med domenemodell og teknisk oppsett (📌 du er her nå!)
2. 🧪 Introduser `xUnit`-baserte tester for kritiske komponenter
3. ⚙️ Konfigurer CI/CD pipeline (GitHub Actions)
4. 📤 Automatiser Windows Service-installasjon i testmiljø
5. 🔭 Legg til observasjon (Serilog, HealthChecks, evt. Prometheus)

---

### 📌 Merk

Systemet er velstrukturert og testvennlig, men **mangler enkelte nøkkelkomponenter for produksjonsklar drift** som testautomatisering, overvåkning og visuell innsikt.


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

## Eksempelflyt (test)

Denne seksjonen viser hvordan systemet kan testes lokalt i simulasjonsmodus, uten fysisk tilkobling til roboter, fres eller D365.


### 🔧 Forutsetninger

- `AppEnvironmentConfig.Testing = true` i `appsettings.json`
- `SimulationController` er aktivert
- Signaler sendes **ikke fysisk** til maskiner
- Systemet kjøres med:
  ```bash
  dotnet run --project LOBGarageDoorProductionControllerService
 ```
🧪 Eksempel på testscenario :


1. Start automatisk kjøring
```bash
curl -X POST http://<host>/Settings/StartStop \
  -H "Content-Type: application/json" -d true
```
2. Simuler at roboter er klare
```bash
curl -X POST http://<host>/Simulation/SignalValue/DOF_OkToSendNewCsvFilesRob1 \
  -H "Content-Type: application/json" -d true
curl -X POST http://<host>/Simulation/SignalValue/DOF_OkToSendNewCsvFilesRob2 \
  -H "Content-Type: application/json" -d true
```
3. Legg inn en manuell produksjonsbatch
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
4. Simuler signaler underveis (valgfritt)
```bash
curl -X POST http://<host>/Simulation/SignalValue/DOF_OrderDone \
  -H "Content-Type: application/json" -d true
```
5. Hent logg / status
```bash
curl http://<host>/Operations/GetFeedback
```
📌 Tips

Bruk SimulationController til å sette alle typer verdier:

-Signal (bool): /Simulation/SignalValue/{name}

-Variabel (double): /Simulation/VariableValue/{name}

-Streng (string): /Simulation/StringVariableValue/{name}

Hvis noe ikke skjer: sjekk loggen via /Operations/GetFeedback

🧪 Med dette kan du teste hele produksjonsløpet uten faktisk utstyr!
