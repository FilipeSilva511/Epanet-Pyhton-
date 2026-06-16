# Roadmap Técnico — Epanet-Python

Proposta de evolução do repositório, ordenada por risco e valor.
Perspectiva: engenharia de software + engenharia hidráulica.

---

## Fase 1 — Fundação

> Sem esta fase, qualquer trabalho futuro está sobre base instável.
> Estimativa: 1–2 semanas.

### 1.1 Módulo compartilhado `epanet_utils.py`

**Problema atual:** funções críticas estão copiadas em 3–4 notebooks.
Corrigir bug em um → outros continuam quebrados silenciosamente.

**Solução:** extrair para módulo Python importável.

```
epanet_utils/
├── __init__.py
├── io.py           ← leitura/escrita de .inp (extrair_pipes, extrair_junctions, update_pipes_inp, update_junctions_inp)
├── roughness.py    ← funções de rugosidade CHW (adjust_roughness_with_limit, ajustar_rugosidade_por_grupo)
├── demand.py       ← demanda e perdas (ajustar_perdas_com_pesos, alterar_demandas)
├── patterns.py     ← lógica de padrão de consumo (definir_pattern, definir_pattern_zona)
├── diagnostics.py  ← encoding, sanidade (analisar_inp, encontrar_erros_utf8)
└── gis.py          ← exportação GIS (ler_secao, construção de GeoDataFrames)
```

**Funções a consolidar (duplicadas hoje):**

| Função | Ocorrências atuais |
|---|---|
| `analisar_inp()` | Tratamento Rede, Tratamento nós, Epanet para gis |
| `ler_secao()` / `parse_section()` | epanet_na_mao, Fusão, Epanet para gis |
| `obter_nos_da_valvula()` | 3× dentro de Tratamento nós |
| `ajustar_perdas_com_pesos()` | epanet_na_mao + Tratamento nós |
| bloco análise hidráulica hora a hora | 4 notebooks |

---

### 1.2 Portabilidade de Paths (`pathlib`)

**Problema atual:** todos os caminhos hardcoded com separador Windows:
```python
# não funciona em Linux/Mac:
pd.read_excel('Tabelas para calibração\\Gama 2\\Nos_parciais GAM2.xlsx')
```

**Solução:** substituir por `pathlib.Path`:
```python
from pathlib import Path
BASE = Path(__file__).parent
pd.read_excel(BASE / 'Tabelas para calibração' / 'Gama 2' / 'Nos_parciais GAM2.xlsx')
```

Escopo: todos os 8 notebooks + módulo compartilhado.

---

### 1.3 Definir Notebook Canônico por Tarefa

**Problema atual:** dois paradigmas paralelos (epyt vs parser manual) fazendo coisas similares. Dois engenheiros podem calibrar o mesmo modelo de formas diferentes.

**Solução:** tabela de responsabilidade clara + arquivar legado.

| Tarefa | Notebook Canônico | Ação no Legado |
|---|---|---|
| Alterar rugosidade CHW | `epanet_na_mao.ipynb` | Deprecar seção em `Tratamento Rede` |
| Alterar demanda / padrões | `epanet_na_mao.ipynb` | Deprecar seção em `Tratamento nós` |
| Análise hidráulica (pressão, vazão) | `Tratamento nós.ipynb` | Manter |
| Cavitação | `Tratamento nós.ipynb` | Manter |
| Exportação GIS | `Epanet para gis.ipynb` | Manter |
| Mescla de modelos | `Fusão.ipynb` | Manter versão Mark 2 apenas |
| `Epanet Versão legado.ipynb` | — | **Arquivar** em `_legado/` |
| `A maior gambiarra de toda.ipynb` | — | **Arquivar** ou deletar |

---

## Fase 2 — Rastreabilidade

> Requisito mínimo para trabalho de engenharia com responsabilidade técnica.
> Estimativa: 1 semana.

### 2.1 Log de Calibração

**Problema atual:** impossível saber quais parâmetros geraram GAM2 V34 vs V30.
Isso é problema em auditoria técnica e em qualquer revisão posterior.

**Solução:** arquivo `calibration_log.jsonl` gerado automaticamente ao salvar .inp.

```json
{
  "timestamp": "2026-06-16T14:32:00",
  "zona": "GAM2",
  "arquivo_entrada": "Epanet/Gama 2/GAM2 V20 ajustada.inp",
  "arquivo_saida": "Epanet Gerados/Gama2/GAM2 V21.inp",
  "operacao": "rugosidade",
  "parametros": {
    "chw_delta": -10,
    "chw_min": 40,
    "chw_max": 150,
    "grupos": ["AMIANTO_ate125", "PVC_125_550"]
  },
  "resultado": {
    "nos_fora_norma_pressao": 12,
    "pressao_media_mca": 28.4,
    "pressao_min_mca": 8.1
  },
  "operador": "carla"
}
```

---

### 2.2 Célula de Configuração Padronizada

**Problema atual:** parâmetros espalhados entre células 30–80, impossível auditoria rápida.

**Solução:** primeira célula de cada notebook define tudo:

```python
# ============================================================
# CONFIGURAÇÃO — editar aqui
# ============================================================
from pathlib import Path

ZONA        = "GAM2"
INP_ENTRADA = Path("Epanet/Gama 2/GAM2 V20 ajustada.inp")
INP_SAIDA   = Path("Epanet Gerados/Gama2/GAM2 V21.inp")

# Calibração de rugosidade
CHW_DELTA   = -10
CHW_MIN     = 40
CHW_MAX     = 150

# Demanda
DEMANDA_EXCEL = Path("Tabelas para calibração/Gama 2/Nos_parciais GAM2.xlsx")
# ============================================================
```

---

### 2.3 Documentar Critérios Hidráulicos

**Problema atual:** critérios de aceitação implícitos na cabeça de quem fez.

**Solução:** seção Markdown no topo de cada notebook de análise:

```markdown
## Critérios de Aceitação

| Parâmetro        | Mínimo | Máximo | Norma           |
|------------------|--------|--------|-----------------|
| Pressão (mca)    | 10     | 50     | ABNT NBR 12218  |
| Velocidade (m/s) | 0.3    | 3.0    | Boas práticas   |
| Perda carga (‰)  | —      | 10     | Boas práticas   |
```

---

## Fase 3 — Automação

> Maior ganho de produtividade para o engenheiro.
> Estimativa: 2–3 semanas.

### 3.1 Calibração Automática de Rugosidade

**Problema atual:** `erro_total()` já existe. Falta o loop.

**Solução:** notebook `calibracao_automatica.ipynb`:

```python
from scipy.optimize import minimize
from epanet_utils.roughness import aplicar_e_simular

def erro_total(chw_deltas, grupos, d_ref):
    aplicar_e_simular(d_ref, grupos, chw_deltas)
    pressoes_sim = d_ref.getNodePressure()
    return sum((p_sim - p_obs)**2 for p_sim, p_obs in zip(pressoes_sim, pressoes_observadas))

resultado = minimize(
    erro_total,
    x0=[0] * len(grupos),
    args=(grupos, d),
    method='Nelder-Mead',
    options={'maxiter': 500, 'xatol': 0.5}
)
```

Calibração que hoje leva 1–2 dias de iteração manual → minutos de computação.

---

### 3.2 Conectar Cluster → Calibração

**Problema atual:** K-Means agrupa tubos por (MATERIAL, Ø, Idade) mas resultado não alimenta calibração.

**Solução:** usar clusters como grupos naturais para CHW diferenciado:

```python
# Cluster.ipynb gera:
rede['cluster'] = kmeans.fit_predict(dados_normalizados)
rede[['ID', 'cluster']].to_csv('grupos_calibracao.csv')

# calibracao_automatica.ipynb consome:
grupos = pd.read_csv('grupos_calibracao.csv').groupby('cluster')['ID'].apply(list).to_dict()
```

---

### 3.3 Relatório Consolidado por Zona

**Problema atual:** 4 arquivos Excel separados por zona, abertos manualmente.

**Solução:** notebook `relatorio_zona.ipynb` que gera tudo de uma vez:

```
relatorio_GAM2_2026-06-16/
├── pressoes_24h.xlsx          ← pressão min/med/max por nó
├── vrps_resumo.xlsx           ← setting, vazão, ZP de cada VRP
├── cavitacao_resumo.xlsx      ← velocidade e perda de carga críticas
├── nos_fora_norma.xlsx        ← nós com pressão < 10 ou > 50 mca
├── nos_GAM2.shp               ← shapefile com tudo
└── rede_GAM2.shp
```

---

## Fase 4 — Excelência

> Para quando a base estiver sólida.
> Estimativa: contínuo.

### 4.1 Testes de Sanidade Hidráulica

Assertions automáticas após simulação, antes de salvar .inp:

```python
def verificar_modelo(d, chw_min=10, chw_max=50, vel_max=3.0):
    pressoes = d.getNodePressure()
    velocidades = d.getLinkVelocity()

    nos_baixa_pressao = [n for p, n in zip(pressoes, d.getNodeNameID()) if p < chw_min]
    nos_alta_pressao  = [n for p, n in zip(pressoes, d.getNodeNameID()) if p > chw_max]
    links_alta_vel    = [l for v, l in zip(velocidades, d.getLinkNameID()) if v > vel_max]

    assert not nos_baixa_pressao, f"Nós com pressão < {chw_min} mca: {nos_baixa_pressao}"
    assert not links_alta_vel,    f"Links com v > {vel_max} m/s: {links_alta_vel}"

    return {"nos_fora_pressao": nos_alta_pressao, "links_alta_vel": links_alta_vel}
```

---

### 4.2 Controle de Versão dos Modelos .inp

```gitattributes
# .gitattributes
*.inp  diff=epanet
*.xlsx -diff
*.xls  -diff
```

Commits descritivos por zona:
```
GAM2: calibração V21 — delta CHW -10 em AMIANTO ≤125mm
      pressão mínima: 8.1 → 12.3 mca | nós fora norma: 12 → 3
```

---

### 4.3 Interface de Linha de Comando

Para operações repetitivas em múltiplas zonas:

```bash
python -m epanet_utils calibrar \
    --zona GAM2 \
    --inp "Epanet/Gama 2/GAM2 V20.inp" \
    --saida "Epanet Gerados/Gama2/GAM2 V21.inp" \
    --delta-chw -10 \
    --material AMIANTO \
    --faixa-diametro ate125

python -m epanet_utils relatorio \
    --zona GAM2 \
    --inp "Epanet Gerados/Gama2/GAM2 V21.inp" \
    --saida "relatorios/GAM2/"
```

---

## Resumo Executivo

| Fase | Item | Impacto | Esforço | Prioridade |
|---|---|---|---|---|
| 1 | Módulo compartilhado | Elimina bugs silenciosos entre notebooks | Médio | **CRÍTICO** |
| 1 | Pathlib | Portabilidade total | Baixo | **CRÍTICO** |
| 1 | Notebook canônico | Processo único por tarefa | Baixo | Alta |
| 2 | Log de calibração | Rastreabilidade técnica auditável | Médio | **CRÍTICO** |
| 2 | Célula de configuração | Auditabilidade imediata | Baixo | Alta |
| 2 | Critérios hidráulicos | Rigor de engenharia documentado | Médio | Alta |
| 3 | Calibração automática | 10× mais rápido | Alto | Alta |
| 3 | Cluster → calibração | Precisão diferenciada por grupo | Médio | Média |
| 3 | Relatório consolidado | Elimina trabalho manual repetitivo | Médio | Média |
| 4 | Testes de sanidade | Modelo nunca salvo com violação de norma | Médio | Média |
| 4 | Git para .inp | Histórico técnico real | Baixo | Média |
| 4 | CLI | Escala para múltiplas zonas | Alto | Baixa |

---

*Documento gerado em 2026-06-16. Rever a cada ciclo de projeto.*
