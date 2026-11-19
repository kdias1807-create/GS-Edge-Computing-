# GS-Edge-Computing-

# Calçada Acessível

**Global Solution 2026.2 · CitizenTech · Edge Computing & Computer Systems**
Equipe [nome da equipe] · Engenharia de Software 1º ano · FIAP [Paulista / Lins]

> Um app colaborativo em que a comunidade marca os problemas e os bons exemplos das calçadas, para todo mundo escolher o melhor caminho. Um sensor com ESP32 também detecta obstáculos e trepidação no caminho e gera ocorrências sozinho.

## Ficha da CitizenTech

| Campo | Nossa resposta |
|---|---|
| Nome da CitizenTech | Calçada Acessível |
| Cidade e bairro | São Paulo – [bairro que vamos observar] |
| Cidadão beneficiado | PcD, idosos e quem anda com carrinho de bebê |
| Problema, em uma frase | Não dá para saber antes quais calçadas estão bloqueadas |
| Como funciona | A comunidade marca problemas e bons exemplos num mapa colaborativo |
| Principais funções | Reportar, consultar o mapa e acompanhar o conserto |
| Link do repositório | [link do GitHub] |
| Indicador de impacto | Ocorrências resolvidas e tempo médio até o conserto |
| ODS conectados | 3 · 10 · 11 |

**Links da entrega**

- Simulação no Wokwi: [colar o link público]
- Vídeo no YouTube: [colar o link]
- App colaborativo (opcional, para demonstração): [link do GitHub Pages]

## O problema

Pessoas com deficiência ou mobilidade reduzida não sabem, antes de sair, quais calçadas do caminho têm buracos ou obstáculos, e não têm um jeito simples de avisar quem pode consertar. Segundo o Censo 2022 do IBGE, são 14,4 milhões de brasileiros com deficiência (7,3% da população com 2 anos ou mais).

## Arquitetura em 3 camadas

```
Camada 1 · Device IoT            Camada 2 · Nuvem IoT              Camada 3 · App
┌─────────────────────┐   MQTT   ┌──────────────────────┐   MQTT   ┌──────────────────────────┐
│ ESP32 + sensores    │ ───────► │ Broker MQTT na nuvem │ ───────► │ App colaborativo (mapa)  │
│ lê, filtra, decide  │  Wi-Fi   │ (HiveMQ, mensagens   │ WebSocket│ reportar, confirmar,     │
│ e alerta sozinho    │          │  retidas = banco)    │ ◄─────── │ acompanhar o conserto    │
└─────────────────────┘          └──────────────────────┘          └──────────────────────────┘
   funciona offline                  guarda as ocorrências            GPS, endereço e foto no celular
```

| Camada | O que faz | Onde está |
|---|---|---|
| 1. Device IoT | Mede, valida, suaviza, classifica e alerta com LEDs e buzzer. Funciona sem internet. | `wokwi/` (simulado no Wokwi) |
| 2. Nuvem IoT | Broker MQTT público (`broker.hivemq.com`). As ocorrências ficam como mensagens retidas, então quem abre o app depois vê o mesmo mapa. | Nuvem |
| 3. App | Mapa colaborativo, painel do sensor, indicadores de impacto e modo subprefeitura. | `app/index.html` |

**Edge:** o ESP32 decide e alerta localmente. Para a nuvem só vão a ocorrência (a cada mudança de estado) e um resumo do estado (a cada 5 s), nunca o fluxo bruto dos sensores.

## O dispositivo (camada 1)

### Componentes

| Componente | Pino ESP32 | Por que está ali |
|---|---|---|
| ESP32 DevKit V1 | – | Processa tudo localmente e fala Wi-Fi/MQTT |
| HC-SR04 (ultrassônico) | TRIG 26, ECHO 27 | Mede a distância até obstáculos (poste, lixo, carro, degrau) |
| MPU6050 (acelerômetro) | SDA 21, SCL 22 | Sente trepidação e impacto (buraco, piso irregular) |
| LDR (módulo) | AO 34 | Mede a luz e identifica trecho mal iluminado |
| LED verde | 18 (resistor 220 Ω) | Caminho livre |
| LED amarelo | 19 (resistor 220 Ω) | Atenção, pouca luz ou falha de sensor |
| LED vermelho | 23 (resistor 220 Ω) | Bloqueio ou falha de sensor |
| Buzzer | 5 | Alerta sonoro para quem não enxerga os LEDs |

O HC-SR04 e o LDR ficam do lado esquerdo da placa, e os atuadores e o MPU6050 do lado direito, para o diagrama não ter fios cruzados. No hardware real, o ECHO do HC-SR04 trabalha em 5 V e pede um divisor de tensão antes do ESP32.

### Lógica de funcionamento

Ciclo de 200 ms: **medir → validar → suavizar → decidir → alertar → enviar**.

1. **Medir** os três sensores.
2. **Validar:** leitura fora da faixa física do sensor (ou MPU6050 que não responde) é descartada.
3. **Suavizar:** média móvel de 5 amostras válidas.
4. **Classificar** por regras e **confirmar** (debounce): o estado só muda depois de 3 ciclos seguidos.
5. **Alertar** com LEDs e buzzer, e **emitir a ocorrência** em JSON.

| Estado | Condição | Gravidade | Resposta |
|---|---|---|---|
| LIVRE | Nenhuma das abaixo | 0 | LED verde, sem som |
| LUZ_BAIXA | LDR indica escuro | 1 | LED amarelo, bipe curto e raro |
| ATENCAO | Distância 30–80 cm ou impacto > 0,25 g | 2 | LED amarelo, bipe lento |
| BLOQUEIO | Distância < 30 cm ou impacto > 0,60 g | 3 | LED vermelho, bipe rápido |
| FALHA_SENSOR | ~2 s sem leitura válida | 1 | LEDs amarelo e vermelho, bipe grave |

**Tratamento de leitura inválida**

| Sensor | O que é inválido | O que o sistema faz |
|---|---|---|
| HC-SR04 | Distância fora de 2–400 cm | Descarta a amostra |
| MPU6050 | Não responde no I2C, não entrega 6 bytes ou passa de 8 g | Descarta a amostra |
| LDR | Valor fora de 0–4095 | Descarta a amostra |

Se um sensor ficar 10 ciclos seguidos (~2 s) sem leitura válida, o sistema entra em FALHA_SENSOR e se recupera sozinho quando as leituras voltam. Sem eco no HC-SR04 significa que não há obstáculo no alcance, e isso não é falha.

**Justificativa dos limites** (premissa: pedestre ou cadeira de rodas a cerca de 0,8 m/s)

| Limite | Valor | Justificativa |
|---|---|---|
| Bloqueio por distância | < 30 cm | A 0,8 m/s restam ~0,4 s até o contato: último momento para parar |
| Atenção por distância | < 80 cm | ~1 s de antecedência para desviar ou frear |
| Atenção por impacto | > 0,25 g | Trepidação perceptível de piso irregular |
| Bloqueio por impacto | > 0,60 g | Impacto forte, típico de buraco ou degrau |
| Média móvel | 5 amostras (~1 s) | Suaviza ruído sem atrasar demais a resposta |
| Debounce | 3 ciclos (~0,6 s) | Evita que um susto isolado dispare alerta |
| Falha de sensor | 10 ciclos (~2 s) | Tolera falhas pontuais antes de declarar defeito |

Os valores de impacto e luz são ajustes de projeto, calibrados na simulação. Confirme-os nos seus testes.

### Mensagens enviadas à nuvem

Tópicos (troque `<ID>` pelo ID da equipe):

| Tópico | Quando | Conteúdo |
|---|---|---|
| `gs2026/calcada-acessivel/<ID>/esp32` | A cada mudança de estado | Ocorrência com os campos do slide 7 |
| `gs2026/calcada-acessivel/<ID>/esp32/status` | A cada 5 s | Estado e leituras suavizadas |
| `gs2026/calcada-acessivel/<ID>/ocorrencias/<uid>` | Quando alguém registra, confirma ou atualiza | Ocorrência completa (mensagem retida) |

```json
{"id":1,"categoria":"problema","tipo":"obstaculo","gravidade":3,"status":"aberta","confirmacoes":0,"dist_cm":24.8,"impacto_g":0.03,"luz":1200,"origem":"esp32"}
```

## Como executar

### 1. Simulação no Wokwi

1. Crie um projeto **ESP32** em wokwi.com.
2. Cole `wokwi/sketch.ino` e `wokwi/diagram.json`. Crie o arquivo `libraries.txt` com o conteúdo de `wokwi/libraries.txt` (biblioteca `PubSubClient`).
3. No `sketch.ino`, troque `TEAM_ID` (`"CHANGE-ME"`) por um nome único da equipe, e preencha os nomes dos integrantes no cabeçalho.
4. Clique em **Start the simulation** (▶) e abra o **Serial Monitor**. A linha `[rede] MQTT conectado` indica que a nuvem está ok. Sem rede, o alerta local continua funcionando.
5. Clique nos sensores para mudar os valores:
   - **Obstáculo:** HC-SR04, 200 → 60 → 20 cm.
   - **Buraco:** MPU6050, aumente a aceleração (por exemplo, Z de 1 g para 2 g).
   - **Pouca luz:** LDR, reduza a luz. Se vier invertido, mude `INVERTER_LDR` para `true`.
   - **Falha de sensor:** pare a simulação, desconecte o fio SDA do MPU6050 e inicie de novo.


## Limitações conhecidas

- O broker é **público e sem login**: qualquer pessoa que souber o ID da equipe pode ler e escrever. O app valida tudo o que recebe, mas um produto real precisaria de back-end próprio com autenticação (por exemplo, FIWARE).
- A foto vai como miniatura (até ~320 px) para caber na mensagem MQTT.
- Confirmações simultâneas de duas pessoas na mesma ocorrência podem se sobrepor (vale a última mensagem).
- O ESP32 não tem GPS: quem dá a localização é o celular.

## Ecossistema e modelo de negócio

Moradores e PcD reportam e confirmam; ONGs de PcD divulgam o app e validam prioridades; subprefeituras recebem a lista e atualizam o status; donos de imóveis cuidam da calçada em frente ao lote; o comércio local apoia reparos e ganha o selo "Calçada Amiga". O app é gratuito para o cidadão, e a receita vem de um painel de dados para prefeituras e construtoras e do selo para comércios.

## ODS

**3** Saúde e bem-estar (menos quedas e lesões) · **10** Redução das desigualdades (mais autonomia para circular) · **11** Cidades sustentáveis (espaço público acessível para todos)

## Equipe

[Nome completo] · RM [_____]
[Nome completo] · RM [_____]
[Nome completo] · RM [_____]
[Nome completo] · RM [_____]
[Nome completo] · RM [_____]

Todos os integrantes também constam no cabeçalho do `sketch.ino`.
