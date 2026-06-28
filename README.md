# TechCorp_HR_Analytics

## 1. Proposta de Implementação em Produção (Databricks)

### Arquitetura Proposta

```
[HR System / SAP] → [Delta Lake Bronze] → [Feature Engineering (Databricks Job)] 
    → [Delta Lake Silver] → [Modelo MLflow] → [Scoring API] → [Dashboard RH]
```

### Pipeline de Produção:
1. **Ingestão diária** dos dados de RH via Databricks Auto Loader (Delta Lake)
2. **Feature Engineering** executado como Databricks Job agendado
3. **Modelo** registrado no **MLflow Model Registry** com versionamento
4. **Scoring batch** diário: todos os funcionários recebem score de risco
5. **Dashboard PowerBI/Tableau** exibe top-N funcionários em risco para o RH
6. **Alertas automáticos** para gestores quando funcionário-chave entra em zona de risco

### Monitoramento (Model Drift):
- **Data drift**: comparar distribuição das features semanalmente (KS-test)
- **Concept drift**: monitorar PR-AUC nos novos casos confirmados mensalmente
- **Retreinamento**: automático quando PR-AUC cair > 5% da baseline

### Métricas de Negócio a Monitorar:
- Taxa de retenção dos funcionários abordados pelo RH
- Custo médio por desligamento real vs. previsto
- ROI das ações preventivas (economia / custo da iniciativa)

## 2. Análise Crítica: Detecção e Correção de Data Leakage

Esta seção documenta os problemas identificados em versões anteriores do notebook e as correções aplicadas — um exercício fundamental de **pensamento crítico** em Data Science.

### Causas identificadas e corrigidas

| # | Problema | Solução aplicada |
|---|----------|-----------------|
| 1 | `AttritionProbability_hidden` no DataFrame — modelo aprendia a variável latente | Removida da função geradora |
| 2 | `Attrition_Target` não removido de X | `drop(columns=['Attrition','Attrition_Target'])` |
| 3 | `PolynomialFeatures.fit_transform()` em toda a base antes do split | `fit()` apenas no treino, `transform()` no teste |
| 4 | Múltiplos splits inconsistentes | Um único split estratificado 80/20 |

### Lição aprendida
> *"Um ROC-AUC = 1.0 em qualquer conjunto de dados real deve ser tratado como sinal de erro, não de sucesso."*

O pipeline final garante:
- **Nenhuma informação do teste** contamina o treino
- **Todas as transformações** (`StandardScaler`, `OneHotEncoder`, `PolynomialFeatures`) são ajustadas SOMENTE no treino
- **SMOTE/ADASYN** são aplicados APÓS o split, nunca antes

## 3. Conclusões e Principais Aprendizados

### Resultados Técnicos
- O **LightGBM** (com ou sem otimização de hiperparâmetros) consistentemente superou os demais modelos em PR-AUC, métrica mais relevante para problemas desbalanceados
- A otimização de **threshold** foi fundamental: o threshold padrão (0.5) subestima sistematicamente a classe minoritária
- O **feature engineering** criou variáveis com poder preditivo superior às originais (`Burnout_Risk`, `Stagnation_Index`, `Commute_Stress`)

### Impacto no Negócio
- O modelo permite que o RH identifique proativamente funcionários de alto risco, concentrando esforços nos **top 10%** (maior precision × recall)
- Redução estimada de **R$ 4,5M+** anuais em custos de attrition com taxa de retenção de 60%
- Melhoria da decisão de quando e como intervir, substituindo intuição por evidência quantitativa