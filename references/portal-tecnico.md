# Portal NFS-e — Notas Tecnicas

Referencia tecnica para automacao do portal nfse.gov.br via chrome-devtools MCP.

---

## Dropdowns Select2

O portal usa a biblioteca Select2 para todos os campos de selecao (municipio, codigo de tributacao, regime especial, etc).

**Nao funciona:** clicar diretamente no option do select nativo.

**Funciona:**
```javascript
window.jQuery('select#idDoSelect')
  .val('valorDaOpcao')
  .trigger('change.select2')
  .trigger('change');
```

Para encontrar o ID do select e os valores das opcoes, inspecionar o DOM antes de interagir.

---

## Campos Readonly

Alguns campos de descricao sao marcados como `readonly` pelo portal. Para preencher:

```javascript
const campo = document.querySelector('textarea#idDoCampo');
campo.removeAttribute('readonly');
campo.value = 'Texto aqui';
campo.dispatchEvent(new Event('input', { bubbles: true }));
campo.dispatchEvent(new Event('change', { bubbles: true }));
```

---

## Radio Buttons

Para selecionar radio buttons (ex: Retencao ISSQN Nao/Sim):

```javascript
document.querySelector('input[name="nomeDoGrupo"][value="NAO"]').click();
```

Nao usar `.checked = true` — o portal escuta o evento de click para atualizar o estado do formulario.

---

## Cascata de Carregamento

O portal tem 3 niveis de cascata que precisam de espera entre selecoes:

### Nivel 1: Municipio
- Apos selecionar o municipio, o campo **Codigo de Tributacao Nacional** fica `disabled` ate carregar os codigos
- Aguardar o campo ficar habilitado: verificar a cada 500ms por ate 10 segundos
- Selector de verificacao: `document.querySelector('#idCampoTributacao').disabled === false`

### Nivel 2: Codigo de Tributacao Nacional
- Apos selecionar o codigo nacional, o campo **Codigo Complementar Municipal** aparece
- O campo pode nao existir no DOM antes da selecao — aguardar o elemento aparecer
- Verificar: `document.querySelector('#idCampoComplementar') !== null`

### Nivel 3: Avancando etapas
- Cada clique em "Avancar" recarrega o conteudo da pagina (SPA parcial)
- Aguardar indicador de etapa mudar antes de interagir com o proximo formulario
- Verificar por elemento especifico da proxima etapa, nao por URL (a URL pode nao mudar)

---

## Selecao de Opcoes em Tabelas

Para campos como PIS/COFINS que usam tabelas de opcoes ao inves de select2:

```javascript
// Clicar na linha da tabela com o valor desejado
document.querySelectorAll('tr.opcao-piscofins').forEach(tr => {
  if (tr.textContent.includes('00 - Nenhum')) tr.click();
});
```

---

## Navegacao entre Etapas

O fluxo tem 4 etapas: Pessoas → Servico → Valores → Revisao

- **Botao Avancar:** `document.querySelector('button[type="submit"].btn-avancar').click()`
- **Verificar etapa atual:** Checar o titulo `<h2>` ou o breadcrumb ativo
- **Nao usar navegacao por URL** — o portal e SPA, a URL nao muda entre etapas

---

## Deteccao de Erros

O portal exibe erros inline apos tentativa de avanco. Para capturar:

```javascript
const erros = Array.from(document.querySelectorAll('.alert-danger, .has-error .help-block'))
  .map(el => el.textContent.trim())
  .filter(t => t.length > 0);
```

Se `erros.length > 0`, reportar ao usuario e aguardar correcao.

---

## Login Automatico

**URL da pagina de login:**
```
https://www.nfse.gov.br/EmissorNacional/Login?ReturnUrl=%2fEmissorNacional
```

### Verificar se ja esta logado

Antes de tentar logar, verificar se a sessao ja esta ativa:

```javascript
// Se a URL atual NÃO contem "Login", ja esta no painel do emissor
const jaLogado = !window.location.href.includes('Login');
```

Se ja logado: pular o fluxo de login e navegar direto para a URL de emissao.

### Selectors do Formulario de Login

Os selectors exatos podem variar — **inspecionar o DOM na primeira execucao**. Padroes esperados:

```javascript
// Campo CPF/usuario (tentar em ordem ate encontrar)
const campoCPF = document.querySelector('input[name*="cpf"]')
  || document.querySelector('input[name*="usuario"]')
  || document.querySelector('input[id*="cpf"]')
  || document.querySelector('input[type="text"]:first-of-type');

// Campo senha
const campoSenha = document.querySelector('input[type="password"]');

// Botao Entrar
const botaoEntrar = document.querySelector('button[type="submit"]')
  || document.querySelector('input[type="submit"]');
```

### Fluxo de Auto Login

```
1. Navegar para URL de login
2. Aguardar pagina carregar (verificar presenca do campo CPF)
3. Preencher campo CPF com cpf_acesso da config
4. Preencher campo senha com senha_acesso da config
5. Clicar botao "Entrar"
6. Aguardar redirecionamento (poll a cada 500ms, timeout 10s)
7. Verificar sucesso: URL deve sair da pagina de Login
8. Se falhou: capturar .alert-danger ou mensagem de erro e reportar ao usuario
```

### Sessao Expirada

Se a sessao expirar durante o fluxo, o portal redireciona para login.

Detectar: verificar se a URL atual contem `Login` ou `acesso.gov.br`.

Ao detectar:
1. Tentar auto re-login automaticamente (se `cpf_acesso` e `senha_acesso` estao configurados)
2. Se re-login bem-sucedido: informar usuario e pedir para reiniciar a nota (dados perdidos)
3. Se re-login falhou ou credenciais nao configuradas: informar usuario e aguardar login manual

---

## Consulta de CNPJ

Duas opcoes — escolher conforme o contexto:

| Opcao | Quando usar | Vantagem |
|-------|-------------|----------|
| **BrasilAPI** (curl) | Fora do fluxo do Chrome, setup, batch | Retorna JSON estruturado, sem abrir pagina |
| **cnpj.biz** (Chrome) | Durante emissao (Chrome ja aberto) | Zero overhead — usar a aba do Chrome que ja existe |

---

### Opcao A: BrasilAPI via curl

```bash
CNPJ_LIMPO=$(echo "$CNPJ" | tr -d './- ')
curl -s https://brasilapi.com.br/api/cnpj/v1/$CNPJ_LIMPO
```

**Campos retornados:**

| Campo | Uso |
|-------|-----|
| `razao_social` | Nome oficial da empresa |
| `nome_fantasia` | Nome fantasia (pode ser vazio) |
| `municipio` | Cidade (portal e retencao ISS) |
| `uf` | Estado |
| `codigo_municipio_ibge` | Comparar com prestador para detectar retencao ISS |
| `descricao_situacao_cadastral` | "ATIVA" ou outro status |
| `situacao_cadastral` | 2 = ATIVA |
| `opcao_pelo_mei` | true/false |
| `opcao_pelo_simples` | true/false |
| `cnae_fiscal_descricao` | Atividade principal |

**Erros:**

| Situacao | Acao |
|----------|------|
| HTTP 404 | CNPJ nao encontrado — pedir ao usuario verificar |
| `situacao_cadastral != 2` | Avisar e confirmar se quer continuar |
| Timeout / erro de rede | Tentar Opcao B (cnpj.biz no Chrome) |

---

### Opcao B: cnpj.biz via Chrome (claude-in-chrome)

Quando o Chrome ja esta aberto (durante emissao), usar a mesma aba ou abrir nova aba:

```
URL: https://cnpj.biz/CNPJ_LIMPO
Exemplo: https://cnpj.biz/50318661000138
```

**Extrair do DOM da pagina:**
```javascript
// Razao social — titulo principal da empresa
const razaoSocial = document.querySelector('h1.company-name, h1, .razao-social')?.textContent?.trim();

// Municipio e UF — aparece em campo de endereco
const municipioUF = document.querySelector('.municipio, .cidade-uf')?.textContent?.trim();

// Situacao cadastral
const situacao = document.querySelector('.situacao-cadastral, .status')?.textContent?.trim();
```

> Os selectors exatos do cnpj.biz podem variar. Se nao encontrar via selector, usar `get_page_text` e parsear o texto retornado — o nome da empresa aparece de forma destacada no inicio da pagina.

**Apos extrair os dados:** fechar a aba ou voltar para a aba do portal nfse.gov.br.

---

## Download de Arquivos

Apos emissao, o portal oferece botoes para baixar PDF (DANFSe) e XML.

- Clicar no botao de download e suficiente — o Chrome baixa automaticamente
- Verificar o arquivo no caminho de download do Chrome antes de mover para o destino configurado
- Nomear os arquivos no padrao: `NFSe_NOME_CLIENTE_MesAno.pdf` / `.xml`

---

## Captura da Chave de Acesso

A chave de acesso aparece na tela de confirmacao apos a emissao. Extrair com:

```javascript
const chave = document.querySelector('.chave-acesso, .codigo-verificacao').textContent.trim();
```

Salvar no log antes de fechar ou navegar.
