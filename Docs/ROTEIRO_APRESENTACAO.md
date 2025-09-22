# 🎯 Roteiro de Apresentação - Sistema de Listas de Compras com Microsserviços

**Duração:** 10 minutos  
**Público:** Desenvolvedores  
**Objetivo:** Demonstrar arquitetura de microsserviços com API Gateway e NoSQL

---

## 📋 **Estrutura da Apresentação (10 min)**

### **1. Introdução e Contexto (2 min)**
### **2. Demonstração da Arquitetura (3 min)**
### **3. Funcionalidades Práticas (4 min)**
### **4. Pontos Técnicos e Conclusão (1 min)**

---

## 🎤 **ROTEIRO DETALHADO**

### **1. INTRODUÇÃO E CONTEXTO (2 minutos)**

#### **O que vamos ver hoje:**
> "Vou demonstrar um sistema de listas de compras construído com arquitetura de microsserviços. O objetivo é mostrar como dividir uma aplicação em serviços independentes que se comunicam através de um API Gateway."

#### **Por que microsserviços?**
- **Escalabilidade independente** de cada serviço
- **Tecnologias diferentes** por domínio
- **Equipes independentes** podem trabalhar em paralelo
- **Falhas isoladas** não afetam todo o sistema

#### **O que implementamos:**
- **3 Microsserviços** independentes
- **1 API Gateway** como ponto único de entrada
- **Bancos NoSQL** baseados em arquivos JSON
- **Service Discovery** automático
- **Circuit Breaker** para resiliência

---

### **2. DEMONSTRAÇÃO DA ARQUITETURA (3 minutos)**

#### **Mostrar a estrutura:**
```bash
# Execute este comando para mostrar a estrutura
Get-ChildItem -Recurse | Where-Object { $_.Name -match "\.(js|json)$" } | Format-Table Name, Length, Directory -AutoSize
```

#### **Explicar os componentes:**
1. **User Service (porta 3001)** - Autenticação e usuários
2. **Item Service (porta 3003)** - Catálogo de produtos  
3. **List Service (porta 3002)** - Listas de compras
4. **API Gateway (porta 3000)** - Ponto único de entrada

#### **Demonstrar Service Discovery:**
```bash
# Mostrar que os serviços se registram automaticamente
curl http://localhost:3000/registry
```
> "Vejam como os serviços se registram automaticamente no registry compartilhado. Isso permite que o API Gateway descubra onde estão os serviços."

#### **Health Checks:**
```bash
# Mostrar saúde de todos os serviços
curl http://localhost:3000/health
```
> "O sistema faz health checks automáticos a cada 30 segundos. Se um serviço falhar, o circuit breaker protege o sistema."

---

### **3. FUNCIONALIDADES PRÁTICAS (4 minutos)**

#### **3.1. Catálogo de Produtos (1 min)**
```bash
# Mostrar itens disponíveis
curl http://localhost:3003/items?limit=3
```
> "O Item Service tem 25 produtos em 5 categorias. Cada serviço tem seu próprio banco NoSQL independente."

#### **3.2. Autenticação (1 min)**
```bash
# Executar demonstração completa
node client-demo.js
```
> "Vou executar o cliente de demonstração que mostra o fluxo completo: registro de usuário, login, busca de produtos e criação de listas."

**Durante a execução, explique:**
- "O sistema registra um novo usuário"
- "Faz login e recebe um token JWT"
- "Busca produtos no catálogo"
- "Cria uma lista de compras"
- "Adiciona itens à lista"

#### **3.3. API Gateway em Ação (1 min)**
```bash
# Mostrar como o gateway roteia as requisições
curl http://localhost:3000/api/items?limit=2
```
> "Todas as requisições passam pelo API Gateway, que roteia para o serviço correto. Isso centraliza o acesso e permite implementar funcionalidades transversais como autenticação e logging."

#### **3.4. Resiliência (1 min)**
> "Se um serviço falhar, o circuit breaker abre após 3 falhas consecutivas, protegendo o sistema de falhas em cascata."

---

### **4. PONTOS TÉCNICOS E CONCLUSÃO (1 minuto)**

#### **Tecnologias utilizadas:**
- **Node.js + Express** para todos os serviços
- **JSON como banco NoSQL** (simplificado para demonstração)
- **JWT para autenticação**
- **Service Discovery via arquivo compartilhado**
- **Circuit Breaker simples** (3 falhas = abrir)

#### **Benefícios demonstrados:**
- ✅ **Serviços independentes** com bancos próprios
- ✅ **Descoberta automática** de serviços
- ✅ **Resiliência** com circuit breaker
- ✅ **Monitoramento** com health checks
- ✅ **Escalabilidade** horizontal por serviço

#### **Conclusão:**
> "Esta implementação demonstra os conceitos fundamentais de microsserviços: serviços independentes, comunicação via API, service discovery e resiliência. É uma base sólida para sistemas distribuídos mais complexos."

---

## 🛠️ **COMANDOS PARA EXECUTAR DURANTE A APRESENTAÇÃO**

### **Preparação (antes da apresentação):**
```bash
# 1. Iniciar todos os serviços
npm start

# 2. Aguardar 30 segundos para inicialização completa
Start-Sleep -Seconds 30
```

### **Durante a apresentação:**
```bash
# 1. Mostrar estrutura do projeto
Get-ChildItem -Recurse | Where-Object { $_.Name -match "\.(js|json)$" } | Format-Table Name, Length, Directory -AutoSize

# 2. Verificar registry de serviços
curl http://localhost:3000/registry

# 3. Health check geral
curl http://localhost:3000/health

# 4. Mostrar catálogo de produtos
curl http://localhost:3003/items?limit=3

# 5. Executar demonstração completa
node client-demo.js

# 6. Mostrar roteamento via gateway
curl http://localhost:3000/api/items?limit=2
```

---

## 💡 **DICAS PARA A APRESENTAÇÃO**

### **Pontos-chave para enfatizar:**
1. **Independência dos serviços** - cada um pode ser desenvolvido, testado e deployado separadamente
2. **Service Discovery** - serviços se registram automaticamente
3. **Resiliência** - circuit breaker protege contra falhas
4. **Observabilidade** - health checks e logs centralizados
5. **Escalabilidade** - cada serviço pode escalar independentemente

### **Se algo der errado:**
- **Serviços não iniciaram:** Execute `npm start` novamente
- **Erro de conexão:** Verifique se as portas 3000-3003 estão livres
- **Registry vazio:** Aguarde mais 30 segundos para registro automático

### **Tempo estimado por seção:**
- **Introdução:** 2 min
- **Arquitetura:** 3 min  
- **Funcionalidades:** 4 min
- **Conclusão:** 1 min
- **Total:** 10 min

---

## 📚 **CONCEITOS PARA EXPLICAR**

### **Microsserviços:**
- Serviços pequenos e independentes
- Comunicação via APIs REST
- Bancos de dados independentes
- Deploy independente

### **API Gateway:**
- Ponto único de entrada
- Roteamento de requisições
- Funcionalidades transversais (auth, logging)
- Agregação de dados

### **Service Discovery:**
- Como serviços se encontram
- Registry centralizado
- Health checks automáticos
- Failover automático

### **Circuit Breaker:**
- Proteção contra falhas em cascata
- Estados: fechado, aberto, meio-aberto
- Recuperação automática

---

**🎯 Objetivo da apresentação: Mostrar que microsserviços não são apenas hype, mas uma arquitetura real e funcional que resolve problemas concretos de sistemas distribuídos.**
