# CLAUDE.md - IT Web Service Cluster

This file provides guidance to Claude Code (claude.ai/code) when working with the full-stack IT Web Service application.

## Project Overview

This is a full-stack enterprise application for managing **CRM**, **Logistics (Logistica)**, and **Maintenance (Mtto)** operations. The system consists of two independent projects that communicate via REST API.

```
it_web_service-cluster/
├── it_web_service-Front/      # React 18 + Vite frontend
└── it_web_service-Backend_R/  # PHP 8.2 REST API backend
```

## Quick Start

### Development Environment

```bash
# Backend (Terminal 1)
cd it_web_service-Backend_R
composer install
# Configure .env with database credentials
php -S localhost:3000
# Or with Docker:
docker-compose up -d    # Runs on port 3000

# Frontend (Terminal 2)
cd it_web_service-Front
npm install
npm run dev             # Runs on port 5173 (Vite default)
```

### Production Build

```bash
# Frontend
cd it_web_service-Front
npm run build           # Creates dist/ folder

# Backend
cd it_web_service-Backend_R
docker build -t it-backend .
```

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           FRONTEND                                   │
│                    React 18 + Vite + Redux                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │  CRM Module │  │   Logistica │  │    Mtto     │                 │
│  │   /crm/*    │  │   /logis/*  │  │   /mtto/*   │                 │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
│         │                │                │                         │
│  ┌──────┴────────────────┴────────────────┴──────┐                 │
│  │              Redux Store + RTK Query           │                 │
│  │         Axios Instances (per module)           │                 │
│  └──────────────────────┬────────────────────────┘                 │
└─────────────────────────┼───────────────────────────────────────────┘
                          │
                          │ HTTP/HTTPS (JSON)
                          │ JWT Authentication
                          │
┌─────────────────────────┼───────────────────────────────────────────┐
│                         ▼                                           │
│  ┌──────────────────────────────────────────────┐                  │
│  │              index.php (Router)               │                  │
│  │         CORS + JWT Token Validation           │                  │
│  └──────────────────────┬───────────────────────┘                  │
│                         │                                           │
│  ┌──────────────────────┴───────────────────────┐                  │
│  │              Controllers                      │                  │
│  │  CRMController │ LogisticaControl │ MttoCtrl │                  │
│  └──────────────────────┬───────────────────────┘                  │
│                         │                                           │
│  ┌──────────────────────┴───────────────────────┐                  │
│  │         Models (ActiveRecord ORM)             │                  │
│  └──────────────────────┬───────────────────────┘                  │
│                         │                                           │
│                    SQL Server                                       │
│                         │                                           │
│                      BACKEND                                        │
│                   PHP 8.2 + Apache                                  │
└─────────────────────────────────────────────────────────────────────┘
```

## Frontend-Backend Communication

### Base URL Configuration

**Frontend** (`src/global/baseURL.js`):
```javascript
export const URL = 'http://localhost:3000/index.php'; // Local
// export const URL = 'https://autorizacion-itweb.ddns.net:409/index.php'; // Production
```

**Backend** (`index.php` CORS headers):
```php
header("Access-Control-Allow-Origin: *");  // Local
// header("Access-Control-Allow-Origin: https://production-url"); // Production
```

### HTTP Client Configuration

The frontend uses two approaches for API calls:

**1. RTK Query** (`src/store/apis/todosApi.js`):
```javascript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const todosApi = createApi({
    baseQuery: fetchBaseQuery({
        baseUrl: URL,
        prepareHeaders: (headers) => {
            const localS = JSON.parse(localStorage.getItem('session'));
            headers.set('Authorization', `Bearer ${localS.token}`);
            headers.set('X-Refresh-Token', `Bearer ${localS.tokenrefresh}`);
            return headers;
        },
    }),
    endpoints: (builder) => ({
        getClientes: builder.query({ query: () => '/getClientes' }),
        // ... more endpoints
    })
});
```

**2. Axios Instances** (`src/store/apis/axios/`):
```javascript
// crmAxios.js - CRM module
export const crmAxios = axios.create({ baseURL: `${URL}/crm` });

// mttoAxios.js - Maintenance module
export const mttoAxios = axios.create({ baseURL: `${URL}` });

// logisAxios.js - Logistics module
export const logisAxios = axios.create({ baseURL: `${URL}/logistica` });
```

All Axios instances include interceptors for:
- Adding JWT tokens to requests
- Handling token refresh on responses
- Centralized error handling (401 redirects)

## Authentication Flow

### Login Sequence

```
Frontend                              Backend
   │                                     │
   │  POST /login                        │
   │  { bd, user, pass }                 │
   │ ──────────────────────────────────► │
   │                                     │ Validate credentials
   │                                     │ Generate JWT tokens
   │  { token, tokenrefresh,             │
   │    user, permisos, ... }            │
   │ ◄────────────────────────────────── │
   │                                     │
   │  Store in localStorage              │
   │  Update Redux authSlice             │
   │                                     │
```

### Token Structure

**Access Token** (short-lived):
- Sent in `Authorization: Bearer <token>` header
- Contains: user, IdEmpleado, IdSucursal, permisos

**Refresh Token** (long-lived):
- Sent in `X-Refresh-Token: Bearer <token>` header
- Used to obtain new access tokens

### Session Persistence

Frontend stores session in `localStorage`:
```javascript
localStorage.setItem('session', JSON.stringify({
    user, token, tokenrefresh, direct,
    idempleado, nombreempleado, permisos,
    IdSucursal, IdCentroOperativo, TipoCambio
}));
```

### Token Validation

On app load, frontend validates existing token:
```javascript
// POST /checkAuth2
// Headers: Authorization + X-Refresh-Token
// Returns: new token if refreshed, or error if expired
```

## API Response Formats

### Backend Standard Response (ResponseHelper)

```json
{
    "success": true,
    "message": "Operación exitosa",
    "data": [...]
}
```

```json
{
    "success": false,
    "message": "Error message",
    "error_code": 500
}
```

### Legacy Response (HTTPreponse)

```json
{
    "status": 200,
    "ok": true,
    "message": "Success",
    "nuevoToken": "new-jwt-token-if-refreshed",
    "data": [...]
}
```

## API Endpoints Summary

| Module | Frontend Route | Backend Prefix | Axios Instance |
|--------|---------------|----------------|----------------|
| Auth | `/login` | `/` | fetch (direct) |
| CRM | `/crmModulo/*` | `/crm/*` | crmAxios |
| Logistics | `/logisticaModulo/*` | `/logistica/*` | logisAxios |
| Maintenance | `/mttoModulo/*` | `/` | mttoAxios |
| General | - | `/generales/*`, `/formatos/*` | - |

### Key Endpoints

**Authentication:**
- `POST /login` - Login and get tokens
- `POST /logout` - End session
- `POST /checkAuth2` - Validate/refresh token

**CRM:**
- `GET /crm/listas/getProspectosBrowser` - List prospects
- `GET /crm/listas/getClientesBrowser` - List clients
- `POST /crm/set/insertCotizacion_v2` - Create quotation
- `GET /crm/listas/getCotizacion` - Get quotation details

**Logistics:**
- `GET /logistica/getRutas` - Get routes
- `GET /logistica/getDestinosHojaRuta` - Route destinations
- `POST /logistica/setStateRfso` - Update RFSO status

**Maintenance:**
- `GET /getCataEquiposRentaAll` - Equipment catalog
- `POST /createOT` - Create work order
- `GET /getOT` - Get work order details

## Request Flow Architecture (Thunks Pattern)

The application uses a layered architecture for API requests. Below is the complete flow using `usePedidoCliente` as a reference example.

### Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND                                         │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 1. COMPONENT/HOOK (usePedidoCliente.js)                                 │ │
│  │    - Calls useGetData() hook                                            │ │
│  │    - Passes thunk action + params                                       │ │
│  │    - Manages local state (data, loading, error)                         │ │
│  └────────────────────────────────┬────────────────────────────────────────┘ │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 2. useGetData HOOK (src/app/hooks/useGetData.js)                        │ │
│  │    - Generic data fetching hook                                         │ │
│  │    - Dispatches thunk to Redux                                          │ │
│  │    - Handles loading states & errors                                    │ │
│  │    - Returns: { data, isLocalLoading, catchErrorAtFetch, getData }      │ │
│  └────────────────────────────────┬────────────────────────────────────────┘ │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 3. THUNK DEFINITION (store/slices/crmSlice/crmThunksModule/)            │ │
│  │    - Defines endpoint path                                              │ │
│  │    - Uses apiThunkGet or apiThunkPost                                   │ │
│  │    - Maps params to API request                                         │ │
│  └────────────────────────────────┬────────────────────────────────────────┘ │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 4. THUNKS BASE (store/slices/crmSlice/thunks-Base/thunksBase.js)        │ │
│  │    - Executes Axios request via crmAxios                                │ │
│  │    - Handles token refresh (nuevoToken)                                 │ │
│  │    - Returns ApiResponse (success/rejected)                             │ │
│  └────────────────────────────────┬────────────────────────────────────────┘ │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 5. AXIOS INSTANCE (store/apis/axios/crmAxios.js)                        │ │
│  │    - Base URL: ${URL}/crm                                               │ │
│  │    - Interceptors: Auth headers, token refresh, error handling          │ │
│  └────────────────────────────────┬────────────────────────────────────────┘ │
│                                   │                                          │
└───────────────────────────────────┼──────────────────────────────────────────┘
                                    │
                                    │  HTTP GET /crm/listas/getPedidoClienteView
                                    │  Headers: Authorization, X-Refresh-Token
                                    │  Params: ?IdPedidoCliente=123
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              BACKEND                                          │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 6. ROUTER (index.php)                                                   │ │
│  │    $router->get('/crm/listas/getPedidoClienteView',                     │ │
│  │                 [CRMController::class, 'getPedidoClienteView']);        │ │
│  └────────────────────────────────┬────────────────────────────────────────┘ │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 7. CONTROLLER (controllers/CRMController.php)                           │ │
│  │    - Validates JWT tokens                                               │ │
│  │    - Sanitizes input ($_GET params)                                     │ │
│  │    - Instantiates Model                                                 │ │
│  │    - Calls Model method                                                 │ │
│  │    - Returns HTTPreponse()                                              │ │
│  └────────────────────────────────┬────────────────────────────────────────┘ │
│                                   │                                          │
│                                   ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ 8. MODEL (models/Crm/PedidoCliente.php)                                 │ │
│  │    - Extends ActiveRecord                                               │ │
│  │    - Contains business logic                                            │ │
│  │    - Executes SQL queries                                               │ │
│  │    - Returns ResponseHelper::success/error                              │ │
│  └────────────────────────────────┬────────────────────────────────────────┘ │
│                                   │                                          │
│                                   ▼                                          │
│                             SQL Server                                       │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Code Example: usePedidoCliente Flow

#### 1. Component/Hook Usage
```javascript
// src/app/crm/components/ViewPedidos/hooks/usePedidoCliente.js
import { useGetData } from "../../../../hooks/useGetData";
import { getThunkGetPedidoClienteView } from "../../../../../store/slices/crmSlice/crmThunksModule/CrmThunksPedidos";

export const usePedidoCliente = ({ idpedido }) => {
    const fetchPedidoCliente = useGetData();

    const fetchData = async () => {
        if (!idpedido) return;
        await fetchPedidoCliente.getData(
            getThunkGetPedidoClienteView,  // Thunk action
            { IdPedidoCliente: idpedido }   // Params
        );
    };

    useEffect(() => { fetchData(); }, []);

    return {
        fetchPedidoCliente,  // { data, isLocalLoading, catchErrorAtFetch }
        fetchData,
    };
};
```

#### 2. useGetData Hook (Generic Data Fetcher)
```javascript
// src/app/hooks/useGetData.js
export const useGetData = () => {
    const [data, setData] = useState([]);
    const [isLocalLoading, setIsLocalLoading] = useState('pending');
    const [catchErrorAtFetch, setCatchErrorAtFetch] = useState({ hasError: false });
    const dispatch = useDispatch();

    const getData = async (thunkAction, object) => {
        setIsLocalLoading('loading');

        const response = await dispatch(thunkAction({ ...object }));

        if (response.success) {
            setData(response.data);
            setIsLocalLoading('fulfilled');
        } else {
            setCatchErrorAtFetch({ hasError: true, message: response.error });
            setIsLocalLoading('rejected');
        }
    };

    return { data, isLocalLoading, catchErrorAtFetch, getData };
};
```

#### 3. Thunk Definition
```javascript
// store/slices/crmSlice/crmThunksModule/CrmThunksPedidos.js
import { apiThunkGet } from "../thunks-Base/thunksBase";

export const getThunkGetPedidoClienteView = (data) =>
    apiThunkGet({
        endpoint: "listas/getPedidoClienteView",  // Appended to /crm/
        params: data,                              // { IdPedidoCliente: 123 }
        thunk: 'getThunkGetPedidoClienteView'
    });
```

#### 4. Thunks Base (API Executor)
```javascript
// store/slices/crmSlice/thunks-Base/thunksBase.js
import { crmAxios } from "../../../apis/axios/crmAxios";
import ApiResponse from "../../../apiClass/responses";

const apiThunkGet = ({ endpoint, params, thunk }) => {
    return async (dispatch) => {
        try {
            const response = await crmAxios({
                url: endpoint,      // "listas/getPedidoClienteView"
                method: "GET",
                params              // Sent as query string
            });

            if (response.data.status !== 200)
                throw new Error(`Error en API: ${thunk}`);

            // Handle token refresh
            if (response.data.nuevoToken)
                dispatch(actualizarToken({ tokenNuevo: response.data.nuevoToken }));

            return ApiResponse.fullfilled(response.data.data);
        } catch (error) {
            return ApiResponse.rejected({ error: error.message });
        }
    };
};

const apiThunkPost = ({ endpoint, params, thunk }) => {
    return async (dispatch) => {
        try {
            const response = await crmAxios({
                url: endpoint,
                method: "POST",
                data: params    // Sent as request body
            });
            // ... same pattern
        } catch (error) {
            return ApiResponse.rejected({ error: error.message });
        }
    };
};
```

#### 5. ApiResponse Class
```javascript
// store/apiClass/responses.js
class ApiResponse {
    constructor(success, data = null, error = null) {
        this.success = success;
        this.data = data;
        this.error = error;
    }

    static fullfilled(data) {
        return new ApiResponse(true, data, null);
    }

    static rejected({ error, status = 500 }) {
        return new ApiResponse(false, null, { error, status });
    }
}
```

#### 6. Backend Router
```php
// index.php
$router->get('/crm/listas/getPedidoClienteView',
    [CRMController::class, 'getPedidoClienteView']);
```

#### 7. Backend Controller
```php
// controllers/CRMController.php
public static function getPedidoClienteView(): void
{
    $headers = getallheaders();

    // 1. Validate authorization headers
    if (!isset($headers['Authorization']) || !isset($headers['X-Refresh-Token']))
        HTTPreponse(0, 401, "Header de autorizacion no encontrado");

    // 2. Validate JWT tokens
    $dataTokenDecoded = validadorTokenCrm(
        $headers['Authorization'],
        $headers['X-Refresh-Token']
    );
    if (!$dataTokenDecoded)
        HTTPreponse(1, 403, "Error en la validacion de los tokens");

    // 3. Sanitize input
    $IdPedidoCliente = $_GET['IdPedidoCliente'] ?? null;
    $IdPedidoCliente = filter_var($IdPedidoCliente, FILTER_VALIDATE_INT);
    if (!$IdPedidoCliente)
        HTTPreponse(0, 403, "Id de cliente no valido");

    // 4. Instantiate model and execute
    $pedido = new PedidoCliente([
        'IdPedidoCliente' => $IdPedidoCliente,
        'db' => $dataTokenDecoded['database']
    ]);
    $resultado = $pedido->getHeaderMainDocView();

    // 5. Return response with optional new token
    $refreshToken = $dataTokenDecoded['nuevoToken'] ?? null;
    HTTPreponse(1, 200, "Exito", $resultado['data'], $refreshToken);
}
```

#### 8. Backend Model
```php
// models/Crm/PedidoCliente.php
class PedidoCliente extends ActiveRecord
{
    public static $tabla = 'OperPedidosCliente';
    public static $primarykey = 'IdPedidoCliente';

    public function getHeaderMainDocView()
    {
        $query = "SELECT PED.*, CLI.RazonSocial, EMP.Vendedor
                  FROM OperPedidosCliente AS PED
                  LEFT JOIN CataClientes CLI ON PED.IdCliente = CLI.IdCliente
                  LEFT JOIN CataEmpleados EMP ON PED.IdEmpleado = EMP.IdEmpleado
                  WHERE PED.IdPedidoCliente = :IdPedidoCliente";

        return ActiveRecord::SQL(
            $this->db,
            $query,
            ['IdPedidoCliente' => $this->IdPedidoCliente]
        );
    }
}
```

### Creating a New Request (Step by Step)

To add a new API request following this pattern:

**1. Create Thunk Definition** (`store/slices/crmSlice/crmThunksModule/`):
```javascript
// For GET requests
export const getThunkGetMyData = (data) =>
    apiThunkGet({
        endpoint: "listas/myEndpoint",
        params: data,
        thunk: 'getThunkGetMyData'
    });

// For POST requests
export const postThunkCreateMyData = (data) =>
    apiThunkPost({
        endpoint: "set/myEndpoint",
        params: data,
        thunk: 'postThunkCreateMyData'
    });
```

**2. Create Custom Hook** (`src/app/[module]/components/[feature]/hooks/`):
```javascript
export const useMyFeature = ({ id }) => {
    const fetchMyData = useGetData();

    const fetchData = async () => {
        await fetchMyData.getData(getThunkGetMyData, { id });
    };

    useEffect(() => { fetchData(); }, [id]);

    return { fetchMyData, fetchData };
};
```

**3. Backend Route** (`index.php`):
```php
$router->get('/crm/listas/myEndpoint', [CRMController::class, 'myMethod']);
```

**4. Backend Controller** (`controllers/CRMController.php`):
```php
public static function myMethod(): void
{
    $headers = getallheaders();
    $dataTokenDecoded = validadorTokenCrm($headers['Authorization'], $headers['X-Refresh-Token']);

    $id = filter_var($_GET['id'], FILTER_VALIDATE_INT);

    $result = MyModel::SQL($dataTokenDecoded['database'], $query, ['id' => $id]);

    HTTPreponse(1, 200, "Exito", $result['data'], $dataTokenDecoded['nuevoToken'] ?? null);
}
```

**5. Use in Component**:
```jsx
const MyComponent = ({ id }) => {
    const { fetchMyData } = useMyFeature({ id });

    if (fetchMyData.isLocalLoading === 'loading') return <Spinner />;
    if (fetchMyData.catchErrorAtFetch.hasError) return <Error />;

    return <div>{JSON.stringify(fetchMyData.data)}</div>;
};
```

## Development Guidelines

### Adding a New Endpoint

**1. Backend** (`index.php`):
```php
$router->get('/crm/listas/newEndpoint', [CRMController::class, 'newMethod']);
```

**2. Backend Controller**:
```php
public function newMethod() {
    $token = validateToken($_SERVER['HTTP_AUTHORIZATION']);
    if (!$token) HTTPreponse(0, 401, 'Unauthorized');

    $result = ActiveRecord::SQL($db, $query, $params);
    HTTPreponse(1, 200, 'Success', $result['data']);
}
```

**3. Frontend RTK Query** (`todosApi.js`):
```javascript
getNewData: builder.query({
    query: (param) => `/crm/listas/newEndpoint?param=${param}`,
}),
```

**4. Frontend Axios** (for mutations):
```javascript
const response = await crmAxios.post('/set/newEndpoint', data);
```

### Environment Switching

To switch between environments:

1. **Frontend**: Edit `src/global/baseURL.js`
2. **Backend**: Edit CORS headers in `index.php`
3. **Backend**: Edit database connection in `.env`

### Debugging API Calls

**Frontend** (Chrome DevTools):
- Network tab: Filter by XHR
- Check request headers for tokens
- Check response for error messages

**Backend**:
- Check Apache/PHP error logs
- Use `error_log()` for debugging
- Check SQL Server connection

## Project-Specific Documentation

For detailed documentation on each project:
- **Frontend**: See `it_web_service-Front/CLAUDE.md`
- **Backend**: See `it_web_service-Backend_R/CLAUDE.md`

## Common Issues

### CORS Errors
- Ensure backend CORS headers match frontend origin
- Check that OPTIONS preflight requests are handled

### 401 Unauthorized
- Token expired: Check token refresh logic
- Wrong token format: Ensure `Bearer ` prefix
- Missing headers: Check Axios interceptors

### Database Connection
- Verify `.env` credentials
- Check SQL Server is accessible from backend
- Verify ODBC Driver 18 is installed (Docker)

### Token Refresh Not Working
- Check `X-Refresh-Token` header is sent
- Verify refresh token is stored in localStorage
- Check backend token validation logic
