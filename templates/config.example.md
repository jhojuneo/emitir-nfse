# Exemplo de Configuracao Preenchida

Copie este bloco para a secao `## Configuracao` do seu SKILL.md e substitua pelos seus dados reais.

```
## Configuracao

empresa_nome: MINHA EMPRESA LTDA
cnpj: 00.000.000/0001-00
municipio: Sao Paulo
uf: SP
regime: Simples Nacional
aliquota: 6.00
codigo_tributacao_nacional: 17.01.01
codigo_complementar_municipal: 17.01.01.001
descricao_padrao: Prestacao de servicos de assessoria e consultoria em marketing digital, incluindo planejamento estrategico, gestao de campanhas publicitarias online e analise de resultados.
caminho_salvamento: ~/Documents/NFS-e/
faturamento_anual_acumulado: 0
```

## Notas sobre cada campo

- **cnpj**: Use o formato com pontos e barras: `00.000.000/0001-00`
- **municipio**: Nome exato como aparece no portal nfse.gov.br
- **regime**: Um dos tres valores exatos: `MEI`, `Simples Nacional`, `Lucro Presumido`
- **aliquota**: Percentual do ISS. MEI: 0 (incluso no DAS). Simples: veja sua faixa. LP: aliquota municipal.
- **codigo_tributacao_nacional**: Codigo de 8 digitos. Ver `references/codigos-tributacao.md`
- **codigo_complementar_municipal**: Especifico do seu municipio. Aparece no portal apos selecionar o nacional.
- **caminho_salvamento**: Caminho onde PDFs e XMLs serao salvos. Use `~` para home directory.
- **faturamento_anual_acumulado**: Total faturado no ano corrente. A skill atualiza automaticamente apos cada emissao. Reset para 0 em janeiro.
