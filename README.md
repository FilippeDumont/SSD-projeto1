# MVP — Predição do Valor FIPE de Veículos de Frota

**Aluno:** Filippe Dumont Lana — **Matrícula:** 202024526
**Disciplina:** Sistema de Suporte a Decisão — **Professor:** André Serrano
**Curso:** Engenharia de Produção
**Tipo de problema:** Regressão supervisionada

---

## Problema

Estimar o Valor FIPE de veículos de uma frota de locação a partir de características
cadastrais e comerciais, permitindo simular cenários de composição de frota antes da
decisão de compra.

O Valor FIPE é a referência usada para precificar seguro, definir valor residual em
contratos de locação e determinar o momento de desmobilização de um ativo.

**Critério de sucesso definido antes da modelagem:** MAPE inferior a 10% no conjunto de
teste, superando um baseline ingênuo.

---

## Resultado

| Métrica | Modelo final | Baseline (média) |
|---|---|---|
| MAPE | **3,83%** | 41,78% |
| MAE | **R$ 4.037** | R$ 17.372 |
| R² | 0,816 | -0,00 |

*Ambos avaliados no mesmo conjunto de teste (210 registros não vistos no treino).*

Modelo final: **Random Forest** com alvo em escala logarítmica, 200 árvores,
profundidade máxima 10. Treinado em 838 registros e avaliado em 210 registros não
vistos durante o treino.

---

## O achado principal: vazamento de dados

Antes da modelagem, uma auditoria testou se alguma variável entregava a resposta
diretamente. O teste contou, para cada par `(Modelo, Ano Modelo)`, quantos valores
distintos de FIPE existiam:

```
Pares (Modelo, Ano Modelo) distintos:   66
Pares com um único Valor FIPE:          66
Percentual determinístico:              100,0%
```

O par determina o Valor FIPE de forma exata em **todos** os casos — resultado esperado,
já que a tabela FIPE é indexada por modelo e ano.

A consequência é decisiva: um modelo treinado com essas variáveis não aprende relação
alguma, apenas memoriza o mapeamento. Exibiria métricas altas e falharia inteiramente
diante de um veículo cujo modelo não constasse do treino — exatamente o caso de uso
pretendido.

**Decisão:** `Modelo` e `Código FIPE` foram excluídos das features. O modelo opera apenas
sobre atributos que descrevem o *tipo* de ativo, não sua identidade. Essa é a decisão de
modelagem mais importante do trabalho.

---

## Escolha do modelo: MAPE em vez de R²

A comparação entre algoritmos revelou um trade-off:

| Modelo | R² (CV) | MAE (CV) | MAPE (CV) |
|---|---|---|---|
| Regressão Linear (log) | **0,922** | R$ 4.156 | 5,09% |
| Ridge (log) | 0,913 | R$ 4.663 | 5,62% |
| Random Forest (log) | 0,899 | **R$ 3.304** | 3,70% |
| Árvore de Decisão (log) | 0,888 | R$ 3.373 | **3,64%** |
| Baseline (média) | -0,008 | R$ 17.486 | 42,87% |

A Regressão Linear lidera em R²; os modelos de árvore lideram em MAE e MAPE. Não é
contradição: o R² é calculado sobre erros ao quadrado e portanto é dominado pelos poucos
veículos de alto valor (Jeep Commander, ~R$ 230 mil). Os modelos de árvore acertam melhor
o veículo **típico** — o automóvel de entrada de ~R$ 65 mil, que compõe a maior parte da
frota. Como a decisão apoiada é avaliar ativos de uma frota majoritariamente de entrada,
o erro relativo no caso típico pesa mais que o ajuste nos extremos.

Entre a Árvore de Decisão e o Random Forest, a diferença de MAPE (3,64% vs 3,70%) é
inferior ao desvio entre folds (0,85 p.p.) — **empate técnico**. O Random Forest foi
selecionado por ser um ensemble: menor variância e menor risco de sobreajuste que uma
árvore única. A otimização de hiperparâmetros usou MAPE como critério.

---

## Tratamento dos dados

Base original: 1.083 registros × 20 colunas. Base final: 1.048 × 7 features.

### Anonimização (LGPD)

A base contém 1.082 placas, 1.082 chassis, 891 Renavams e **750 nomes de clientes** —
dados pessoais de terceiros. Esses campos foram usados apenas para checar duplicidade
(via hash SHA-256 com sal aleatório) e **removidos integralmente** da base publicada.
Remoção é mais forte que pseudonimização: identificadores de baixa entropia, como placas,
são recuperáveis por força bruta quando o algoritmo e o sal são conhecidos.

> **Atenção:** o arquivo `.xlsx` original **não deve ser versionado** neste repositório.
> Apenas a versão anonimizada (`veiculos_anonimizado.csv`) é publicável.

### Limpeza

| Problema | Tratamento | Volume |
|---|---|---|
| Coluna 100% vazia (`Chave Reserva`) | Removida | 1 coluna |
| Colunas constantes (variância zero) | Removidas | 5 colunas |
| Linha totalizadora do relatório (R$ 73.054.389) | Removida | 1 registro |
| Valor FIPE zero ou fora de faixa | Removidos | 34 registros |
| Dados pessoais | Removidos após checagem de duplicidade | 4 colunas |

Descarte total: 3,2% dos registros.

### Engenharia de features

- **`Idade`** — anos desde o ano-modelo; torna a depreciação diretamente interpretável.
- **`Categoria`** — separa automóvel de motocicleta. A frota tem 98 motos Yamaha, em
  patamar de preço ~4× inferior. Sem essa marcação, o modelo trataria moto como carro barato.
- **Agrupamento de montadoras raras** — marcas com menos de 10 unidades viram `OUTRAS`,
  evitando categorias de 1–2 registros que o one-hot encoding transformaria em ruído.

---

## Variáveis mais importantes

1. `Categoria = MOTOCICLETA` — divisor de patamar de preço da frota
2. `Grupo = ASSINATURA` — **proxy**, não causa: a modalidade contratual não determina o
   valor do veículo, mas concentra um perfil específico de ativo
3. `Montadora` — posicionamento de marca
4. `Ano Modelo` / `Idade` — depreciação

---

## Estrutura do repositório

```
├── MVP_FIPE_Veiculos.ipynb      # Notebook principal (executado, com saídas)
├── Relatorio_MVP_FIPE.pdf       # Relatório completo
├── veiculos_anonimizado.csv     # Base limpa, sem identificadores
├── veiculos_modelagem.csv       # Recorte final usado no modelo
└── README.md
```

## Como reproduzir

```bash
pip install pandas numpy scikit-learn matplotlib seaborn openpyxl
jupyter notebook MVP_FIPE_Veiculos.ipynb
```

O notebook roda de ponta a ponta. `RANDOM_STATE = 42` garante reprodutibilidade.
No Google Colab, basta abrir e executar — os dados são carregados direto da URL.

---

## Limitações

- **Frota homogênea:** predominam Fiat, Citroën e Peugeot dos anos 2023–2025. Não aplicar
  a marcas ou faixas de valor ausentes da base.
- **Recorte estático:** fotografia única, sem série temporal. Não modela evolução do valor.
- **`Grupo` é proxy:** mudanças na política comercial invalidariam a relação e exigiriam
  retreinamento.
- **Cauda superior sub-representada:** poucos veículos acima de R$ 150 mil, o que explica
  a maior dispersão dos resíduos nessa faixa.

## Próximos passos

Incorporar **quilometragem** e **histórico de manutenção** — ausentes nesta base e
provavelmente o maior ganho disponível, por serem os fatores que diferenciam o valor de
dois veículos idênticos em modelo e ano.

---

## Fonte

Base do repositório [`andrelmsunb/Previsao_Valor_FIPE_Veiculos`](https://github.com/andrelmsunb/Previsao_Valor_FIPE_Veiculos).
