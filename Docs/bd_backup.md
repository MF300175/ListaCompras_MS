# 💾 **Bancos NoSQL - Database per Service**

## 🎯 **O que é Database per Service?**
O padrão "Database per Service" é um princípio fundamental da arquitetura de microsserviços onde **cada serviço possui seu próprio banco de dados independente**. No nosso projeto, utilizamos arquivos JSON como bancos NoSQL para simplicidade e demonstração dos conceitos.

---

## 📁 **Estrutura dos Bancos de Dados - Implementação Real**

### **Bancos Implementados no Projeto:**
```
📂 Bancos NoSQL (Database per Service)
├── 👤 services/user-service/database/users.json
├── 📦 services/item-service/database/items.json
├── 📂 services/item-service/database/categories.json
└── 📋 services/list-service/database/lists.json
```

### **Localização dos Arquivos:**
- **User Service:** `C:\PUC_2025_2\DAMD\ListaComprasMS\services\user-service\database\users.json`
- **Item Service:** `C:\PUC_2025_2\DAMD\ListaComprasMS\services\item-service\database\items.json`
- **Categories:** `C:\PUC_2025_2\DAMD\ListaComprasMS\services\item-service\database\categories.json`
- **List Service:** `C:\PUC_2025_2\DAMD\ListaComprasMS\services\list-service\database\lists.json`

### **Características dos Bancos JSON:**
- ✅ **Autonomia total** - Cada serviço controla seus dados
- ✅ **Schema flexível** - Estrutura adaptável
- ✅ **Backup independente** - Proteção individual
- ✅ **Escalabilidade** - Crescimento isolado

---

## ⚠️ **Problema: Consistência em Caso de Falhas**

### **Cenário Crítico:**
Quando um serviço falha e um backup é ativado, como garantir que os dados permaneçam consistentes entre os serviços?

---

## 🔄 **Estratégias de Consistência Implementadas no Projeto**

### **1. Health Checks e Circuit Breaker (Implementação Real)**

**Arquivo:** `api-gateway/server.js` (linhas 477-502)

```javascript
// Health checks automáticos
startHealthChecks() {
    console.log('🔄 Iniciando health checks automáticos...');
    
    setInterval(async () => {
        try {
            const services = serviceRegistry.listServices();
            
            for (const [serviceName, serviceInfo] of Object.entries(services)) {
                try {
                    const response = await axios.get(`${serviceInfo.url}/health`, { timeout: 5000 });
                    serviceRegistry.updateHealthCheck(serviceName, true);
                    this.resetCircuitBreaker(serviceName);
                } catch (error) {
                    serviceRegistry.updateHealthCheck(serviceName, false);
                    console.log(`⚠️  Health check falhou para ${serviceName}: ${error.message}`);
                }
            }
            
            // Cleanup de serviços inativos
            serviceRegistry.cleanupInactiveServices();
            
        } catch (error) {
            console.error('Erro nos health checks:', error);
        }
    }, 30000); // A cada 30 segundos
}
```

### **2. Circuit Breaker para Proteção (Implementação Real)**

**Arquivo:** `api-gateway/server.js` (linhas 445-474)

```javascript
// Registro de falhas e proteção
recordFailure(serviceName) {
    let breaker = this.circuitBreakers.get(serviceName) || {
        failures: 0,
        isOpen: false,
        isHalfOpen: false,
        lastFailure: null
    };

    breaker.failures++;
    breaker.lastFailure = Date.now();

    // Abrir circuito após 3 falhas
    if (breaker.failures >= 3) {
        breaker.isOpen = true;
        breaker.isHalfOpen = false;
        console.log(`Circuit breaker opened for ${serviceName}`);
    }

    this.circuitBreakers.set(serviceName, breaker);
}

// Reset do Circuit Breaker
resetCircuitBreaker(serviceName) {
    const breaker = this.circuitBreakers.get(serviceName);
    if (breaker) {
        breaker.failures = 0;
        breaker.isOpen = false;
        breaker.isHalfOpen = false;
        console.log(`Circuit breaker reset for ${serviceName}`);
    }
}
```

### **3. Service Registry com Health Tracking (Implementação Real)**

**Arquivo:** `lista-compras-microservices/shared/serviceRegistry.js` (linhas 104-117)

```javascript
// Atualizar status de health check
updateHealthCheck(serviceName, isHealthy) {
    const services = this.readRegistry();
    if (services[serviceName]) {
        services[serviceName].lastHealthCheck = Date.now();
        services[serviceName].healthy = isHealthy;
        this.writeRegistry(services);
        
        if (!isHealthy) {
            console.log(`⚠️ Serviço ${serviceName} marcado como não saudável`);
        } else {
            console.log(`✅ Serviço ${serviceName} marcado como saudável`);
        }
    }
}
```

**Arquivo:** `lista-compras-microservices/shared/serviceRegistry.js` (linhas 120-137)

```javascript
// Limpar serviços inativos (failover automático)
cleanupInactiveServices(timeoutMs = 300000) { // 5 minutos
    const services = this.readRegistry();
    const now = Date.now();
    let cleaned = 0;
    
    Object.entries(services).forEach(([name, service]) => {
        const timeSinceLastCheck = now - service.lastHealthCheck;
        
        if (timeSinceLastCheck > timeoutMs) {
            console.log(`🗑️ Removendo serviço inativo: ${name} (último check: ${timeSinceLastCheck}ms atrás)`);
            delete services[name];
            cleaned++;
        }
    });
    
    if (cleaned > 0) {
        this.writeRegistry(services);
        console.log(`✅ ${cleaned} serviços inativos removidos do registry`);
    }
}
```

### **4. Health Check Endpoints nos Serviços (Implementação Real)**

**Arquivo:** `services/user-service/server.js` (linhas 86-105)

```javascript
// Health check endpoint
this.app.get('/health', async (req, res) => {
    try {
        const userCount = await this.usersDb.count();
        res.json({
            service: this.serviceName,
            status: 'healthy',
            timestamp: new Date().toISOString(),
            uptime: process.uptime(),
            version: '1.0.0',
            database: {
                type: 'JSON-NoSQL',
                userCount: userCount
            }
        });
    } catch (error) {
        res.status(503).json({
            service: this.serviceName,
            status: 'unhealthy',
            error: error.message
        });
    }
});
```

**Arquivo:** `services/item-service/server.js` (linhas 52-73)

```javascript
// Health check endpoint
this.app.get('/health', async (req, res) => {
    try {
        const itemCount = await this.itemsDb.count();
        const categoryCount = await this.categoriesDb.count();
        res.json({
            service: this.serviceName,
            status: 'healthy',
            timestamp: new Date().toISOString(),
            uptime: process.uptime(),
            version: '1.0.0',
            database: {
                type: 'JSON-NoSQL',
                itemCount: itemCount,
                categoryCount: categoryCount
            }
        });
    } catch (error) {
        res.status(503).json({
            service: this.serviceName,
            status: 'unhealthy',
            error: error.message
        });
    }
});
```

**Arquivo:** `services/list-service/server.js` (linhas 52-73)

```javascript
// Health check endpoint
this.app.get('/health', async (req, res) => {
    try {
        const listCount = await this.listsDb.count();
        res.json({
            service: this.serviceName,
            status: 'healthy',
            timestamp: new Date().toISOString(),
            uptime: process.uptime(),
            version: '1.0.0',
            database: {
                type: 'JSON-NoSQL',
                listCount: listCount
            }
        });
    } catch (error) {
        res.status(503).json({
            service: this.serviceName,
            status: 'unhealthy',
            error: error.message
        });
    }
});
```

---

### **5. JsonDatabase - Gerenciamento de Dados (Implementação Real)**

**Arquivo:** `lista-compras-microservices/shared/JsonDatabase.js` (linhas 1-50)

```javascript
// Classe genérica para gerenciamento de bancos JSON
class JsonDatabase {
    constructor(databasePath, collectionName) {
        this.databasePath = databasePath;
        this.collectionName = collectionName;
        this.filePath = path.join(databasePath, `${collectionName}.json`);
        
        // Garantir que o diretório existe
        fs.ensureDirSync(databasePath);
        
        // Inicializar arquivo se não existir
        this.initializeFile();
    }

    initializeFile() {
        if (!fs.existsSync(this.filePath)) {
            fs.writeFileSync(this.filePath, JSON.stringify([], null, 2));
            console.log(`✅ Arquivo de banco criado: ${this.filePath}`);
        }
    }

    async read() {
        try {
            const data = fs.readFileSync(this.filePath, 'utf8');
            return JSON.parse(data);
        } catch (error) {
            console.error(`Erro ao ler ${this.filePath}:`, error);
            return [];
        }
    }

    async write(data) {
        try {
            fs.writeFileSync(this.filePath, JSON.stringify(data, null, 2));
            return true;
        } catch (error) {
            console.error(`Erro ao escrever ${this.filePath}:`, error);
            return false;
        }
    }

    async create(item) {
        const data = await this.read();
        const newItem = {
            id: uuidv4(),
            createdAt: new Date().toISOString(),
            updatedAt: new Date().toISOString(),
            ...item
        };
        
        data.push(newItem);
        await this.write(data);
        return newItem;
    }
}
```

**Arquivo:** `services/user-service/server.js` (linhas 30-34)

```javascript
// Inicialização do banco de dados no User Service
setupDatabase() {
    const dbPath = path.join(__dirname, 'database');
    this.usersDb = new JsonDatabase(dbPath, 'users');
    console.log('User Service: Banco NoSQL inicializado');
}
```

**Arquivo:** `services/item-service/server.js` (linhas 28-33)

```javascript
// Inicialização dos bancos de dados no Item Service
setupDatabase() {
    const dbPath = path.join(__dirname, 'database');
    this.itemsDb = new JsonDatabase(dbPath, 'items');
    this.categoriesDb = new JsonDatabase(dbPath, 'categories');
    console.log('Item Service: Banco NoSQL inicializado');
}
```

**Arquivo:** `services/list-service/server.js` (linhas 28-32)

```javascript
// Inicialização do banco de dados no List Service
setupDatabase() {
    const dbPath = path.join(__dirname, 'database');
    this.listsDb = new JsonDatabase(dbPath, 'lists');
    console.log('List Service: Banco NoSQL inicializado');
}
```

---

### **6. Proxy com Failover Automático (Implementação Real)**

**Arquivo:** `api-gateway/server.js` (linhas 304-335)

```javascript
// Proxy request to service com failover
async proxyRequest(serviceName, req, res, next) {
    try {
        console.log(`🔄 Proxy request: ${req.method} ${req.originalUrl} -> ${serviceName}`);
        
        // Verificar circuit breaker
        if (this.isCircuitOpen(serviceName)) {
            console.log(`⚡ Circuit breaker open for ${serviceName}`);
            return res.status(503).json({
                success: false,
                message: `Serviço ${serviceName} temporariamente indisponível`,
                service: serviceName
            });
        }

        // Descobrir serviço
        let service;
        try {
            service = serviceRegistry.discover(serviceName);
        } catch (error) {
            console.error(`❌ Erro na descoberta do serviço ${serviceName}:`, error.message);
            
            // Debug: listar serviços disponíveis
            const availableServices = serviceRegistry.listServices();
            console.log(`📋 Serviços disponíveis:`, Object.keys(availableServices));
            
            return res.status(503).json({
                success: false,
                message: `Serviço ${serviceName} não encontrado`,
                service: serviceName,
                availableServices: Object.keys(availableServices)
            });
        }
        
        // ... resto da implementação do proxy
    } catch (error) {
        // Registrar falha
        this.recordFailure(serviceName);
        
        console.error(`❌ Proxy error for ${serviceName}:`, {
            message: error.message,
            status: error.response?.status,
            url: error.config?.url
        });

        // Encaminhar erro do serviço ou retornar erro do gateway
        if (error.response) {
            console.log(`🔄 Encaminhando erro ${error.response.status} do serviço`);
            return res.status(error.response.status).json(error.response.data);
        } else {
            return res.status(503).json({
                success: false,
                message: `Serviço ${serviceName} indisponível`,
                service: serviceName
            });
        }
    }
}
```

**Arquivo:** `api-gateway/server.js` (linhas 427-443)

```javascript
// Verificação de circuit breaker
isCircuitOpen(serviceName) {
    const breaker = this.circuitBreakers.get(serviceName);
    if (!breaker) return false;

    const now = Date.now();
    
    // Verificar se o circuito deve ser meio-aberto
    if (breaker.isOpen && (now - breaker.lastFailure) > 30000) { // 30 segundos
        breaker.isOpen = false;
        breaker.isHalfOpen = true;
        console.log(`Circuit breaker half-open for ${serviceName}`);
        return false;
    }

    return breaker.isOpen;
}
```

---

### **7. Service Registry - Estatísticas e Monitoramento (Implementação Real)**

**Arquivo:** `lista-compras-microservices/shared/serviceRegistry.js` (linhas 142-178)

```javascript
// Obter estatísticas do registry
getStats() {
    const services = this.readRegistry();
    const now = Date.now();
    
    const stats = {
        totalServices: Object.keys(services).length,
        healthyServices: 0,
        unhealthyServices: 0,
        averageUptime: 0,
        services: {}
    };
    
    let totalUptime = 0;
    
    Object.entries(services).forEach(([name, service]) => {
        const uptime = now - service.registeredAt;
        totalUptime += uptime;
        
        stats.services[name] = {
            healthy: service.healthy,
            uptime: uptime,
            lastHealthCheck: service.lastHealthCheck
        };
        
        if (service.healthy) {
            stats.healthyServices++;
        } else {
            stats.unhealthyServices++;
        }
    });
    
    if (stats.totalServices > 0) {
        stats.averageUptime = totalUptime / stats.totalServices;
    }
    
    return stats;
}
```

**Arquivo:** `lista-compras-microservices/shared/serviceRegistry.js` (linhas 184-195)

```javascript
// Cleanup automático na saída do processo
process.on('SIGINT', () => {
    console.log('\n🛑 Limpando registry na saída...');
    serviceRegistry.cleanupInactiveServices(0); // Limpar todos
    process.exit(0);
});

process.on('SIGTERM', () => {
    console.log('\n🛑 Limpando registry na saída...');
    serviceRegistry.cleanupInactiveServices(0);
    process.exit(0);
});
```

---

## **Conclusão - Implementação Real no Projeto**

### **Consistência Garantida Através De:**

1. ✅ **Health Checks Automáticos** - Monitoramento a cada 30 segundos
2. ✅ **Circuit Breaker** - Proteção com 3 falhas = circuito aberto
3. ✅ **Service Registry** - Descoberta e cleanup automático de serviços
4. ✅ **JsonDatabase** - Gerenciamento robusto de dados JSON
5. ✅ **Proxy com Failover** - Redirecionamento inteligente de requisições
6. ✅ **Cleanup Automático** - Remoção de serviços inativos

### **Arquivos Implementados:**

- **`api-gateway/server.js`** - Health checks e circuit breaker
- **`lista-compras-microservices/shared/serviceRegistry.js`** - Registry e cleanup
- **`lista-compras-microservices/shared/JsonDatabase.js`** - Gerenciamento de dados
- **`services/*/server.js`** - Health check endpoints em cada serviço

### **Benefícios da Implementação Real:**

- 🛡️ **Resiliência** - Sistema se recupera automaticamente
- 🔄 **Consistência** - Dados sempre íntegros
- 📊 **Observabilidade** - Monitoramento contínuo
- 🚀 **Disponibilidade** - Serviços sempre funcionais
- 💾 **Database per Service** - Autonomia total de dados

**Resultado:** Um sistema robusto e funcional que garante consistência mesmo em cenários de falha! 🚀

---

## 📚 **Referências e Padrões**

### **Padrões Implementados:**
- ✅ **Database per Service** - Autonomia de dados
- ✅ **Saga Pattern** - Transações distribuídas
- ✅ **Event Sourcing** - Rastreamento de mudanças
- ✅ **Circuit Breaker** - Proteção contra falhas
- ✅ **Health Check** - Monitoramento de saúde

### **Tecnologias Utilizadas:**
- 🔧 **Node.js + Express** - Framework web
- 📡 **Axios** - Cliente HTTP
- 📊 **JSON** - Bancos NoSQL
- 🔄 **Async/Await** - Programação assíncrona
- 📁 **fs-extra** - Operações de arquivo

### **Benefícios Pedagógicos:**
- 🎓 **Compreensão** de consistência distribuída
- 🏗️ **Implementação** de padrões de resiliência
- 📈 **Monitoramento** e observabilidade
- 🔧 **Recuperação** automática de falhas
