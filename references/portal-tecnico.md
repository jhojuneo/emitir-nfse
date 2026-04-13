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

## Sessao Expirada

Se a sessao gov.br expirar durante o fluxo, o portal redireciona para login.

Detectar: verificar se a URL atual contem `login` ou `acesso.gov.br`.

Ao detectar: informar o usuario, aguardar novo login, retomar do ponto onde parou (os dados ja preenchidos serao perdidos — o usuario precisara recomecar a nota).

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
