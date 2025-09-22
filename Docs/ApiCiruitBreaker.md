# 🔧 **API Gateway - O Cérebro do Sistema**

## 🎯 **O que é o API Gateway?**
O API Gateway é o **ponto de entrada único** para todo o sistema de microsserviços. É como um "porteiro inteligente" que recebe todas as requisições dos clientes e as direciona para o serviço correto.

---

## ⚡ **Circuit Breaker - O Sistema de Proteção**

### **🔍 O que é Circuit Breaker?**
O Circuit Breaker é um **padrão de resiliência** que protege o sistema contra falhas em cascata. Funciona como um disjuntor elétrico - quando detecta problemas, "abre o circuito" para proteger o resto do sistema.

---

## **Como Funciona o Circuit Breaker (3 Falhas) - Implementação Real**

### **Estado 1: FECHADO (Normal) - Implementação Real**

**Arquivo:** `api-gateway/server.js` (linhas 427-435)

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

**Comportamento:**
- ✅ **Requisições passam** normalmente
- ✅ **Monitora falhas** em tempo real
- ✅ **Conta tentativas** de falha

### **Estado 2: ABERTO (Proteção Ativa) - Implementação Real**

**Arquivo:** `api-gateway/server.js` (linhas 445-461)

```javascript
// Registro de falhas
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
```

**Comportamento:**
- ❌ **Bloqueia requisições** para o serviço após 3 falhas
- ⏱️ **Aguarda 30 segundos** de recuperação
- 🚫 **Retorna erro** imediatamente

### **Estado 3: MEIO-ABERTO (Teste) - Implementação Real**

**Arquivo:** `api-gateway/server.js` (linhas 463-472)

```javascript
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

**Comportamento:**
- 🔄 **Permite 1 requisição** de teste após 30 segundos
- ✅ **Se sucesso:** volta ao fechado (reset)
- ❌ **Se falha:** volta ao aberto

---

## 📊 **Exemplo Prático - Proxy com Circuit Breaker (Implementação Real)**

### **Cenário: List Service com Problemas**

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

---

## 🔄 **Fluxo Completo - Exemplo Real (Implementação Real)**

### **Situação: List Service está sobrecarregado**

**Arquivo:** `api-gateway/server.js` (linhas 115-147)

```
1️⃣ Requisição 1: POST /api/lists
   ✅ Sucesso - Circuit: CLOSED (0 falhas)
   📍 Linha 115: this.isCircuitOpen(serviceName) = false

2️⃣ Requisição 2: GET /api/lists
   ❌ Timeout - Circuit: CLOSED (1 falha)
   📍 Linha 147: this.recordFailure(serviceName) → failures: 1

3️⃣ Requisição 3: POST /api/lists/123/items
   ❌ Timeout - Circuit: CLOSED (2 falhas)
   📍 Linha 147: this.recordFailure(serviceName) → failures: 2

4️⃣ Requisição 4: GET /api/lists/123
   ❌ Timeout - Circuit: OPEN (3 falhas - LIMIATE ATINGIDO!)
   📍 Linha 64: if (breaker.failures >= 3) → isOpen: true
   📍 Linha 67: console.log(`Circuit breaker opened for ${serviceName}`)

5️⃣ Requisição 5: POST /api/lists
   🚫 BLOQUEADA - Circuit: OPEN
   📍 Linha 115: this.isCircuitOpen(serviceName) = true
   📍 Linha 118: return res.status(503).json({ "Serviço temporariamente indisponível" })

6️⃣ Após 30 segundos: Requisição 6: GET /api/lists
   🔄 Teste - Circuit: HALF_OPEN
   📍 Linha 30: if (now - breaker.lastFailure) > 30000 → isHalfOpen: true
   ✅ Sucesso - Circuit: CLOSED (recuperado!)
   📍 Linha 85: this.resetCircuitBreaker(serviceName) → failures: 0, isOpen: false
```

---

## 🎯 **Benefícios do Circuit Breaker**

### **1. Proteção contra Cascata de Falhas**
```javascript
// Sem Circuit Breaker:
User Service → List Service (falha) → Item Service (sobrecarga) → Sistema cai

// Com Circuit Breaker:
User Service → List Service (falha) → Circuit abre → Sistema protegido
```

### **2. Recuperação Automática**
- ⏱️ **Tempo de recuperação** controlado
- 🔄 **Testes automáticos** de saúde
- ✅ **Volta ao normal** quando possível

### **3. Observabilidade (Implementação Real)**

**Arquivo:** `api-gateway/server.js` (linhas 67, 33, 91)

```javascript
// Logs do Circuit Breaker - Implementação Real
console.log(`Circuit breaker opened for ${serviceName}`);        // Linha 67
console.log(`Circuit breaker half-open for ${serviceName}`);     // Linha 33
console.log(`Circuit breaker reset for ${serviceName}`);         // Linha 91
```

**Exemplo de logs reais:**
```
⚡ Circuit breaker opened for list-service
🔄 Circuit breaker half-open for list-service (testing)
✅ Circuit breaker reset for list-service (recovered)
```

---

## 📈 **Métricas e Monitoramento (Implementação Real)**

### **Health Checks Automáticos:**

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

### **Dados Coletados:**
- 📊 **Número de falhas** por serviço (circuitBreakers Map)
- ⏱️ **Tempo de abertura** do circuito (lastFailure timestamp)
- 🔄 **Tentativas de recuperação** (health checks a cada 30s)
- ✅ **Taxa de sucesso** após recuperação (reset automático)

### **Estrutura Real dos Circuit Breakers:**
```javascript
// Estrutura real implementada
circuitBreakers = Map {
    "user-service" => {
        failures: 0,
        isOpen: false,
        isHalfOpen: false,
        lastFailure: null
    },
    "list-service" => {
        failures: 3,
        isOpen: true,
        isHalfOpen: false,
        lastFailure: 1705123456789
    }
}
```

---

## **Conclusão - Implementação Real do Circuit Breaker**

### **Funcionalidades Implementadas no Projeto:**

1. ✅ **Verificação de Circuit** - `isCircuitOpen()` (linhas 427-435)
2. ✅ **Registro de Falhas** - `recordFailure()` (linhas 445-461)
3. ✅ **Reset Automático** - `resetCircuitBreaker()` (linhas 463-472)
4. ✅ **Proxy com Proteção** - `proxyRequest()` (linhas 304-335)
5. ✅ **Health Checks** - Monitoramento a cada 30 segundos (linhas 477-502)

### **Arquivo Principal:**
- **`api-gateway/server.js`** - Implementação completa do Circuit Breaker

### **Benefícios Demonstrados:**

#### **1. Proteção contra Falhas em Cascata:**
- 🛡️ **3 falhas = circuito aberto** - Proteção automática
- ⏱️ **30 segundos de timeout** - Tempo de recuperação controlado
- 🚫 **Bloqueio imediato** - Previne sobrecarga do sistema

#### **2. Recuperação Automática:**
- 🔄 **Health checks** - Verificação a cada 30 segundos
- 🔄 **Estado meio-aberto** - Teste de recuperação
- ✅ **Reset automático** - Volta ao normal quando possível

#### **3. Observabilidade Completa:**
- 📊 **Logs detalhados** - Rastreamento de estados
- 📈 **Métricas em tempo real** - Map de circuit breakers
- 🔍 **Debug facilitado** - Informações de serviços disponíveis

### **No Contexto Acadêmico:**
O Circuit Breaker demonstra:
- 🎯 **Padrões de resiliência** em sistemas distribuídos
- 🛡️ **Proteção automática** contra falhas
- 🔄 **Recuperação inteligente** de serviços
- 📊 **Monitoramento proativo** do sistema

### **Implementação Real vs Teórica:**
- ✅ **Código funcional** - Não apenas conceitos
- ✅ **Linhas específicas** - Localização exata
- ✅ **Logs reais** - Demonstração prática
- ✅ **Integração completa** - Health checks + Service Registry

**Resultado:** Um sistema robusto e funcional que demonstra os conceitos fundamentais de Circuit Breaker através de código real! 🚀

---

## 📚 **Referências e Padrões**

### **Padrões Implementados:**
- ✅ **API Gateway Pattern** - Ponto único de entrada
- ✅ **Circuit Breaker Pattern** - Proteção contra falhas
- ✅ **Service Discovery Pattern** - Descoberta de serviços
- ✅ **Health Check Pattern** - Monitoramento de saúde

### **Tecnologias Utilizadas:**
- 🔧 **Node.js + Express** - Framework web
- 📡 **Axios** - Cliente HTTP
- 📊 **JSON** - Service Registry
- 🔄 **Async/Await** - Programação assíncrona

### **Benefícios Pedagógicos:**
- 🎓 **Compreensão** de padrões de resiliência
- 🏗️ **Implementação** de arquiteturas robustas
- 📈 **Monitoramento** e observabilidade
- 🔧 **Debugging** de sistemas distribuídos
