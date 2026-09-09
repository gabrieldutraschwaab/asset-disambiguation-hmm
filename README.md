# Desambiguação sequencial de ativos com HMM

Protótipo experimental para desambiguar ativos industriais visualmente idênticos combinando observações posicionais, um grafo de movimentação e modelos ocultos de Markov (HMM).
Neste repositório foi implementada e avaliada a etapa de desambiguação sequencial da arquitetura proposta. As etapas anteriores de percepção, estimação de profundidade e localização não foram implementadas. Para testar o HMM, foram utilizadas posições cadastradas de seis válvulas e observações geradas pela adição de erros aleatórios de localização. As 144 mil observações resultam da combinação de quatro rotas, seis níveis de ruído, mil repetições por condição e seis ativos por trajetória.

## Objetivo

Avaliar, sob níveis controlados de incerteza posicional, se a associação sequencial baseada em HMM reduz identificações incorretas em comparação com uma associação espacial independente.


## Métodos comparados

* **Espacial:** associa cada observação independentemente ao ativo mais provável.
* **Forward:** incorpora o histórico de observações e o prior de movimentação para realizar inferência sequencial online.
* **Viterbi:** encontra a sequência global de estados mais provável.
* **Abstenção:** suspende a identificação quando a confiança ou a separação entre candidatos não satisfaz os limiares fixados na validação.

Os estados de identidade do HMM são exclusivamente as seis válvulas V1 a V6. Os pontos intermediários do grafo representam apenas caminhos de movimentação e não são candidatos de identidade. As probabilidades de emissão são calculadas a partir da distância de Mahalanobis entre a posição observada e a posição cadastrada de cada válvula, considerando a incerteza da observação. O prior de movimentação é obtido a partir do grafo, atribuindo maior probabilidade às transições realizadas por trajetos mais curtos.

## Cenário sintético

A planta contém um módulo central não atravessável, dois corredores laterais e seis válvulas visualmente idênticas distribuídas em faces opostas.

Foram avaliadas quatro rotas:

* rotas primárias: `circuito_horario` e `circuito_antihorario`;
* rotas de estresse: `troca_central_longa` e `inspecao_com_saltos`.

A rota `inspecao_com_saltos` viola deliberadamente o prior de movimentação para delimitar uma condição de falha do modelo sequencial.

## Desenho experimental

Os parâmetros foram selecionados exclusivamente no conjunto de validação e mantidos fixos no teste final.

| Parâmetro               |                              Valor |
| ----------------------- | ---------------------------------: |
| Sementes de validação   |                        1000 a 1199 |
| Sementes de teste       |                      10000 a 10999 |
| Escala de transição     |                              2,0 m |
| Fatores de ruído        | 0,50; 0,75; 1,00; 1,25; 1,50; 2,00 |
| Rotas avaliadas         |                                  4 |
| Condições de teste      |                                 24 |
| Linhas no teste final   |                            144.000 |
| Reamostragens bootstrap |                             10.000 |
| Unidade de reamostragem |                   semente completa |

Limiares fixados na validação:

| Método   | Confiança mínima | Separação mínima |
| -------- | ---------------: | ---------------: |
| Espacial |             0,60 |             0,25 |
| Forward  |             0,80 |             0,00 |

Os conjuntos de validação e teste não compartilham sementes.

## Resultados

### Acurácia sem abstenção

| Grupo             | Espacial | Forward | Viterbi | Ganho Forward | Ganho Viterbi |
| ----------------- | -------: | ------: | ------: | ------------: | ------------: |
| Geral             |   82,10% |  85,65% |  88,34% |      +3,55 pp |      +6,24 pp |
| Rotas primárias   |   82,14% |  91,09% |  96,24% |      +8,95 pp |     +14,11 pp |
| Rotas de estresse |   82,06% |  80,21% |  80,43% |      −1,85 pp |      −1,62 pp |

Nas rotas primárias, o bootstrap pareado por semente produziu:

* Forward: ganho de 8,95 p.p., com IC 95% de 8,46 a 9,44 p.p.;
* Viterbi: ganho de 14,11 p.p., com IC 95% de 13,56 a 14,63 p.p.

### Operação com abstenção nas rotas primárias

| Método   | Cobertura | Acurácia seletiva | Identificações incorretas |
| -------- | --------: | ----------------: | ------------------------: |
| Espacial |    82,92% |            87,70% |                    10,20% |
| Forward  |    81,25% |            97,24% |                     2,24% |

Com os limiares definidos na validação, o Forward apresentou:

* redução de cobertura de 1,67 p.p., com IC 95% de 1,27 a 2,06 p.p.;
* ganho de acurácia seletiva de 9,54 p.p., com IC 95% de 9,07 a 10,01 p.p.;
* redução absoluta de identificações incorretas de 7,96 p.p., com IC 95% de 7,56 a 8,36 p.p.;
* redução relativa de identificações incorretas de 78,02%, com IC 95% de 75,08% a 80,82%.

## Interpretação

No cenário sintético, o prior de movimentação melhora substancialmente a desambiguação quando a sequência observada é compatível com o grafo.

O ganho não é universal. Na rota `inspecao_com_saltos`, o histórico induz transições incompatíveis com o prior e pode tornar Forward e Viterbi inferiores ao método espacial. Essa condição delimita o principal limite observado: um modelo sequencial depende da adequação entre sua dinâmica de transição e o movimento realizado.

A regra de abstenção reduz fortemente as identificações incorretas nas rotas primárias, com uma pequena perda de cobertura. Esses resultados não demonstram ainda transferência para uma instalação offshore real.

## Estrutura do repositório

```text
.
├── notebooks/
│   └── 01_cenario_sintetico.ipynb
├── resultados/
│   ├── figuras/
│   ├── manifesto_teste_final.json
│   ├── parametros_selecionados.json
│   ├── pontos_operacao_validacao.csv
│   ├── resultados_validacao.csv.gz
│   ├── resultados_teste.csv.gz
│   └── resumos em CSV
├── README.md
└── requirements.txt
```

O manifesto final registra os parâmetros do experimento e os hashes SHA-256 dos 15 artefatos derivados do teste.

SHA-256 do resultado bruto final:

```text
b411865b3082bce9d9989a55685634a11c5c5e7e507a34e626fb4f421ecff79e
```

## Reprodução

O experimento foi executado com Python 3.13.5.

```bash
git clone https://github.com/gabrieldutraschwaab/asset-disambiguation-hmm.git
cd asset-disambiguation-hmm

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt

jupyter lab
```

No JupyterLab, abra `notebooks/01_cenario_sintetico.ipynb` e execute as células sequencialmente.

A execução integral inclui a geração de 144.000 linhas de teste e 10.000 reamostragens bootstrap. O tempo depende do hardware. A execução também atualiza os artefatos em `resultados/`.

Para verificar o arquivo bruto já publicado:

```bash
sha256sum resultados/resultados_teste.csv.gz
```

## Limitações

* posições e incertezas geradas sinteticamente;
* ausência de detecção visual e estimação de profundidade reais;
* ausência de integração com VIO, UWB ou EKF;
* dinâmica de transição fixa;
* ausência de avaliação de latência e consumo de memória;
* ausência de validação em uma planta offshore real;
* sensibilidade a trajetórias incompatíveis com o prior de movimentação.

## Próximas etapas

1. Medir latência e consumo de memória dos métodos espacial, Forward e Viterbi.
2. Avaliar estratégias de detecção ou adaptação diante de incompatibilidade com o prior.
3. Transferir o experimento para uma topologia representativa de um cenário offshore.
4. Substituir progressivamente as observações sintéticas por saídas de componentes reais de percepção e localização.
