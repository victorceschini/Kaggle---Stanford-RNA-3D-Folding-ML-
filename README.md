# Relatório técnico: RNA 3D Structure Prediction — Kaggle Competition

## Grupo:  
- **Integrante 1:** Victor Gabriel Ceschini Menezes

## Resumo do projeto 
### Objetivo
Prever as coordenadas tridimensionais (x, y, z) de cada resíduo de moléculas de RNA a partir da sequência primária, gerando submissões compatíveis com o formato exigido pela competição.

### Abordagem
Pipeline híbrido que combina transferência de template, engenharia de features por resíduo, modelos de regressão multi‑output (Random Forest e XGBoost) e refinamento geométrico para produzir múltiplas candidatas por alvo e agregar previsões em ```submission.csv```.

**Link para a competição**  
Kaggle: **https://www.kaggle.com/competitions/stanford-rna-3d-folding-2**

---

## Descrição do problema
Prever estruturas 3D de RNA é um problema de regressão multivariada com desafios práticos: variação de comprimento entre sequências, quebras de cadeia, lacunas em templates e necessidade de agregar múltiplas predições por resíduo. A métrica principal adotada foi RMSE sobre coordenadas.

---

## Datasets utilizados
Biopython: **https://www.kaggle.com/datasets/kami1976/biopython-cp312**

---

## Dados e Análise Exploratória
### Fontes de dados
- Sequências e coordenadas de treinamento (templates);
- Sequências de teste e mapa de segmentos de cadeia;
- Recursos auxiliares (índices de k‑mers, metadados de templates).

### Análises Realizadas
- Distribuição de comprimentos: histograma e estatísticas descritivas; identificação de caudas longas;
- Segmentação de cadeias: frequência de quebras e distribuição de comprimentos de segmentos;
- Estatísticas de coordenadas: média, desvio padrão e amplitude por eixo; detecção de outliers;
- Cobertura por templates: proporção de alvos com templates de alta similaridade;
- Dados faltantes: contagem de posições com coordenadas ausentes.

---

## Metodologia e Implementação
### Visão geral do pipeline
- Recuperação de templates: pré‑filtro por comprimento e score rápido (aligner ou k‑mer) para obter candidatos.
- Adaptação de template: mapeamento vetorizado das coordenadas do template para a sequência consulta via alinhamento; interpolação para preencher lacunas.
- Geração de candidatos: até 5 candidatos por alvo aplicando transformações controladas (ruído, hinge, jitter, smooth wiggle) para diversidade.
- Predição de resíduos: features por resíduo (posição, posição normalizada, janela one‑hot, coordenadas do template) e predição de (dx, dy, dz) com modelos multi‑output.
- Refinamento geométrico: aplicação de restrições vetorizadas para ajustar ligações i,i+1, distâncias i,i+2, suavização Laplaciana e auto‑evitação leve.
- Agregação e exportação: montar submission.csv com até 5 candidatos por resíduo; preservar NaN até conversão final se exigida.

### Engenharia de features
- Posicionais: pos, pos_norm, seq_len.
- Contexto local: janela one‑hot ao redor da posição.
- Âncoras: coordenadas adaptadas do template (ax, ay, az).
- Similaridade: score normalizado do alinhamento ou Jaccard de k‑mers.

### Modelos e validação
* Modelos:
  * RandomForestRegressor (multi‑output nativo).
  * XGBoost XGBRegressor encapsulado em MultiOutputRegressor.
  * Ensemble: média simples; opção de média ponderada por RMSE de CV.

- Validação: GroupKFold por target_id para evitar vazamento entre resíduos do mesmo alvo.
- Métrica: RMSE por fold e média de folds.
- Pré‑processamento: remoção de linhas com NaN/inf antes do treino usando np.isfinite.

### Robustez e reprodutibilidade
- Salvamento de artefatos: joblib para modelos e feature_cols em ordem determinística (sorted).
- Metadata: model_metadata com window, feature_cols_len, versões de pacotes, seed e flag preserve_nan.
- Fallbacks: k‑mer scoring quando Biopython/aligner não disponível; straight‑line per chain quando nenhum template é encontrado.

### Justificativas para o uso dos modelos
Escolhi modelos baseados em árvores (Random Forest e XGBoost) porque combinam robustez a ruído, capacidade de modelar relações não lineares entre features heterogêneas e facilidade de integração em um fluxo de trabalho multi‑output; o ensemble (média simples ou ponderada) reduz variância e melhora estabilidade das previsões. Ademais, tais modelos geralmente trazem bons resultados nas competições do Kaggle.

---

## Relato, Resultados e Experiência na Competição
### Implementação
Usei um notebook público com bom desempenho como referência para entender a lógica geral da solução e as etapas específicas de predição de estruturas de RNA. A partir desse notebook base, adaptei e reescrevi as funções necessárias para mapeamento de templates, alinhamento e interpolação de coordenadas, e implementei a lógica dos modelos de aprendizado de máquina (Random Forest e XGBoost) — que não estavam presentes na solução original.

### Desempenho de validação
- RF CV mean RMSE: 30.9320
- XGB CV mean RMSE: 37.4067
- Ensemble CV mean RMSE: 32.1382

### Submissões e leaderboard
Número de submissões: 2
Best score: 0.286
Melhor posição pública: 633 (data: 16/02/26)

<img width="1393" height="234" alt="image" src="https://github.com/user-attachments/assets/cfb8bb34-faa6-4004-9d36-d75b833b848a" />

<img width="1523" height="670" alt="image" src="https://github.com/user-attachments/assets/e87f56d2-92fb-48b4-beac-2a4d6317f961" />

### Desafios enfrentados
- Compatibilidade de pacotes com a versão do Python do kernel;
- Tentar garantir uma ordem determinística de feature_cols;
- Complexidade do problema.

--- 

## Paralelo com possíveis problemas reais
### Planejamento e otimização de redes de sensores ambientais para o monitoramento de qualidade de água em bacias hidrográficas estaduais
Por que é análogo:
- Em ambos os casos temos entidades sequenciais/espaciais (resíduos ao longo de uma cadeia de RNA ↔ pontos de medição ao longo de um rio ou rede de drenagem).
- Existe informação de template/histórico (estruturas conhecidas de rios, medições históricas, mapas topográficos) que pode ser transferida para novos trechos com ajustes locais.
- Precisamos prever valores contínuos por posição (coordenadas 3D ou concentrações de poluentes) e impor restrições físicas/geométricas (distâncias mínimas entre sensores, conectividade de rede, limites topográficos).

### Modelagem de conformações de proteínas/peptídeos para pesquisa em biotecnologia local
Por que é análogo:
- Entrada sequencial com informação local e global: tanto em RNA quanto em proteínas temos uma sequência linear (nucleotídeos ou aminoácidos) cuja conformação 3D depende de contexto local (vizinhança imediata) e de efeitos de longo alcance (interações distantes).
- Existência de templates úteis: para muitas proteínas/peptídeos existem estruturas experimentais ou modelos homólogos (PDB, modelos de homologia) que servem como templates iniciais — exatamente como usamos estruturas de RNA conhecidas.
- Necessidade de ajuste fino: o template fornece uma boa aproximação, mas variantes (mutação de resíduos, pequenas diferenças de sequência, condições ambientais) exigem correções locais — o papel do modelo residual.
