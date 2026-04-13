# API REST NFS-e Nacional — Referencia Tecnica

Documentacao tecnica da API do Emissor Publico Nacional para emissao programatica de NFS-e via certificado digital.

**Documentacao oficial:** https://www.gov.br/nfse/pt-br/biblioteca/documentacao-tecnica/apis-prod-restrita-e-producao

---

## Base URLs

| Ambiente | URL |
|----------|-----|
| Producao | `https://sefin.nfse.gov.br/SefinNacional` |
| Testes (prod. restrita) | `https://sefin.producaorestrita.nfse.gov.br/SefinNacional` |
| Swagger producao | `https://adn.nfse.gov.br/contribuintes/docs/index.html` |
| Swagger testes | `https://adn.producaorestrita.nfse.gov.br/contribuintes/docs/index.html` |

**Autenticacao:** Mutual TLS com certificado digital A1 (.pfx/.p12).
Todos os requests usam `curl --cert {certificado_pfx}:{certificado_senha}`.

---

## Endpoints

### POST /nfse — Emissao

Envia DPS (XML assinado) e retorna NFS-e gerada sincronamente.

```bash
curl -s --cert ~/certificado.pfx:SENHA \
  -X POST https://sefin.nfse.gov.br/SefinNacional/nfse \
  -H "Content-Type: application/xml" \
  -d @dps_assinada.xml
```

**Sucesso (200):** Retorna XML da NFS-e com `<chaveAcesso>`.
**Erro (400/422):** Retorna `<xMotivo>` com descricao do problema.

---

### GET /nfse/{chaveAcesso} — Consulta

```bash
curl -s --cert ~/certificado.pfx:SENHA \
  "https://sefin.nfse.gov.br/SefinNacional/nfse/CHAVE_ACESSO"
```

---

### GET /parametros_municipais/{codMunicipio}/convenio

Parametros do municipio necessarios para construir a DPS.

```bash
# codMunicipio = codigo_municipio_ibge retornado pela BrasilAPI
curl -s --cert ~/certificado.pfx:SENHA \
  "https://sefin.nfse.gov.br/SefinNacional/parametros_municipais/3143302/convenio"
```

Retorna: serie DPS, numero inicial, regras do municipio.

---

### GET /parametros_municipais/{codMunicipio}/{CPF_ou_CNPJ}

Retencoes a que o contribuinte esta sujeito no municipio.

```bash
curl -s --cert ~/certificado.pfx:SENHA \
  "https://sefin.nfse.gov.br/SefinNacional/parametros_municipais/3143302/50318661000138"
```

---

### GET /dps/{id} — Verificar DPS

Recupera chave de acesso da NFS-e pelo identificador da DPS.

```bash
curl -s --cert ~/certificado.pfx:SENHA \
  "https://sefin.nfse.gov.br/SefinNacional/dps/ID_DA_DPS"
```

---

### POST /nfse/{chaveAcesso}/eventos — Cancelamento

```bash
curl -s --cert ~/certificado.pfx:SENHA \
  -X POST https://sefin.nfse.gov.br/SefinNacional/nfse/CHAVE/eventos \
  -H "Content-Type: application/xml" \
  -d @evento_cancelamento.xml
```

---

## Schema DPS (Template Base)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<DPS xmlns="http://www.sped.fazenda.gov.br/nfse" versao="1.00">
  <infDPS Id="DPS{CNPJ_LIMPO}{DATA_EMISSAO_YYYYMMDD}{SERIE}{NUMERO}">
    <tpAmb>1</tpAmb>
    <!-- 1=Producao | 2=Homologacao -->
    <dhEmi>2026-04-13T10:00:00-03:00</dhEmi>
    <verAplic>emitir-nfse-skill-1.0</verAplic>
    <serie>1</serie>
    <nDPS>1</nDPS>
    <!-- Numero sequencial — incrementar a cada emissao -->
    <dCompet>2026-04-13</dCompet>
    <!-- Data de competencia (DD/MM/YYYY convertida para YYYY-MM-DD) -->
    <subst>
      <chSubstda/>
      <!-- Deixar vazio para emissao nova. Preencher para substituicao. -->
    </subst>
    <prest>
      <CNPJ>{CNPJ_PRESTADOR_SEM_PONTUACAO}</CNPJ>
      <IM>{INSCRICAO_MUNICIPAL}</IM>
      <!-- IM = inscricao municipal do prestador -->
    </prest>
    <toma>
      <!-- Use CNPJ para pessoa juridica ou CPF para pessoa fisica -->
      <CNPJ>{CNPJ_TOMADOR_SEM_PONTUACAO}</CNPJ>
      <!-- OU: <CPF>{CPF_TOMADOR_SEM_PONTUACAO}</CPF> -->
    </toma>
    <serv>
      <cServ>
        <cTribNac>{CODIGO_TRIBUTACAO_NACIONAL}</cTribNac>
        <!-- Ex: 17.01.01 -->
        <cTribMun>{CODIGO_COMPLEMENTAR_MUNICIPAL}</cTribMun>
        <!-- Ex: 17.01.01.001 -->
        <xDescServ>{DESCRICAO_SERVICO}</xDescServ>
      </cServ>
    </serv>
    <valores>
      <vServPrest>
        <vServ>{VALOR_COM_PONTO_DECIMAL}</vServ>
        <!-- Ex: 3000.00 -->
      </vServPrest>
      <trib>
        <tribMun>
          <tribISSQN>1</tribISSQN>
          <!-- 1=Operacao tributavel -->
          <cPaisResult>1058</cPaisResult>
          <!-- 1058 = Brasil -->
          <BM>
            <cBM>{CODIGO_MUNICIPIO_IBGE}</cBM>
            <!-- Codigo IBGE do municipio do prestador -->
          </BM>
          <pAliq>{ALIQUOTA_COM_PONTO_DECIMAL}</pAliq>
          <!-- Ex: 2.01 -->
        </tribMun>
        <totTrib>
          <vTotTrib>{VALOR_ISS}</vTotTrib>
          <!-- Calcular: valor * aliquota / 100 -->
        </totTrib>
      </trib>
    </valores>
  </infDPS>
</DPS>
```

> **IMPORTANTE:** O schema completo e mais extenso. Este template cobre os campos basicos para o regime Simples Nacional. Para MEI (ISS incluso no DAS) e Lucro Presumido, os campos de tributo variam. Consultar o Anexo I da documentacao oficial antes da primeira emissao em producao.

---

## Assinatura Digital do XML

A DPS deve ser assinada digitalmente antes do envio. Processo via bash:

```bash
# 1. Extrair chave privada e certificado do .pfx
openssl pkcs12 -in ~/certificado.pfx -nocerts -nodes \
  -out /tmp/nfse_key.pem -passin pass:SENHA_DO_CERTIFICADO

openssl pkcs12 -in ~/certificado.pfx -clcerts -nokeys \
  -out /tmp/nfse_cert.pem -passin pass:SENHA_DO_CERTIFICADO

# 2. Assinar o XML (requer xmlsec1: brew install xmlsec1)
xmlsec1 --sign \
  --privkey-pem /tmp/nfse_key.pem,/tmp/nfse_cert.pem \
  --id-attr:Id infDPS \
  --output /tmp/dps_assinada.xml \
  /tmp/dps.xml

# 3. Limpar chaves temporarias imediatamente
rm -f /tmp/nfse_key.pem /tmp/nfse_cert.pem

# 4. Enviar
curl -s --cert ~/certificado.pfx:SENHA_DO_CERTIFICADO \
  -X POST https://sefin.nfse.gov.br/SefinNacional/nfse \
  -H "Content-Type: application/xml" \
  -d @/tmp/dps_assinada.xml

# 5. Limpar DPS temporaria
rm -f /tmp/dps.xml /tmp/dps_assinada.xml
```

---

## Campos que variam por Regime

| Campo DPS | MEI | Simples Nacional | Lucro Presumido |
|-----------|-----|-----------------|----------------|
| `<tribISSQN>` | 3 (nao tributavel — ISS no DAS) | 1 | 1 |
| `<pAliq>` | omitir | aliquota da config | aliquota municipal |
| `<retISSQN>` | Nao | Nao* | Nao* |
| `<vISSRetido>` | 0 | 0* | 0* |

> *Se tomador informou retencao: marcar `<retISSQN>2</retISSQN>` e `<vISSRetido>valor</vISSRetido>`

---

## MCP nfse-nacional (Consulta Complementar)

O MCP `mcp-nfse-nacional` permite **consultar** notas ja emitidas diretamente no Claude Code. Nao emite — apenas busca, detalha e baixa PDF.

**Configurar em `~/.claude/settings.json`:**
```json
{
  "mcpServers": {
    "nfse-nacional": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "mcp-nfse-nacional"],
      "env": {
        "CERT_FILE": "/caminho/absoluto/certificado.pfx",
        "CERT_PASSWORD": "senha_do_certificado",
        "STORAGE_PATH": "~/Documents/NFS-e/xml"
      }
    }
  }
}
```

**Tools disponiveis:**
- `nfse_buscar` — busca notas por periodo (data_inicio, data_fim)
- `nfse_detalhes` — detalhes completos por chave de acesso
- `nfse_pdf` — baixa PDF (DANFSe) por chave de acesso

---

## Codigos de Erro Comuns

| Codigo | Descricao | Solucao |
|--------|-----------|---------|
| 400 | XML invalido ou mal formado | Verificar schema e encoding UTF-8 |
| 401 | Certificado invalido ou expirado | Renovar certificado |
| 422 | Regra de negocio violada | Ler `<xMotivo>` — ex: CNPJ invalido, aliquota incorreta |
| 500 | Erro interno da SEFIN | Tentar novamente em alguns minutos |

---

## Seguranca

- **Nunca** salvar `certificado.pfx` em repositorio git (adicionar ao `.gitignore`)
- **Nunca** incluir a senha do certificado no SKILL.md ao compartilhar
- Chaves extraidas para `/tmp/` devem ser removidas imediatamente apos uso
- O certificado A1 tem validade tipica de 1 a 3 anos — monitorar vencimento
