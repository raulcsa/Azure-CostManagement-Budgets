#  Azure Multi-Team Cost Management con Ownership Tags
 
 
---
 
##  Descripción
 
Este proyecto demuestra cómo aplicar **Azure Cost Management**, **Resource Tags** y **Budgets** en un escenario multi-equipo real. Cada equipo tiene su propio Resource Group, tags de propiedad y centro de costo, y se automatiza un reporte de gasto semanal por email mediante Logic Apps.
 


##  Arquitectura
 
```
│
├── rg-frontend  [team=frontend | costcenter=CC-001 | owner=Ana]
│   ├── App Service Plan (F1 - gratuito)
│   └── Web App (Node.js 18)
│
├── rg-backend   [team=backend  | costcenter=CC-002 | owner=Carlos]
│   ├── Function App (Python 3.10)
│   ├── Storage Account
│   └── Logic App → Reporte semanal por email
│
└── rg-data      [team=data     | costcenter=CC-003 | owner=Maria]
    └── Storage Account (análisis de datos)
 
Cost Management
├── Budget: $15 USD (scope: suscripción, filtro: project=multi-team-demo)
│   ├── Alerta 80%  → Email
│   └── Alerta 100% → Email
└── Cost Analysis → Group by tag: team / costcenter
```
 
---
 
##  Esquema de Tags
 
Todos los recursos siguen esta convención de etiquetado:
 
| Tag | Valores posibles | Descripción |
|-----|-----------------|-------------|
| `team` | `frontend` \| `backend` \| `data` | Equipo propietario del recurso |
| `owner` | nombre de persona | Responsable directo |
| `costcenter` | `CC-001` \| `CC-002` \| `CC-003` | Centro de costo para chargeback |
| `environment` | `dev` \| `prod` | Entorno de despliegue |
| `project` | `multi-team-demo` | Nombre del proyecto (usado en filtros de Budget) |
 
---
 
##  Despliegue paso a paso
 
### 1. Login y configuración inicial
 
```bash
# Iniciar sesión en Azure
az login
 
# Verificar suscripción activa
az account list --output table
 
 

```
 
### 2. Crear Resource Groups con Tags
 
```bash
# Equipo Frontend
az group create \
  --name rg-frontend \
  --location eastus \
  --tags team=frontend costcenter=CC-001 owner=Ana \
         environment=dev project=multi-team-demo
 
# Equipo Backend
az group create \
  --name rg-backend \
  --location eastus \
  --tags team=backend costcenter=CC-002 owner=Carlos \
         environment=dev project=multi-team-demo
 
# Equipo Data
az group create \
  --name rg-data \
  --location eastus \
  --tags team=data costcenter=CC-003 owner=Maria \
         environment=dev project=multi-team-demo
 
# Verificar los 3 resource groups
az group list \
  --query "[?tags.project=='multi-team-demo'].{name:name, team:tags.team}" \
  --output table
```

<img width="706" height="134" alt="image" src="https://github.com/user-attachments/assets/03179c67-02f1-4fe1-9f8b-4cae1be5f6e9" />
 
### 3. Desplegar recursos por equipo
 
#### Equipo Frontend — App Service (gratuito)
 
```bash
# Plan App Service F1 
az appservice plan create \
  --name plan-frontend \
  --resource-group rg-frontend \
  --sku F1 \
  --is-linux
 
# Web App con Node.js
az webapp create \
  --resource-group rg-frontend \
  --plan plan-frontend \
  --name webapp-frontend-demo-$RANDOM \
  --runtime "NODE:22-lts" \
  --tags team=frontend costcenter=CC-001 owner=Ana project=multi-team-demo
```

<img width="1101" height="546" alt="image" src="https://github.com/user-attachments/assets/5881d543-ccaf-4fe6-922c-2eb04a271480" />

#### Equipo Backend — Function App + Storage
 
```bash
# Storage Account para Function App
az storage account create \
  --name sabackenddemo$RANDOM \
  --resource-group rg-backend \
  --location eastus \
  --sku Standard_LRS \
  --tags team=backend costcenter=CC-002 project=multi-team-demo
 
# Function App 
az functionapp create \
  --resource-group rg-backend \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.10 \
  --functions-version 4 \
  --name func-backend-demo-$RANDOM \
  --storage-account nombre-de-storage-account \
  --os-type Linux \
  --tags team=backend costcenter=CC-002 owner=Carlos project=multi-team-demo
```

<img width="1679" height="559" alt="image" src="https://github.com/user-attachments/assets/6dfe39c4-c144-4ae0-8856-b01d4d47cef2" />
 
#### Equipo Data — Storage Account
 
```bash
az storage account create \
  --name sadatademo$RANDOM \
  --resource-group rg-data \
  --location eastus \
  --sku Standard_LRS \
  --tags team=data costcenter=CC-003 owner=Maria project=multi-team-demo
```

<img width="1088" height="545" alt="image" src="https://github.com/user-attachments/assets/4904026f-5129-495b-8a01-38fe0bd48103" />
 
### 4. Configurar Budgets y Alertas
 
En el [Portal de Azure](https://portal.azure.com) → **Cost Management + Billing → Budgets → + Add**:
 
| Campo | Valor |
|-------|-------|
| Nombre | `budget-multi-team` |
| Scope | Tu suscripción de estudiante |
| Reset period | Monthly |
| Amount | `$15 USD` |
| Filtro por tag | `project = multi-team-demo` |

<img width="1626" height="766" alt="image" src="https://github.com/user-attachments/assets/14c5ae12-5d6d-431c-acb0-e0be8ac45f95" />
 
**Alertas:**
- 80% ($12) → Email del proyecto
- 100% ($15) → Email del proyecto

<img width="1657" height="708" alt="image" src="https://github.com/user-attachments/assets/fead22f2-ef8f-4802-a12b-0b77e3627d88" />

También puedes crearlo desde CLI:
 
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
START_DATE=$(date +%Y-%m-01)
END_DATE="2025-12-31"
 
az consumption budget create \
  --budget-name "budget-multi-team" \
  --amount 15 \
  --category Cost \
  --time-grain Monthly \
  --start-date $START_DATE \
  --end-date $END_DATE \
  --resource-group rg-backend
```
 
### 5. Logic App — Reporte Semanal por Email
 
#### 5.1 Crear el Logic App
 
```bash
az logic workflow create \
  --resource-group rg-backend \
  --location norwayeast \
  --name la-reporte-costos \
  --definition '{"definition":{"$schema":"https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#","actions":{},"triggers":{},"outputs":{}}}' \
  --tags team=backend costcenter=CC-002 project=multi-team-demo
```
 
#### 5.2 Habilitar Managed Identity
 
```bash
az resource update \
  --resource-group rg-backend \
  --name la-reporte-costos \
  --resource-type "Microsoft.Logic/workflows" \
  --set identity.type=SystemAssigned
```
 
#### 5.3 Asignar rol de lectura de costos
 
```bash
LOGIC_APP_PRINCIPAL=$(az resource show \
  --resource-group rg-backend \
  --name la-reporte-costos \
  --resource-type "Microsoft.Logic/workflows" \
  --query "identity.principalId" -o tsv)

SUBSCRIPTION_ID=$(az account show --query id -o tsv)

az role assignment create \
  --assignee $LOGIC_APP_PRINCIPAL \
  --role "Cost Management Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```

<img width="1679" height="275" alt="image" src="https://github.com/user-attachments/assets/05149b1f-c348-4e52-b4c9-728a4d94ca4b" />
 
#### 5.4 Flujo del Logic App (en el Designer del Portal)
 
```
TRIGGER: Recurrence
├── Frequency: Week
├── Interval: 1
└── On: Monday at 08:00 AM
 
PASO 1: HTTP (llamada a Cost Management API)
├── Method: POST
├── URI: https://management.azure.com/subscriptions/{subId}/providers
│         /Microsoft.CostManagement/query?api-version=2023-11-01
├── Authentication: Managed Identity
└── Body:
    {
      "type": "ActualCost",
      "timeframe": "WeekToDate",
      "dataset": {
        "granularity": "None",
        "aggregation": {
          "totalCost": { "name": "Cost", "function": "Sum" }
        },
        "grouping": [
          { "type": "TagKey", "name": "team" },
          { "type": "TagKey", "name": "costcenter" }
        ]
      }
    }
 
PASO 2: Send an email (Office 365 Outlook o Gmail)
├── To: tu@correo.com
├── Subject: [Azure] Reporte Semanal de Costos por Equipo
└── Body: @{body('HTTP')['properties']['rows']}
```

<img width="1352" height="786" alt="image" src="https://github.com/user-attachments/assets/05178a58-bc17-41a6-bf35-23d2ccb36955" />

<img width="1360" height="842" alt="image" src="https://github.com/user-attachments/assets/501b9407-ef9a-4117-9c04-4ffa4e7a5ade" />

<img width="1355" height="829" alt="image" src="https://github.com/user-attachments/assets/d1b6c67f-1c53-426b-8210-d55b178f6faf" />

---
 
##  Visualizar costos en Cost Analysis
 
Una vez que los recursos lleven 24 horas activos:
 
1. Ve a **Cost Management → Cost Analysis**
2. Cambia el rango a **"Este mes"**
3. En **"Group by"** → selecciona **Tag → `team`**
4. Verás barras separadas por `frontend`, `backend`, `data`
5. Repite agrupando por **`costcenter`** para la vista de chargeback
---
 
##  Limpieza de recursos
 
 
```bash
# Eliminar todos los Resource Groups del proyecto
az group delete --name rg-frontend --yes --no-wait
az group delete --name rg-backend  --yes --no-wait
az group delete --name rg-data     --yes --no-wait
 
# Verificar que se eliminaron
az group list \
  --query "[?tags.project=='multi-team-demo'].name" \
  --output table
```
 
---
 
 

 
