# emitir-nfse

Automacao completa de emissao de NFS-e no portal nacional [nfse.gov.br](https://nfse.gov.br) via Claude Code.

Funciona para prestadores de servico: **MEI, Simples Nacional e Lucro Presumido**.

---

## O que faz

- Emite NFS-e no portal nacional sem precisar preencher formularios manualmente
- Salva PDF (DANFSe) e XML automaticamente em pastas organizadas por mes
- Mantem log de todas as notas emitidas
- Gera resumo mensal para enviar ao contador
- Avisa quando o limite anual do MEI estiver proximo
- Suporta emissao em lote (varias notas de uma vez)

---

## Pre-requisitos

1. **Claude Code** — [instalar](https://claude.ai/code)
2. **Google Chrome** — [baixar](https://chrome.google.com)
3. **Extensao Claude in Chrome** — instalar na Chrome Web Store (buscar por "Claude in Chrome" ou "claude-devtools")
4. **Conta gov.br** com acesso habilitado ao portal nfse.gov.br
5. **CNPJ ativo** com inscricao municipal

---

## Instalacao

### Opcao 1: Clone direto

```bash
git clone https://github.com/SEU_USUARIO/emitir-nfse-skill ~/.claude/skills/emitir-nfse
```

### Opcao 2: Manual

Baixe o repositorio como ZIP e extraia em `~/.claude/skills/emitir-nfse/`

### Opcao 3: Via ClawHub (em breve)

```bash
claude skill add emitir-nfse
```

---

## Primeiro uso — Setup em 2 minutos

Apos instalar, abra o Claude Code e execute:

```
/emitir-nfse setup
```

O assistente vai te guiar para preencher:
- Dados da sua empresa (nome, CNPJ, municipio, regime)
- Aliquota de ISS
- Codigo de tributacao do seu servico
- Descricao padrao
- Onde salvar os PDFs/XMLs

---

## Como usar

### Emitir uma nota

```
/emitir-nfse 12.345.678/0001-90 3000 25/03/2026 "Assessoria em marketing digital"
```

### Emitir em lote

```
/emitir-nfse batch
```

### Ver resumo do mes

```
/emitir-nfse resumo 03/2026
```

### Gerenciar clientes frequentes

```
/emitir-nfse clientes
```

---

## Como funciona

A skill usa o MCP **claude-in-chrome** para controlar o Chrome e preencher automaticamente o formulario do portal nfse.gov.br. O Claude navega pelas 4 etapas (Tomador → Servico → Valores → Emissao), aguarda os carregamentos do portal, e finaliza o download dos arquivos.

Voce so precisa:
1. Estar logado no gov.br no Chrome (a skill avisa se precisar logar)
2. Confirmar os dados antes da emissao (a skill mostra um resumo e pede confirmacao)

---

## Estrutura dos arquivos

```
emitir-nfse/
├── SKILL.md                     # Skill principal
├── references/
│   ├── portal-tecnico.md        # Notas tecnicas do portal
│   ├── regimes-tributarios.md   # MEI, Simples, Lucro Presumido
│   └── codigos-tributacao.md    # Codigos de servico mais comuns
└── templates/
    ├── config.example.md        # Exemplo de configuracao
    └── clientes.example.json    # Exemplo de lista de clientes
```

---

## Licenca

MIT — use, modifique e distribua livremente.
