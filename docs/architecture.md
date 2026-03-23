# Arquitetura — NewExportToHASS

> **Fork de** [adizanni/ExportToHASS](https://github.com/adizanni/ExportToHASS)
> **Versão:** 1.0.0 | **Licença:** GNU GPL v3 | **Java target:** 1.7 | **SweetHome3D mínimo:** 6.5

---

## Sumário

1. [Visão geral](#1-visão-geral)
2. [Estrutura do repositório](#2-estrutura-do-repositório)
3. [Arquitetura de componentes](#3-arquitetura-de-componentes)
4. [Fluxo de execução](#4-fluxo-de-execução)
5. [Modelo de dados](#5-modelo-de-dados)
6. [Pipeline de build e release](#6-pipeline-de-build-e-release)
7. [Formato de saída (.sh3p / ZIP)](#7-formato-de-saída-sh3p--zip)
8. [Diferenças em relação ao ExportToHASS original](#8-diferenças-em-relação-ao-exporthass-original)
9. [Timeline do repositório original](#9-timeline-do-repositório-original)

---

## 1. Visão geral

O **NewExportToHASS** é um plugin para o SweetHome3D que exporta modelos 3D domésticos para o formato Wavefront OBJ, estruturado especificamente para integração com o [`floor3d-card`](https://github.com/adizanni/floor3d-card) do Home Assistant.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1e3a5f", "primaryTextColor": "#ffffff", "primaryBorderColor": "#4a90d9", "lineColor": "#4a90d9", "secondaryColor": "#2d6a9f", "tertiaryColor": "#0d2137"}}}%%
graph LR
    classDef sh3d fill:#1e3a5f,stroke:#4a90d9,color:#fff,rx:8
    classDef plugin fill:#1a5c38,stroke:#4caf50,color:#fff,rx:8
    classDef hass fill:#7b2d00,stroke:#ff6d00,color:#fff,rx:8
    classDef output fill:#4a2060,stroke:#9c27b0,color:#fff,rx:8

    SH3D["🏠 SweetHome3D\nModelo 3D"]:::sh3d
    PLUGIN["🔌 NewExportToHASS\nPlugin"]:::plugin
    ZIP["📦 home.zip\n(OBJ + MTL + texturas + JSON)"]:::output
    HASS["🏡 Home Assistant\n/config/www/"]:::hass
    CARD["📱 floor3d-card\nVisualização 3D"]:::hass

    SH3D -- "Tools → Export obj to HASS" --> PLUGIN
    PLUGIN -- "Gera" --> ZIP
    ZIP -- "Usuário descompacta" --> HASS
    HASS --> CARD
```

**Problema resolvido:** O plugin original usa nomes de objeto definidos pelo usuário (em vez de índices auto-gerados) para garantir IDs estáveis entre exportações. Este fork renomeia o pacote Java e o identificador do plugin para permitir coexistência com o ExportToHASS original instalado simultaneamente.

---

## 2. Estrutura do repositório

```
NewExportToHASS/
├── src/
│   └── com/eteks/sweethome3d/plugin/
│       └── newexporthass/                  ← pacote renomeado (era: exporthass)
│           ├── ExportHass.java             ← ponto de entrada do plugin
│           ├── OBJWriter.java              ← motor de exportação OBJ/MTL
│           ├── OBJMaterial.java            ← extensão de material para ray-tracing
│           └── ApplicationPlugin.properties← descriptor lido pelo SweetHome3D
├── .github/
│   └── workflows/
│       └── release.yml                     ← CI/CD: build + GitHub Release automático
├── docs/
│   └── architecture.md                     ← este documento
├── build.xml                               ← script Ant para compilar e empacotar
├── SweetHome3D-6.6.jar                     ← dependência principal (framework)
├── j3dcore.jar                             ← Java3D — core
├── j3dutils.jar                            ← Java3D — utilitários
├── vecmath.jar                             ← matemática vetorial
├── json-20210307.jar                       ← serialização JSON
├── README.md
└── LICENSE                                 ← GNU GPL v3
```

---

## 3. Arquitetura de componentes

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#0d2137", "primaryTextColor": "#ffffff", "primaryBorderColor": "#4a90d9", "lineColor": "#4a90d9", "edgeLabelBackground": "#1e3a5f", "clusterBkg": "#0a1929", "clusterBorder": "#4a90d9"}}}%%
classDiagram
    direction TB

    class Plugin {
        <<SweetHome3D API>>
        +getHome() Home
        +getUserPreferences() UserPreferences
        +getHomeController() HomeController
    }

    class PluginAction {
        <<SweetHome3D API>>
        +execute()
        +putPropertyValue(prop, value)
        +setEnabled(boolean)
    }

    class ExportHass {
        +getActions() PluginAction[]
    }

    class ExportHassAction {
        -resourceBaseName String
        +execute()
        -exportHomeStructure(home, factory, name, file) File
    }

    class OBJWriter {
        <<Motor de exportação>>
        +writeNodeInZIPFile(node, file, level, entry, header)$
        +writeNode(node)
        +close()
        -writeMTLFile(mtlFile, appearances)
        -writeTextures(textureDir, appearances)
    }

    class OBJMaterial {
        <<Extensão de Material>>
        -opticalDensity Float
        -illuminationModel Integer
        -sharpness Float
        +getOpticalDensity() float
        +getIlluminationModel() int
        +getSharpness() float
        +cloneNodeComponent(forceDuplicate) NodeComponent
    }

    class Material {
        <<Java3D API>>
    }

    Plugin <|-- ExportHass : extends
    PluginAction <|-- ExportHassAction : extends
    ExportHass *-- ExportHassAction : inner class
    ExportHassAction ..> OBJWriter : usa (static)
    OBJWriter ..> OBJMaterial : referencia
    Material <|-- OBJMaterial : extends
```

### Responsabilidades

| Classe | Responsabilidade |
|---|---|
| `ExportHass` | Ponto de entrada; registra a ação no menu do SweetHome3D |
| `ExportHassAction` | Controla o diálogo de arquivo e orquestra a exportação |
| `OBJWriter` | Converte a cena Java3D para OBJ/MTL e empacota em ZIP |
| `OBJMaterial` | Estende `javax.media.j3d.Material` com propriedades extras de ray-tracing |

---

## 4. Fluxo de execução

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1a1a2e", "primaryTextColor": "#e0e0e0", "primaryBorderColor": "#7c4dff", "lineColor": "#7c4dff", "secondaryColor": "#16213e", "tertiaryColor": "#0f3460", "noteBkgColor": "#1a1a2e", "noteTextColor": "#e0e0e0", "noteBorderColor": "#7c4dff", "activationBorderColor": "#7c4dff", "activationBkgColor": "#16213e"}}}%%
sequenceDiagram
    autonumber
    actor Usuário
    participant SH3D as SweetHome3D
    participant EA as ExportHassAction
    participant FCM as FileContentManager
    participant EHS as exportHomeStructure()
    participant OBJ as OBJWriter
    participant FS as Sistema de Arquivos

    Usuário->>SH3D: Tools → Export obj to HASS
    SH3D->>EA: execute()

    EA->>FCM: Cria ContentManager com filtro .zip
    EA->>FCM: showSaveDialog(homeView, ...)
    FCM-->>EA: caminho do arquivo ZIP destino

    EA->>EHS: exportHomeStructure(home, factory, "Home.obj", destFile)

    Note over EHS: Clona o Home para processamento isolado
    EHS->>EHS: Itera home.getHomeObjects()

    loop Para cada HomeObject
        alt HomeFurnitureGroup
            EHS->>EHS: Itera getAllFurniture()
            EHS->>EHS: Node.setName("lvlNNN" + grupo + "_" + item)
        else HomePieceOfFurniture
            EHS->>EHS: Node.setName("lvlNNN" + nome)
        else Wall
            EHS->>EHS: Node.setName("lvlNNN" + "wall_" + idx)
        else Room
            EHS->>EHS: Node.setName("lvlNNN" + "room_" + idx)
        else Outro
            EHS->>EHS: Node.setName("lvlNNN" + "object_" + idx)
        end
        EHS->>EHS: root.addChild(node)
    end

    EHS->>OBJ: writeNodeInZIPFile(root, destFile, 0, "home.obj", header)

    OBJ->>FS: Cria pasta temporária
    OBJ->>FS: Escreve home.obj (vértices, normais, faces)
    OBJ->>FS: Escreve home.mtl (materiais)
    OBJ->>FS: Extrai texturas (.png)
    OBJ->>FS: Escreve objects.json (lista de object_ids)
    OBJ->>FS: Empacota tudo em .zip
    OBJ->>FS: Remove pasta temporária

    OBJ-->>EHS: File (zipFile)
    EHS-->>EA: File
    EA->>Usuário: JOptionPane "Model successfully exported"
```

---

## 5. Modelo de dados

### Hierarquia de objetos exportados

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1b3a1b", "primaryTextColor": "#ccffcc", "primaryBorderColor": "#4caf50", "lineColor": "#4caf50", "edgeLabelBackground": "#1b3a1b"}}}%%
graph TD
    classDef level fill:#1b3a1b,stroke:#4caf50,color:#ccffcc
    classDef obj fill:#1a3a5c,stroke:#4a90d9,color:#cce5ff
    classDef skip fill:#3a1a1a,stroke:#f44336,color:#ffcccc
    classDef out fill:#3a2a00,stroke:#ff9800,color:#ffe0b2

    HOME["🏠 Home (clonado)"]:::level

    HOME --> ENV["HomeEnvironment\n⛔ ignorado"]:::skip
    HOME --> CAM["Camera\n⛔ ignorado"]:::skip
    HOME --> LVL["Level\n⛔ ignorado"]:::skip
    HOME --> GRP["HomeFurnitureGroup\n✔ processado"]:::obj
    HOME --> FUR["HomePieceOfFurniture\n✔ processado"]:::obj
    HOME --> WAL["Wall\n✔ processado"]:::obj
    HOME --> ROM["Room\n✔ processado"]:::obj
    HOME --> OTH["Outro\n✔ processado"]:::obj

    GRP --> GI["Itera getAllFurniture()\nlvlNNN + grupo_item"]:::out
    FUR --> FI["lvlNNN + nome"]:::out
    WAL --> WI["lvlNNN + wall_idx"]:::out
    ROM --> RI["lvlNNN + room_idx"]:::out
    OTH --> OI["lvlNNN + object_idx"]:::out
```

### Estrutura do ZIP exportado

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#2a0a4a", "primaryTextColor": "#e8d5ff", "primaryBorderColor": "#9c27b0", "lineColor": "#9c27b0", "edgeLabelBackground": "#2a0a4a"}}}%%
graph LR
    classDef zip fill:#2a0a4a,stroke:#9c27b0,color:#e8d5ff
    classDef file fill:#1a0a2e,stroke:#7c4dff,color:#e8d5ff
    classDef tex fill:#1a2a0a,stroke:#4caf50,color:#e8d5ff

    ZIP["📦 home.zip"]:::zip
    OBJ["📄 home.obj\n— vértices v x y z\n— normais vn x y z\n— faces f v1 v2 v3"]:::file
    MTL["📄 home.mtl\n— newmtl\n— Kd r g b\n— map_Kd textura.png"]:::file
    JSON["📄 objects.json\n— lista de object_ids\npara o floor3d-card"]:::file
    TEX["🖼️ textura_N.png\n(N texturas extraídas\ndos materiais)"]:::tex

    ZIP --> OBJ
    ZIP --> MTL
    ZIP --> JSON
    ZIP --> TEX
```

---

## 6. Pipeline de build e release

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1a1a1a", "primaryTextColor": "#ffffff", "primaryBorderColor": "#ff6d00", "lineColor": "#ff6d00", "edgeLabelBackground": "#1a1a1a"}}}%%
flowchart TD
    classDef trigger fill:#7b2d00,stroke:#ff6d00,color:#fff
    classDef step fill:#1e3a5f,stroke:#4a90d9,color:#fff
    classDef artifact fill:#1a5c38,stroke:#4caf50,color:#fff
    classDef release fill:#4a2060,stroke:#9c27b0,color:#fff

    TAG["🏷️ git tag v1.0.0\ngit push --tags"]:::trigger
    GHA["⚙️ GitHub Actions\nbuild-and-release job\n(ubuntu-latest)"]:::step
    JDK["☕ Setup JDK 11\n(temurin)"]:::step
    ANT["🐜 Install Ant"]:::step
    PATCH["📝 Patch versão\nbuild.xml + properties"]:::step
    BUILD["🔨 ant package\n→ dist/*.sh3p"]:::artifact
    REL["🚀 GitHub Release\nNewExportToHASS v1.0.0\n+ .sh3p como asset"]:::release

    TAG --> GHA
    GHA --> JDK --> ANT --> PATCH --> BUILD --> REL
```

### Build local

```bash
# Pré-requisito: JDK 7+ e Ant instalados
ant clean    # limpa bin/ e dist/
ant compile  # compila src/ → bin/
ant package  # empacota bin/ → dist/NewExportToHASSPlugin-1.0.0-sh3d6.6.sh3p
ant          # alias: clean + compile + package
```

### Ciclo de release via tag

```bash
git tag v1.0.0
git push origin v1.0.0
# → GitHub Actions detecta a tag e cria o release automaticamente
```

---

## 7. Formato de saída (.sh3p / ZIP)

O arquivo `.sh3p` é um arquivo ZIP consumido diretamente pelo SweetHome3D. Estrutura interna:

```
NewExportToHASSPlugin-1.0.0-sh3d6.6.sh3p
├── ApplicationPlugin.properties    ← descriptor na raiz
├── com/
│   └── eteks/
│       └── sweethome3d/
│           └── plugin/
│               └── newexporthass/  ← pacote renomeado
│                   ├── ExportHass.class
│                   ├── ExportHass$ExportHassAction.class
│                   ├── OBJWriter.class
│                   ├── OBJWriter$ComparableAppearance.class
│                   └── OBJMaterial.class
└── lib/
    └── json-20210307.jar           ← dependência empacotada
```

**Campos críticos em `ApplicationPlugin.properties`:**

```properties
name=New Export Wavefront Obj for HASS   # identificador único do plugin
class=com.eteks.sweethome3d.plugin.newexporthass.ExportHass
version=1.0.0
applicationMinimumVersion=6.5
javaMinimumVersion=1.5
```

---

## 8. Diferenças em relação ao ExportToHASS original

| Aspecto | ExportToHASS (original) | NewExportToHASS (fork) |
|---|---|---|
| **Pacote Java** | `com.eteks.sweethome3d.plugin.exporthass` | `com.eteks.sweethome3d.plugin.newexporthass` |
| **Nome do plugin** | `Export Wavefront Obj for HASS` | `New Export Wavefront Obj for HASS` |
| **Versão inicial** | `0.1.0` | `1.0.0` |
| **Arquivo .sh3p** | `ExportToHASSPlugin.sh3p` | `NewExportToHASSPlugin-<ver>-sh3d6.6.sh3p` |
| **Build system** | Eclipse manual | Ant (`build.xml`) |
| **CI/CD** | Nenhum | GitHub Actions (`release.yml`) |
| **Coexistência** | Conflita com o fork se ambos instalados | Coexiste com o original sem conflito |
| **Docs** | README básico | README + `docs/architecture.md` |

> As classes, lógica de exportação e comportamento funcional são **idênticos** ao upstream. Apenas o namespace e a infraestrutura de build foram alterados.

---

## 9. Timeline do repositório original

A linha do tempo abaixo documenta a evolução do repositório [adizanni/ExportToHASS](https://github.com/adizanni/ExportToHASS), do qual este fork deriva.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"primaryColor": "#1e3a5f", "primaryTextColor": "#ffffff", "primaryBorderColor": "#4a90d9", "lineColor": "#4a90d9", "edgeLabelBackground": "#0d2137"}}}%%
timeline
    title ExportToHASS — Evolução do Repositório Original

    section 2021
        Jul 2021 : 🚀 v0.0.1 — Primeiro release público
                 : Plugin básico de exportação OBJ para HASS
        Jul 2021 : 🔖 v0.0.2 — Correções iniciais
                 : Mesmo commit do v0.0.1

    section 2021 — Q3/Q4
        Jul 2021 : 🔖 v0.0.3 — Correção de nomeação de materiais
                 : Remove gestão do til (~) em nomes de materiais

    section 2022
        Jan 2022 : 🔖 v0.0.3-sh3d6.6 — Build para SweetHome3D 6.6
                 : Novo sufixo de versão com versão do SH3D
        Fev 2022 : 🔖 v0.1.0-sh3d6.6 — Suporte a múltiplos níveis
                 : Prefixos lvl000, lvl001… para floor3d-card
                 : Gestão de HomeFurnitureGroup
                 : 4.758+ downloads registrados

    section 2024
        Fev 2024 : 📄 Atualização da licença para GNU GPL v3
                 : Último commit registrado no repositório original

    section 2026 (fork)
        Mar 2026 : 🍴 NewExportToHASS v1.0.0
                 : Renomeação do pacote Java: newexporthass
                 : Build Ant + GitHub Actions CI/CD
                 : Documentação de arquitetura
```

### Detalhamento das versões upstream

| Tag | Data | Destaques |
|---|---|---|
| `v0.0.1` | Jul 2021 | Primeira versão pública; exportação básica OBJ/MTL/ZIP |
| `v0.0.2` | Jul 2021 | Mesmo commit; tag alternativa para distribuição inicial |
| `v0.0.3` | Jul 2021 | Remove tratamento de til (`~`) em nomes de materiais |
| `v0.0.3-sh3d6.6` | Jan 2022 | Recompilação com SweetHome3D 6.6; introduz sufixo `-sh3dX.Y` |
| `v0.1.0-sh3d6.6` | Fev 2022 | **Maior release:** suporte a múltiplos andares (`lvlNNN`), grupos de mobília, +4.7k downloads |
| *(sem tag)* | Fev 2024 | Atualização de licença para GNU GPL v3 — último commit no upstream |

### Padrão de versionamento adotado pelo upstream

```
MAJOR.MINOR.PATCH[-sh3dX.Y]
│     │     │     └── versão do SweetHome3D requerida (a partir de v0.0.3-sh3d6.6)
│     │     └── patch: correções de bugs
│     └── minor: novos recursos não-quebra-de-compatibilidade
└── major: mudanças significativas de arquitetura
```

Este fork (NewExportToHASS) adota o mesmo padrão, iniciando em `1.0.0` para distinguir claramente da linha upstream.
