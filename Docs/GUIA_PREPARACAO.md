# 📚 Guia de Preparação - Sistema de Microsserviços

## 🎯 **Objetivo**
Este guia te ajudará a entender o sistema antes da apresentação, para que você possa explicar com confiança cada componente.

---

## 🏗️ **ARQUITETURA DO SISTEMA**

### **Visão Geral:**
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   User Service  │    │   Item Service  │    │   List Service  │
│   (porta 3001)  │    │   (porta 3003)  │    │   (porta 3002)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   API Gateway   │
                    │   (porta 3000)  │
                    └─────────────────┘
```

### **O que cada serviço faz:**

#### **1. User Service (porta 3001)**
- **Função:** Gerenciar usuários e autenticação
- **Banco:** `services/user-service/database/users.json`
- **Endpoints principais:**
  - `POST /auth/register` - Registrar usuário
  - `POST /auth/login` - Login (retorna JWT)
  - `GET /users/:id` - Buscar usuário
- **Dados iniciais:** Usuário admin (admin@listacompras.com / admin123)

#### **2. Item Service (porta 3003)**
- **Função:** Catálogo de produtos
- **Banco:** `services/item-service/database/items.json` e `categories.json`
- **Endpoints principais:**
  - `GET /items` - Listar produtos
  - `GET /items/:id` - Buscar produto específico
  - `GET /search?q=termo` - Buscar produtos
- **Dados iniciais:** 25 produtos em 5 categorias (Alimentos, Limpeza, Higiene, Bebidas, Padaria)

#### **3. List Service (porta 3002)**
- **Função:** Gerenciar listas de compras
- **Banco:** `services/list-service/database/lists.json`
- **Endpoints principais:**
  - `POST /lists` - Criar lista
  - `GET /lists` - Listar listas do usuário
  - `POST /lists/:id/items` - Adicionar item à lista
- **Funcionalidades:** Calcula totais automaticamente

#### **4. API Gateway (porta 3000)**
- **Função:** Ponto único de entrada e roteamento
- **Recursos:**
  - Service Discovery automático
  - Circuit Breaker (3 falhas = abrir)
  - Health checks a cada 30 segundos
  - Dashboard agregado
- **Roteamento:**
  - `/api/auth/*` → User Service
  - `/api/users/*` → User Service
  - `/api/items/*` → Item Service
  - `/api/lists/*` → List Service

---

## 🔧 **COMPONENTES TÉCNICOS**

### **1. Service Discovery**
- **Arquivo:** `lista-compras-microservices/shared/services-registry.json`
- **Como funciona:** Cada serviço se registra automaticamente na inicialização
- **Benefício:** API Gateway descobre serviços dinamicamente

### **2. Bancos NoSQL**
- **Implementação:** `JsonDatabase.js` - classe personalizada
- **Características:**
  - Arquivos JSON como banco de dados
  - Operações CRUD completas
  - Busca textual
  - Timestamps automáticos

### **3. Circuit Breaker**
- **Implementação:** Simples (3 falhas = abrir circuito)
- **Estados:**
  - **Fechado:** Requisições passam normalmente
  - **Aberto:** Bloqueia requisições (proteção)
  - **Meio-aberto:** Testa se serviço recuperou

### **4. Autenticação JWT**
- **Biblioteca:** `jsonwebtoken`
- **Fluxo:** Login → Token JWT → Token em todas as requisições
- **Expiração:** 24 horas

---

## 📁 **ESTRUTURA DE ARQUIVOS**

```
ListaComprasMS/
├── package.json                    # Configuração principal
├── client-demo.js                  # Cliente de demonstração original
├── demo-apresentacao.js            # Script para apresentação
├── ROTEIRO_APRESENTACAO.md         # Este roteiro
├── api-gateway/
│   ├── package.json
│   └── server.js                   # API Gateway
├── lista-compras-microservices/
│   ├── package.json
│   └── shared/
│       ├── JsonDatabase.js         # Banco NoSQL
│       └── serviceRegistry.js      # Service Discovery
└── services/
    ├── user-service/
    │   ├── package.json
    │   └── server.js               # User Service
    ├── item-service/
    │   ├── package.json
    │   └── server.js               # Item Service
    └── list-service/
        ├── package.json
        └── server.js               # List Service
```

---

## 🚀 **COMANDOS ESSENCIAIS**

### **Inicialização:**
```bash
# Instalar dependências (só na primeira vez)
npm install
cd services/user-service && npm install
cd ../item-service && npm install  
cd ../list-service && npm install
cd ../../api-gateway && npm install
cd ..

# Iniciar todos os serviços
npm start
```

### **Verificação:**
```bash
# Health check geral
curl http://localhost:3000/health

# Registry de serviços
curl http://localhost:3000/registry

# Demonstração para apresentação
node demo-apresentacao.js
```

---

## 💡 **CONCEITOS PARA EXPLICAR**

### **1. Por que microsserviços?**
- **Escalabilidade:** Cada serviço pode escalar independentemente
- **Tecnologia:** Diferentes serviços podem usar tecnologias diferentes
- **Equipes:** Equipes podem trabalhar independentemente
- **Falhas:** Falha em um serviço não derruba todo o sistema

### **2. Service Discovery**
- **Problema:** Como o API Gateway sabe onde estão os serviços?
- **Solução:** Registry compartilhado onde serviços se registram
- **Benefício:** Serviços podem mudar de porta sem quebrar o sistema

### **3. Circuit Breaker**
- **Problema:** Se um serviço falha, pode causar falha em cascata
- **Solução:** Circuit breaker detecta falhas e bloqueia requisições
- **Benefício:** Sistema continua funcionando mesmo com falhas parciais

### **4. API Gateway**
- **Problema:** Cliente precisa conhecer todos os serviços
- **Solução:** Ponto único de entrada que roteia requisições
- **Benefício:** Centraliza autenticação, logging, rate limiting

---

## 🎭 **CENÁRIOS DE DEMONSTRAÇÃO**

### **Cenário 1: Sistema Saudável**
```bash
curl http://localhost:3000/health
```
**O que mostrar:** 3 serviços saudáveis, uptime, estatísticas

### **Cenário 2: Service Discovery**
```bash
curl http://localhost:3000/registry
```
**O que mostrar:** Serviços registrados automaticamente, PIDs diferentes

### **Cenário 3: Catálogo de Produtos**
```bash
curl http://localhost:3003/items?limit=3
```
**O que mostrar:** 25 produtos em 5 categorias, banco NoSQL independente

### **Cenário 4: Fluxo Completo**
```bash
node demo-apresentacao.js
```
**O que mostrar:** Registro → Login → JWT → Criação de lista → Adição de itens

### **Cenário 5: Roteamento via Gateway**
```bash
curl http://localhost:3000/api/items?limit=2
```
**O que mostrar:** Gateway roteia para Item Service, proxy funcionando

---

## 🛠️ **TROUBLESHOOTING**

### **Se serviços não iniciarem:**
```bash
# Verificar se portas estão livres
netstat -an | findstr "3000 3001 3002 3003"

# Reiniciar serviços
Ctrl+C (parar)
npm start
```

### **Se registry estiver vazio:**
```bash
# Aguardar mais tempo para registro
Start-Sleep -Seconds 30
curl http://localhost:3000/registry
```

### **Se health check falhar:**
```bash
# Verificar serviços individuais
curl http://localhost:3001/health  # User Service
curl http://localhost:3002/health  # List Service  
curl http://localhost:3003/health  # Item Service
```

---

## 🎯 **PONTOS-CHAVE PARA ENFATIZAR**

1. **Independência:** Cada serviço é independente (porta, banco, deploy)
2. **Descoberta:** Serviços se registram automaticamente
3. **Resiliência:** Circuit breaker protege contra falhas
4. **Observabilidade:** Health checks e logs centralizados
5. **Escalabilidade:** Cada serviço pode escalar independentemente
6. **Tecnologia:** Bancos NoSQL flexíveis para cada domínio

---

## 📝 **ROTEIRO DE EMERGÊNCIA (se algo der errado)**

### **Se a demonstração falhar:**
1. **Explique os conceitos** teoricamente
2. **Mostre o código** dos arquivos principais
3. **Demonstre a estrutura** de diretórios
4. **Explique os benefícios** de microsserviços

### **Pontos de backup:**
- Arquitetura em slides ou desenho
- Explicação dos componentes
- Benefícios de microsserviços vs monolito
- Casos de uso reais

---

**🎯 Lembre-se: O objetivo é mostrar que microsserviços são uma arquitetura real e funcional, não apenas teoria!**
