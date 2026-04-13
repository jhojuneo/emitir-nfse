# Regimes Tributarios — Referencia para NFS-e

---

## Comparativo Rapido

| Regime | Limite Anual | ISS | PIS | COFINS | CSLL | IRPJ |
|--------|-------------|-----|-----|--------|------|------|
| **MEI** | R$ 81.000 | Incluso no DAS | — | — | — | — |
| **Simples Nacional** | R$ 4.800.000 | 2% a 5% (LC 116) | Conforme anexo | Conforme anexo | — | — |
| **Lucro Presumido** | R$ 78.000.000 | 2% a 5% (municipal) | 0,65% | 3% | 2,88% | 4,8% |

---

## MEI — Microempreendedor Individual

**Limite:** R$ 81.000/ano (R$ 6.750/mes)

**Tributacao:** Pagamento unico via DAS (Documento de Arrecadacao do Simples). O ISS ja esta incluso.

**NFS-e:**
- O portal identifica automaticamente o regime ao consultar o CNPJ
- Aliquota no campo de valores: **nao ha percentual de ISS separado** (incluso no DAS)
- Preencher campos de retencao como **NAO** — MEI nao tem ISS retido na fonte pelo tomador

**Atencao:** Se o faturamento acumulado no ano atingir R$ 81.000, o MEI e automaticamente desenquadrado. A skill rastreia `faturamento_anual_acumulado` para alertar antes de atingir o limite.

**Alertas:**
- 80% do limite = R$ 64.800 → aviso amarelo
- 95% do limite = R$ 76.950 → aviso vermelho, considerar abertura de ME

---

## Simples Nacional

**Limite:** R$ 4.800.000/ano

**Tributacao:** Aliquota varia por faixa de faturamento e pelo Anexo da atividade.

**Servicos de consultoria/assessoria (Anexo III):**

| Faixa | Receita Bruta 12 meses | Aliquota | ISS |
|-------|------------------------|----------|-----|
| 1a | Ate R$ 180.000 | 6,00% | 2,00% |
| 2a | R$ 180.001 a R$ 360.000 | 11,20% | 2,00% |
| 3a | R$ 360.001 a R$ 720.000 | 13,50% | 3,00% |
| 4a | R$ 720.001 a R$ 1.800.000 | 16,00% | 3,50% |
| 5a | R$ 1.800.001 a R$ 3.600.000 | 21,00% | 4,50% |
| 6a | R$ 3.600.001 a R$ 4.800.000 | 33,00% | 5,00% |

**NFS-e:** Preencher a aliquota efetiva do ISS conforme a faixa atual. Para empresas na faixa 1 com deducao de folha, pode haver reducao — verificar com contador.

**Retencao ISS:**
- Se o tomador esta no mesmo municipio do prestador: **sem retencao** (em geral)
- Se o tomador esta em municipio diferente: **pode haver retencao** — verificar legislacao do municipio do tomador

---

## Lucro Presumido

**Limite:** R$ 78.000.000/ano

**Tributacao:** ISS municipal (2% a 5%) + PIS (0,65%) + COFINS (3%) + CSLL (2,88%) + IRPJ (4,8% sobre presuncao de 32%)

**NFS-e:**
- Preencher ISS com a aliquota municipal (consultar legislacao do municipio)
- Marcar PIS/COFINS conforme regime: **01 - Cumulative** (Lucro Presumido usa PIS/COFINS cumulativo)
- Selecionar "Tipo de Retencao: Retidos" se o tomador e pessoa juridica e realiza a retencao na fonte

**Retencao PIS/COFINS/CSLL:** Tomadores PJ que pagam por servicos de consultoria geralmente reteem 4,65% (PIS 0,65% + COFINS 3% + CSLL 1%). Verificar se o contrato ou a legislacao exige.

---

## ISS — Regras de Retencao

**Regra geral (LC 116/2003):**
- ISS e devido no municipio onde o servico e prestado
- Para servicos de consultoria/assessoria, geralmente e o municipio do **prestador**

**Excecoes onde o ISS e devido no municipio do tomador:**
- Construcao civil, demolicao, reparos
- Limpeza, manutencao de imoveis
- Servicos de diversoes, lazer em local fixo

**Consequencia pratica para consultoria/marketing digital:**
- Na maioria dos casos: ISS devido ao municipio do prestador, **sem retencao pelo tomador**
- Verificar sempre o municipio do tomador e a legislacao local

---

## Perguntas Frequentes

**P: Tomador de outro estado pode reter o ISS?**
R: Somente se a legislacao municipal do tomador prever a retencao para aquele tipo de servico. Para consultoria, a maioria dos municipios nao exige retencao.

**P: MEI precisa emitir NFS-e?**
R: Depende do municipio. Alguns municipios obrigam MEI a emitir NFS-e. Outros aceitam recibo. Verificar a legislacao municipal.

**P: Posso emitir NFS-e retroativa?**
R: Sim, com data de competencia retroativa. Mas verifique se o municipio tem prazo limite para emissao (alguns tem 30 ou 60 dias apos a competencia).
