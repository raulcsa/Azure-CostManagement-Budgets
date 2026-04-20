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
 
### 3. Desplegar recursos por equipo
 
#### Equipo Frontend — App Service (gratuito)
 
```bash
# Plan App Service F1 (completamente gratis)
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
  --runtime "NODE:18-lts" \
  --tags team=frontend costcenter=CC-001 owner=Ana project=multi-team-demo
```
 
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
  --storage-account <NOMBRE_STORAGE_ANTERIOR> \
  --tags team=backend costcenter=CC-002 owner=Carlos project=multi-team-demo
```
 
#### Equipo Data — Storage Account
 
```bash
az storage account create \
  --name sadatademo$RANDOM \
  --resource-group rg-data \
  --location eastus \
  --sku Standard_LRS \
  --tags team=data costcenter=CC-003 owner=Maria project=multi-team-demo
```
 
### 4. Configurar Budgets y Alertas
 
En el [Portal de Azure](https://portal.azure.com) → **Cost Management + Billing → Budgets → + Add**:
 
| Campo | Valor |
|-------|-------|
| Nombre | `budget-multi-team` |
| Scope | Tu suscripción de estudiante |
| Reset period | Monthly |
| Amount | `$15 USD` |
| Filtro por tag | `project = multi-team-demo` |
 
**Alertas:**
- 80% ($12) → Email del proyecto
- 100% ($15) → Email del proyecto
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
  --location eastus \
  --name la-reporte-costos \
  --tags team=backend costcenter=CC-002 project=multi-team-demo
```
 
#### 5.2 Habilitar Managed Identity
 
```bash
az logic workflow identity assign \
  --resource-group rg-backend \
  --name la-reporte-costos \
  --identity-type SystemAssigned
```
 
#### 5.3 Asignar rol de lectura de costos
 
```bash
LOGIC_APP_PRINCIPAL=$(az logic workflow show \
  --resource-group rg-backend \
  --name la-reporte-costos \
  --query "identity.principalId" -o tsv)
 
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
 
az role assignment create \
  --assignee $LOGIC_APP_PRINCIPAL \
  --role "Cost Management Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```
 
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
 
 

 
