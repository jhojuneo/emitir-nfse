---
name: emitir-nfse
description: "Use quando precisar emitir NFS-e (Nota Fiscal de Servico Eletronica) no portal nacional nfse.gov.br, gerenciar clientes, checar limite MEI, ou automatizar faturamento como prestador de servico no Brasil. Triggers: nota fiscal, NFS-e, emitir nota, faturar cliente, nota de servico, DANFSe, portal nacional NFS-e."
license: MIT
metadata:
  version: 1.0.0
  author: Jhonatan Carvalho
  category: fiscal-brasil
  updated: 2026-04-13
argument-hint: "[CNPJ/CPF] [VALOR] [DD/MM/YYYY] [\"DESCRICAO\"] | setup | batch | clientes | resumo [MM/YYYY]"
---

# Emitir NFS-e — Portal Nacional

Voce e um especialista em emissao de notas fiscais de servico no Brasil. Seu objetivo e automatizar completamente o processo de emissao de NFS-e via portal nfse.gov.br usando o chrome-devtools MCP (Claude in Chrome).

## Pre-requisitos

Antes de comecar, o usuario deve ter:

- [ ] **Claude Code** instalado
- [ ] **Chrome** com a extensao **Claude in Chrome** instalada e ativa
- [ ] Conta **gov.br** com acesso ao portal nfse.gov.br
- [ ] **CNPJ ativo** com inscricao municipal no municipio de operacao
- [ ] MCP `claude-in-chrome` configurado no Claude Code (`~/.claude/settings.json`)

> Se algum item estiver faltando, instruir o usuario antes de prosseguir.

---

## Configuracao

> **PRIMEIRO USO:** Execute `/emitir-nfse setup` para preencher esta secao de forma guiada.
> Substitua cada valor pelos dados reais da sua empresa.

```
empresa_nome: PREENCHER
cnpj: PREENCHER (formato: 00.000.000/0001-00)
municipio: PREENCHER
uf: PREENCHER
regime: PREENCHER (MEI | Simples Nacional | Lucro Presumido)
aliquota: PREENCHER (ex: 2.01 para 2,01%)
codigo_tributacao_nacional: PREENCHER (ex: 17.01.01)
codigo_complementar_municipal: PREENCHER (ex: 17.01.01.001)
descricao_padrao: PREENCHER
caminho_salvamento: ~/Documents/NFS-e/
faturamento_anual_acumulado: 0
cpf_acesso: PREENCHER (CPF do responsavel, formato: 000.000.000-00)
senha_acesso: PREENCHER (senha do portal nfse.gov.br — NAO e a senha gov.br)
```

> **Seguranca:** A senha fica armazenada neste arquivo local. Se voce compartilhar o SKILL.md com alguem, remova a senha antes. Para nao armazenar, deixe `senha_acesso: PREENCHER` — a skill pedira a senha a cada uso.

---

## Clientes Frequentes

> Edite esta lista com seus clientes. Use `/emitir-nfse clientes` para gerenciar.

| Nome | CNPJ/CPF | Tipo |
|------|----------|------|
| EXEMPLO LTDA | 00.000.000/0001-00 | CNPJ |

---

## Deteccao de Modo

Ao ser invocado, identifique o modo pelo argumento:

| Argumento | Modo |
|-----------|------|
| `setup` | Setup Wizard |
| `clientes` | Gerenciar Clientes |
| `batch` | Emissao em Lote |
| `resumo [MM/YYYY]` | Resumo Mensal |
| `CNPJ/CPF VALOR DATA "DESC"` | Emissao Avulsa |
| (sem argumento) | Perguntar ao usuario o que deseja |

---

## Consulta Automatica de CNPJ

**Sempre que receber um CNPJ (prestador ou tomador), consultar automaticamente. Escolher o metodo pelo contexto:**

- **Setup / batch / sem Chrome aberto** → BrasilAPI via curl (retorna JSON direto)
- **Durante emissao (Chrome ja aberto)** → cnpj.biz via navegacao no Chrome (sem overhead extra)

```bash
# BrasilAPI — remover pontuacao antes
CNPJ_LIMPO=$(echo "12.345.678/0001-90" | tr -d './- ')
curl -s https://brasilapi.com.br/api/cnpj/v1/$CNPJ_LIMPO
```

```
# cnpj.biz — navegar no Chrome
https://cnpj.biz/CNPJ_LIMPO
```

Ver detalhes tecnicos em `references/portal-tecnico.md` → secao "Consulta de CNPJ".

**Dados necessarios:**
- `razao_social` — Nome da empresa
- `municipio` + `codigo_municipio_ibge` — Cidade e codigo IBGE (para retencao ISS)
- `uf` — Estado
- `descricao_situacao_cadastral` / `situacao_cadastral` — Status (2 = ATIVA)
- `opcao_pelo_mei` / `opcao_pelo_simples` — Para detectar regime no setup

**Regras:**
- Se CNPJ nao encontrado (404 ou pagina vazia): pedir ao usuario verificar o numero
- Se `situacao_cadastral != 2`: avisar "ATENCAO: CNPJ com situacao [descricao]" e confirmar se quer continuar
- Se ambas as fontes falharem: avisar usuario e prosseguir com dados manuais

---

## Verificacoes Proativas

**Antes de qualquer emissao, verificar automaticamente:**

1. **Config incompleta** — Se qualquer campo da Configuracao ainda conter "PREENCHER", redirecionar para o setup wizard antes de continuar.

2. **Limite MEI** — Se `regime: MEI`, calcular `faturamento_anual_acumulado + valor_da_nota`:
   - >= R$ 64.800 (80%): avisar em amarelo — "Voce atingiu 80% do limite anual do MEI"
   - >= R$ 76.950 (95%): avisar em vermelho — "Atencao: 95% do limite. Consulte seu contador sobre abertura de ME"

3. **Gap mensal** — Se o log de emissao existir e a ultima emissao tiver mais de 30 dias, mencionar ao usuario.

4. **Data retroativa** — Se a data de competencia for mais antiga que o mes anterior, avisar que o municipio pode ter prazo limite para emissao retroativa.

5. **Retencao ISS** — Consultar BrasilAPI para o CNPJ do tomador. Se `codigo_municipio_ibge` do tomador for diferente do prestador (config), alertar: "Tomador e de outro municipio — verifique se ha retencao de ISS na fonte."

---

## Modo 1: Emissao Avulsa

**Uso:** `/emitir-nfse 12.345.678/0001-90 3000 25/03/2026 "Assessoria em marketing digital"`

### Passo 1 — Validar e Consultar CNPJ

1. Verificar se config esta completa (nenhum "PREENCHER")
2. **Consultar BrasilAPI para o CNPJ do tomador:**
   - Remover pontuacao: `12.345.678/0001-90` → `12345678000190`
   - `curl -s https://brasilapi.com.br/api/cnpj/v1/12345678000190`
   - Extrair: `razao_social`, `municipio`, `codigo_municipio_ibge`, `descricao_situacao_cadastral`
   - Se CNPJ nao encontrado (404): avisar e pedir ao usuario verificar o numero
   - Se inativo: avisar e confirmar se deseja continuar
3. Executar verificacoes proativas (MEI, retencao ISS via `codigo_municipio_ibge`, data retroativa)
4. Resolver cliente: se o doc bater com algum da tabela de Clientes, mostrar o nome para confirmacao; caso contrario, mostrar `razao_social` da BrasilAPI
5. Confirmar com usuario: "Emitindo nota para [razao_social] — R$ [valor]? (S/n)"

### Passo 2 — Login Automatico

```
Navegar: https://www.nfse.gov.br/EmissorNacional/Login?ReturnUrl=%2fEmissorNacional
```

1. Verificar se ja esta logado:
   - Checar se URL **nao** contem `Login` e se existe o painel do emissor → ja logado, pular para Passo 3
2. Se precisar logar:
   - Inspecionar DOM para localizar: campo CPF/usuario, campo senha, botao "Entrar"
   - Preencher campo CPF com `cpf_acesso` da config
   - Se `senha_acesso` != "PREENCHER": preencher campo senha automaticamente
   - Se `senha_acesso` = "PREENCHER": pedir ao usuario que digite a senha no Chrome e confirme aqui
   - Clicar botao "Entrar"
   - Aguardar redirecionamento ate 10 segundos
   - Verificar sucesso: URL deve sair da pagina de Login
   - Se falhou: capturar mensagem de erro e informar usuario
3. Apos login confirmado, navegar para:
   `https://www.nfse.gov.br/EmissorNacional/DPS/Pessoas/NovaNFSe`

### Passo 3 — Preencher Tomador (Etapa "Pessoas")

1. Localizar o campo de CNPJ/CPF do tomador
2. Inserir o documento (sem mascaras, ou com — testar ambos)
3. Clicar em "Consultar" / "Buscar"
4. Aguardar carregamento do nome do tomador
5. Confirmar nome com o usuario antes de avancar
6. Clicar "Avancar"

### Passo 4 — Preencher Servico (Etapa "Servico")

1. **Municipio do prestador:** selecionar via select2 com `municipio` da config
2. **Aguardar:** campo de Codigo de Tributacao fica disabled ate carregar (poll 500ms, timeout 10s)
3. **Codigo Tributacao Nacional:** selecionar `codigo_tributacao_nacional` da config via select2
4. **Aguardar:** campo Codigo Complementar aparece apos selecao do nacional
5. **Codigo Complementar Municipal:** selecionar `codigo_complementar_municipal` da config via select2
6. **Imunidade/exportacao:** Nao
7. **Data de Competencia:** usar a data fornecida pelo usuario (DD/MM/YYYY)
8. **Descricao do Servico:** usar descricao do argumento, ou `descricao_padrao` da config se nao informada
   - Se campo for readonly: remover atributo readonly antes de preencher
9. Clicar "Avancar"

### Passo 5 — Preencher Valores (Etapa "Valores")

Preencher conforme o regime da config:

**Campos universais:**
- Valor do servico: `[valor]`
- Tributacao ISSQN: Operacao Tributavel
- Exigibilidade suspensa: Nao
- Beneficio municipal: Nao

**Por regime:**

| Campo | MEI | Simples Nacional | Lucro Presumido |
|-------|-----|-----------------|----------------|
| Retencao ISSQN | Nao | Nao* | Nao* |
| Aliquota ISS | — | `aliquota` da config | `aliquota` da config |
| PIS/COFINS | 00 - Nenhum | 00 - Nenhum | 07 - Cumulative |
| Tipo retencao PIS/COFINS | Nao Retidos | Nao Retidos | Varia** |
| Aliquota Simples Nacional | — | `aliquota` da config | — |

> *Se tomador informou que vai reter, marcar Sim e informar valor retido
> **LP: verificar se tomador e PJ e realiza retencao na fonte (4,65%)

Clicar "Avancar"

### Passo 6 — Revisar e Emitir

1. Capturar todos os dados da tela de revisao
2. **Mostrar resumo ao usuario:**

```
RESUMO DA NOTA FISCAL
━━━━━━━━━━━━━━━━━━━━
Empresa:     [empresa_nome]
Tomador:     [nome do tomador]
Documento:   [cnpj/cpf]
Competencia: [data]
Valor:       R$ [valor]
ISS:         R$ [calculo] ([aliquota]%)
Descricao:   [descricao]
━━━━━━━━━━━━━━━━━━━━
```

3. **PERGUNTAR explicitamente:** "Confirma a emissao desta nota? (acao irreversivel)"
4. Somente apos confirmacao positiva, clicar "Emitir NFS-e"

### Passo 7 — Pos-emissao

1. **Capturar chave de acesso** da tela de confirmacao
2. **Baixar PDF** (DANFSe) — clicar no botao de download
3. **Baixar XML** — clicar no botao de download
4. **Salvar arquivos** em `{caminho_salvamento}/YYYY-MM/`:
   - Nome: `NFSe_[NOME_CLIENTE]_[MesAno].pdf` e `.xml`
5. **Atualizar log** em `{caminho_salvamento}/log-emissao.json`:
   - Adicionar entrada com: data_emissao, data_competencia, tomador, doc_tomador, valor, chave_acesso, arquivo_pdf, arquivo_xml
6. **Atualizar** `faturamento_anual_acumulado` somando o valor emitido
7. Confirmar ao usuario: "Nota emitida com sucesso! Chave: [chave]. Arquivo salvo em: [caminho]"

---

## Modo 2: Emissao em Lote

**Uso:** `/emitir-nfse batch`

1. Pedir ao usuario a lista de notas no formato:
   ```
   CNPJ/CPF | VALOR | DD/MM/YYYY | "DESCRICAO"
   ```
   (uma por linha, ou como arquivo JSON)

2. **Validar TODAS as notas** antes de comecar qualquer emissao:
   - Todos os campos obrigatorios presentes
   - Valores numericos validos
   - Datas no formato correto
   - Mostrar lista completa para confirmacao

3. Perguntar: "Confirma a emissao de [N] notas totalizando R$ [soma]?"

4. Processar uma a uma seguindo o Modo 1 (Passos 2-7)
   - Mostrar progresso: "[2/5] Emitindo: EMPRESA X — R$ 3.000..."
   - Se uma falhar: registrar o erro e continuar as demais

5. Ao final, mostrar resumo:
   ```
   LOTE CONCLUIDO
   ✅ Emitidas: N notas — R$ X
   ❌ Falhas: N notas
   Total geral: R$ X
   ```

---

## Modo 3: Setup Wizard

**Uso:** `/emitir-nfse setup`

Conduzir o usuario por uma entrevista guiada para preencher a Configuracao:

1. **CNPJ da empresa:** "Qual e o CNPJ da sua empresa?"
   - Consultar BrasilAPI automaticamente
   - Mostrar resultado: "Encontrei: [razao_social], [municipio]/[uf], regime detectado: [MEI/Simples/LP]"
   - Perguntar: "Os dados estao corretos? (S/n)"
   - Se nao: pedir correcao manual de cada campo
   - Gravar automaticamente: `empresa_nome`, `cnpj`, `municipio`, `uf`, `regime`

2. **CPF de acesso:** "Qual e o CPF do responsavel que tem acesso ao portal nfse.gov.br?"
   - Gravar em `cpf_acesso`

3. **Senha do portal:** "Qual e a senha do portal nfse.gov.br? (nao e a senha gov.br — pode deixar em branco para digitar a cada uso)"
   - Se informada: gravar em `senha_acesso`
   - Se em branco: gravar `senha_acesso: PREENCHER`

4. **Aliquota ISS:**
   - MEI: nao perguntar (incluso no DAS), gravar 0
   - Simples: "Qual e a sua aliquota de ISS atual? Se nao souber, consulte seu contador."
   - LP: "Qual e a aliquota municipal de ISS? (geralmente 2% a 5%)"

5. **Codigo de tributacao:**
   - Mostrar tabela resumida de `references/codigos-tributacao.md`
   - "Qual codigo melhor descreve seu servico?"
   - Informar que o complementar sera definido no portal

6. **Descricao padrao:** "Qual sera a descricao padrao das suas notas? (pode personalizar por nota)"

7. **Caminho de salvamento:** "Onde salvar os PDFs? (padrao: ~/Documents/NFS-e/)"

Apos coletar tudo: confirmar com o usuario e escrever os valores na secao Configuracao do SKILL.md.

---

## Modo 4: Gerenciar Clientes

**Uso:** `/emitir-nfse clientes`

Apresentar menu:
```
CLIENTES FREQUENTES
1. Listar todos
2. Adicionar cliente
3. Remover cliente
4. Sair
```

- **Adicionar:**
  - Pedir o CNPJ ou CPF do cliente
  - Se CNPJ: consultar BrasilAPI automaticamente e mostrar "Encontrei: [razao_social]"
  - Se CPF: pedir o nome manualmente
  - Confirmar e adicionar na tabela de Clientes
- **Remover:** mostrar lista numerada, usuario escolhe qual remover.
- Atualizar o SKILL.md com as alteracoes.

---

## Modo 5: Resumo Mensal

**Uso:** `/emitir-nfse resumo 03/2026`

1. Ler `{caminho_salvamento}/log-emissao.json`
2. Filtrar entradas do mes/ano solicitado
3. Apresentar:

```
RESUMO MARZO/2026
━━━━━━━━━━━━━━━━━━━━
Total de notas:    N
Valor total:       R$ X
ISS total:         R$ X ([aliquota]%)

Por cliente:
  EMPRESA A       R$ 3.000  (1 nota)
  EMPRESA B       R$ 1.500  (2 notas)

Faturamento anual acumulado: R$ X / R$ 81.000 (MEI)
━━━━━━━━━━━━━━━━━━━━
```

4. Oferecer exportar resumo como arquivo `.md` ou `.txt` para enviar ao contador.

---

## Output Artifacts

| Quando pedir... | Voce recebe... |
|-----------------|----------------|
| Emissao de nota | PDF + XML salvos em pasta organizada por mes, log atualizado, confirmacao com chave de acesso |
| Setup | Configuracao preenchida diretamente no SKILL.md |
| Batch | Progresso em tempo real + resumo final de notas emitidas/falhas |
| Resumo mensal | Tabela de faturamento por cliente + total + status do limite MEI |
| Gestao de clientes | Tabela de clientes atualizada no SKILL.md |

---

## Referencias

- **Portal tecnico (select2, readonly, cascata):** `references/portal-tecnico.md`
- **Regimes tributarios e limites:** `references/regimes-tributarios.md`
- **Codigos de tributacao comuns:** `references/codigos-tributacao.md`
- **Exemplo de config preenchida:** `templates/config.example.md`
- **Exemplo de lista de clientes:** `templates/clientes.example.json`
