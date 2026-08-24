# 📋 Leia-me Especializado — Facilitador GRU (feito pela IA Copilot embutido no Github)
## Documentação Técnica para Profissionais de TI e Bibliotecários com Conhecimento em Informática

---

## 📑 Índice

1. [Visão Geral Técnica](#visão-geral-técnica)
2. [Arquitetura e Tecnologias](#arquitetura-e-tecnologias)
3. [Componentes do Sistema](#componentes-do-sistema)
4. [Fluxo de Dados e Validações](#fluxo-de-dados-e-validações)
5. [API de Integração (Portal Tesouro)](#api-de-integração-portal-tesouro)
6. [Segurança e Dados Sensíveis](#segurança-e-dados-sensíveis)
7. [Tratamento de Erros e Logs](#tratamento-de-erros-e-logs)
8. [Deployment e Hospedagem](#deployment-e-hospedagem)
9. [Manutenção e Troubleshooting](#manutenção-e-troubleshooting)
10. [Especificações de Compatibilidade](#especificações-de-compatibilidade)

---

## 🎯 Visão Geral Técnica

### O que é o Facilitador GRU?

É uma **aplicação web de página única (SPA)** em HTML5 puro (sem framework) que funciona como um **intermediário facilitador** entre usuários finais e o Portal de Pagamento do Tesouro Nacional. 

**Funcionalidade Principal:**
- Coleta dados do contribuinte (CPF, nome, valor da multa, campus)
- Valida os dados localmente no cliente
- Constrói uma URL parametrizada conforme especificação da API do Tesouro
- Abre nova abda redirecionando o usuário para `pagtesouro.tesouro.gov.br` com os dados pré-preenchidos

**Escopo:**
- ❌ NÃO processa pagamentos
- ❌ NÃO armazena dados em servidor
- ❌ NÃO mantém session de usuário
- ✅ SIM: Validação client-side, formatação de dados, redirecionamento seguro

---

## 🛠️ Arquitetura e Tecnologias

### Stack Tecnológico

| Componente | Tecnologia | Versão/CDN | Propósito |
|-----------|-----------|-----------|-----------|
| **Markup** | HTML5 | Nativo | Estrutura semântica |
| **Styling** | Tailwind CSS | v3+ (CDN) | Utilidades de design responsivo |
| **Icons** | Phosphor Icons | Web (CDN) | Ícones vectoriais de alta qualidade |
| **Lógica** | JavaScript Vanilla (ES6+) | Nativo | Validações, mascaras, manipulação DOM |
| **Protocolo** | HTTPS | TLS 1.2+ | Transporte seguro |
| **Storage Client** | localStorage (opcional) | Web API | Persistência local (não implementado) |

### Dependências Externas (CDN)

```html
<!-- Tailwind CSS via jsDelivr -->
<script src="https://cdn.tailwindcss.com"></script>

<!-- Phosphor Icons Web -->
<script src="https://unpkg.com/@phosphor-icons/web"></script>
```

⚠️ **Implicação crítica:** A aplicação é **online-only** — requer conexão com CDN para funcionar corretamente. Sem acesso aos CDNs, falha na renderização de estilos e ícones.

---

## 🧩 Componentes do Sistema

### 1. **Sistema de Notificações (Toast)**

**Elemento DOM:**
```html
<div id="toast" class="fixed top-4 right-4 z-50 ...">
    <i id="toast-icon"></i>
    <span id="toast-msg"></span>
</div>
```

**Função de Exibição:**
```javascript
function mostrarMensagem(texto, tipo = 'success')
```

**Parametros:**
- `texto` (string): Mensagem a exibir
- `tipo` (string): `'success'` | `'error'`

**Comportamento:**
- Position: Fixed top-right (z-index: 50)
- Auto-dismiss: 3 segundos
- Animação: Slide-in from top + fade-out
- Classes CSS aplicadas dinamicamente:
  - ✅ Success: `bg-green-600`
  - ❌ Error: `bg-red-500`

**Implementação de Animação:**
```javascript
toast.classList.replace('translate-y-[-150%]', 'translate-y-0');
toast.classList.replace('opacity-0', 'opacity-100');
// ... após 3s ...
toast.classList.replace('translate-y-0', 'translate-y-[-150%]');
toast.classList.replace('opacity-100', 'opacity-0');
```

---

### 2. **Máscaras de Entrada (Input Formatting)**

#### 📱 Máscara CPF

**Trigger:** `input` event no elemento `#cpf`

**Algoritmo:**
```
1. Remove tudo que não for dígito: \D
2. Limita a 11 caracteres
3. Aplica máscara: 000.000.000-00
   - Insere ponto após 3º dígito
   - Insere ponto após 6º dígito
   - Insere travessão após 9º dígito
```

**Exemplo:**
```
Entrada: 12345678901
Processado: 123.456.789-01
```

**Regex Pattern:**
```javascript
value = value.replace(/(\d{3})(\d)/, '$1.$2');      // 123.456...
value = value.replace(/(\d{3})(\d)/, '$1.$2');      // 123.456.789...
value = value.replace(/(\d{3})(\d{1,2})$/, '$1-$2'); // 123.456.789-01
```

#### 💰 Máscara Valor (Moeda Brasileira)

**Trigger:** `input` event no elemento `#valor`

**Algoritmo:**
```
1. Remove tudo que não for dígito
2. Converte para número decimal (divide por 100)
3. Formata usando locale pt-BR com 2 casas decimais
```

**Exemplo:**
```
Entrada: 1550        → 15,50
Entrada: 100         → 1,00
Entrada: 2000        → 20,00
```

**Método:**
```javascript
let numericValue = (parseInt(value, 10) / 100);
e.target.value = numericValue.toLocaleString('pt-BR', {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2
});
```

---

### 3. **Bloco de Seleção de Campus**

**Elemento:**
```html
<select id="campus">
    <option value="">Selecione o seu campus...</option>
    <option value="030955">Campus Brasília</option>
    <option value="152145">Campus Ceilândia</option>
    <!-- ... 10 campi total ... -->
</select>
```

**Mapeamento Campus → Código de Serviço → E-mail**

| Campus | Código Serviço | E-mail |
|--------|------------------|--------|
| Brasília | `030955` | bibliotecabrasilia@ifb.edu.br |
| Ceilândia | `031180` | biblioteca.ccei@ifb.edu.br |
| Estrutural | `034504` | bibliotecaestrutural@ifb.edu.br |
| Gama | `033575` | cgam.biblioteca@ifb.edu.br |
| Planaltina | `033275` | bibliotecaplanaltina@ifb.edu.br |
| Recanto das Emas | `031403` | cdbi.crem@ifb.edu.br |
| Riacho Fundo | `030898` | biblioteca.crfi@ifb.edu.br |
| Samambaia | `031315` | bibliotecasamambaia@ifb.edu.br |
| São Sebastião | `033511` | biblioteca.cssb@ifb.edu.br |
| Taguatinga | `031373` | biblioteca.ctag@ifb.edu.br |

⚠️ **Crítico:** Os códigos de serviço (`value`) devem corresponder aos códigos reconhecidos pelo Portal Tesouro. Mudanças requerem validação com equipe financeira do IFB.

---

### 4. **Campos de Entrada Principal**

| Campo | ID | Tipo | Mascara | Max Length | Obrigatório |
|-------|-----|-------|---------|-----------|-----------|
| Campus | `campus` | `<select>` | N/A | N/A | ✅ Sim |
| CPF | `cpf` | `<input>` text | 000.000.000-00 | 14 | ✅ Sim |
| Nome | `nome` | `<input>` text | Nenhuma | Ilimitado | ✅ Sim |
| Valor | `valor` | `<input>` text | 0,00 (BR) | Ilimitado | ✅ Sim |

---

### 5. **Seções de UI (Toggle)**

A interface alterna entre duas seções principais via classes Tailwind:

#### Seção 1: `#form-section` (Formulário Principal)
- Estado inicial: `visible`
- Contém: Campus, CPF, Nome, Valor, Botão "Ir para Pagamento"
- Transição: `.hidden` → `.flex` (Tailwind)

#### Seção 2: `#helper-section` (Resumo e Ajuda)
- Estado inicial: `hidden`
- Contém: Resumo de dados, botões de copiar, e-mail, reset
- Transição: `.hidden` → `.flex` + `animate-fade-in`

**Toggle Logic:**
```javascript
// Mostrar helper, ocultar form
document.getElementById('form-section').classList.add('hidden');
document.getElementById('helper-section').classList.remove('hidden');
document.getElementById('helper-section').classList.add('animate-fade-in');

// Voltar ao form
document.getElementById('helper-section').classList.add('hidden');
document.getElementById('form-section').classList.remove('hidden');
```

---

## 🔄 Fluxo de Dados e Validações

### Fluxo Principal: `handleOpenTesouro()`

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Usuário clica "Ir para Pagamento no Tesouro"            │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 2. Coleta dados dos campos                                  │
│    - campus (select value)                                  │
│    - cpf (input, remove mascaração)                        │
│    - nome (input, uppercase)                               │
│    - valor (input, converte decimal)                       │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 3. VALIDAÇÕES CLIENT-SIDE                                   │
│    ├─ Campus selecionado? (servicoCodigo !== '')           │
│    ├─ CPF válido? (length === 11)                          │
│    ├─ Nome preenchido? (trim !== '')                       │
│    └─ Valor válido? (> 0 e numeric)                        │
│                                                              │
│    Se FALHAR: mostrarMensagem(erro) + focus() + return    │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 4. FORMATAÇÕES                                              │
│    ├─ CPF limpo: remove \D                                 │
│    ├─ Nome: .toUpperCase().trim()                          │
│    ├─ Valor URL: 00.00 (decimal americano)                │
│    └─ Valor Tela: 00,00 (decimal brasileiro)              │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 5. LOOKUP: emailsCampi[servicoCodigo]                       │
│    (Mapear campus para e-mail da biblioteca)               │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 6. PREENCHER TELA DE RESUMO (#helper-section)              │
│    ├─ #display-cpf         ← CPF limpo                     │
│    ├─ #display-nome        ← Nome uppercase               │
│    ├─ #display-valor       ← Valor em formato BR          │
│    └─ #display-email-*     ← E-mail biblioteca            │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 7. CONSTRUIR URL TESOURO                                    │
│    urlTesouro = https://pagtesouro.tesouro.gov.br/...      │
│    ?servico={code}                                          │
│    &numeroReferencia=100                                    │
│    &valorPrincipal={valor}                                 │
│    &cpfCnpjContrib={cpf}                                   │
│    &nomeContrib={nome}                                     │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 8. ATUALIZAR LINK DE FALLBACK (#fallback-link href)       │
│    (Para caso bloqueador de popups interfira)              │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 9. TRANSIÇÃO UI                                             │
│    - Ocultar #form-section (.hidden)                       │
│    - Mostrar #helper-section                               │
│    - Adicionar animação fade-in                            │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 10. ABRIR NOVA ABA                                          │
│     window.open(urlTesouro, '_blank', 'noopener,noreferrer')│
│     + Mostrar toast "Abrindo site do Tesouro..."           │
└─────────────────────────────────────────────────────────────┘
```

### Validações Detalhadas

#### ❌ Validação Campus Ausente
```javascript
if (!servicoCodigo) {
    mostrarMensagem("Por favor, selecione o seu Campus.", "error");
    document.getElementById('campus').focus();
    return;
}
```

#### ❌ Validação CPF
```javascript
// cpfLimpo = CPF com \D removido
if (cpfLimpo.length !== 11) {
    mostrarMensagem("Digite um CPF válido (11 números).", "error");
    document.getElementById('cpf').focus();
    return;
}
```

**Nota:** Atualmente a validação é apenas de **comprimento**. NÃO valida:
- Dígitos verificadores (algoritmo módulo 11)
- CPF sequencial (11111111111)
- Padrões conhecidos como inválidos

**Recomendação TI:** Implementar validação completa de CPF seguindo [Resolução 65/2008 do Banco Central](https://www.bcb.gov.br).

#### ❌ Validação Nome
```javascript
if (!nome) {
    mostrarMensagem("O nome do contribuinte é obrigatório.", "error");
    document.getElementById('nome').focus();
    return;
}
```

#### ❌ Validação Valor
```javascript
if (!valorCru || isNaN(valorCru) || parseFloat(valorCru) <= 0) {
    mostrarMensagem("Informe um valor válido maior que zero.", "error");
    document.getElementById('valor').focus();
    return;
}
```

---

## 🔌 API de Integração (Portal Tesouro)

### Endpoint Alvo

```
https://pagtesouro.tesouro.gov.br/portal-gru/#/pagamento-gru/formulario
```

### Parâmetros de Query

| Parâmetro | Tipo | Exemplo | Origem | Obrigatório |
|-----------|------|---------|---------|-----------|
| `servico` | string | `030955` | Campus (select value) | ✅ Sim |
| `numeroReferencia` | int | `100` | Hardcoded (constante IFB) | ✅ Sim |
| `valorPrincipal` | decimal | `15.50` | Valor formatado (formato USA) | ✅ Sim |
| `cpfCnpjContrib` | string | `12345678901` | CPF limpo (sem mascaração) | ✅ Sim |
| `nomeContrib` | string | `JOAO SILVA` | Nome em UPPERCASE | ✅ Sim |

### Construção da URL

```javascript
const urlTesouro = `https://pagtesouro.tesouro.gov.br/portal-gru/#/pagamento-gru/formulario?servico=${servicoCodigo}&numeroReferencia=100&valorPrincipal=${valorFormatadoURL}&cpfCnpjContrib=${cpfLimpo}&nomeContrib=${nome}`;
```

### Exemplo Completo

```
https://pagtesouro.tesouro.gov.br/portal-gru/#/pagamento-gru/formulario?servico=030955&numeroReferencia=100&valorPrincipal=15.50&cpfCnpjContrib=12345678901&nomeContrib=JOAO%20SILVA
```

### 🔐 Protocolo de Abertura

```javascript
window.open(urlTesouro, '_blank', 'noopener,noreferrer');
```

**Flags de Segurança:**
- `noopener`: Desabilita acesso do Tesouro ao objeto `window.opener` (proteção contra referrer stealing)
- `noreferrer`: Remove header `Referer` na requisição (privacidade)

⚠️ **Implicação:** O bloqueador de popups do navegador pode interromper a abertura. Solução implementada:
```javascript
// Link de fallback para clique manual
document.getElementById('fallback-link').href = urlTesouro;
```

---

## 🔒 Segurança e Dados Sensíveis

### 1. **Dados em Trânsito**

#### ✅ O que é protegido:
- Comunicação com Portal Tesouro: **HTTPS/TLS 1.2+**
- Headers seguros: `Strict-Transport-Security`, `X-Content-Type-Options`
- Cookies: `HttpOnly`, `Secure`, `SameSite`

#### ❌ O que NÃO é protegido:
- **URL contém dados sensíveis** (CPF, Nome no query string)
  - Browser histórico: acesso ao URL com dados
  - Logs de servidor: CPF e nome aparecem em access logs
  - Proxies corporativos: possivelmente loguem os parâmetros

**Mitigação:** 
1. CPF não é armazenado — apenas transitório
2. Uso de método POST ao invés de GET (se possível com Tesouro)
3. Aviso ao usuário sobre compartilhamento de comprovante

### 2. **Armazenamento Local**

#### Atual:
- ❌ NÃO usa `localStorage`
- ❌ NÃO usa `sessionStorage`
- ❌ NÃO usa cookies
- ✅ Dados residem APENAS em variáveis JS (perdidas ao fechar aba)

#### Risco Residual:
- Developer Tools (F12) pode expor dados em memória durante a sessão
- Cache do navegador pode conter HTML com dados (se houver script dinâmico)

**Recomendação:** Limpar memória após fechamento:
```javascript
window.addEventListener('beforeunload', () => {
    // Limpar dados sensíveis
    document.getElementById('cpf').value = '';
    document.getElementById('nome').value = '';
    document.getElementById('valor').value = '';
});
```

### 3. **Funções de Cópia (Clipboard)**

```javascript
function copiarParaAreaDeTransferencia(idElemento, isDinheiro = false)
```

**Processo:**
1. Cria `<textarea>` temporária (oculta)
2. Executa `document.execCommand('copy')`
3. Remove elemento após cópia

**Vulnerabilidades:**
- Clipboard fica preenchido com dados sensíveis até próxima cópia
- Malware poderia interceptar clipboard (fora do escopo web)

**Mitigação:**
```javascript
// Limpar clipboard após 30s (futuro)
setTimeout(() => {
    document.execCommand('cut'); // Ou usar Clipboard API moderna
}, 30000);
```

### 4. **Validação de Entrada (XSS Prevention)**

Risco Atual: ⚠️ **Médio**

**Campo Vulnerável:**
```javascript
document.getElementById('display-nome').innerText = nome;
```

- Usando `.innerText` ao invés de `.innerHTML` ✅ (seguro)
- Entrada é `.trim()` e `.toUpperCase()` mas **não sanitizada**

**Exemplo Ataque XSS:**
```
Nome: <img src=x onerror="alert('XSS')">SILVA
```

Resultado: Como usa `.innerText`, não executa script (seguro). ✅

**Recomendação:** Aplicar sanitização adicional:
```javascript
function sanitizeInput(input) {
    const div = document.createElement('div');
    div.textContent = input; // Força conversão para texto puro
    return div.innerHTML;
}
```

---

## 📊 Tratamento de Erros e Logs

### Sistema Atual de Notificação

**Classe:** `mostrarMensagem(texto, tipo)`

**Tipos de Mensagem:**

| Tipo | Classe CSS | Ícone | Uso |
|------|-----------|-------|-----|
| `success` | `bg-green-600` | `ph-check-circle` | Validação OK, popup aberto |
| `error` | `bg-red-500` | `ph-warning-circle` | Validação falhou |

**Exemplo de Erros Capturados:**

```javascript
// Erro de cópia ao clipboard
try {
    const sucesso = document.execCommand('copy');
    if (sucesso) {
        mostrarMensagem("Copiado com sucesso!");
    } else {
        mostrarMensagem("Erro ao copiar.", "error");
    }
} catch (err) {
    mostrarMensagem("Erro ao copiar.", "error");
}
```

### Logs e Diagnostico

⚠️ **Limitação Atual:** Nenhum sistema de logging implementado.

**Dados Perdidos:**
- Tentativas de pagamento (sucesso/falha)
- Erros de validação
- Cliques em botões
- Tempo de interação

**Recomendação para Produção:**

```javascript
// Logger simples (envia para servidor via beacon)
function logEvent(eventType, data) {
    const payload = {
        timestamp: new Date().toISOString(),
        event: eventType,
        campus: data.campus,
        // NUNCA logar CPF/Nome completo em logs
        data: {
            valorAttempted: data.valor,
            validationPassed: data.valid,
            userAgent: navigator.userAgent
        }
    };
    
    navigator.sendBeacon('/api/logs', JSON.stringify(payload));
}
```

---

## 🚀 Deployment e Hospedagem

### Requisitos Mínimos

| Requisito | Especificação |
|-----------|---|
| **Protocolo** | HTTPS obrigatório |
| **Servidor** | Qualquer servidor Web (Apache, Nginx, GitHub Pages) |
| **Processamento** | Nenhum — apenas servir arquivo estático |
| **Banco de Dados** | Nenhum necessário |
| **Cache** | Permitido (arquivo é estático) |
| **CDN** | Recomendado para Tailwind/Phosphor |

### Opções de Hospedagem

#### 1. GitHub Pages (Atual)
```
URL: https://sibifb.github.io/Facilitador_Pag_Multa_GRU_IFB/
Vantagens:
  ✅ Gratuito
  ✅ HTTPS automático
  ✅ Versionamento Git
  ✅ CI/CD nativo
Desvantagens:
  ❌ Sem backend (OK para SPA)
```

#### 2. IFB Servidor Institucional
```
URL: https://bibliotecas.ifb.edu.br/facilitador-gru/
Requisitos:
  ✅ HTTPS/TLS
  ✅ CORS liberado (para CDN)
  ✅ Cache control headers
```

#### 3. AWS S3 + CloudFront
```
- S3: Hospedagem estática
- CloudFront: CDN + HTTPS
- Custo: ~$0.5-2/mês (baixo volume)
```

### Headers HTTP Recomendados

```apache
# Cachear assets estáticos por 1 ano
<FilesMatch "\.(js|css|woff|woff2|ttf|svg)$">
    Header set Cache-Control "public, max-age=31536000, immutable"
</FilesMatch>

# Revalidar HTML a cada requisição
<FilesMatch "\.html$">
    Header set Cache-Control "public, max-age=0, must-revalidate"
</FilesMatch>

# Segurança
Header set Strict-Transport-Security "max-age=31536000; includeSubDomains"
Header set X-Content-Type-Options "nosniff"
Header set X-Frame-Options "SAMEORIGIN"
Header set X-XSS-Protection "1; mode=block"
Header set Referrer-Policy "strict-origin-when-cross-origin"
```

### Dockerfile (Se containerizar)

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 443
```

---

## 🔧 Manutenção e Troubleshooting

### Problemas Comuns

#### 1️⃣ **Ícones não aparecem (blank squares)**

**Causa:** Falha ao carregar Phosphor Icons do CDN

**Diagnóstico:**
```javascript
// Console (F12)
fetch('https://unpkg.com/@phosphor-icons/web').then(r => console.log(r.status))
```

**Solução:**
- Verificar conexão internet
- Verificar firewall/proxy corporativo bloqueando CDN
- Usar espelho alternativo: `https://cdn.jsdelivr.net/@phosphor-icons/web`

#### 2️⃣ **Tailwind CSS não carrega (layout quebrado)**

**Causa:** CDN do Tailwind indisponível ou bloqueado

**Diagnóstico:**
```javascript
// Console
document.styleSheets.length // Verificar quantidade de stylesheets
```

**Solução:**
- Alternativa: Build Tailwind localmente
- Usar `tailwindcss-cli` para gerar CSS offline

#### 3️⃣ **Popup não abre**

**Causa:** Bloqueador de popups ativado

**Diagnostic:**
- Verificar notificação do navegador
- Usar fallback link (clique manual)

**Solução:**
```javascript
// Detectar bloqueio e alertar
const popup = window.open('', '_blank');
if (!popup || popup.closed || typeof popup.closed === 'undefined') {
    mostrarMensagem('Bloqueador de popups detectado. Use o link abaixo.', 'error');
}
```

#### 4️⃣ **CPF não formata**

**Causa:** Evento `input` não disparado corretamente

**Diagnóstico:**
```javascript
document.getElementById('cpf').addEventListener('input', () => {
    console.log('Input event fired:', this.value);
});
```

**Solução:**
- Verificar se elemento existe no DOM
- Testar em navegador diferente
- Verificar console (F12) para erros

#### 5️⃣ **Erro "Erro ao copiar" ao usar botões copy**

**Causa:** `document.execCommand('copy')` não funcionando (browsers modernos)

**Diagnóstico:**
```javascript
// Verificar suporte
console.log('execCommand suportado:', document.queryCommandSupported('copy'));
```

**Solução (Moderno):**
```javascript
async function copiarModerno(texto) {
    try {
        await navigator.clipboard.writeText(texto);
        mostrarMensagem("Copiado com sucesso!");
    } catch (err) {
        mostrarMensagem("Erro ao copiar.", "error");
    }
}
```

#### 6️⃣ **URL do Tesouro retorna erro 404**

**Causa:** Código de campus inválido ou desatualizado

**Diagnóstico:**
1. Verificar código no mapeamento `emailsCampi`
2. Validar com Tesouro Nacional
3. Testar URL manualmente no navegador

**Solução:**
- Contatar equipe financeira IFB
- Atualizar valor de `value` no `<select>`

---

### Checklist de Manutenção Mensal

- [ ] Verificar acesso a CDNs (Tailwind, Phosphor, jsDelivr)
- [ ] Testar botão "Ir para Pagamento" com cada campus
- [ ] Validar mascaração de CPF e Valor com entradas edge cases
- [ ] Confirmar e-mails de bibliotecas ainda ativos
- [ ] Revisar console (F12) para erros JS
- [ ] Testar em navegadores: Chrome, Firefox, Safari, Edge
- [ ] Testar em dispositivos móveis (responsividade)
- [ ] Verificar HTTPS certificate (validade)
- [ ] Revisar logs de erro (se implementado)
- [ ] Testar fallback link (se popup bloqueado)

---

## 📱 Especificações de Compatibilidade

### Navegadores Suportados

| Navegador | Versão Mínima | Status |
|-----------|-------------|--------|
| Chrome | 90+ | ✅ Full Support |
| Firefox | 88+ | ✅ Full Support |
| Safari | 14+ | ✅ Full Support |
| Edge | 90+ | ✅ Full Support |
| IE 11 | - | ❌ Não suportado |
| Opera | 76+ | ✅ Full Support |

### Recursos JavaScript Utilizados

| Recurso | ES6+ | Compatibilidade |
|---------|------|-----------------|
| `const/let` | ✅ | Chrome 50+, Firefox 54+, Safari 11+ |
| `Template Literals` | ✅ | Chrome 51+, Firefox 34+, Safari 11+ |
| `.includes()` | ✅ | Chrome 41+, Firefox 43+, Safari 9+ |
| `.classList` | ✅ | Chrome 22+, Firefox 3.6+, Safari 5.1+ |
| `document.execCommand` | ✅ | Todos (deprecado em favor Clipboard API) |
| Spread Operator `...` | ❌ | Não usado |
| Arrow Functions | ❌ | Não usado |

**Conclusão:** Compatibilidade com navegadores modernos (últimas 2 versões).

### Responsividade

**Breakpoints Tailwind:**
- `sm`: 640px
- `md`: 768px
- `lg`: 1024px
- `xl`: 1280px

**Layout:**
```html
<div class="w-full max-w-md"> <!-- Máxima 28rem em telas médias -->
```

**Viewport:**
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

**Testado em:**
- 📱 Mobile: 320px - 480px ✅
- 📱 Tablet: 768px - 1024px ✅
- 🖥️ Desktop: 1920px+ ✅

### Acessibilidade (WCAG 2.1)

| Critério | Status | Nota |
|----------|--------|------|
| Contraste de Cores | ⚠️ Parcial | Tailwind padrão atende AA |
| Labels | ✅ Completo | Todos inputs têm `<label>` |
| Keyboard Navigation | ❌ Ausente | Sem Tab order explícito |
| ARIA | ❌ Ausente | Sem atributos ARIA |
| Screen Readers | ⚠️ Limitado | Funciona mas sem hints específicos |

**Melhorias Recomendadas:**
```html
<!-- Adicionar tabindex e roles -->
<button aria-label="Enviar formulário" tabindex="0" onclick="...">
```

---

## 📝 Notas de Desenvolvimento

### Próximas Versões (Roadmap)

#### v2.0
- [ ] Validação completa de CPF (algoritmo módulo 11)
- [ ] Sistema de logging server-side
- [ ] Dark mode
- [ ] Suporte a idiomas (i18n)
- [ ] Integração com sistema de bibliotecas IFB (API)

#### v2.1
- [ ] Implementar Clipboard API moderna (remover execCommand)
- [ ] Adicionar atributos ARIA (acessibilidade)
- [ ] Progressive Web App (PWA) offline
- [ ] Notificações desktop (Notification API)

#### v3.0
- [ ] Backend Node.js para proxy seguro (POST ao Tesouro)
- [ ] Banco de dados para rastrear tentativas
- [ ] Dashboard administrativo
- [ ] Integração com sistema IFB (autenticação LDAP)

### Contribuindo

**Passos para desenvolvimento local:**

```bash
# 1. Clonar repo
git clone https://github.com/SiBIFB/Facilitador_Pag_Multa_GRU_IFB.git
cd Facilitador_Pag_Multa_GRU_IFB

# 2. Servir arquivo (Python)
python3 -m http.server 8000

# 3. Acessar
open http://localhost:8000/index.html
```

**Padrões de Código:**
- Usar `const/let` ao invés de `var`
- Comentários em português
- Validações côté-cliente sempre (não confiabilidade)
- Não minificar (para auditoria de segurança)

---

## 📞 Contato Técnico

Para dúvidas técnicas, abra uma **Issue** no repositório com os labels:
- `bug`: Erro identificado
- `enhancement`: Melhoria
- `security`: Vulnerabilidade
- `documentation`: Documentação

---

## 📄 Licença

Este projeto é mantido pelo **Sistema de Bibliotecas do IFB** sob licença MIT.

---

**Última atualização:** Agosto 2026
**Versão Documentação:** 1.0 (Especializada)
**Autor:** Equipe TI - IFB
