# Captação de Leads via Dados Abertos de CNPJ

Pipeline de ETL (Extract, Transform, Load) que extrai, filtra e organiza empresas de um setor e região específicos, a partir da base pública de CNPJ da Receita Federal, com o objetivo de gerar listas de leads B2B qualificados de forma gratuita e escalável.

## Objetivo

Este projeto nasceu de uma necessidade real: gerar uma lista de empresas-alvo (leads) para um cliente de serviços B2B, seguindo um Perfil de Cliente Ideal (ICP) definido por ele — localização, setor de atuação e porte financeiro.

O objetivo técnico foi validar se era possível construir esse processo **sem depender de ferramentas pagas de prospecção** (como Clay, Apollo, Econodata), usando apenas dados públicos oficiais do governo brasileiro, tratados via Python.

## Contexto e motivação

A primeira abordagem testada foi capturar empresas via Google Maps e tentar confirmar CNPJ/dados oficiais uma por uma, por nome — esse método se mostrou pouco confiável (nomes ambíguos geravam correspondências erradas) e não escalável manualmente.

A solução foi inverter a lógica: em vez de partir de uma fonte incerta (Maps) e tentar validar contra a Receita, o projeto passou a **partir direto da base oficial da Receita Federal**, aplicando os filtros do ICP (localização, setor, porte) diretamente sobre o dado primário — eliminando o problema de correspondência ambígua.

## Sobre a fonte de dados

Os **Dados Abertos do CNPJ** são disponibilizados publicamente pela Receita Federal em `dadosabertos.rfb.gov.br/CNPJ/dados_abertos_cnpj/`, com atualização mensal. É a base cadastral completa de todas as empresas registradas no Brasil (ativas e inativas), dividida em arquivos temáticos:

- **Estabelecimentos** — endereço, município, CNAE (atividade econômica), situação cadastral, telefone
- **Empresas** — razão social, natureza jurídica, porte, capital social
- **Sócios** — nome dos sócios/administradores de cada empresa
- Tabelas de referência (`Cnaes`, `Municipios`, etc.) — traduzem os códigos numéricos usados nos arquivos principais

Cada categoria é dividida em 10 arquivos (`0` a `9`) por um critério técnico interno (hash), **não por região** — por isso o pipeline sempre processa as 10 partes de cada categoria, mesmo filtrando apenas 3 cidades.

## Critérios de filtro aplicados (ICP do cliente)

| Critério | Valor |
|---|---|
| Municípios | São Paulo capital (`7107`), Jundiaí (`6619`), São José dos Campos (`7099`) |
| Setor (CNAE) | `6201500`, `6201501`, `6202300`, `6203100`, `6204000` — desenvolvimento de software e consultoria em TI |
| Situação cadastral | Ativa (`02`) |
| Porte | ME + EPP (`01` + `03`) |

## Metodologia (pipeline)

1. **Extract** — download dos arquivos `.zip` mensais da Receita Federal
2. **Transform** — leitura em blocos (*chunks*) para não sobrecarregar a memória, com filtro aplicado em cada etapa:
   - `Estabelecimentos`: filtro por município + CNAE + situação ativa → gera a lista-base de `cnpj_basico`
   - `Empresas`: cruzamento pelos `cnpj_basico` já filtrados, com filtro adicional de porte
   - `Sócios`: cruzamento pelos mesmos `cnpj_basico`, trazendo o nome do decisor
3. **Load** — consolidação dos resultados em CSVs tratados, prontos para consulta/análise

## Estrutura do projeto

```
captacao-leads-cnpj/
├── notebooks/          # notebook Jupyter com todo o processamento
├── data/                # dados brutos e processados (não versionado)
├── docs/                # documentação do cliente (não versionado, confidencial)
├── sql/                 # scripts de criação de banco (em desenvolvimento)
└── README.md
```

## Tecnologias utilizadas

- Python (pandas, zipfile)
- Jupyter Notebook
- Dados Abertos CNPJ — Receita Federal

## Resultados parciais

- Estabelecimentos filtrados (município + CNAE + situação ativa): **57.097**
- Empresas filtradas (+ porte ME/EPP): **50.844**

## Pontos de atenção identificados

- O campo `porte_empresa` sozinho não distingue microempreendedor individual/empresário individual de empresas formalmente constituídas (LTDA, S/A) — foram encontradas razões sociais que são nomes de pessoa física classificadas como porte "ME". Está em andamento o cruzamento com o campo `natureza_juridica` para refinar esse filtro.
- O campo de e-mail na base da Receita costuma vir vazio — não é uma fonte confiável para esse dado.
- A base não informa número de funcionários — apenas porte declarado (proxy indireto de faturamento).

## Confidencialidade

Os critérios de filtro foram definidos em conjunto com um cliente real, mas o nome do cliente e o documento de ICP original não são versionados neste repositório (`docs/` está no `.gitignore`) por questão de confidencialidade comercial.

## Autor

Sérgio Fonseca Leite Junior — [github.com/engsergioleite](https://github.com/engsergioleite)
Projeto desenvolvido durante transição de carreira para Análise de Dados.
