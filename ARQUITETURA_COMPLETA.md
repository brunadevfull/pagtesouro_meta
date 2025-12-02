# Arquitetura Completa: PAPEM → DGOM → PagTesouro Nacional (STN)

## Visão Geral do Sistema

O sistema implementa uma arquitetura de três camadas onde **toda requisição da PAPEM passa obrigatoriamente pela DGOM** antes de chegar ao PagTesouro Nacional (STN).

```
┌──────────────┐         ┌──────────────┐         ┌────────────────────┐
│   PAPEM      │  HTTPS  │    DGOM      │  HTTPS  │  PagTesouro        │
│  (Frontend)  ├────────→│  (Gateway)   ├────────→│  Nacional (STN)    │
│   PHP        │  JWT    │   Node.js    │  JWT    │   API REST         │
└──────────────┘         └──────────────┘         └────────────────────┘
```

---

## 1. ARQUITETURA DE COMUNICAÇÃO

### 1.1 PAPEM → DGOM

**Tecnologia:** PHP (cURL)

**Classe Principal:** `PagTesouroClasse.php`

#### Conexão e Endpoints

| Parâmetro | Valor |
|-----------|-------|
| **URL Produção** | `https://pagtesouro.dgom.mb:3000/handle` |
| **URL Homologação** | `https://dpagtesourohmg.mb:3000/handle` |
| **Método HTTP** | POST |
| **Content-Type** | application/json |
| **Porta** | 3000 (HTTPS) |

#### Autenticação

**Tipo:** Bearer Token (JWT)

**Token PAPEM:**
```
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI3NzMyMDAifQ.X92vQ2oBESAP...
```

**Payload do Token:**
- `alg`: RS256 (RSA-SHA256)
- `sub`: "773200" (Código UG PAPEM)

**Implementação em PHP:**
```php
curl_setopt($ch, CURLOPT_HTTPHEADER, array(
    'Content-Type: application/json',
    'Authorization: Bearer '.$chave
));
```

#### Certificado SSL

**Arquivo:** `MarinhadoBrasilAutoridadeCertificadoradaRECIM-chain.pem`

**Configuração:**
```php
curl_setopt($ch, CURLOPT_CAINFO, "MarinhadoBrasilAutoridadeCertificadoradaRECIM-chain.pem");
```

**Autoridade Certificadora:** Marinha do Brasil - RECIM

#### Métodos Disponíveis

A classe `PagTesouroClasse.php` oferece diferentes métodos para tipos específicos de servidores:

1. **`servidorCivil()`** - @PAGTESOURO_PAPEM/PagTesouroClasse.php:4
   - Servidores civis
   - Códigos de serviço: 11859 (padrão), 11860 (SISRES)

2. **`militarAtivo()`** - @PAGTESOURO_PAPEM/PagTesouroClasse.php:168
   - Militares ativos
   - Inclui NIP (Número de Identificação Pessoal)

3. **`sisresOutros()`** - @PAGTESOURO_PAPEM/PagTesouroClasse.php:360
   - Outros tipos de servidores SISRES

4. **`sisresSC()`** - @PAGTESOURO_PAPEM/PagTesouroClasse.php:535
   - Servidores civis SISRES

5. **`recAtivosB()`** - @PAGTESOURO_PAPEM/PagTesouroClasse.php:710
   - Receita de ativos
   - Inclui valores de exercícios anteriores e atuais

---

### 1.2 DGOM → PagTesouro Nacional

**Tecnologia:** Node.js (Express + Axios)

**Arquivo Principal:** `@dgom_pagtesouro/pgt.js`

#### URLs de Ambiente

| Ambiente | URL Base | Status |
|----------|----------|--------|
| **Homologação** | `https://valpagtesouro.tesouro.gov.br/api/gru/` | Disponível |
| **Produção** | `https://pagtesouro.tesouro.gov.br/api/gru/` | **Ativo** |

**Configuração Atual:** - @dgom_pagtesouro/pgt.js:72
```javascript
var ender = prd_ender; // Produção ativa
```

#### Endpoints da STN

1. **Criar Solicitação de Pagamento (GRU)**
   ```
   POST /api/gru/solicitacao-pagamento
   ```

2. **Consultar Status de Pagamento**
   ```
   GET /api/gru/pagamentos/{idPagamento}
   ```

#### Autenticação por Serviço

A DGOM utiliza diferentes tokens JWT para cada categoria de serviço:

| Categoria | Token | Linha |
|-----------|-------|-------|
| **PAPEM** | `tokenAcessoPAPEM` | pgt.js:78 |
| **CCCPM** | `tokenAcessoCCCPM` | pgt.js:76 |
| **CCCPM2** | `tokenAcessoCCCPM2` | pgt.js:77 |
| **Padrão** | `tokenAcesso` | pgt.js:75 |

**Seleção do Token:** - @dgom_pagtesouro/pgt.js:175
```javascript
var token = tokenAcesso;
if (value.cat_servico == "CCCPM") token = tokenAcessoCCCPM;
if (value.cat_servico == "CCCPM2") token = tokenAcessoCCCPM2;
if (value.cat_servico == "PAPEM") token = tokenAcessoPAPEM;
```

#### Configuração de Proxy

**Proxy Corporativo:** - @dgom_pagtesouro/pgt.js:182-184
```javascript
proxy: {
    host: 'proxy-1dn.mb',
    port: 6060
}
```

**Autenticação do Proxy:**
```javascript
headers: {
    'Proxy-Autorization': aut
}
```

---

## 2. FLUXO DE DADOS BIDIRECIONAL

### 2.1 Fluxo de Criação de GRU (/handle)

#### Passo 1: PAPEM → DGOM

**Payload Enviado:**
```json
{
  "cat": "PAPEM",
  "codigoServico": 11859,
  "vencimento": "09122025",
  "competencia": "102025",
  "nomeContribuinte": "Nome do Servidor",
  "cnpjCpf": "12345678901",
  "valorPrincipal": 1500.00,
  "valorDescontos": 0,
  "valorOutrasDeducoes": 0,
  "valorMulta": 0,
  "valorJuros": 0,
  "valorOutrosAcrescimos": 0,
  "nomeUG": "SIGLA_OM",
  "cod_om": "008",
  "cat_servico": "PAPEM",
  "codRubrica": "00176",
  "nomeRubrica": "GRATIFICAÇÃO NATALINA",
  "motivo": "Motivo do recolhimento",
  "tributavel": 1,
  "nomeOC": "SIGLA_OC",
  "cod_oc": "015",
  "valorBrutoExercAnt": 1200.00,
  "codSiapeNip": "1234567",
  "NatDev": "GRATIFICAÇÃO NATALINA"
}
```

#### Passo 2: DGOM - Transformações

**2.1 Consulta ao Banco de Dados** - @dgom_pagtesouro/pgt.js:143-149

Busca o sequencial da referência:
```sql
SELECT COALESCE(MAX(CD_REF_SEQ), 0) AS seq
FROM pagtesouro.tb_pgto
WHERE cd_cpf = $1 AND cat_servico = 'PAPEM'
```

**2.2 Montagem do Código de Referência** - @dgom_pagtesouro/pgt.js:94-108

```javascript
function montaref(result) {
    // Sequencial com 4 dígitos
    cd_referencia = result.toString();
    while (cd_referencia.length < 4)
        cd_referencia = "0" + cd_referencia;

    // Remove primeiros 3 dígitos do CNPJ
    var cnpjCpf_edit = value.cnpjCpf;
    if (cnpjCpf_edit.length == 14)
        cnpjCpf_edit = cnpjCpf_edit.substring(3);

    // Monta referência: 73200 + CPF/CNPJ + Sequencial
    if (value.cat == "PAPEM")
        value.referencia = "73200" + cnpjCpf_edit + cd_referencia;
    else
        value.referencia = value.nomeUG + cnpjCpf_edit + cd_referencia;
}
```

**Exemplo:**
- CPF: `12345678901`
- Sequencial: `0001`
- Referência: `732001234567890100001`

**2.3 Ajuste de Campos Específicos PAPEM** - @dgom_pagtesouro/pgt.js:116-124

```javascript
if (value.cat == "PAPEM") {
    value.cat_servico = "PAPEM";
    value.nomeUG = "773200";  // UG PAPEM
    value.id_servico = "52";  // ID interno DGOM
}
```

**2.4 Criptografia de Dados Sensíveis** - @dgom_pagtesouro/pgt.js:207-218

```javascript
// AES-128-CBC
var crypto = require('crypto');
var cipher = crypto.createCipheriv('aes-128-cbc', key, iv);

// Nome criptografado
var nome_encrypted = cipher.update(value.nomeContribuinte, 'utf8', 'base64');
nome_encrypted += cipher.final('base64');

// CPF criptografado
cipher = crypto.createCipheriv('aes-128-cbc', key, iv);
var cnpjCpf_encrypted = cipher.update(value.cnpjCpf, 'utf8', 'base64');
cnpjCpf_encrypted += cipher.final('base64');
```

#### Passo 3: DGOM → STN

**Payload Transformado:**
```json
{
  "codigoServico": 11859,
  "vencimento": "09122025",
  "competencia": "102025",
  "nomeContribuinte": "Nome do Servidor",
  "cnpjCpf": "12345678901",
  "valorPrincipal": 1500.00,
  "valorDescontos": 0,
  "valorOutrasDeducoes": 0,
  "valorMulta": 0,
  "valorJuros": 0,
  "valorOutrosAcrescimos": 0,
  "referencia": "732001234567890100001",
  "nomeUG": "773200"
}
```

**Headers:**
```javascript
headers: {
    'Authorization': 'Bearer ' + tokenAcessoPAPEM,
    'Proxy-Autorization': aut
}
```

#### Passo 4: STN → DGOM (Resposta)

**Resposta de Sucesso:**
```json
{
  "idPagamento": "202512091234567890",
  "dataCriacao": "2025-12-09T14:30:00Z",
  "situacao": {
    "codigo": "CRIADO",
    "data": "2025-12-09T14:30:00Z"
  },
  "proximaUrl": "https://pagtesouro.tesouro.gov.br/gru/visualizar?numero=202512091234567890",
  "codigoBarras": "00000000000000000000000000000000000000000000"
}
```

**Resposta de Erro:**
```json
[
  {
    "codigo": "C0005",
    "descricao": "Serviço inexistente"
  },
  {
    "codigo": "C0010",
    "descricao": "Parâmetros incompatíveis"
  }
]
```

#### Passo 5: DGOM - Persistência no Banco

**Registro no PostgreSQL** - @dgom_pagtesouro/pgt.js:224-237

```sql
INSERT INTO pagtesouro.tb_pgto (
    id_pgto, id_servico, dt_criacao, ds_situacao, dt_situacao,
    cd_servico, cd_om, dt_vencimento, dt_competencia,
    nome, cd_cpf, vr_descontos, vr_deducoes, vr_multa,
    vr_juros, vr_oacresc, vr_principal, cd_referencia,
    cd_ref_seq, cat_servico, cod_rubrica, nome_rubrica,
    motivo, tributavel, nomeoc, vr_bruto_ex_ant,
    vr_ex_atu, natdev, cod_siapenip, cod_oc, cod_om
) VALUES (...)
```

**Campos Criptografados:**
- `nome`: AES-128-CBC (base64)
- `cd_cpf`: AES-128-CBC (base64)

#### Passo 6: DGOM → PAPEM (Resposta Final)

A DGOM repassa a resposta da STN para a PAPEM sem modificações:
```javascript
response.send(resposta);
```

**Tratamento de Erro na PAPEM:** - @PAGTESOURO_PAPEM/PagTesouroClasse.php:148-162
```php
if (is_array($result)) {
    // Erro: exibe mensagens
    foreach ($result as $erro) {
        echo 'ERRO: ' . $erro->codigo . " - " . $erro->descricao;
    }
} else {
    // Sucesso: abre URL de pagamento
    echo '<script>window.open("' . $result->proximaUrl . '", "_blank");</script>';
}
```

---

### 2.2 Fluxo de Consulta de Status (/update)

#### Passo 1: PAPEM → DGOM

**Payload:**
```json
{
  "id_pgto": "202512091234567890",
  "cat_servico": "PAPEM"
}
```

#### Passo 2: DGOM → STN

**Requisição:**
```
GET /api/gru/pagamentos/202512091234567890
Authorization: Bearer {tokenAcessoPAPEM}
```

#### Passo 3: STN → DGOM (Resposta)

**Status: CONCLUIDO**
```json
{
  "idPagamento": "202512091234567890",
  "valor": 1500.00,
  "situacao": {
    "codigo": "CONCLUIDO",
    "data": "2025-12-09T15:30:00Z"
  },
  "tipoPagamentoEscolhido": "PIX",
  "nomePSP": "Banco do Brasil",
  "transacaoPSP": "BB20251209153000"
}
```

**Outros Status Possíveis:**
- `CRIADO`: GRU criada, aguardando pagamento
- `EM_PROCESSAMENTO`: Pagamento em processamento
- `CONCLUIDO`: Pagamento confirmado
- `CANCELADO`: GRU cancelada
- `EXPIRADO`: GRU vencida

#### Passo 4: DGOM - Atualização no Banco

**Update Principal:** - @dgom_pagtesouro/pgt.js:325-326
```sql
UPDATE pagtesouro.tb_pgto
SET ds_tp_pgto = $1,
    vr_pago = $2,
    ds_nomepsp = $3,
    cd_transacpsp = $4,
    ds_situacao = $5,
    dt_situacao = TO_TIMESTAMP($6, 'YYYY-MM-DD"T"HH24:MI:SSZ')
WHERE id_pgto = $7
```

#### Passo 5: Integração com SINGRA (Apenas CCIM)

**Condição:** - @dgom_pagtesouro/pgt.js:341
```javascript
if (request.body.cat_servico == "CCIM") {
    // Envia dados para SINGRA
}
```

**Payload SINGRA:**
```json
{
  "id_pgto": "202512091234567890",
  "cpf": "12345678901",
  "situacao": "CONCLUIDO",
  "tp_pgto": "PIX",
  "dt_pgto": "2025-12-09T15:30:00Z",
  "nome_PSP": "Banco do Brasil",
  "id_transac": "202512091234567890",
  "vr_pago": 1500.00
}
```

**Endpoint SINGRA:**
```
POST https://www.api-singra.dabm.mb/pagtesouro/pagamento
Authorization: Basic {base64(admin:pwssingra)}
```

**Update SINGRA_OK:** - @dgom_pagtesouro/pgt.js:417-418
```sql
UPDATE pagtesouro.tb_pgto
SET singra_ok = 1
WHERE id_pgto = $1
```

#### Passo 6: DGOM → PAPEM

**Resposta CCIM (com SINGRA):**
```json
["1 ok"]
// ou
["1 fail", "erro_cod: descrição"]
```

**Resposta Outros Serviços:**
```json
["1"]
// ou
"0"
```

---

## 3. IMPLEMENTAÇÃO TÉCNICA

### 3.1 Autenticação JWT

#### Estrutura do Token

**Algoritmo:** RS256 (RSA-SHA256)

**Formato:**
```
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI3NzMyMDAifQ.{signature}
```

**Header (decodificado):**
```json
{
  "alg": "RS256"
}
```

**Payload (decodificado):**
```json
{
  "sub": "773200"
}
```

**Subject Codes:**
- `773200`: PAPEM
- Outros códigos para CCCPM, CCCPM2, etc.

#### Validação no STN

O PagTesouro Nacional valida:
1. Assinatura RSA do token
2. Subject (código da UG)
3. Permissões do código de serviço
4. Compatibilidade UG × Serviço

---

### 3.2 Certificados SSL/TLS

#### DGOM (Servidor)

**Arquivos:** - @dgom_pagtesouro/pgt.js:52-54
```javascript
var options = {
    key: fs.readFileSync('pagtesouro.key'),   // Chave privada
    cert: fs.readFileSync('pagtesouro.pem')   // Certificado
};
```

**Servidor HTTPS:** - @dgom_pagtesouro/pgt.js:508-509
```javascript
var server = https.createServer(options, app).listen(3000);
```

#### PAPEM (Cliente)

**Certificado RECIM:** - @PAGTESOURO_PAPEM/PagTesouroClasse.php:134
```php
curl_setopt($ch, CURLOPT_CAINFO,
    "MarinhadoBrasilAutoridadeCertificadoradaRECIM-chain.pem");
```

#### Cadeia de Certificados

**DGOM → STN:**
```javascript
https.globalAgent.options.ca = rootCas +
    fs.readFileSync('/var/www/html/pagtesouro/recim-chain.pem');
```

---

### 3.3 Tratamento de Erros

#### Códigos de Erro do PagTesouro Nacional

| Código | Descrição | Causa Provável |
|--------|-----------|----------------|
| **C0005** | Serviço inexistente | Código de serviço inválido |
| **C0010** | Parâmetros incompatíveis | Formato de dados incorreto |
| **C0015** | Token de acesso inválido | JWT expirado ou inválido |
| **C0020** | UG do token ≠ UG do serviço | Incompatibilidade de permissões |

#### Tratamento na DGOM - @dgom_pagtesouro/pgt.js:241-263

```javascript
try {
    const response2 = await axios.post(ender + 'solicitacao-pagamento', value, options);
    // Sucesso: processa resposta
} catch (error) {
    var erro = new Object();

    if (typeof(error.response) !== 'undefined') {
        // Erro da API STN
        for (const [key, value] of Object.entries(error.response.data)) {
            geralog("Erro: " + value["codigo"] + " - " + value["descricao"]);
            erro[value["codigo"]] = value["descricao"];
        }
        erro["situacao"] = { codigo: 'CORRIGIR' };
        response.send(erro);
    } else {
        // Erro de rede/timeout
        erro["situacao"] = { codigo: 'ERRO' };
        response.send(erro);
    }
}
```

#### Tratamento na PAPEM - @PAGTESOURO_PAPEM/PagTesouroClasse.php:141-163

```php
if ($result === false) {
    echo 'Erro: ' . curl_error($ch);
} else {
    $result = json_decode($result);

    if (is_array($result)) {
        // Array = Erro
        foreach ($result as $erro) {
            echo 'ERRO: ' . $erro->codigo . " - " . $erro->descricao;
        }
    } else {
        // Object = Sucesso
        echo '<script>window.open("' . $result->proximaUrl . '", "_blank");</script>';
    }
}
```

---

### 3.4 Logging e Monitoramento

#### Sistema de Logs da DGOM - @dgom_pagtesouro/pgt.js:61-63

```javascript
function geralog(texto) {
    console.log(new Date().toLocaleString() + " - " + texto);
}
```

#### Pontos de Log Principais

**1. Recebimento de Dados:**
```javascript
geralog(" Dados para GRU Recebidos!");
geralog("======= INÍCIO DOS DADOS ORIGINAIS ========");
console.log(request.body);
geralog("======= FIM DOS DADOS ORIGINAIS ========");
```

**2. Envio para STN:**
```javascript
geralog(" Emitindo POST-REQUEST para " + ender + 'solicitacao-pagamento');
```

**3. Resposta da STN:**
```javascript
geralog("Resposta PagTesouro recebida!");
geralog("======= INÍCIO DA RESPOSTA ========");
console.log(response2.data);
geralog("======= FIM DA RESPOSTA ========");
```

**4. Sucesso:**
```javascript
geralog("Sucesso na criação da GRU. ID Pagamento: " + resposta["idPagamento"]);
geralog("Registro no Banco de Dados efetuado com sucesso!");
```

**5. Erro:**
```javascript
geralog("Acesso: " + cont + " - Resposta do REQUEST com Erro!");
geralog("Erro: " + value["codigo"] + " - " + value["descricao"]);
```

#### Contador de Requisições

```javascript
var cont = 0;
cont = cont + 1;  // Incrementado a cada requisição
```

---

### 3.5 Segurança Implementada

#### Headers de Segurança - @dgom_pagtesouro/pgt.js:17-36

**1. HSTS (HTTP Strict Transport Security):**
```javascript
res.setHeader('Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload');
```

**2. Cache Control:**
```javascript
res.setHeader('Cache-Control', 'no-cache, no-store');
res.setHeader('Pragma', 'no-cache');
```

**3. X-Frame-Options:**
```javascript
res.setHeader('X-Frame-Options', 'SAMEORIGIN');
```

**4. CORS:**
```javascript
res.setHeader('Access-Control-Allow-Origin', '127.0.0.1');
res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT');
res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
res.setHeader('Access-Control-Allow-Credentials', 'true');
```

#### Criptografia de Dados

**Algoritmo:** AES-128-CBC

**Campos Criptografados:**
- Nome do contribuinte
- CPF/CNPJ

**Chave e IV:**
```javascript
key = Buffer.from("#####", 'utf8');   // 16 bytes
iv = Buffer.from('#####', 'utf8');    // 16 bytes
```

---

## 4. DIAGRAMA DE SEQUÊNCIA

### 4.1 Criação de GRU

```mermaid
sequenceDiagram
    participant PAPEM as PAPEM<br/>(PHP Frontend)
    participant DGOM as DGOM<br/>(Node.js Gateway)
    participant DB as PostgreSQL<br/>(Banco de Dados)
    participant STN as PagTesouro<br/>(STN API)

    Note over PAPEM: Usuário solicita<br/>criação de GRU

    PAPEM->>DGOM: POST /handle<br/>{dados do contribuinte}<br/>Authorization: Bearer {JWT}

    Note over DGOM: Recebe requisição<br/>da PAPEM

    DGOM->>DB: SELECT MAX(CD_REF_SEQ)<br/>WHERE cpf = ? AND cat_servico = 'PAPEM'
    DB-->>DGOM: seq = 1234

    Note over DGOM: Monta código de referência:<br/>73200 + CPF + seq<br/>(ex: 732001234567890101234)

    Note over DGOM: Ajusta campos PAPEM:<br/>nomeUG = "773200"<br/>id_servico = "52"

    Note over DGOM: Criptografa:<br/>- nome (AES-128-CBC)<br/>- cpf (AES-128-CBC)

    DGOM->>STN: POST /api/gru/solicitacao-pagamento<br/>{payload transformado}<br/>Authorization: Bearer {tokenPAPEM}<br/>Proxy: proxy-1dn.mb:6060

    alt Sucesso (situacao.codigo = "CRIADO")
        STN-->>DGOM: 200 OK<br/>{idPagamento, proximaUrl, situacao}

        DGOM->>DB: INSERT INTO tb_pgto<br/>(id_pgto, nome_encrypted,<br/>cpf_encrypted, valores, ...)
        DB-->>DGOM: OK

        DGOM-->>PAPEM: 200 OK<br/>{resposta da STN}

        Note over PAPEM: Abre proximaUrl<br/>em nova janela

    else Erro (array de erros)
        STN-->>DGOM: 400/500<br/>[{codigo, descricao}, ...]

        Note over DGOM: Processa erros:<br/>erro.situacao = "CORRIGIR"

        DGOM-->>PAPEM: 400<br/>{erros da STN}

        Note over PAPEM: Exibe mensagens<br/>de erro ao usuário
    end
```

### 4.2 Consulta de Status

```mermaid
sequenceDiagram
    participant PAPEM as PAPEM<br/>(PHP Frontend)
    participant DGOM as DGOM<br/>(Node.js Gateway)
    participant DB as PostgreSQL<br/>(Banco de Dados)
    participant STN as PagTesouro<br/>(STN API)
    participant SINGRA as SINGRA<br/>(Sistema Interno)

    Note over PAPEM: Cron ou usuário<br/>solicita atualização

    PAPEM->>DGOM: POST /update<br/>{id_pgto, cat_servico}<br/>Authorization: Bearer {JWT}

    Note over DGOM: Seleciona token<br/>baseado em cat_servico

    DGOM->>STN: GET /api/gru/pagamentos/{id_pgto}<br/>Authorization: Bearer {token}<br/>Proxy: proxy-1dn.mb:6060

    alt Status = CONCLUIDO
        STN-->>DGOM: 200 OK<br/>{valor, situacao, tipoPagamento,<br/>nomePSP, transacaoPSP}

        DGOM->>DB: UPDATE tb_pgto<br/>SET ds_tp_pgto, vr_pago,<br/>ds_nomepsp, cd_transacpsp,<br/>ds_situacao, dt_situacao<br/>WHERE id_pgto = ?
        DB-->>DGOM: OK

        alt cat_servico = "CCIM"
            Note over DGOM: Integração com SINGRA

            DGOM->>SINGRA: POST /pagtesouro/pagamento<br/>{id_pgto, cpf, situacao,<br/>tp_pgto, vr_pago}<br/>Authorization: Basic {admin}

            alt SINGRA Sucesso
                SINGRA-->>DGOM: sucesso: true

                DGOM->>DB: UPDATE tb_pgto<br/>SET singra_ok = 1<br/>WHERE id_pgto = ?
                DB-->>DGOM: OK

                DGOM-->>PAPEM: ["1 ok"]

            else SINGRA Erro
                SINGRA-->>DGOM: sucesso: false<br/>erro_Msg: "..."

                DGOM-->>PAPEM: ["1 fail", "erro_cod: descrição"]
            end

        else Outros Serviços
            DGOM-->>PAPEM: ["1"]
        end

    else Status != CONCLUIDO
        STN-->>DGOM: 200 OK<br/>{situacao diferente de CONCLUIDO}

        DGOM->>DB: UPDATE tb_pgto<br/>SET ds_situacao, dt_situacao<br/>WHERE id_pgto = ?
        DB-->>DGOM: OK

        DGOM-->>PAPEM: "1"
    end

    Note over PAPEM: Atualiza interface<br/>com novo status
```

---

## 5. PAPEL DE CADA SISTEMA NA ARQUITETURA

### 5.1 PAPEM (Frontend/Cliente)

**Responsabilidades:**

1. **Interface com o Usuário**
   - Coleta dados do contribuinte
   - Validação de formulários (CPF, valores, datas)
   - Exibição de mensagens de erro/sucesso

2. **Comunicação com DGOM**
   - Prepara payloads JSON
   - Envia requisições HTTPS via cURL
   - Processa respostas (sucesso/erro)

3. **Abertura de URLs de Pagamento**
   - Recebe `proximaUrl` da resposta
   - Abre em nova janela para pagamento

4. **Diferentes Perfis de Servidores**
   - Servidores civis
   - Militares ativos
   - SISRES (diversos tipos)
   - Receita de ativos

**Tecnologias:**
- PHP
- cURL para HTTPS
- JSON encode/decode
- Certificado RECIM para SSL

**Localização:** `@PAGTESOURO_PAPEM/`

**Arquivo Principal:** `PagTesouroClasse.php`

---

### 5.2 DGOM (Gateway Intermediário)

**Responsabilidades:**

1. **Gateway de API**
   - Recebe requisições da PAPEM
   - Encaminha para PagTesouro Nacional
   - Retorna respostas para PAPEM

2. **Gerenciamento de Tokens**
   - Mantém múltiplos tokens JWT (PAPEM, CCCPM, etc.)
   - Seleciona token correto baseado em `cat_servico`
   - Renova tokens quando necessário

3. **Transformações de Dados**
   - Monta código de referência único
   - Ajusta campos específicos (UG, código de serviço)
   - Normaliza dados entre PAPEM e STN

4. **Persistência em Banco de Dados**
   - Registra todas as GRUs criadas
   - Criptografa dados sensíveis (AES-128-CBC)
   - Mantém histórico de transações
   - Atualiza status de pagamentos

5. **Proxy Corporativo**
   - Encaminha requisições através do proxy MB
   - Gerencia autenticação do proxy

6. **Integração com Sistemas Internos**
   - SINGRA (para categoria CCIM)
   - Atualização de saldos

7. **Logging e Auditoria**
   - Registra todas as operações
   - Timestamps de requisições/respostas
   - Contador de acessos

8. **Segurança**
   - HTTPS obrigatório
   - Headers de segurança (HSTS, CORS, X-Frame-Options)
   - Cache control
   - Certificados SSL/TLS

**Tecnologias:**
- Node.js + Express
- Axios para requisições HTTP
- PostgreSQL para persistência
- Crypto para criptografia AES-128-CBC
- SSL/TLS

**Endpoints:**
- `POST /handle` - Criar GRU
- `POST /update` - Consultar status

**Localização:** `@dgom_pagtesouro/`

**Arquivo Principal:** `pgt.js`

---

### 5.3 PagTesouro Nacional (STN)

**Responsabilidades:**

1. **API Oficial do Governo Federal**
   - Processa solicitações de pagamento (GRU)
   - Gera códigos de barras
   - Integra com PSPs (Provedores de Serviço de Pagamento)

2. **Validações**
   - Valida tokens JWT (autenticação)
   - Verifica permissões (UG × Serviço)
   - Valida formato e valores dos dados

3. **Geração de URLs de Pagamento**
   - Retorna `proximaUrl` para interface de pagamento
   - Suporta múltiplos métodos (PIX, boleto, cartão)

4. **Processamento de Pagamentos**
   - Recebe confirmações dos PSPs
   - Atualiza status das GRUs
   - Disponibiliza API de consulta

5. **Retorno de Status**
   - `CRIADO`: GRU criada
   - `EM_PROCESSAMENTO`: Pagamento em andamento
   - `CONCLUIDO`: Pagamento confirmado
   - `CANCELADO`: GRU cancelada
   - `EXPIRADO`: GRU vencida

**Endpoints:**
- `POST /api/gru/solicitacao-pagamento` - Criar GRU
- `GET /api/gru/pagamentos/{id}` - Consultar status

**URLs:**
- **Homologação:** `https://valpagtesouro.tesouro.gov.br/api/gru/`
- **Produção:** `https://pagtesouro.tesouro.gov.br/api/gru/`

**Documentação Oficial:**
- SISGRU (Sistema de Gestão de GRU)
- Credenciamento via órgão federal

---

## 6. CONSIDERAÇÕES IMPORTANTES

### 6.1 Por Que a DGOM é Necessária?

**Razões Técnicas:**

1. **Múltiplos Tokens JWT**
   - Cada serviço requer token específico
   - DGOM centraliza o gerenciamento
   - Facilita renovação e manutenção

2. **Proxy Corporativo**
   - Acesso à STN requer proxy MB
   - DGOM gerencia a autenticação do proxy
   - Único ponto de configuração

3. **Transformações de Dados**
   - Código de referência único e sequencial
   - Ajustes específicos por categoria
   - Normalização de formatos

4. **Persistência e Auditoria**
   - Banco de dados centralizado
   - Histórico completo de transações
   - Criptografia de dados sensíveis

5. **Integração com Sistemas Internos**
   - SINGRA para CCIM
   - Outros sistemas futuros

6. **Isolamento de Complexidade**
   - PAPEM não precisa conhecer detalhes da STN
   - Facilita manutenção e atualizações
   - Reduz acoplamento

---

### 6.2 URLs de Homologação vs Produção

#### DGOM

| Ambiente | URL | Porta | Uso Atual |
|----------|-----|-------|-----------|
| **Produção** | `https://pagtesouro.dgom.mb:3000` | 3000 | ✅ Ativo |
| **Homologação** | `https://dpagtesourohmg.mb:3000` | 3000 | Disponível |

#### PagTesouro Nacional (STN)

| Ambiente | URL Base | Proxy | Uso Atual |
|----------|----------|-------|-----------|
| **Produção** | `https://pagtesouro.tesouro.gov.br/api/gru/` | proxy-1dn.mb:6060 | ✅ Ativo |
| **Homologação** | `https://valpagtesouro.tesouro.gov.br/api/gru/` | proxy-1dn.mb:6060 | Disponível |

**Configuração Atual:** - @dgom_pagtesouro/pgt.js:66-73
```javascript
var hmg_ender = 'https://valpagtesouro.tesouro.gov.br/api/gru/';
var prd_ender = 'https://pagtesouro.tesouro.gov.br/api/gru/';

var ender = prd_ender;  // ⚠️ PRODUÇÃO ATIVA
```

---

### 6.3 Banco de Dados PostgreSQL

#### Esquema Principal

**Tabela:** `pagtesouro.tb_pgto`

**Campos Principais:**

| Campo | Tipo | Descrição | Criptografado |
|-------|------|-----------|---------------|
| `id_pgto` | VARCHAR | ID da GRU (STN) | ❌ |
| `id_servico` | INT | ID interno DGOM | ❌ |
| `dt_criacao` | TIMESTAMP | Data de criação | ❌ |
| `ds_situacao` | VARCHAR | Status (CRIADO, CONCLUIDO, etc.) | ❌ |
| `cd_servico` | INT | Código do serviço | ❌ |
| `cd_om` | VARCHAR | Código da OM | ❌ |
| `dt_vencimento` | DATE | Data de vencimento | ❌ |
| `dt_competencia` | VARCHAR | Competência (MMAAAA) | ❌ |
| `nome` | TEXT | Nome do contribuinte | ✅ AES-128-CBC |
| `cd_cpf` | TEXT | CPF/CNPJ | ✅ AES-128-CBC |
| `vr_principal` | DECIMAL | Valor principal | ❌ |
| `vr_descontos` | DECIMAL | Descontos | ❌ |
| `vr_pago` | DECIMAL | Valor pago | ❌ |
| `cd_referencia` | VARCHAR | Código de referência completo | ❌ |
| `cd_ref_seq` | INT | Sequencial da referência | ❌ |
| `cat_servico` | VARCHAR | Categoria (PAPEM, CCCPM, etc.) | ❌ |
| `singra_ok` | INT | Flag integração SINGRA | ❌ |
| `ds_tp_pgto` | VARCHAR | Tipo de pagamento (PIX, etc.) | ❌ |
| `ds_nomepsp` | VARCHAR | Nome do PSP | ❌ |
| `cd_transacpsp` | VARCHAR | Transação do PSP | ❌ |

**Configuração de Conexão:** - @dgom_pagtesouro/pgt.js:129-136
```javascript
const { Pool } = require('pg')
const pool = new Pool({
    user: '#',
    host: '#',
    database: '#',
    schema: '#',
    password: '#',
    port: 5432
})
```

---

### 6.4 Códigos de Serviço

| Código | Descrição | Categoria |
|--------|-----------|-----------|
| **11859** | Serviço padrão PAPEM | PAPEM |
| **11860** | SISRES PAPEM | PAPEM |
| **1541** | Serviço homologação | Teste |
| **23** | Exemplo oficial (documentação STN) | Exemplo |

**Seleção Automática:** - @PAGTESOURO_PAPEM/PagTesouroClasse.php:11-15
```php
if ($natureza_despesa == "SISRES") {
    $codigoServico = 11860;
} else {
    $codigoServico = 11859;
}
```

---

## 7. FLUXOGRAMAS ADICIONAIS

### 7.1 Tratamento de Erro Completo

```
┌─────────────────────────────────────────────────────────────┐
│                        PAPEM                                │
│  Envia requisição POST /handle                              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│                        DGOM                                 │
│  1. Recebe requisição                                       │
│  2. Valida formato                                          │
│  3. Consulta sequencial no DB                               │
│  4. Monta código de referência                              │
│  5. Criptografa dados sensíveis                             │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
              ┌──────────┴──────────┐
              │  Envia para STN     │
              └──────────┬──────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                 │
        ↓                                 ↓
┌───────────────┐              ┌──────────────────┐
│   Sucesso     │              │      Erro        │
│ (situacao =   │              │  (array de       │
│  "CRIADO")    │              │   erros)         │
└───────┬───────┘              └────────┬─────────┘
        │                               │
        ↓                               ↓
┌───────────────┐              ┌──────────────────┐
│ DGOM:         │              │ DGOM:            │
│ - INSERT DB   │              │ - Processa erros │
│ - Responde    │              │ - situacao:      │
│   PAPEM       │              │   "CORRIGIR"     │
└───────┬───────┘              └────────┬─────────┘
        │                               │
        ↓                               ↓
┌───────────────┐              ┌──────────────────┐
│ PAPEM:        │              │ PAPEM:           │
│ - Abre        │              │ - Exibe erros    │
│   proximaUrl  │              │ - Permite        │
│               │              │   correção       │
└───────────────┘              └──────────────────┘
```

---

## 8. PONTOS DE ATENÇÃO PARA DESENVOLVIMENTO

### 8.1 Segurança

✅ **Implementado:**
- HTTPS obrigatório em todas as camadas
- Tokens JWT (RS256)
- Certificados SSL/TLS (RECIM)
- Criptografia AES-128-CBC para dados sensíveis
- Headers de segurança (HSTS, CORS, X-Frame-Options)
- Cache control

⚠️ **Melhorias Possíveis:**
- Rotação automática de tokens
- Rate limiting na DGOM
- Validação de origem das requisições
- Logs estruturados (JSON) para SIEM

---

### 8.2 Performance

✅ **Implementado:**
- Conexões HTTPS persistentes
- Pool de conexões PostgreSQL
- Async/await para operações I/O

⚠️ **Melhorias Possíveis:**
- Cache de consultas frequentes (Redis)
- Fila de processamento para alta carga
- Balanceamento de carga (múltiplas instâncias DGOM)
- Compressão de respostas (gzip)

---

### 8.3 Monitoramento

✅ **Implementado:**
- Logging de todas as operações
- Timestamps de requisições
- Contador de acessos

⚠️ **Melhorias Possíveis:**
- Dashboard de monitoramento
- Alertas automáticos (Slack, email)
- Métricas de performance (Prometheus + Grafana)
- APM (Application Performance Monitoring)
- Health checks

---

### 8.4 Resiliência

⚠️ **Melhorias Necessárias:**
- Retry automático com backoff exponencial
- Circuit breaker para falhas da STN
- Fallback strategies
- Timeout configurável por endpoint
- Dead letter queue para requisições falhadas

---

### 8.5 Manutenibilidade

✅ **Implementado:**
- Separação de responsabilidades (PAPEM, DGOM, STN)
- Código comentado
- Configuração centralizada

⚠️ **Melhorias Possíveis:**
- Variáveis de ambiente para configurações
- Documentação de API (Swagger/OpenAPI)
- Testes automatizados (unit, integration, e2e)
- CI/CD pipeline
- Versionamento de API

---

## 9. CONCLUSÃO

A arquitetura **PAPEM → DGOM → PagTesouro Nacional** implementa um sistema robusto de três camadas onde:

1. **PAPEM** atua como frontend, coletando dados e apresentando interfaces aos usuários
2. **DGOM** serve como gateway inteligente, gerenciando tokens, transformando dados, persistindo informações e integrando com sistemas internos
3. **PagTesouro Nacional (STN)** processa os pagamentos e retorna os status

**A DGOM é essencial** para:
- Gerenciar múltiplos tokens JWT
- Encaminhar requisições através do proxy corporativo
- Manter histórico de transações com dados criptografados
- Integrar com sistemas internos (SINGRA)
- Isolar a complexidade da STN

**O fluxo bidirecional** garante:
- Criação de GRUs com código de referência único
- Consulta periódica de status
- Atualização automática de pagamentos concluídos
- Tratamento adequado de erros em todas as camadas

**A implementação técnica** utiliza:
- JWT (RS256) para autenticação
- SSL/TLS com certificados RECIM
- AES-128-CBC para criptografia
- PostgreSQL para persistência
- Logging extensivo para auditoria

Este documento serve como referência completa para manutenção, desenvolvimento e evolução do sistema.

---

**Última Atualização:** 2025-12-02
**Versão:** 1.0
**Repositório:** pagtesouro_meta
