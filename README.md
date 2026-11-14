# CaminhoLivre

**Global Solution 2026.2 · CitizenTech · Engenharia de Software 1º ano · FIAP**
**Disciplina: Edge Computing & Computer Systems — "o sensor que decide sozinho"**

---

## Ficha da CitizenTech

| Campo | Preenchimento |
|---|---|
| **Nome da CitizenTech** | CaminhoLivre |
| **Cidade e bairro** | São Paulo, região da Av. Paulista (trajeto da estação de metrô até a FIAP) `[confirmar o trecho escolhido]` |
| **Cidadão beneficiado** | Pessoa com mobilidade reduzida que circula pelo bairro: cadeirantes, idosos e quem empurra carrinho de bebê. `[citar a persona/entrevistado]` |
| **Problema, em uma frase** | Pessoas com mobilidade reduzida não têm como saber, antes de sair de casa, quais calçadas do trajeto têm buracos, degraus ou falta de rampa, e acabam se arriscando ou desistindo de sair. |
| **Como funciona** | Um dispositivo com ESP32 acoplado à cadeira ou ao carrinho mede vibração e inclinação, decide localmente se o trecho é Normal, Atenção ou Crítico e alerta por LED e buzzer, mesmo sem internet. Quando há conexão, envia a telemetria por MQTT para um servidor, de onde os dados podem alimentar o mapa de barreiras do app web. |
| **O que cada parte faz** | **Sensor (Edge, este repositório):** ESP32 + MPU6050, lógica de borda e alertas locais, com envio MQTT opcional. **Lógica (Python):** cadastro de barreiras e trajetos, nota de acessibilidade, persistência em JSON. **Matemática:** custo do trajeto como função da distância e das barreiras, com otimização. **Interface e Interação (React):** app com CRUD de barreiras e nota por rota. **Experiência:** atores, requisitos, regras de negócio e protótipo. **Pitch:** história da pessoa entrevistada. |
| **Link do repositório** | `[colar o link do repositório]` |
| **Indicador de impacto** | Número de barreiras mapeadas e removidas; porcentagem do trajeto-piloto com nota de acessibilidade calculada; redução da nota de risco do trajeto após a escolha da rota sugerida. |
| **ODS conectados** | **ODS 10** (Redução das desigualdades): garante o direito de circular a quem tem mobilidade reduzida. **ODS 11** (Cidades e comunidades sustentáveis): torna o espaço público mais acessível e seguro. **ODS 3** (Saúde e bem-estar): reduz o risco de quedas e acidentes. |

---

## Equipe

Grupo: `Play Masters`

| Nome completo | RM |
|---|---|
| `Enrico Bertolacini` | `570999` |
| `Matheus Sá Teles` | `571719` |
| `Julia Lima da Silva` | `569203` |
| `Pedro Antonio Borges` | `572549` |
| `Guilherme Alvejan` | `570835` |

## Links da entrega

- Simulação no Wokwi: `[colar o link público]`
- Vídeo no YouTube: `[colar o link do vídeo]`

---

## 1. Descrição do projeto

O CaminhoLivre é um dispositivo de borda (edge) que "sente" a calçada. Acoplado a uma cadeira de rodas ou a um carrinho de bebê, ele mede a vibração e a inclinação do equipamento enquanto a pessoa se desloca. O microcontrolador processa essas leituras **localmente**, sem depender de internet, e responde com alertas visuais e sonoros que indicam se o trecho é tranquilo, irregular ou crítico.

Muitos problemas da cidade acontecem onde a conectividade é fraca ou inexistente. Por isso a decisão e o alerta acontecem no próprio dispositivo. Quando há conexão, o dispositivo também publica o estado e as leituras em um servidor MQTT, o que permite que a experiência de uma pessoa ajude a alimentar o mapa de barreiras usado por outras.

O ciclo de funcionamento é: **medir → validar → suavizar → decidir → alertar**.

## 2. Componentes utilizados

| Componente | Função | Por que está no projeto |
|---|---|---|
| ESP32 DevKit | Microcontrolador | Lê o sensor, decide e responde sozinho; tem Wi-Fi para o envio opcional à nuvem |
| MPU6050 | Acelerômetro/giroscópio (I2C) | Mede a vibração e a inclinação do equipamento |
| 3 LEDs (verde, amarelo, vermelho) | Resposta visual | Mostram o estado de relance |
| 3 resistores de 220 Ω | Proteção dos LEDs | Limitam a corrente de cada LED |
| Buzzer | Resposta sonora | Avisa quem não pode olhar para o dispositivo durante o deslocamento |

### Ligações

| Pino do ESP32 | Conectado a |
|---|---|
| GPIO 21 | MPU6050 SDA |
| GPIO 22 | MPU6050 SCL |
| 3V3 / GND | MPU6050 VCC / GND |
| GPIO 25 | LED verde (via resistor de 220 Ω) |
| GPIO 26 | LED amarelo (via resistor de 220 Ω) |
| GPIO 27 | LED vermelho (via resistor de 220 Ω) |
| GPIO 14 | Buzzer (positivo) |
| GND | Cátodo dos LEDs e negativo do buzzer |

## 3. Lógica de funcionamento

### Medição e tratamento das leituras

1. **Leitura:** o MPU6050 é lido 10 vezes por segundo (a cada 100 ms, controlado com `millis()`, sem `delay()`).
2. **Validação:** a leitura é descartada se a comunicação I2C falhar, se algum valor for inválido ou se o módulo da aceleração estiver fora da faixa física (menor que 0,10 g ou maior que 3,50 g).
3. **Suavização:** média móvel das últimas 5 leituras de vibração e de inclinação.
4. **Confirmação:** o estado só muda depois de 3 leituras seguidas apontando o mesmo novo estado, para evitar alarme falso por um solavanco isolado.

### Grandezas calculadas

- **Vibração:** quanto o módulo da aceleração se afasta de 1 g (gravidade), em g.
- **Inclinação:** ângulo entre o eixo Z do sensor e a vertical, em graus.

### Estados e respostas

| Estado | Condição | Resposta local |
|---|---|---|
| **NORMAL** | Vibração ≤ 0,15 g **e** inclinação ≤ 4,8° | LED verde aceso, sem som |
| **ATENÇÃO** | Vibração > 0,15 g **ou** inclinação > 4,8° | LED amarelo aceso e bip curto a cada 2 s |
| **CRÍTICO** | Vibração > 0,40 g **ou** inclinação > 10° | LED vermelho aceso e bips rápidos |
| **ERRO_SENSOR** | Leitura inválida | LED amarelo piscando, sem som |

### Justificativa dos limites

- **Inclinação:** 4,8° corresponde a cerca de 8,33% de inclinação, que é o valor máximo de rampa da norma de acessibilidade NBR 9050. `[confirmar o número na norma antes da entrega]`
- **Vibração:** 0,15 g e 0,40 g são valores adotados para esta simulação. Em um protótipo físico, precisariam ser calibrados com testes reais em cadeira de rodas e carrinho.

### Saída no Serial Monitor

A cada 0,5 s o dispositivo imprime a vibração e a inclinação suavizadas e o estado atual. As mudanças de estado e as mensagens da nuvem são impressas na hora.

## 4. Como executar a simulação

1. Acesse o projeto no Wokwi pelo link da seção "Links da entrega".
2. Clique em **Play** e abra o Serial Monitor. Deve aparecer `Estado: NORMAL` e o LED verde aceso.
3. Com a simulação rodando, clique no **MPU6050** para abrir o painel de sensores e altere a aceleração. Espere cerca de 1 segundo para o estado mudar.

| Para ver | Ajuste no painel do MPU6050 |
|---|---|
| Normal | X = 0, Y = 0, Z = 1 g |
| Atenção por vibração | Z = 1,3 g |
| Atenção por inclinação | X = 0,12 g, Z = 1 g |
| Crítico | Z = 1,6 g (ou X = 0,25 g) |
| Erro de sensor | X = 0, Y = 0, Z = 0 |

### Rodando em um projeto novo

1. Crie um projeto **ESP32 (Arduino)** no Wokwi.
2. Cole o conteúdo de `sketch.ino` na aba do código e o de `diagram.json` na aba do diagrama.
3. Para usar a nuvem, crie um arquivo `libraries.txt` com a linha `PubSubClient` e altere `USAR_NUVEM` para `1` no código. Sem a nuvem, não é necessária nenhuma biblioteca.

## 5. Comunicação com o servidor (opcional)

O envio para a nuvem **não é obrigatório** e o alerta local nunca depende dele. Quando ativado (`USAR_NUVEM = 1`), o dispositivo:

- conecta ao Wi-Fi simulado do Wokwi (`Wokwi-GUEST`) em segundo plano;
- conecta por MQTT ao servidor disponibilizado pelo professor, `3.93.68.31`, porta `1883`, sem autenticação;
- publica no tópico `fiap/gs/caminholivre/play_masters/telemetria` `[confirmar o tópico usado]`, a cada 2 s e imediatamente a cada mudança de estado;
- se o Wi-Fi ou o servidor ficarem indisponíveis, continua medindo e alertando normalmente e tenta reconectar a cada 10 s.

Exemplo de mensagem publicada:

```json
{"dispositivo":"caminholivre-esp32-01","estado":"NORMAL","vibracao_g":0.00,"inclinacao_graus":0.0}
```

Para acompanhar as mensagens, use qualquer cliente MQTT (por exemplo, MQTTX ou MyMQTT) conectado ao servidor e assinando o tópico do grupo. Não assine tópicos genéricos, pois o servidor é compartilhado com outros grupos. As mensagens contêm apenas estado e leituras do sensor, sem dados pessoais.

## 6. Limitações e próximos passos

- O projeto é uma **simulação** no Wokwi. Os limites de vibração precisam de calibração com um protótipo físico.
- O dispositivo **não envia localização**. A associação de cada leitura a um trecho da calçada ainda precisa ser feita (por exemplo, com GPS ou marcação do trajeto).
- O servidor MQTT apenas repassa mensagens; a leitura desses dados pelo app web é um próximo passo.
- Possíveis evoluções: calibração em campo, envio de localização, histórico dos trechos e integração com o mapa de barreiras do app.

