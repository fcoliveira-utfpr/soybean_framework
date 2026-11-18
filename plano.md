# Planejamento experimental — AgroWDN-BR
## Previsão integrada de preço, produtividade e margem da soja no Brasil com arquitetura Wide-and-Deep e fusão de dados multifonte

---

## 1. Objetivo

Adaptar o framework Wide-and-Deep (AgroWDN) de Ghonge e Kulkarni (2026) ao contexto brasileiro e estendê-lo para três tarefas acopladas:

1. previsão do preço da soja ao produtor em horizontes de 1, 3 e 6 meses;
2. previsão da produtividade municipal ou regional;
3. estimativa probabilística da margem líquida (R$ ha⁻¹), unindo as duas previsões aos custos de produção.

**Escala inicial:** Paraná, por núcleo regional da SEAB/DERAL e por município.
**Escala de expansão:** principais estados produtores (MT, PR, RS, GO, MS).

---

## 2. Lacuna e justificativa

- O AgroWDN foi desenvolvido para o mercado indiano, que tem preços domésticos regulados e pouca ligação com bolsas internacionais. No Brasil, o preço ao produtor depende de três componentes: CBOT, câmbio e prêmio de porto (*basis*). Essa estrutura exige outro desenho de variáveis.
- Revisões recentes apontam a falta de modelos de previsão do preço da soja voltados ao Brasil, apesar de o país ser o maior produtor e exportador mundial (Mello; Kumar; Nascimento).
- Zheng et al. (2026) mostram que a rentabilidade, e não apenas a produtividade, limita a produção de soja. Ainda assim, a maioria dos estudos modela produtividade e preço separadamente.
- Já existe no grupo um pipeline de previsão de produtividade com dependência espaço-temporal e intervalos de previsão (`soybean_yield_forecasts`), e também um pipeline de margem com custos CONAB (`aquacrop_ml`). Este trabalho conecta os dois.

---

## 3. Perguntas e hipóteses

| # | Pergunta | Hipótese |
|---|---|---|
| Q1 | A arquitetura Wide-and-Deep supera *baselines* estatísticos e de ML no preço ao produtor no Brasil? | H1: o AgroWDN-BR reduz o RMSE em pelo menos 10% frente ao *random walk* e ao SARIMAX em h ≥ 3 meses (teste de Diebold-Mariano, p < 0,05). |
| Q2 | Variáveis agrometeorológicas por estádio fenológico agregam informação ao preço além das variáveis de mercado? | H2: o bloco de clima e oferta melhora o desempenho em safras com anomalia climática (2008/09, 2011/12, 2019/20, 2021/22, 2022/23 no RS). |
| Q3 | O acoplamento produtividade → oferta → preço → margem melhora a estimativa de risco econômico em relação a modelos isolados? | H3: o modelo acoplado produz intervalos de margem mais bem calibrados (cobertura nominal de 90% ± 3%) e um CRPS menor. |
| Q4 | Quais fatores determinam o preço e a margem em cada fase da safra? | H4: câmbio e CBOT dominam na entressafra; clima e estimativas da CONAB ganham peso entre R1 e R5. |

---

## 4. Área, período e resolução temporal

- **Espacial:** 22 núcleos regionais da SEAB/DERAL (PR). A agregação municipal vem do IBGE/PAM, e a máscara de soja do MapBiomas pondera as variáveis climáticas.
- **Temporal:** 2004–2025, com ajuste conforme a série mais curta disponível.
- **Frequência do modelo:** mensal para preço e margem e anual (por safra) para produtividade. Há ainda uma versão semanal exploratória com CEPEA e B3.
- **Calendário de safra:** a safra vai de setembro a agosto, e as janelas fenológicas são definidas pela data de semeadura do ZARC e pela fenologia calibrada (ver seção 6.3).

---

## 5. Variáveis-alvo

| Alvo | Fonte | Frequência | Observação |
|---|---|---|---|
| Preço recebido pelo produtor (R$ sc⁻¹ 60 kg) | SEAB/DERAL | semanal/mensal, por núcleo | alvo principal |
| Indicador CEPEA/ESALQ Paranaguá | CEPEA | diário → mensal | alvo secundário e variável de referência |
| Produtividade (kg ha⁻¹) | IBGE/PAM; CONAB | anual | reaproveita `soybean_yield_forecasts` |
| Margem líquida (R$ ha⁻¹) | derivada | por safra | receita − COE/COT (custos CONAB) |

**Definição da margem:**

MB = Ŷ × P̂ − COE      MLiq = Ŷ × P̂ − COT

Ŷ é a produtividade prevista, P̂ o preço previsto na janela de comercialização, e COE/COT são os custos operacional efetivo e total da CONAB, pela mesma abordagem de subtração da Renda de Fatores usada no `aquacrop_ml`.

---

## 6. Dados de entrada

### 6.1 Mercado e macroeconomia (bloco Deep — séries temporais)

| Variável | Fonte | Acesso |
|---|---|---|
| Futuro de soja CBOT (1º vencimento, *spread* entre vencimentos) | CME Group | histórico público / API |
| Futuro de soja B3 (SJC) | B3 | dados históricos |
| Câmbio PTAX (R$/US$) | BCB/SGS (série 1) | API `python-bcb` |
| Prêmio de exportação em Paranaguá (*basis*) | CEPEA / corretoras | mensal |
| Selic, IPCA (deflator) | BCB/SGS | API |
| Diesel ao produtor | ANP | mensal |
| Preço de fertilizantes (KCl, MAP) | CONAB / IMEA | mensal |
| Frete Cascavel–Paranaguá | ESALQ-LOG (SIFRECA) | mensal |
| Farelo e óleo de soja (*crush margin*) | CBOT / CEPEA | diário → mensal |

### 6.2 Oferta, demanda e comércio (bloco Deep + Wide)

| Variável | Fonte |
|---|---|
| Estimativas mensais de safra (área, produção, produtividade) | CONAB (levantamentos de safra) |
| Estoques finais Brasil/EUA/mundo | CONAB; USDA WASDE/PSD |
| Exportações de soja em grão (volume, US$ FOB, destino China) | ComexStat (NCM 1201) |
| Importações chinesas de soja | USDA FAS / GACC |
| Line-ups portuários | agências marítimas (opcional) |
| Estimativa de produção dos EUA e da Argentina | USDA WASDE |

### 6.3 Biofísicas e agrometeorológicas (bloco Deep, agregadas por janela fenológica)

| Variável | Fonte / ferramenta | Repositório |
|---|---|---|
| Precipitação, Tmax, Tmin, Rs, UR, vento diários | asset GEE `inmet_pr_diario`; grade Xavier (BR-DWGD); NASA POWER | `agrometeorologiapy` |
| ETo (FAO-56 PM) e balanço hídrico | `agrometeorologiapy` | `agrometeorologiapy` |
| ISNA (índice de satisfação das necessidades de água) por decêndio | balanço hídrico sequencial | `aquacrop_ml` |
| SPI-1, SPI-3 | grade de precipitação | `spi-enso` |
| Índices de El Niño/La Niña (ONI, SOI) | NOAA CPC | `spi-enso` |
| GPP (MOD17A2H), NDVI/EVI (MOD13Q1) mascarados por soja | GEE + MapBiomas | — |
| Graus-dia e dias com Tmax > 32 °C em R1–R5 | derivado | `agrometeorologiapy` |
| Duração das fases (semeadura–R1, R1–R5, R5–R7) | fenologia calibrada (CROPGRO-Soybean, emulador bayesiano) | calibração bayesiana |

**Janelas fenológicas:** as variáveis climáticas devem ser agregadas pelas fases do ciclo estimadas pelo modelo fenológico calibrado, e não por mês do calendário. É a contribuição agronômica que falta no AgroWDN original.

### 6.4 Variáveis estáticas e categóricas (bloco Wide)

| Variável | Fonte |
|---|---|
| Núcleo regional / município (*embedding* ou *one-hot*) | IBGE / SEAB |
| Classe climática Köppen e Thornthwaite | `climas_brasil` |
| Textura e água disponível do solo | MapBiomas Solo / SoilGrids |
| Altitude | SRTM |
| Mês do ano e fase da safra (plantio, crescimento, colheita, entressafra) | derivado |
| Fase ENSO (El Niño, Neutro, La Niña) | NOAA |
| *Dummies* de eventos: guerra comercial EUA–China (2018–19), COVID (2020), conflito Rússia–Ucrânia (2022), choques tarifários (2025) | derivado |

**Produtos cruzados no bloco Wide** (*cross-product transformations*, como no Wide & Deep original):
- núcleo × mês;
- fase ENSO × fase da safra;
- classe Köppen × janela fenológica.

---

## 7. Arquitetura proposta (AgroWDN-BR)

```
                 ┌──────────────────────────────┐
 Wide input ───► │ Linear + cross-products      │ ──┐
 (categóricas,   └──────────────────────────────┘   │
  estáticas)                                         │
                                                     ▼
 Deep input ───► ┌──────────────────────────────┐   ┌─────────┐   ┌────────────────────┐
 (séries 12–24   │ Encoder temporal             │──►│ Fusão   │──►│ Cabeças multitarefa│
  meses: mercado,│ (LSTM bidirecional ou TFT)   │   │ (concat │   │ • preço h=1,3,6    │
  clima, oferta) │ + atenção temporal           │   │ + MLP)  │   │ • produtividade    │
                 └──────────────────────────────┘   └─────────┘   │ • quantis (0.05,   │
                                                                  │   0.5, 0.95)       │
                                                                  └────────────────────┘
```

### 7.1 Variantes a comparar

| Variante | Descrição |
|---|---|
| **V1 — AgroWDN fiel** | réplica do desenho de [1], só com variáveis de mercado, clima e comércio |
| **V2 — AgroWDN-BR** | V1 + variáveis fenológicas, ISNA, *basis* e câmbio decompostos |
| **V3 — cascata** | produtividade (`soybean_yield_forecasts`) → oferta agregada → preço → margem (Monte Carlo) |
| **V4 — multitarefa conjunta** | cabeças compartilhadas para preço e produtividade, com perda ponderada |
| **V5 — guiada por conhecimento** | V4 com restrição física na cabeça de produtividade, que não pode passar da produtividade limitada por água estimada via ISNA/AquaCrop (ligação com a proposta de *physics-guided DL*) |

### 7.2 Função de perda

- **Pontual:** MSE sobre os alvos padronizados.
- **Probabilística:** *pinball loss* para os quantis 0,05, 0,50 e 0,95.
- **Multitarefa:** L = λ₁·L_preço + λ₂·L_prod + λ₃·L_quantil, com λ ajustado por incerteza homoscedástica (Kendall et al., 2018) ou por busca.

### 7.3 Hiperparâmetros (busca com Optuna)

- Janela de entrada: 6, 12, 24 meses
- *Embedding* das categóricas: 4–16
- LSTM/TFT: 1–3 camadas, 32–256 unidades, *dropout* 0,1–0,4
- Taxa de aprendizado: 1e-4 a 1e-2 (AdamW), *early stopping* na validação temporal

> Confirmar no texto completo de [1] a configuração original (camadas, ativação, otimizador) para a variante V1.

---

## 8. Engenharia de atributos

1. **Defasagens:** 1, 2, 3, 6 e 12 meses para preço, câmbio, CBOT e exportações.
2. **Estatísticas móveis:** média e desvio-padrão de 3 e 6 meses, mais a volatilidade realizada da CBOT.
3. **Decomposição do preço:**
   P_BR ≈ (CBOT + prêmio) × câmbio × 2,2046 × 0,6
   Usar os componentes separadamente e o resíduo (*basis* interno = DERAL − paridade Paranaguá − frete).
4. **Anomalias climáticas** em relação à normal 1991–2020 por janela fenológica.
5. **Revisões de estimativa:** variação entre levantamentos consecutivos da CONAB (Δ produção), como "surpresa de oferta".
6. **Variáveis de calendário:** meses até a colheita e fração da safra já comercializada (quando houver dado).
7. **Controle de vazamento de informação (crítico):** cada variável entra só a partir da data em que fica pública. A PAM sai cerca de 10 meses após a safra, e a CONAB publica em datas fixas. Montar uma base com as versões publicadas em cada data (*vintage*).

---

## 9. Validação

### 9.1 Esquema de validação (correção em relação ao plano anterior)

O *k-fold* aleatório **não é adequado** para séries temporais, porque mistura passado e futuro e infla as métricas. Usar:

| Esquema | Finalidade |
|---|---|
| **Origem móvel com janela expansível** (*rolling-origin*), passo de 1 mês, teste a partir de 2016 | desempenho temporal |
| **Blocked/purged CV com embargo** de *h* meses entre treino e teste | otimização de hiperparâmetros sem vazamento |
| **Leave-one-season-out** (safra inteira fora) | produtividade e margem |
| **Validação espacial em blocos** (deixar núcleos regionais inteiros de fora) | generalização espacial |
| **Teste de estresse** em safras extremas (2011/12, 2019/20, 2021/22) e choques de mercado (2020–2022) | robustez |

### 9.2 Baselines

*Random walk* (ingênuo) · média sazonal · SARIMA/SARIMAX · ETS · VAR (CBOT, câmbio, preço) · Random Forest · XGBoost/LightGBM · LSTM puro · TFT · N-HiTS · AgroWDN (V1).

### 9.3 Métricas

| Tipo | Métricas |
|---|---|
| Erro pontual | RMSE, MAE, MAPE, sMAPE, **MASE** (escala frente ao *random walk*) |
| Ajuste | R² (apenas complementar) |
| Direção | acurácia direcional (acerto do sinal de variação) |
| Comparação estatística | teste de **Diebold-Mariano** (com correção de Harvey); *Model Confidence Set* |
| Probabilística | cobertura empírica dos intervalos, largura média, **CRPS**, *pinball loss* |
| Econômica | valor de decisão (ver seção 11.3) |

---

## 10. Interpretabilidade (XAI)

1. **SHAP:** TreeSHAP para modelos de *ensemble* e DeepSHAP/GradientSHAP para o AgroWDN-BR, com importância global e por fase da safra.
2. **Pesos do bloco Wide:** interpretação direta dos coeficientes lineares e dos produtos cruzados.
3. **Atenção temporal** (TFT/LSTM com atenção): quais meses da janela pesam na previsão.
4. **LIME:** explicações locais em eventos extremos (por exemplo, o pico de preço de 2020/21).
5. **Ablação por bloco de dados:**
   mercado → + câmbio/*basis* → + oferta (CONAB/USDA) → + comércio (ComexStat) → + clima mensal → + clima por fenologia → + sensoriamento remoto.
   Comparar o ΔRMSE e o ΔCRPS em cada etapa.

---

## 11. Avaliação econômica

### 11.1 Margem probabilística

- Monte Carlo (n = 10.000) amostrando dos quantis previstos de Ŷ e P̂, mantendo a correlação empírica entre resíduos de produtividade e preço (cópula gaussiana ou t).
- Resultados por núcleo e por safra: E[MLiq], P(MLiq < 0), VaR₅% e CVaR₅% da margem.

### 11.2 Cenários

- Fases ENSO (El Niño, Neutro, La Niña), com base na relação SPI–ENSO do grupo.
- Câmbio (±15%) e CBOT (percentis 10/50/90).
- Custos de insumos (fertilizantes +30%, cenário 2022).

### 11.3 Valor econômico da previsão (*backtest* de comercialização)

Comparar, em 2016–2025, estratégias de venda do produtor representativo:
- (a) venda integral na colheita;
- (b) venda escalonada fixa (1/3 em março, maio e julho);
- (c) venda orientada pelo modelo (vender quando P̂(t+h) − custo de armazenagem < P(t)).

Métrica: receita média e desvio-padrão em R$ ha⁻¹ e o índice de Sharpe da estratégia.

### 11.4 Integração com o framework multicritério

Substituir o preço fixo no TOPSIS do projeto multicritério da soja no Paraná pela distribuição prevista de preço e margem, transformando a escolha da data de semeadura em uma decisão sob risco.

---

## 12. Integração com os repositórios existentes

| Repositório | Papel neste experimento |
|---|---|
| `agrometeorologiapy` | ETo, balanço hídrico, ISNA, graus-dia |
| `soybean_yield_forecasts` | módulo de produtividade (V3) e intervalos de previsão |
| `aquacrop_ml` | rotina de margem com custos CONAB e TOPSIS |
| `climas_brasil` | classes Köppen e Thornthwaite como variáveis estáticas do bloco Wide |
| asset GEE `inmet_pr_diario` | clima diário por estação → agregação por núcleo |
| calibração bayesiana da fenologia | duração das fases para definir as janelas |
| novo repositório `agrowdn_br` | coleta, *vintage database*, modelos, validação, XAI |

**Estrutura sugerida do novo repositório:**

```
agrowdn_br/
├── data/
│   ├── raw/            # CEPEA, DERAL, B3, CBOT, BCB, CONAB, ComexStat, USDA
│   ├── vintage/        # versões das variáveis por data de publicação
│   └── processed/
├── src/
│   ├── ingest/         # um módulo por fonte
│   ├── features/       # lags, decomposição do preço, janelas fenológicas
│   ├── models/         # baselines, wide_deep.py, tft.py, multitask.py
│   ├── validation/     # rolling_origin.py, purged_cv.py, dm_test.py
│   ├── xai/            # shap, lime, ablation
│   └── economics/      # monte_carlo_margin.py, backtest_marketing.py
├── notebooks/          # Colab
└── configs/            # YAML de experimentos (Optuna)
```

---

## 13. Cronograma (12 meses)

| Mês | Atividade |
|---|---|
| 1–2 | Coleta e montagem da base com controle de datas de publicação; harmonização espacial (núcleos e municípios) |
| 3 | Engenharia de atributos; janelas fenológicas; análise exploratória (estacionariedade, quebras estruturais — Bai-Perron) |
| 4 | Baselines estatísticos e de ML |
| 5–6 | AgroWDN V1 (réplica) e V2 (BR); otimização com Optuna |
| 7 | Variantes V3–V5 (cascata, multitarefa, guiada por conhecimento) |
| 8 | Validação completa, testes DM, estresse em safras extremas |
| 9 | XAI e ablação |
| 10 | Margem probabilística, cenários e *backtest* de comercialização |
| 11–12 | Redação dos artigos e publicação do código e dos dados |

---

## 14. Produtos previstos

1. **Artigo 1:** previsão do preço da soja ao produtor no Brasil com Wide-and-Deep e fusão multifonte (lacuna explícita na literatura). Periódicos-alvo: *Computers and Electronics in Agriculture*, *International Journal of Forecasting*, *Agricultural Economics*.
2. **Artigo 2:** acoplamento produtividade–preço–margem com previsão probabilística e valor econômico. Periódicos-alvo: *Agricultural Systems*, *European Journal of Agronomy*.
3. **Base de dados aberta:** séries harmonizadas por núcleo regional, com as versões por data de publicação (*data paper*).
4. TCC e iniciação científica derivados: módulos de ingestão, *backtest* de comercialização.

---

## 15. Riscos e limitações

| Risco | Mitigação |
|---|---|
| Séries anuais curtas para produtividade (~20 safras) | modelagem em painel (núcleos × safras), *transfer learning* entre estados, regularização forte |
| Quebras estruturais (2020–2022) | janelas móveis, teste de Bai-Perron, *dummies* de regime |
| Vazamento de informação | base com versões por data de publicação; embargo na validação |
| Licenciamento de dados (CEPEA, B3) | verificar termos de uso; usar DERAL e BCB como base pública |
| Sobreajuste de modelos profundos com poucos dados mensais (~250 meses) | comparar com *ensembles* rasos; reportar o MASE frente ao *random walk* |
| Preço é processo quase passeio aleatório no curto prazo | foco em h ≥ 3 meses e em previsão probabilística, não só pontual |

---

## 16. Referências adicionais a incorporar

CHENG, H.-T. et al. Wide & Deep learning for recommender systems. In: WORKSHOP ON DEEP LEARNING FOR RECOMMENDER SYSTEMS, 1., 2016. **Proceedings** [...]. New York: ACM, 2016. p. 7–10. DOI: 10.1145/2988450.2988454.

LIM, B. et al. Temporal Fusion Transformers for interpretable multi-horizon time series forecasting. **International Journal of Forecasting**, v. 37, n. 4, p. 1748–1764, 2021. DOI: 10.1016/j.ijforecast.2021.03.012.

MELLO, F. D. da C.; KUMAR, P.; NASCIMENTO, E. G. S. Advancements in soybean price forecasting: impact of AI and critical research gaps in global markets. [Verificar periódico, volume e DOI.]

DIEBOLD, F. X.; MARIANO, R. S. Comparing predictive accuracy. **Journal of Business & Economic Statistics**, v. 13, n. 3, p. 253–263, 1995.

KENDALL, A.; GAL, Y.; CIPOLLA, R. Multi-task learning using uncertainty to weigh losses for scene geometry and semantics. In: IEEE/CVF CVPR, 2018. p. 7482–7491.
