# 🔍 **Service Registry - Descoberta de Serviços**

## 🎯 **O que é Service Registry?**
O Service Registry é um padrão fundamental da arquitetura de microsserviços que atua como um **"catálogo telefônico"** dos serviços. Ele permite que os serviços se registrem automaticamente e sejam descobertos dinamicamente, eliminando a necessidade de configuração manual de endereços.

---

## 📁 **Implementação Real no Projeto**

### **Arquivo Principal:**
- **`lista-compras-microservices/shared/serviceRegistry.js`** - Implementação completa do Service Registry

### **Arquivo de Dados:**
- **`lista-compras-microservices/shared/services-registry.json`** - Armazenamento das informações dos serviços

---

## 🔧 **Funcionalidades Implementadas**

### **1. Registro Automático de Serviços (Implementação Real)**

**Arquivo:** `lista-compras-microservices/shared/serviceRegistry.js` (linhas 14-36)

```javascript
// Registrar um serviço
register(serviceName, serviceUrl, port) {
    const services = this.readRegistry();
    
    const serviceInfo = {
        name: serviceName,
        url: serviceUrl,
        port: port,
        registeredAt: Date.now(),
        lastHealthCheck: Date.now(),
        healthy: true
    };
    
    services[serviceName] = serviceInfo;
    this.writeRegistry(services);
    
    console.log(`✅ Serviço registrado: ${serviceName} em ${serviceUrl}:${port}`);
    return serviceInfo;
}
```

### **2. Descoberta Dinâmica de Serviços (Implementação Real)**

**Arquivo:** `lista-compras-microservices/shared/serviceRegistry.js` (linhas 38-52)

```javascript
// Descobrir um serviço
discover(serviceName) {
    const services = this.readRegistry();
    
    if (!services[serviceName]) {
        throw new Error(`Serviço ${serviceName} não encontrado no registry`);
    }
    
    if (!services[serviceName].healthy) {
        throw new Error(`Serviço ${serviceName} está marcado como não saudável`);
    }
    
    console.log(`🔍 Serviço descoberto: ${serviceName} -> ${services[serviceName].url}`);
    return services[serviceName];
}
```

### **3. Listagem de Serviços Disponíveis (Implementação Real)**

**Arquivo:** `lista-compras-microservices/shared/serviceRegistry.js` (linhas 54-62)

```javascript
// Listar todos os serviços
listServices() {
    const services = this.readRegistry();
    console.log(`📋 Serviços registrados: ${Object.keys(services).join(', ')}`);
    return services;
}
```

### **4. Health Checks Automáticos (Implementação Real)**

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

### **5. Cleanup Automático de Serviços Inativos (Implementação Real)**

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

---

## 🔄 **Como os Serviços se Registram (Implementação Real)**

### **User Service:**
**Arquivo:** `services/user-service/server.js` (linhas 107-116)

```javascript
// Registrar serviço no registry
async registerService() {
    try {
        await serviceRegistry.register(this.serviceName, this.serviceUrl, this.port);
        console.log(`✅ ${this.serviceName} registrado com sucesso`);
    } catch (error) {
        console.error(`❌ Erro ao registrar ${this.serviceName}:`, error.message);
    }
}
```

### **Item Service:**
**Arquivo:** `services/item-service/server.js` (linhas 75-84)

```javascript
// Registrar serviço no registry
async registerService() {
    try {
        await serviceRegistry.register(this.serviceName, this.serviceUrl, this.port);
        console.log(`✅ ${this.serviceName} registrado com sucesso`);
    } catch (error) {
        console.error(`❌ Erro ao registrar ${this.serviceName}:`, error.message);
    }
}
```

### **List Service:**
**Arquivo:** `services/list-service/server.js` (linhas 75-84)

```javascript
// Registrar serviço no registry
async registerService() {
    try {
        await serviceRegistry.register(this.serviceName, this.serviceUrl, this.port);
        console.log(`✅ ${this.serviceName} registrado com sucesso`);
    } catch (error) {
        console.error(`❌ Erro ao registrar ${this.serviceName}:`, error.message);
    }
}
```

---

## 🌐 **Como o API Gateway Usa o Service Registry (Implementação Real)**

### **Descoberta de Serviços no Gateway:**
**Arquivo:** `api-gateway/server.js` (linhas 371-388)

```javascript
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
```

### **Health Checks Automáticos no Gateway:**
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

---

## 📊 **Estatísticas e Monitoramento (Implementação Real)**

### **Estatísticas do Registry:**
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

---

## 🛡️ **Cleanup Automático na Saída (Implementação Real)**

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

## 📁 **Estrutura do Arquivo services-registry.json**

### **Exemplo de Conteúdo:**
```json
{
  "user-service": {
    "name": "user-service",
    "url": "http://localhost:3001",
    "port": 3001,
    "registeredAt": 1705123456789,
    "lastHealthCheck": 1705123456789,
    "healthy": true
  },
  "item-service": {
    "name": "item-service",
    "url": "http://localhost:3003",
    "port": 3003,
    "registeredAt": 1705123456790,
    "lastHealthCheck": 1705123456790,
    "healthy": true
  },
  "list-service": {
    "name": "list-service",
    "url": "http://localhost:3002",
    "port": 3002,
    "registeredAt": 1705123456791,
    "lastHealthCheck": 1705123456791,
    "healthy": true
  }
}
```

---

## 🎯 **Benefícios da Implementação**

### **1. Descoberta Dinâmica:**
- ✅ **Sem configuração manual** - Serviços se registram automaticamente
- ✅ **Flexibilidade** - Novos serviços são descobertos automaticamente
- ✅ **Escalabilidade** - Fácil adição/remoção de instâncias

### **2. Resiliência:**
- ✅ **Health monitoring** - Verificação contínua da saúde dos serviços
- ✅ **Cleanup automático** - Remoção de serviços inativos
- ✅ **Failover** - Redirecionamento para serviços saudáveis

### **3. Observabilidade:**
- ✅ **Estatísticas** - Monitoramento de uptime e saúde
- ✅ **Logs detalhados** - Rastreamento de registros e descobertas
- ✅ **Debugging** - Listagem de serviços disponíveis

---

## **Conclusão - Service Registry Funcional**

### **Implementação Completa:**
- ✅ **Registro automático** - Todos os serviços se registram ao iniciar
- ✅ **Descoberta dinâmica** - API Gateway encontra serviços automaticamente
- ✅ **Health checks** - Monitoramento contínuo a cada 30 segundos
- ✅ **Cleanup automático** - Remoção de serviços inativos
- ✅ **Estatísticas** - Monitoramento completo do registry

### **Arquivos Envolvidos:**
- **`serviceRegistry.js`** - Lógica principal do registry
- **`services-registry.json`** - Armazenamento dos dados
- **`api-gateway/server.js`** - Uso do registry no gateway
- **`services/*/server.js`** - Registro dos serviços

**Resultado:** Um Service Registry completamente funcional que garante descoberta dinâmica e resiliência do sistema! 🚀
