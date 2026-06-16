# CLAUDE.md — Epanet-Python

Repositório de notebooks para modelagem hidráulica de redes de distribuição de água no Distrito Federal. Trabalho central: calibração de rugosidade CHW, alocação de demanda, análise de VRPs e exportação para GIS.

## Zonas Cobertas

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

## Estrutura de Diretórios

```
/
├── *.ipynb                   ← notebooks de processamento (ver abaixo)
├── expansao_limpo.inp        ← modelo EPANET de expansão de rede
├── comparativo_final_vrps.csv
├── Epanet/                   ← modelos de trabalho por zona
├── Epanet Gerados/           ← versões geradas pelos scripts (V1..V34+)
├── Epanet Cavitacao/         ← modelos para análise de cavitação
├── Cavitacao/                ← resultados Excel de cavitação por zona/cenário
├── Tabelas para calibração/  ← dados de entrada (Excel, XLS) por zona
├── Media horaria/            ← pressões e VRPs exportadas hora a hora (Excel)
├── Shape/                    ← Shapefiles GIS de saída (EPSG:31983)
└── Fusão/                    ← arquivos .inp fontes para mescla de modelos
```

## Notebooks

### `epanet_na_mao.ipynb` — Toolkit Principal (parser manual)
Não usa epyt. Lê e reescreve .inp como texto puro. Mais portável e controlável.

**Funções principais:**
- `extrair_pipes()` / `extrair_junctions()` — lê seções do .inp
- `update_pipes_inp()` / `update_junctions_inp()` — reescreve seções no .inp
- `adjust_roughness_with_limit(df, ids, delta, limit)` — ajusta CHW com teto/piso
- `ajustar_perdas_com_pesos(perda_total, df)` — distribui perda por Join_Count
- `definir_pattern(row)` / `definir_pattern_zona(row)` — atribui padrão por DMC>RAP>UDA
- `encontrar_erros_utf8(caminho)` — diagnóstico de encoding

### `Tratamento Rede.ipynb` — Rugosidade via epyt
Calibra CHW usando engine EPANET. Agrupa tubos por (material, faixa_diâmetro) e aplica delta CHW dentro de limites.

**Funções principais:**
- `ajustar_rugosidade(d, dict_ids, lista_ids, ajuste, rug_min, rug_max)`
- `ajustar_rugosidade_por_grupo(d, dict_ids, ids_por_grupo, faixas, ajuste_geral)`

### `Tratamento nós.ipynb` — Demanda, Padrões, Cavitação via epyt
O mais amplo (242 células). Cobre análise de VRPs, pressões horárias, cavitação, demanda, padrões e redução.

**Função de calibração:**
- `erro_total()` — função objetivo para scipy.optimize.minimize

### `Epanet para gis.ipynb` — EPANET → Shapefile
Exporta resultados de simulação para GIS.
- Nós (Point): pressão 24h + zona de pressão (ZP)
- Rede (LineString): Q, V, HL horárias
- VRPs (Point): localização com setting

**Algoritmo ZP:** fecha cada VRP sequencialmente → identifica nós afetados → atribui à VRP de menor área.

### `Cluster.ipynb` — Segmentação ML de Tubos
K-Means em (DIAMETRO, MATERIAL, CHW, Idade, UDA). Elbow + Silhouette. Desconectado do fluxo de calibração.

### `Fusão.ipynb` — Mescla de Modelos
`merge_inps(arq1, arq2, saida)` — combina dois .inp preservando todas as seções.

### `Epanet Versão legado.ipynb`
Predecessor de Tratamento Rede + Tratamento nós. Mantido para referência.

### `A maior gambiarra de toda.ipynb`
Merge ad-hoc GIS → EPANET via pandas. 4 células.

## Nomenclatura de Objetos EPANET

```
VRP.{local}.{número}          → Válvula Redutora de Pressão
DMC.{local}.{número}          → Distrito de Medição e Controle
RAP.{local}.{número}          → Reservatório de Apoio
UDA.{local}.{número}          → Unidade de Distribuição de Água
VZ{n}.{tipo}.{local}.{número} → Padrão de consumo (Pattern)
```

**Hierarquia de Pattern:** DMC > RAP/UDA (lógica em `definir_pattern()`)

## Dependências Python

```
epyt            # interface EPANET engine
pandas          # manipulação de dados
numpy           # cálculos
geopandas       # exportação Shapefile
shapely         # geometrias Point/LineString
scikit-learn    # K-Means, StandardScaler
scipy           # optimize.minimize (calibração)
pyproj          # transformação CRS
openpyxl        # leitura/escrita Excel
chardet         # detecção de encoding
matplotlib      # gráficos
```

## CRS

- Entrada/saída GIS: **EPSG:31983** (UTM zona 23S SIRGAS2000)
- Conversão geográfica quando necessário: EPSG:31983 → EPSG:4674

## Problemas Conhecidos

1. **Paths Windows hardcoded** (`\\`) — não rodam diretamente no Linux sem adaptação
2. **Encoding inconsistente** — .inp podem ser UTF-8 ou Latin-1; usar `chardet` para detectar
3. **Arquivos `_temp.inp`** — gerados automaticamente no processo de escrita, não são versões manuais
4. **Funções duplicadas** — `analisar_inp()`, `ler_secao()`, etc. redefinidas em múltiplos notebooks
5. **Estado implícito** — `df_junctions` e outros DataFrames modificados ao longo de muitas células sem encapsulamento

## Fluxo Geral de Trabalho

```
Cadastro GIS (Excel/XLS)
        ↓
epanet_na_mao.ipynb   [rugosidade, demanda, padrões → .inp]
        ↓
Epanet Gerados/       [versões versionadas V1..VN]
        ↓
Tratamento nós.ipynb  [análise hidráulica, cavitação, ajuste fino]
        ↓
Epanet para gis.ipynb [export → Shape/]
Tratamento nós.ipynb  [export → Media horaria/, Cavitacao/]
```
