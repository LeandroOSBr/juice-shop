# 📘 Relatório Técnico Detalhado de Vulnerabilidades (OWASP ZAP)

> 💡 **Visualização Interativa:** Para navegar visualmente pelos relatórios completos do OWASP ZAP com abas e dashboard comparativo, baixe o arquivo **[Juice-shop Relatorio Unificado.html](https://github.com/LeandroOSBr/juice-shop/blob/master/RELATORIO/Juice-shop%20Relatorio%20Unificado.html)** e abra-o localmente no seu navegador.

Este documento fornece uma análise aprofundada de cada uma das **16 categorias de vulnerabilidades e más configurações** identificadas pelo **OWASP ZAP 2.16.1** durante a varredura dinâmica no **OWASP Juice Shop** sem proteção de WAF, contrastando-as com o comportamento quando protegido pelo **AWS WAF**.

---

## 📑 Índice de Vulnerabilidades

1. [🔴 Risco Alto: Injeção SQL (CWE-89)](#1-injeção-sql-cwe-89)
2. [🟠 Risco Médio: Configuração Incorreta Entre Domínios - CORS (CWE-264)](#2-configuração-incorreta-entre-domínios---cors-cwe-264)
3. [🟠 Risco Médio: Content Security Policy - CSP Ausente (CWE-693)](#3-content-security-policy---csp-ausente-cwe-693)
4. [🟠 Risco Médio: Anti-clickjacking Header Ausente (CWE-1021)](#4-anti-clickjacking-header-ausente-cwe-1021)
5. [🟠 Risco Médio: Session ID in URL Rewrite (CWE-598)](#5-session-id-in-url-rewrite-cwe-598)
6. [🟠 Risco Médio: Vulnerable JS Library (CWE-1395)](#6-vulnerable-js-library-cwe-1395)
7. [🟡 Risco Baixo: Cross-Domain JavaScript Source File Inclusion (CWE-829)](#7-cross-domain-javascript-source-file-inclusion-cwe-829)
8. [🟡 Risco Baixo: Divulgação de Data e Hora - Unix (CWE-497)](#8-divulgação-de-data-e-hora---unix-cwe-497)
9. [🟡 Risco Baixo: Divulgação de IP Privado (CWE-497)](#9-divulgação-de-ip-privado-cwe-497)
10. [🟡 Risco Baixo: Vazamento de Versão no Cabeçalho Server (CWE-497)](#10-vazamento-de-versão-no-cabeçalho-server-cwe-497)
11. [🟡 Risco Baixo: Strict-Transport-Security - HSTS Ausente (CWE-319)](#11-strict-transport-security---hsts-ausente-cwe-319)
12. [🟡 Risco Baixo: X-Content-Type-Options Header Missing (CWE-693)](#12-x-content-type-options-header-missing-cwe-693)
13. [🔵 Informativo: Comentários Suspeitos no Código-Fonte (CWE-615)](#13-comentários-suspeitos-no-código-fonte-cwe-615)
14. [🔵 Informativo: Modern Web Application](#14-modern-web-application)
15. [🔵 Informativo: Diretivas de Cache-Control Re-examine (CWE-525)](#15-diretivas-de-cache-control-re-examine-cwe-525)
16. [🔵 Informativo: Retrieved from Cache (CWE-525)](#16-retrieved-from-cache-cwe-525)

---

## 1. Injeção SQL (CWE-89)

- **Severidade:** 🔴 Alto | **Confiança:** Baixo | **Ocorrências:** 1
- **Scanner ZAP:** Active Scanner (`40018`)
- **Classificação:** [CWE-89](https://cwe.mitre.org/data/definitions/89.html), WASC-19, OWASP Top 10 2021 - A03 (Injection)

### Descrição Técnica
A injeção de SQL ocorre quando dados fornecidos pelo usuário são inseridos diretamente em uma consulta SQL sem validação, escape ou parametrização adequada. Isso permite que um atacante manipule a lógica da consulta original para ler dados restritos, alterar registros ou executar comandos no banco.

### Evidência Detectada no Juice Shop (Sem WAF)
- **Requisição:**
  ```http
  GET /juice-shop/rest/products/search?q=%27%28 HTTP/1.1
  Host: 3.22.70.177
  ```
- **Parâmetro:** `q`
- **Payload:** `q='(`
- **Comportamento:** A aplicação processou a aspa não tratada e o parêntese, resultando em erro de sintaxe SQL ou retorno anômalo.

### Correção no Código (Remediação Definitiva)
No backend (Node.js / Express / Sequelize / SQLite):
```javascript
// ❌ VULNERÁVEL (Concatenação de string):
db.query(`SELECT * FROM Products WHERE name LIKE '%${req.query.q}%'`);

// ✅ SEGURO (Uso de consultas parametrizadas / Prepared Statements):
db.query(
  'SELECT * FROM Products WHERE name LIKE :searchTerm',
  { replacements: { searchTerm: `%${req.query.q}%` }, type: QueryTypes.SELECT }
);
```

### Mitigação no WAF
- **Regra AWS WAF:** `AWSManagedRulesSQLiRuleSet`
- **Ação:** O WAF inspeciona a query string `q` antes de repassar ao backend. Ao detectar o caractere de escape `'` associado a operadores, retorna imediatamente **HTTP 403 Forbidden**.

---

## 2. Configuração Incorreta Entre Domínios - CORS (CWE-264)

- **Severidade:** 🟠 Médio | **Confiança:** Médio | **Ocorrências:** 34
- **Scanner ZAP:** Passive Scanner (`10098`)
- **Classificação:** [CWE-264](https://cwe.mitre.org/data/definitions/264.html), WASC-14

### Descrição Técnica
O cabeçalho `Access-Control-Allow-Origin` estava configurado de forma permissiva (`*` ou refletindo qualquer origem com credenciais), permitindo que sites maliciosos façam requisições autenticadas em nome do usuário e leiam respostas sensíveis.

### Correção no Código
Restringir origens permitidas no middleware CORS do Express:
```javascript
const cors = require('cors');

const allowedOrigins = ['https://meudominio.com.br'];
app.use(cors({
  origin: function(origin, callback){
    if(!origin || allowedOrigins.indexOf(origin) !== -1){
      callback(null, true);
    } else {
      callback(new Error('Bloqueado por CORS'));
    }
  },
  credentials: true
}));
```

---

## 3. Content Security Policy - CSP Ausente (CWE-693)

- **Severidade:** 🟠 Médio | **Confiança:** Alto | **Ocorrências:** 13
- **Scanner ZAP:** Passive Scanner (`10038`)
- **Classificação:** [CWE-693](https://cwe.mitre.org/data/definitions/693.html), WASC-15

### Descrição Técnica
O cabeçalho HTTP `Content-Security-Policy` não foi retornado pelo servidor. O CSP é a principal linha de defesa no navegador contra ataques de Cross-Site Scripting (XSS), Clickjacking e injeção de dados.

### Correção no Código / Servidor
Adicionar cabeçalho CSP rigoroso (via biblioteca `helmet` no Node.js ou no Nginx):
```javascript
const helmet = require('helmet');
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", "data:"]
  }
}));
```

---

## 4. Anti-clickjacking Header Ausente (CWE-1021)

- **Severidade:** 🟠 Médio | **Confiança:** Médio | **Ocorrências:** 8
- **Scanner ZAP:** Passive Scanner (`10020`)
- **Classificação:** [CWE-1021](https://cwe.mitre.org/data/definitions/1021.html)

### Descrição Técnica
A aplicação não inclui o cabeçalho `X-Frame-Options` ou a diretiva CSP `frame-ancestors`. Isso permite que a página seja renderizada dentro de uma tag `<iframe` em um site sob controle de um invasor, facilitando ataques de *UI Redressing* / Clickjacking.

### Correção
Configurar `X-Frame-Options`:
```http
X-Frame-Options: DENY
# ou
X-Frame-Options: SAMEORIGIN
```

---

## 5. Session ID in URL Rewrite (CWE-598)

- **Severidade:** 🟠 Médio | **Confiança:** Médio | **Ocorrências:** 31
- **Scanner ZAP:** Passive Scanner (`10011`)
- **Classificação:** [CWE-598](https://cwe.mitre.org/data/definitions/598.html)

### Descrição Técnica
Identificadores de sessão foram passados ou reescritos como parte da URL (query string ou path). Isso expõe tokens de sessão nos históricos de navegação, logs de servidores intermediários, cabeçalhos `Referer` e capturas de tela.

### Correção
Armazenar tokens e IDs de sessão exclusivamente em cookies com as flags de proteção ativadas:
```http
Set-Cookie: session_id=xyz; Secure; HttpOnly; SameSite=Strict
```

---

## 6. Vulnerable JS Library (CWE-1395)

- **Severidade:** 🟠 Médio | **Confiança:** Médio | **Ocorrências:** 1
- **Scanner ZAP:** Passive Scanner (`10003`)
- **Classificação:** [CWE-1395](https://cwe.mitre.org/data/definitions/1395.html)

### Descrição Técnica
Uso de bibliotecas JavaScript de frontend obsoletas contendo vulnerabilidades conhecidas com identificadores CVE públicos.

### Correção
- Atualizar dependências frontend utilizando ferramentas de auditoria contínua (`npm audit`, `Snyk`, `Dependabot`).
- Remover bibliotecas não utilizadas.

---

## 7. Cross-Domain JavaScript Source File Inclusion (CWE-829)

- **Severidade:** 🟡 Baixo | **Confiança:** Médio | **Ocorrências:** 2
- **Scanner ZAP:** Passive Scanner (`10017`)
- **Classificação:** [CWE-829](https://cwe.mitre.org/data/definitions/829.html)

### Descrição Técnica
Inclusão de scripts JavaScript hospedados em domínios de terceiros (ex: CDNs públicas) sem garantia de integridade. Se a CDN for comprometida, o script malicioso será executado no contexto da aplicação.

### Correção
Implementar **Subresource Integrity (SRI)** nas tags de script:
```html
<script src="https://cdn.exemplo.com/lib.js"
        integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
        crossorigin="anonymous"></script>
```

---

## 8. Divulgação de Data e Hora - Unix (CWE-497)

- **Severidade:** 🟡 Baixo | **Confiança:** Baixo | **Ocorrências:** 4
- **Scanner ZAP:** Passive Scanner (`10096`)
- **Classificação:** [CWE-497](https://cwe.mitre.org/data/definitions/497.html)

### Descrição Técnica
Timestamps em formato Epoch Unix foram identificados no corpo de respostas HTTP. Embora de baixo impacto isoladamente, permitem que atacantes sincronizem ataques baseados em tempo ou identifiquem janelas de geração de tokens previsíveis.

---

## 9. Divulgação de IP Privado (CWE-497)

- **Severidade:** 🟡 Baixo | **Confiança:** Médio | **Ocorrências:** 2
- **Scanner ZAP:** Passive Scanner (`10062`)
- **Classificação:** [CWE-497](https://cwe.mitre.org/data/definitions/497.html)

### Descrição Técnica
Endereços IP de intervalos privados (RFC 1918, ex: `10.x.x.x`, `172.16.x.x` ou `192.168.x.x`) vazaram em cabeçalhos HTTP ou respostas de erro, revelando o endereçamento interno da infraestrutura de nuvem.

### Correção
Configurar proxies reversos e servidores web para remover cabeçalhos que revelem IPs internos (como `X-Forwarded-For` não higienizados ou mensagens de stack trace).

---

## 10. Vazamento de Versão no Cabeçalho Server (CWE-497)

- **Severidade:** 🟡 Baixo | **Confiança:** Alto | **Ocorrências:** 44
- **Scanner ZAP:** Passive Scanner (`10036`)
- **Classificação:** [CWE-497](https://cwe.mitre.org/data/definitions/497.html)

### Descrição Técnica
O cabeçalho `Server` revela o software e a versão exata do servidor web/backend (ex: `Express`, `nginx/1.18.0`).

### Correção
Desativar ou ofuscar cabeçalhos de identificação:
```javascript
// Express
app.disable('x-powered-by');
```
```nginx
# Nginx
server_tokens off;
```

---

## 11. Strict-Transport-Security - HSTS Ausente (CWE-319)

- **Severidade:** 🟡 Baixo | **Confiança:** Médio | **Ocorrências:** 10
- **Scanner ZAP:** Passive Scanner (`10035`)
- **Classificação:** [CWE-319](https://cwe.mitre.org/data/definitions/319.html)

### Descrição Técnica
O cabeçalho `Strict-Transport-Security` não foi configurado. Isso permite que um atacante em rede compartilhada force o downgrade da conexão para HTTP em texto claro (SSL Stripping).

### Correção
Habilitar HTTPS obrigatório e adicionar o cabeçalho:
```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

---

## 12. X-Content-Type-Options Header Missing (CWE-693)

- **Severidade:** 🟡 Baixo | **Confiança:** Médio | **Ocorrências:** 15
- **Scanner ZAP:** Passive Scanner (`10021`)
- **Classificação:** [CWE-693](https://cwe.mitre.org/data/definitions/693.html)

### Descrição Técnica
A ausência do cabeçalho `X-Content-Type-Options: nosniff` faz com que certos navegadores tentem adivinhar o tipo MIME do arquivo (MIME-sniffing), podendo interpretar arquivos de imagem ou texto como scripts executáveis.

### Correção
```http
X-Content-Type-Options: nosniff
```

---

## 13. Comentários Suspeitos no Código-Fonte (CWE-615)

- **Severidade:** 🔵 Informativo | **Confiança:** Baixo | **Ocorrências:** 1
- **Scanner ZAP:** Passive Scanner (`10027`)
- **Classificação:** [CWE-615](https://cwe.mitre.org/data/definitions/615.html)

### Descrição Técnica
Foram identificados comentários em arquivos HTML/JS com termos como `TODO`, `FIXME`, `BUG` ou anotações de desenvolvimento que podem expor lógica interna ou vulnerabilidades não resolvidas.

---

## 14. Modern Web Application

- **Severidade:** 🔵 Informativo | **Confiança:** Médio | **Ocorrências:** 1
- **Scanner ZAP:** Passive Scanner (`10109`)

### Descrição Técnica
O scanner identificou que o alvo é uma Single Page Application (SPA) moderna baseada em frameworks reativos (Angular/React), orientando o auditor sobre a necessidade de testes adicionais via DOM/AJAX Spider.

---

## 15. Diretivas de Cache-Control Re-examine (CWE-525)

- **Severidade:** 🔵 Informativo | **Confiança:** Baixo | **Ocorrências:** 27
- **Scanner ZAP:** Passive Scanner (`10015`)
- **Classificação:** [CWE-525](https://cwe.mitre.org/data/definitions/525.html)

### Descrição Técnica
Endpoints com dados de usuário não definiram explicitamente cabeçalhos para evitar cache (`no-store`, `no-cache`), podendo permitir que informações sensíveis permaneçam salvas no disco de computadores compartilhados.

### Correção
```http
Cache-Control: no-store, no-cache, must-revalidate, max-age=0
Pragma: no-cache
```

---

## 16. Retrieved from Cache (CWE-525)

- **Severidade:** 🔵 Informativo | **Confiança:** Médio | **Ocorrências:** 1
- **Scanner ZAP:** Passive Scanner (`10050`)

### Descrição Técnica
Respostas servidas a partir de um cache intermediário ou local, indicando comportamento de armazenamento temporário ativo na rota testada.

---

## Resumo Comparativo: Ação do WAF em Cada Categoria

| Categoria de Vulnerabilidade | Bloqueado pelo WAF? | Mecanismo de Ação do AWS WAF |
| :--- | :---: | :--- |
| **Injeção SQL (Alto)** | ✅ **SIM** | Regra de assinatura `AWSManagedRulesSQLiRuleSet` bloqueia caracteres anômalos na URI e Body com HTTP 403. |
| **CORS / Cabeçalhos Ausentes (Médio/Baixo)** | ✅ **MITIGADO NO SCAN** | O WAF pode ser configurado para adicionar cabeçalhos de resposta via ALB/CloudFront; no scan, o ZAP não progrediu devido ao bloqueio de requisições de mapeamento. |
| **Vazamento de Versão / Server Header** | ✅ **MITIGADO NO SCAN** | O ALB atua como proxy reverso, substituindo ou mascarando os cabeçalhos de resposta do servidor interno. |

---

*Documento gerado como material complementar de estudo de segurança ofensiva e defensiva.*
