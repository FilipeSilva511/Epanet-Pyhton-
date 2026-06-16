# Epanet-Python

Ferramentas Python para modelagem hidráulica de sistemas de distribuição de água no Distrito Federal. Calibração de rugosidade CHW, alocação de demanda, análise de VRPs e exportação para GIS.

---

## Contexto

Repositório de trabalho para engenharia hidráulica de redes de distribuição do DF. Cada notebook automatiza uma etapa do fluxo de calibração e análise de modelos EPANET (.inp).

**Zonas cobertas:**

| Sigla | Região Administrativa |
|---|---|
| SAM | Samambaia |
| GAM / GAM2 | Gama / Gama 2 |
| SSB | São Sebastião |
| BRZ | Brazlândia |
| PRN / ITP | Paranoá-Itapoã |
| NBN / CND | Núcleo Bandeirante / Candangolândia |
| RCE / RF2 | Recanto das Emas / Riacho Fundo 2 |
| VCP | Vicente Pires |
| AGC / AGQ | Água Quente |

---

## Dependências

```bash
pip install epyt pandas numpy geopandas shapely scikit-learn scipy pyproj openpyxl chardet matplotlib
```

> **Atenção:** Os caminhos nos notebooks estão em formato Windows (`\\`). Em Linux/Mac, ajustar para `/` ou usar `pathlib.Path`.
> Ver [ROADMAP.md](ROADMAP.md) — item 1.2 para solução definitiva.

---

## Estrutura de Diretórios

```
/
├── README.md
├── CLAUDE.md                 ← documentação técnica detalhada para IA e devs
├── ROADMAP.md                ← proposta de evolução técnica
│
├── epanet_na_mao.ipynb       ← toolkit principal (parser manual de .inp)
├── Tratamento Rede.ipynb     ← calibração de rugosidade via epyt
├── Tratamento nós.ipynb      ← demanda, padrões, cavitação via epyt
├── Epanet para gis.ipynb     ← exportação EPANET → Shapefile
├── Cluster.ipynb             ← segmentação ML de tubos (K-Means)
├── Fusão.ipynb               ← mescla de dois modelos .inp
├── Epanet Versão legado.ipynb  ← versão anterior (referência)
├── A maior gambiarra de toda.ipynb  ← merge ad-hoc pontual
│
├── expansao_limpo.inp        ← modelo EPANET de expansão de rede
├── comparativo_final_vrps.csv
│
├── Epanet/                   ← modelos .inp de trabalho (por zona)
├── Epanet Gerados/           ← versões geradas pelos scripts (V1..VN)
├── Epanet Cavitacao/         ← modelos para análise de cavitação
├── Cavitacao/                ← resultados Excel de cavitação (por zona/cenário)
├── Tabelas para calibração/  ← dados de entrada: rede, nós, demanda (Excel)
├── Media horaria/            ← pressões e VRPs hora a hora (Excel)
├── Shape/                    ← Shapefiles GIS de saída (EPSG:31983)
└── Fusão/                    ← .inp fontes para mescla de modelos
```

---

## Notebooks — Qual Usar para Quê

| Tarefa | Notebook |
|---|---|
| Alterar rugosidade CHW no .inp | `epanet_na_mao.ipynb` |
| Alterar demanda e padrões de consumo | `epanet_na_mao.ipynb` |
| Trocar diâmetros de tubos | `epanet_na_mao.ipynb` |
| Mesclar dois modelos .inp | `Fusão.ipynb` |
| Análise hidráulica (pressão, vazão, headloss) | `Tratamento nós.ipynb` |
| Análise de cavitação por VRP | `Tratamento nós.ipynb` |
| Exportar resultados para QGIS/ArcGIS | `Epanet para gis.ipynb` |
| Identificar zona de pressão (ZP) por VRP | `Epanet para gis.ipynb` |
| Agrupar tubos para calibração diferenciada | `Cluster.ipynb` |

---

## Fluxo de Trabalho

```
1. ENTRADA
   Cadastro GIS (Excel/XLS por zona em Tabelas para calibração/)

2. PREPARAÇÃO DO MODELO
   epanet_na_mao.ipynb
   ├── Ler .inp da zona
   ├── Atribuir demanda por nó (merge com cadastro + Thiessen)
   ├── Atribuir padrão de consumo (DMC > RAP > UDA)
   ├── Ajustar rugosidade CHW por grupo (material, diâmetro)
   └── Salvar novo .inp em Epanet Gerados/

3. ANÁLISE HIDRÁULICA
   Tratamento nós.ipynb
   ├── Rodar simulação 24h
   ├── Exportar pressões hora a hora → Media horaria/
   ├── Analisar VRPs (setting, zona de pressão, day-night)
   └── Gerar tabela de cavitação → Cavitacao/

4. EXPORTAÇÃO GIS
   Epanet para gis.ipynb
   ├── Nós com pressões 24h → Shape/{zona}/nos.shp
   ├── Rede com Q, V, HL → Shape/{zona}/rede.shp
   └── VRPs com ZP → Shape/{zona}/vrp.shp

5. ITERAÇÃO
   Comparar resultados com medições de campo
   Ajustar CHW / demanda → voltar ao passo 2
```

---

## Nomenclatura de Objetos

```
VRP.{local}.{número}           → Válvula Redutora de Pressão
DMC.{local}.{número}           → Distrito de Medição e Controle
RAP.{local}.{número}           → Reservatório de Apoio
UDA.{local}.{número}           → Unidade de Distribuição de Água
VZ{n}.{tipo}.{local}.{número}  → Pattern de consumo
```

**Hierarquia de Pattern:**
`DMC` > `RAP` > `UDA` — implementada em `definir_pattern()` no `epanet_na_mao.ipynb`.

---

## Parâmetros Hidráulicos Principais

| Parâmetro | Mínimo | Máximo | Norma |
|---|---|---|---|
| Pressão (mca) | 10 | 50 | ABNT NBR 12218 |
| Velocidade (m/s) | 0,3 | 3,0 | Boas práticas |
| CHW — AMIANTO ≤125mm | 40 | 100 | Calibração |
| CHW — PVC ≤125mm | 100 | 150 | Calibração |
| CHW — PEAD | 120 | 150 | Calibração |

---

## CRS

- **Trabalho:** EPSG:31983 (UTM zona 23S / SIRGAS2000)
- **Geográfico:** EPSG:4674 (SIRGAS2000) — usado em conversões pontuais

---

## Problemas Conhecidos

1. **Paths Windows** — caminhos com `\\` não funcionam em Linux/Mac sem ajuste manual
2. **Encoding dos .inp** — alguns arquivos são UTF-8, outros Latin-1. Usar `analisar_inp()` para detectar antes de rodar
3. **Arquivos `_temp.inp`** — gerados automaticamente no processo de escrita de seções. Não são versões manuais — podem ser ignorados
4. **Funções duplicadas** — mesma função definida em múltiplos notebooks. Ver [ROADMAP.md](ROADMAP.md) para solução
5. **Estado entre células** — notebooks com 100+ células dependem de variáveis definidas em células anteriores. Executar sempre do início

---

## Evolução Planejada

Ver [ROADMAP.md](ROADMAP.md) para proposta técnica completa com fases, justificativas e exemplos de código.

Resumo das próximas ações prioritárias:
1. Criar módulo `epanet_utils.py` consolidando funções duplicadas
2. Substituir paths hardcoded por `pathlib.Path`
3. Implementar log automático de calibração
4. Automatizar loop de calibração via `scipy.optimize`
