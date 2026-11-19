# Calçada Acessível · Sensor com ESP32

**Global Solution 2026.2 · CitizenTech · Edge Computing & Computer Systems**
Equipe [nome da equipe] · Engenharia de Software 1º ano · FIAP [Paulista / Lins]

> Calçada Acessível é um app colaborativo em que a comunidade marca os problemas e os bons exemplos das calçadas, para todo mundo escolher o melhor caminho. Esta entrega é o **sensor com ESP32** da solução: um dispositivo que acompanha a pessoa, detecta obstáculos, trepidação e falta de luz, decide sozinho e alerta na hora, sem depender de internet.

**Links da entrega**

| Item | Link |
|---|---|
| Simulação no Wokwi | [https://wokwi.com/projects/481161279072109569 |
| Vídeo no YouTube | [https://youtu.be/wBfUwIu19S8 |

**Índice:** [Ficha](#ficha-da-citizentech) · [Problema](#o-problema) · [Descrição](#descrição-do-projeto) · [Componentes](#componentes-utilizados) · [Circuito](#circuito) · [Lógica](#lógica-de-funcionamento) · [Como executar](#como-executar-a-simulação) · [Evolução](#evolução-futura) · [ODS](#ods) · [Equipe](#equipe)

## Ficha da CitizenTech

| Campo | Nossa resposta |
|---|---|
| Nome da CitizenTech | Calçada Acessível |
| Cidade e bairro | São Paulo – [bairro que vamos observar] |
| Cidadão beneficiado | PcD, idosos e quem anda com carrinho de bebê |
| Problema, em uma frase | Não dá para saber antes quais calçadas estão bloqueadas |
| Como funciona | A comunidade marca problemas e bons exemplos num mapa colaborativo |
| Principais funções | Reportar, consultar o mapa e acompanhar o conserto |
| Indicador de impacto | Ocorrências resolvidas e tempo médio até o conserto |
| ODS conectados | 3 · 10 · 11 |

## O problema

Pessoas com deficiência ou mobilidade reduzida não sabem, antes de sair, quais calçadas do caminho têm buracos ou obstáculos, e não têm um jeito simples de avisar quem pode consertar. Segundo o Censo 2022 do IBGE, são 14,4 milhões de brasileiros com deficiência, 7,3% da população com 2 anos ou mais.

Quem usa cadeira de rodas, bengala, andador ou carrinho de bebê só descobre o obstáculo quando já está na frente dele. Os casos mais comuns são buracos e desníveis, degraus e rampas fora da norma, postes, lixo e carros na calçada, trechos mal iluminados, falta de piso tátil e calçada estreita ou inexistente.

## Descrição do projeto

O dispositivo fica preso à cadeira de rodas, ao carrinho de bebê ou à bengala e acompanha a pessoa pela calçada. Ele lê o ambiente e **processa tudo localmente (Edge Computing)**, seguindo o ciclo **medir, decidir, alertar**:

- **Medir:** distância até obstáculos, trepidação e impacto, e luminosidade.
- **Decidir:** valida e suaviza as leituras e classifica em um de 5 estados.
- **Alertar:** acende LEDs e toca o buzzer em ritmos diferentes, mesmo sem internet.

Quando o estado muda, o dispositivo gera uma **Ocorrência em JSON** com os mesmos campos do app da Calçada Acessível (gravidade de 0 a 3), pronta para alimentar o mapa colaborativo.

**Por que Edge Computing**

- A resposta é imediata, sem latência de rede.
- Funciona sem internet, onde a conectividade é fraca ou inexistente.
- Só o dado relevante (a ocorrência) é gerado, não o fluxo bruto dos sensores.

## Componentes utilizados

| Componente | Pino ESP32 | Por que está ali |
|---|---|---|
| ESP32 DevKit V1 | – | Processa tudo localmente |
| HC-SR04 (ultrassônico) | TRIG 26, ECHO 27 | Mede a distância até obstáculos (poste, lixo, carro, degrau) |
| MPU6050 (acelerômetro) | SDA 21, SCL 22 | Sente trepidação e impacto (buraco, piso irregular) |
| LDR (módulo) | AO 34 | Mede a luz e identifica trecho mal iluminado |
| LED verde | 18, com resistor de 220 Ω | Caminho livre |
| LED amarelo | 19, com resistor de 220 Ω | Atenção, pouca luz ou falha de sensor |
| LED vermelho | 23, com resistor de 220 Ω | Bloqueio ou falha de sensor |
| Buzzer | 5 | Alerta sonoro para quem não enxerga os LEDs |

## Circuito

O HC-SR04 e o LDR ficam do lado esquerdo da placa. O MPU6050, os LEDs e o buzzer ficam do lado direito. Assim o diagrama não tem fios cruzados.

| De | Para | Função |
|---|---|---|
| HC-SR04 VCC e GND | VIN e GND | Alimentação |
| HC-SR04 TRIG | D26 | Dispara o pulso |
| HC-SR04 ECHO | D27 | Recebe o eco |
| LDR VCC e GND | VIN e GND | Alimentação |
| LDR AO | D34 | Leitura analógica da luz |
| MPU6050 VCC e GND | 3V3 e GND | Alimentação |
| MPU6050 SDA e SCL | D21 e D22 | Comunicação I2C |
| LED verde, amarelo e vermelho | D18, D19 e D23, cada um com resistor de 220 Ω | Estado do sistema |
| Cátodo dos três LEDs | GND | Retorno |
| Buzzer | D5 e GND | Alerta sonoro |

Para o hardware real: o ECHO do HC-SR04 trabalha em 5 V e pede um divisor de tensão antes do ESP32, e o módulo LDR deve ser alimentado em 3V3 para não passar de 3,3 V na entrada analógica. No Wokwi isso não faz diferença.

## Lógica de funcionamento

O ciclo se repete a cada 200 ms: **medir, validar, suavizar, decidir e alertar**.

1. **Medir** os três sensores.
2. **Validar:** leitura fora da faixa física do sensor (ou MPU6050 que não responde) é descartada.
3. **Suavizar:** média móvel das últimas 5 amostras válidas.
4. **Decidir:** classifica por regras de prioridade e só troca de estado depois de 3 ciclos seguidos (debounce).
5. **Alertar:** acende o LED, toca o buzzer e emite a ocorrência em JSON no Serial Monitor.

### Estados e respostas

| Estado | Condição | Gravidade | Resposta |
|---|---|---|---|
| LIVRE | Nenhuma das abaixo | 0 | LED verde, sem som |
| LUZ_BAIXA | LDR indica escuro | 1 | LED amarelo, bipe curto e raro |
| ATENCAO | Distância de 30 a 80 cm ou impacto > 0,25 g | 2 | LED amarelo, bipe lento |
| BLOQUEIO | Distância < 30 cm ou impacto > 0,60 g | 3 | LED vermelho, bipe rápido |
| FALHA_SENSOR | ~2 s sem leitura válida | 1 | LEDs amarelo e vermelho, bipe grave |

As regras são avaliadas em ordem: bloqueio, atenção, luz baixa e, por fim, livre.

### Limites escolhidos e justificativa

Premissa: pedestre ou cadeira de rodas a cerca de 0,8 m/s.

| Limite | Valor | Justificativa |
|---|---|---|
| Bloqueio por distância | < 30 cm | A 0,8 m/s restam ~0,4 s até o contato: último momento para parar |
| Atenção por distância | < 80 cm | ~1 s de antecedência para desviar ou frear |
| Atenção por impacto | > 0,25 g | Trepidação perceptível de piso irregular |
| Bloqueio por impacto | > 0,60 g | Impacto forte, típico de buraco ou degrau |
| Luz baixa | LDR > 3000 de 4095 | Trecho escuro. Se a resposta vier invertida, use `INVERTER_LDR` |
| Média móvel | 5 amostras (~1 s) | Suaviza ruído sem atrasar demais a resposta |
| Debounce | 3 ciclos (~0,6 s) | Evita que um susto isolado dispare alerta |
| Falha de sensor | 10 ciclos (~2 s) | Tolera falhas pontuais antes de declarar defeito |

O impacto é quanto a aceleração total se afasta de 1 g (a gravidade). Os valores de impacto e de luz são ajustes de projeto, calibrados na simulação. Confirme-os nos seus testes.

### Tratamento de leitura inválida

| Sensor | O que é inválido | O que o sistema faz |
|---|---|---|
| HC-SR04 | Distância fora de 2 a 400 cm | Descarta a amostra |
| MPU6050 | Não responde no I2C, não entrega 6 bytes ou passa de 8 g | Descarta a amostra |
| LDR | Valor fora de 0 a 4095 | Descarta a amostra |

Se um sensor ficar 10 ciclos seguidos (~2 s) sem leitura válida, o sistema entra em FALHA_SENSOR e se recupera sozinho quando as leituras voltam. Sem eco no HC-SR04 significa que não há obstáculo no alcance, e isso não é falha.

### Saída no Serial Monitor

A cada segundo, uma linha de status mostra o estado e os valores suavizados:

```
[status] ATENCAO      dist=55 cm | impacto=0.02 g | luz=800 | falhas(dist/imu)=0/0
```

Quando o estado muda, é emitida a ocorrência, com os campos do app (categoria, tipo, gravidade, status e confirmações) mais os valores medidos:

```json
{"id":1,"categoria":"problema","tipo":"obstaculo","gravidade":3,"status":"aberta","confirmacoes":0,"dist_cm":24.8,"impacto_g":0.03,"luz":1200}
```

| Estado | categoria | tipo |
|---|---|---|
| BLOQUEIO ou ATENCAO por distância | problema | obstaculo |
| BLOQUEIO ou ATENCAO por impacto | problema | buraco |
| LUZ_BAIXA | problema | mal_iluminado |
| LIVRE | ponto_positivo | bem_conservada |
| FALHA_SENSOR | alerta_sistema | sensor_sem_leitura |

## Como executar a simulação

1. Abra o link do Wokwi acima (ou crie um projeto **ESP32** em wokwi.com e cole `wokwi/sketch.ino` e `wokwi/diagram.json`).
2. Clique em **Start the simulation** (▶) e abra o **Serial Monitor**.
3. Clique no sensor que quer alterar e mude o valor no controle que aparece.
4. Espere cerca de 1 segundo depois de cada mudança, porque o debounce leva 3 ciclos para confirmar.

| Para ver | O que fazer | Resultado esperado |
|---|---|---|
| Caminho livre | HC-SR04 em 200 cm | LED verde, sem som |
| Atenção | HC-SR04 em 60 cm | LED amarelo, bipe lento, ocorrência de gravidade 2 |
| Bloqueio | HC-SR04 em 20 cm | LED vermelho, bipe rápido, ocorrência de gravidade 3 |
| Buraco | MPU6050 com aceleração maior (por exemplo, Z de 1 g para 2 g) | Ocorrência do tipo `buraco` |
| Pouca luz | LDR com menos luz | LED amarelo, bipe curto e raro, tipo `mal_iluminado` |
| Falha de sensor | Pare a simulação, desconecte o fio SDA do MPU6050 e inicie de novo | Em ~2 s, LEDs amarelo e vermelho, bipe grave, `alerta_sistema` |

Mantenha a distância do HC-SR04 entre 2 e 400 cm: fora disso a leitura é considerada inválida.

## Evolução futura

Num cenário real, o ESP32 enviaria cada ocorrência por Wi-Fi (MQTT) para um back-end (como FIWARE) e para o app da Calçada Acessível. O app acrescentaria GPS, endereço e foto, e permitiria acompanhar o status (aberta, confirmada, encaminhada e resolvida) e medir os indicadores de impacto: problemas mapeados, % resolvidos, tempo médio até o conserto e trechos bem avaliados por bairro.

**Ecossistema:** moradores e PcD reportam e confirmam; ONGs de PcD divulgam e validam prioridades; subprefeituras recebem a lista e atualizam o status; donos de imóveis cuidam da calçada em frente ao lote; o comércio local apoia reparos e ganha o selo "Calçada Amiga". O app seria gratuito para o cidadão, com receita de um painel de dados para prefeituras e construtoras e do selo para comércios.

## ODS

| ODS | Contribuição |
|---|---|
| **3** Saúde e bem-estar | Menos quedas e lesões de idosos e PcD |
| **10** Redução das desigualdades | Mais autonomia para circular pela cidade |
| **11** Cidades sustentáveis | Espaço público acessível para todos |

## Estrutura do repositório



## Equipe

| Nome | RM |
|---|---|
| [Kaio Nincao Maia Dias] | [_568834____] |
| [Leonardo Gonçalves Cardoso da Fonseca] | [_569475____] |
| [Juliano Galhardo de Oliveira] | [__572493___] |
| [Kaue fernando jaques lopes] | [_571353____] |
| [Matheus Martins Santos] | [_570884____] |

Todos os integrantes também constam no cabeçalho do `sketch.ino`.
