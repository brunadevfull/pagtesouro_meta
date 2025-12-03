# Autenticação PAPEM → DGOM → STN

## 📋 Sumário Executivo

Este documento descreve o fluxo de autenticação entre os sistemas:
- **PAPEM** (Sistema de Pagamento da Escola de Aprendizes-Marinheiros)
- **DGOM** (Diretoria Geral de Orçamento da Marinha) - atua como **proxy**
- **STN** (Secretaria do Tesouro Nacional via PagTesouro)

**Conclusão**: DGOM funciona como **proxy de autenticação**, repassando o token JWT da PAPEM para a STN sem validação intermediária.

---

## 🔐 Modelo de Autenticação

### Token JWT Único

**Token**: Subject "773200" (código da UG PAPEM)
```
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI3NzMyMDAifQ...
```

**Características**:
- **Algoritmo**: RS256 (RSA-SHA256)
- **Subject**: "773200" (identificador da PAPEM)
- **Emissor**: STN (possui chave privada para assinar)
- **Validador**: STN (possui chave pública para validar)

---

## 🔄 Fluxo Completo de Autenticação

```
┌──────────────────────────────────────────────────────────────────┐
│                   FLUXO DE AUTENTICAÇÃO                           │
└──────────────────────────────────────────────────────────────────┘

1. PAPEM (Origem)
   ├─ Token hardcoded no código
   │  Arquivo: @PAGTESOURO_PAPEM/PagTesouroClasse.php:6
   │  $chave = "eyJhbGci...773200..."
   │
   ├─ Prepara requisição HTTPS
   │  URL: https://pagtesouro.dgom.mb:3000/handle
   │  Header: Authorization: Bearer <token>
   │  Arquivo: PagTesouroClasse.php:137, 329, 506, 681, 858
   │
   └─ Valida certificado SSL da DGOM
      Arquivo: MarinhadoBrasilAutoridadeCertificadoradaRECIM-chain.pem

                    ↓ HTTPS POST /handle

2. DGOM (Proxy)
   ├─ ❌ NÃO valida o token JWT recebido
   │  Arquivo: @dgom_pagtesouro/pgt.js:80-271
   │  app.post('/handle', ...) sem middleware de autenticação
   │
   ├─ Identifica categoria do serviço
   │  if (value.cat_servico == "PAPEM")
   │  Arquivo: pgt.js:122
   │
   ├─ Seleciona token apropriado
   │  token = tokenAcessoPAPEM  // Mesmo subject "773200"
   │  Arquivo: pgt.js:175
   │
   ├─ Gera número de referência (sequencial)
   │  Query: SELECT nextval('seqgru')
   │  Arquivo: pgt.js:142-147
   │
   └─ Repassa para STN (mesmo token)
      Header: Authorization: Bearer <token>
      Arquivo: pgt.js:179-180

                    ↓ HTTPS via proxy-1dn.mb:6060

3. STN (Validador Final)
   ├─ ✅ VALIDA o token JWT
   │  - Verifica assinatura RS256 com chave pública
   │  - Valida subject "773200" (autoriza operações da PAPEM)
   │  - Verifica expiração e outras claims
   │
   ├─ Processa solicitação de pagamento
   │  URL: https://pagtesouro.tesouro.gov.br/api/gru/solicitacao-pagamento
   │
   └─ Retorna resposta (GRU ou erro)

                    ↓ Response flow

4. DGOM (Persistência)
   ├─ Recebe resposta da STN
   ├─ Armazena dados criptografados (AES-128-CBC)
   │  Arquivo: pgt.js:253-266
   └─ Repassa resposta para PAPEM (sem modificação)

                    ↓ JSON Response

5. PAPEM (Destino)
   └─ Recebe GRU ou mensagem de erro
```

---

## 🏗️ Arquitetura de Autenticação

### Papéis dos Componentes

| Componente | Papel | Validação JWT | Função Principal |
|------------|-------|---------------|------------------|
| **PAPEM** | Cliente | ❌ Não | Envia requests com token |
| **DGOM** | Proxy | ❌ Não | Roteia e repassa token |
| **STN** | Servidor | ✅ Sim | Valida e autoriza operações |

### Por que DGOM não valida?

**Razões arquiteturais**:
1. **Chave Pública**: STN possui a chave pública para validar RS256
2. **Zona de Confiança**: PAPEM e DGOM estão na mesma rede interna (MB)
3. **Proxy Corporativo**: DGOM é intermediário técnico, não de segurança
4. **Validação Centralizada**: STN mantém controle total sobre autorização

---

## 🔍 Detalhes Técnicos

### 1. PAPEM → DGOM

**Configuração HTTP**:
```php
// @PAGTESOURO_PAPEM/PagTesouroClasse.php:137
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    'Content-Type: application/json',
    'Authorization: Bearer '.$chave
));
```

**Endpoint**:
- **Produção**: `https://pagtesouro.dgom.mb:3000/handle`
- **Homologação**: `https://dpagtesourohmg.mb:3000/handle`

**SSL/TLS**:
- Protocolo: TLS 1.2+ (HTTPS)
- Certificado: RECIM (Rede de Comunicações Integradas da Marinha)
- Validação: One-way (PAPEM valida DGOM, mas não o inverso)

### 2. DGOM - Processamento

**Recepção sem validação**:
```javascript
// @dgom_pagtesouro/pgt.js:80
app.post('/handle', cors(corsOptions), (request,response) => {
    // ❌ SEM MIDDLEWARE DE AUTENTICAÇÃO
    geralog(" Dados para GRU Recebidos!");
    console.log(request.body);
    var value = request.body;
    // ... continua processamento
});
```

**Seleção de Token por Categoria**:
```javascript
// @dgom_pagtesouro/pgt.js:172-175
var token = tokenAcesso;                              // Default
if (value.cat_servico == "CCCPM") token = tokenAcessoCCCPM;
if (value.cat_servico == "CCCPM2") token = tokenAcessoCCCPM2;
if (value.cat_servico == "PAPEM") token = tokenAcessoPAPEM;  // UG 773200
```

**Identificação da PAPEM**:
```javascript
// @dgom_pagtesouro/pgt.js:122
value.nomeUG = "773200";  // Código da PAPEM
```

### 3. DGOM → STN

**Configuração da Requisição**:
```javascript
// @dgom_pagtesouro/pgt.js:178-186
const options = {
    headers: {
        'Authorization': 'Bearer ' + token,  // Token da PAPEM (773200)
        'Proxy-Autorization': aut
    },
    proxy: {
        host: 'proxy-1dn.mb',  // Proxy corporativo da Marinha
        port: 6060
    }
};
```

**Endpoint STN**:
```
https://pagtesouro.tesouro.gov.br/api/gru/solicitacao-pagamento
```

**Proxy Corporativo**:
- Host: `proxy-1dn.mb`
- Porta: 6060
- Função: Gateway da Marinha para internet

---

## 🔐 Análise de Segurança

### Modelo de Confiança

```
┌───────────────────────────────────────────────────────────┐
│              ZONA DE CONFIANÇA (Intranet MB)              │
│                                                           │
│  PAPEM ←──────────────────→ DGOM                         │
│  (773200)    HTTPS          (proxy)                      │
│                                                           │
│  Confiança: Implícita (mesma rede corporativa)           │
│  Segurança: Firewall interno + HTTPS                     │
└───────────────────────────────────────────────────────────┘
                         │
                         │ proxy-1dn.mb:6060
                         │
┌───────────────────────────────────────────────────────────┐
│         ZONA NÃO CONFIÁVEL (Internet Pública)             │
│                                                           │
│  STN (PagTesouro Nacional)                               │
│  ✅ Valida JWT RS256                                      │
│  ✅ Verifica subject "773200"                             │
│  ✅ Autoriza operações                                    │
│                                                           │
│  Confiança: Zero Trust - valida TUDO                     │
└───────────────────────────────────────────────────────────┘
```

### Pontos Fortes

✅ **Criptografia End-to-End**: HTTPS em todas as conexões
✅ **Token Centralizado**: Mesmo token (773200) em toda cadeia
✅ **Validação Forte**: RS256 (assinatura RSA) na STN
✅ **Certificados RECIM**: Infraestrutura PKI da Marinha
✅ **Persistência Criptografada**: AES-128-CBC no banco DGOM
✅ **Proxy Corporativo**: Controle de saída para internet

### Vulnerabilidades Identificadas

❌ **DGOM sem autenticação**: Aceita qualquer request na porta 3000
```javascript
// Endpoint desprotegido
app.post('/handle', cors(corsOptions), (request,response) => {
    // Sem validação de token
    // Sem autenticação de cliente
    // Sem rate limiting
});
```

❌ **Token hardcoded**: JWT exposto no código-fonte
```php
// @PAGTESOURO_PAPEM/PagTesouroClasse.php:6
$chave="eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI3NzMyMDAifQ...";
```

❌ **SSL verification desabilitada**:
```php
// @PAGTESOURO_PAPEM/PagTesouroClasse.php:132-133
//curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 1);  // COMENTADO
//curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, 1);  // COMENTADO
```

❌ **Sem mTLS**: Não há autenticação mútua de certificados
- DGOM não solicita certificado de cliente da PAPEM
- Qualquer cliente que alcance a porta 3000 pode enviar requests

❌ **CORS limitado**: Apenas 127.0.0.1, mas sem autenticação adicional
```javascript
// @dgom_pagtesouro/pgt.js:68-72
var corsOptions = {
    origin: ['http://127.0.0.1:3000'],
    optionsSuccessStatus: 200
};
```

### Mitigações Existentes

🛡️ **Firewall Interno**: DGOM provavelmente está atrás de firewall
🛡️ **Rede Isolada**: Intranet da Marinha (não acessível da internet)
🛡️ **Validação Final**: STN valida tudo antes de processar
🛡️ **Logging**: Todas operações são registradas

---

## 📊 Comparação: Cenários Possíveis vs. Realidade

| Cenário | Descrição | Status |
|---------|-----------|--------|
| **A - JWT Only** | PAPEM envia JWT, DGOM repassa para STN | ✅ **REAL** |
| **B - mTLS** | Certificados SSL mútuos PAPEM ↔ DGOM | ❌ Não implementado |
| **C - JWT + mTLS** | Ambos mecanismos combinados | ❌ Não implementado |
| **D - API Key** | Chave estática simples | ❌ Não usado |

---

## 📁 Referências de Código

### PAPEM

**Arquivo Principal**: `@PAGTESOURO_PAPEM/PagTesouroClasse.php`

| Linha | Descrição |
|-------|-----------|
| 6, 170, 361, 536, 711 | Definição do token JWT |
| 137, 329, 506, 681, 858 | Configuração do header `Authorization: Bearer` |
| 10, 180, 372, 547, 722 | URLs de produção (pagtesouro.dgom.mb:3000) |
| 175, 367, 542, 717 | URLs de homologação (dpagtesourohmg.mb:3000) |
| 134, 326, 503, 678, 855 | Certificado SSL (RECIM-chain.pem) |
| 132-133, 324-325 | SSL verification (comentado) |

**Certificado**: `@PAGTESOURO_PAPEM/MarinhadoBrasilAutoridadeCertificadoradaRECIM-chain.pem`
- Emissor: Marinha do Brasil - Autoridade Certificadora RECIM
- Validade: 2023-02-10 até 2033-02-07

### DGOM

**Arquivo Principal**: `@dgom_pagtesouro/pgt.js`

| Linha | Descrição |
|-------|-----------|
| 75-78 | Declaração de tokens por categoria |
| 80-271 | Endpoint `/handle` (sem autenticação) |
| 122 | Identificação PAPEM (nomeUG = "773200") |
| 142-147 | Geração de número de referência (sequencial) |
| 172-175 | Seleção de token por `cat_servico` |
| 179-180 | Headers para STN (`Authorization: Bearer`) |
| 181-184 | Configuração do proxy corporativo |
| 253-266 | Persistência criptografada (AES-128-CBC) |
| 274-506 | Endpoint `/update` (sem autenticação) |

**Arquivo de Configuração**: `@dgom_pagtesouro/server.txt`
- Linha 104: Template `tokenAcessoPAPEM = " xxxxxxxx"`
- Linha 153: Identificação UG PAPEM "773200"

**Certificados DGOM**:
- `pagtesouro.key` - Chave privada do servidor
- `pagtesouro.pem` - Certificado do servidor
- `recim-chain.pem` - Cadeia RECIM para validar STN

---

## 🎯 Conclusão

### Arquitetura Real

**DGOM opera como proxy de autenticação**:
1. ✅ Recebe token JWT da PAPEM (subject "773200")
2. ❌ **NÃO valida** o token recebido
3. ✅ Identifica categoria de serviço (`cat_servico = "PAPEM"`)
4. ✅ Seleciona o token apropriado (`tokenAcessoPAPEM`)
5. ✅ Repassa o **mesmo token** para STN
6. ✅ STN valida e autoriza a operação
7. ✅ DGOM armazena dados criptografados
8. ✅ DGOM retorna resposta para PAPEM

### Princípio de Segurança

**"Trust but Verify at the Border"**
- **Confiança interna**: PAPEM ↔ DGOM (mesma rede)
- **Verificação na borda**: STN valida tudo (internet)

### Responsabilidades

| Sistema | Responsabilidade de Segurança |
|---------|-------------------------------|
| **PAPEM** | Proteger token JWT no código |
| **DGOM** | Logging, roteamento, persistência |
| **STN** | Validação, autorização, auditoria |

---

## 📞 Contato e Manutenção

**Sistemas Envolvidos**:
- **PAPEM**: Escola de Aprendizes-Marinheiros
- **DGOM**: Diretoria Geral de Orçamento da Marinha
- **STN**: Secretaria do Tesouro Nacional

**Documentação Original**:
- PagTesouro Nacional: https://pagtesouro.tesouro.gov.br/
- API Docs: https://pagtesouro.tesouro.gov.br/api/gru/

**Última Atualização**: 2025-12-03

---

## 🔄 Histórico de Revisões

| Data | Versão | Descrição |
|------|--------|-----------|
| 2025-12-03 | 1.0 | Documentação inicial do fluxo de autenticação |
