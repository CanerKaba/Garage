# LOB Garage Door – Produksjonskontroller

Dette repoet styrer produksjonen av garasjeporter på fabrikkgulvet: plukk/lagring, løfteanlegg, fresecelle og to robotceller. Systemet integrerer mot Dynamics 365 F&O (produksjonsordre, BOM, lager) og orkestrerer kjøringen via en bakgrunnstjeneste.

**Status:** Utkast. Basert på disse filene:
> - `Application/Services/ProductionExecutionService.cs`
> - `LOBGarageDoorProductionControllerService/Controllers/OperationsController.cs`
> - `LOBGarageDoorProductionControllerService/Controllers/SettingsController.cs`
> - `LOBGarageDoorProductionControllerService/Controllers/SimulationController.cs`

Ukjente deler er merket Ukjent og fylles inn når flere filer kommer.

## Innhold
- [Arkitektur](#arkitektur)
- [Teknologier](#teknologier)
- [Domenemodell](#domenemodell)
- [Flyter](#flyter)
- [API](#api)
- [Konfigurasjon](#konfigurasjon)
- [Feilsøking](#feilsøking)
- [Veikart](#veikart)

## Arkitektur

```mermaid
flowchart LR
    subgraph Client
      Operator["Operator / HMI / Eksterne systemer"]
    end

    subgraph Service["Service-prosjekt (ASP.NET Core)"]
      API[["HTTP API\n(Operations, Settings, Simulation)"]]
      Hosted["ProductionExecutionService\n(BackgroundService)"]
    end

    subgraph Application["Application-laget"]
      Sched["ProductionSchedulingService"]
      ProdFx["ProductionFunctionsService"]
      LiftSvc["LiftService"]
      MillFile["MillingMachineFileProcessingService"]
      RobFile["RobotFileProcessingService"]
      D365Data["ID365DataProcessingService"]
      D365Act["ID365ActionProcessingService"]
      WMS["WarehouseManagementService"]
      IO["IRobotOutboundMessageProcessingService"]
      Settings["ISettingsService\n(runtime state)"]
      Log["ILoggingService"]
      MillSvc["IMillingMachineService"]
    end

    subgraph Domain
      Entities["Entities:\nProduction, Lift, Robot, MillingMachine,\nConfiguration"]
      Enums["Enums"]
    end

    subgraph Integrations
      D365["Dynamics 365 F&O"]
      Robots["Robotceller\n10.5.15.21 (R1)\n10.5.15.73 (R2)"]
      Milling["Fresemaskin"]
      Storage["Lager/WMS"]
      Files["CSV / Filutveksling"]
    end

    Operator --> API
    API --> Settings
    API --> Hosted
    Hosted --> Application
    Application --> Domain
    Application --> Integrations
    Hosted <--> IO
```

### Roller og ansvar (utdrag)

ProductionExecutionService: Sentralt orkestreringsloop (2s intervall). Leser state fra ISettingsService, henter produksjonsordre (manuell/automatisk), genererer filer for Robot1/Robot2 og fresecelle, håndterer håndtrykk via IO-signaler.

HTTP API: Kontroller for operatør-kommandoer (Operations), driftsmodi (Settings) og simulering i testmiljø (Simulation).

Integrasjoner:

D365 Data/Action (lese BOM/picklist, opprette/starte ordre)

WMS (plukk-/tray-posisjon, lagerkvantum)

Robot IO (digitale signaler) + CSV-filutveksling

Fresemaskin (filutveksling)
