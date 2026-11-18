# FOMEZERO — Physical Computing, Embedded AI, Robotics & Cognitive IoT

Global Solution 2026.2 · FIAP · Tecnólogo em Inteligência Artificial · Turma 2TIAPR-2026

## Integrantes

- Erick Miguel — RM 564896
- Leonardo Melo — RM 567097
- Luna Oliveira — RM 564215

## 1. Proposta

O FOMEZERO parte da inversão da lógica do consumo exacerbado: utilizar a tecnologia para facilitar a distribuição, aproximando estabelecimentos com alimentos disponíveis para doação de ONGs, cozinhas comunitárias e pessoas em situação de vulnerabilidade.

A proposta CitizenTech organiza essa colaboração em torno do lote de doação. O objetivo é apoiar o aproveitamento de alimentos, a rastreabilidade das doações e o reconhecimento dos doadores, sem apresentar benefícios sociais ou ambientais como resultados já comprovados.

Este diretório contém o módulo físico de monitoramento das condições ambientais associadas ao lote. Seus sensores e alertas apoiam a avaliação humana; não certificam a segurança do alimento.

## 2. Escopo e estado da integração

- Dispositivo físico ESP32 com sensores e sinalização local.
- Comunicação MQTT entre dispositivo e infraestrutura local.
- Processamento por Node-RED, armazenamento em InfluxDB e visualização em Grafana.
- Integração demonstrada no aplicativo por importação de logs históricos coletados em momentos reais de uso.
- Não há MQTT ao vivo na interface do aplicativo.
- A simulação de memristor da disciplina Cluster Computing, Computação Neuromórfica e Supercomputadores é independente: não recebe sinais físicos do ESP32 nem mensagens MQTT.
- O Grafana deste módulo é um painel de telemetria. Não é o dashboard Analytics de Governança, que permanece apenas planejado em README.

## 3. Arquitetura

```text
DHT22 + HC-SR04
       |
       v
     ESP32 ----> OLED + LED RGB
       |
       | Wi-Fi / MQTT
       v
    Mosquitto
       |
       v
    Node-RED ----> InfluxDB ----> Grafana

Logs físicos históricos ----> importação no aplicativo
```

A infraestrutura física e a importação histórica são percursos distintos. A existência de comunicação MQTT no dispositivo não significa que o navegador recebe dados ao vivo.

## 4. Componentes e software

| Componente | Função |
|---|---|
| ESP32 | Leitura, lógica de alerta e publicação MQTT |
| DHT22 | Temperatura e umidade do ambiente próximo ao sensor |
| HC-SR04 | Distância e inferência de presença de obstáculo |
| OLED SSD1306 | Interface local |
| LED RGB | Indicação de conectividade e estado de alerta |
| Mosquitto | Broker MQTT |
| Node-RED | Processamento do fluxo e encaminhamento ao armazenamento |
| InfluxDB | Persistência das séries temporais |
| Grafana | Visualização de telemetria |

O projeto utiliza PlatformIO com framework Arduino e ambiente `esp32dev`. As bibliotecas declaradas em `platformio.ini` incluem PubSubClient, DHT sensor library, Adafruit Unified Sensor, Adafruit SSD1306 e Adafruit GFX Library.

## 5. Identificação e regras do firmware

| Parâmetro | Configuração |
|---|---|
| Lote demonstrativo | `LOTE-2026-0001` |
| Dispositivo | `esp32-caixa-01` |
| Leitura dos sensores | A cada 2 segundos |
| Publicação de telemetria | A cada 10 segundos |
| Ativação do alerta | Temperatura acima de 5 °C por 30 segundos |
| Normalização | Temperatura abaixo de 4,5 °C por 60 segundos |
| Presença inferida | Distância medida inferior a 30 cm |
| Monitor serial | 115200 baud |

Os limiares e intervalos são regras demonstrativas do firmware. Não constituem norma sanitária universal. A recuperação do estado atual não deve apagar o registro histórico da ocorrência.

### LED RGB

- Azul: ausência de conexão Wi-Fi ou MQTT.
- Vermelho: alerta ativo, quando há conectividade.
- Verde: conectividade disponível e alerta inativo.

A indicação de falha de rede tem prioridade sobre a indicação de alerta no LED. Verde não significa alimento seguro para consumo.

## 6. Mensagens MQTT

Prefixo utilizado pelo lote demonstrativo:

```text
fomezero/gs2026/rm564896/lote/LOTE-2026-0001
```

| Sufixo | Conteúdo |
|---|---|
| `/telemetria` | Leituras e estado do dispositivo |
| `/alerta` | Evento térmico e normalização |
| `/status` | Estado de conexão online/offline |
| `/comando` | Canal assinado pelo dispositivo; o callback atual apenas registra o recebimento no serial |

O payload de telemetria utiliza `fomezero.telemetria.v1` e inclui `lote_id`, `device_id`, `seq`, `ts`, `ts_valido`, `uptime_ms`, `temp_c`, `umid_pct`, `caixa_presente`, `distancia_cm`, `limite_c`, `evento`, `estado` e `origem`.

### Qualidade dos dados

- Conferir `ts_valido` antes de interpretar o horário como sincronizado.
- O firmware utiliza valores substitutos em determinadas falhas de leitura: -127 °C para temperatura e zero para umidade/distância. Esses valores não devem ser interpretados automaticamente como medições físicas válidas.
- `caixa_presente` indica uma inferência por distância, não identidade, conteúdo ou entrega.
- A fila em memória comporta oito mensagens. Quando cheia, descarta a mais antiga. Não garante histórico completo durante desconexões e não persiste após reinicialização.
- A confirmação de publicação não deve ser confundida com prova de persistência definitiva no banco.

## 7. Estrutura principal

```text
iot/
├── platformio.ini
├── src/
│   ├── main.cpp
│   └── fomezero_interface.cpp
├── include/
│   └── fomezero_interface.h
├── infra/
│   ├── compose.yaml
│   ├── mosquitto/
│   ├── node-red/
│   └── grafana/
├── backend/
├── firmware/
├── backups/
├── diagram.json
└── wokwi.toml
```

A configuração física descrita neste README corresponde ao firmware de `src/main.cpp`. Arquivos de diagrama e simulação não substituem evidência da montagem física.

## 8. Preparação da infraestrutura local

Pré-requisitos: Docker com Compose, PlatformIO, dispositivo e montagem compatível com o firmware, além de rede acessível ao ESP32.

Na pasta `iot/infra`, criar localmente um arquivo `.env` com as variáveis utilizadas por `compose.yaml`:

```dotenv
INFLUX_USER=defina_localmente
INFLUX_PASSWORD=defina_uma_senha_local
INFLUX_TOKEN=defina_um_token_local
INFLUX_ORG=fomezero
INFLUX_BUCKET=fomezero
GRAFANA_USER=defina_localmente
GRAFANA_PASSWORD=defina_outra_senha_local
```

Os valores acima são marcadores de configuração, não credenciais publicadas. Não versionar o `.env` preenchido.

A partir da raiz do repositório:

```bash
cd iot/infra
docker compose up -d
docker compose ps
```

| Serviço no computador hospedeiro | Endereço / porta |
|---|---|
| Mosquitto | Porta MQTT 1884, encaminhada à 1883 do contêiner |
| Node-RED | http://localhost:1882 |
| InfluxDB | http://localhost:8088 |
| Grafana | http://localhost:3002 |

O Compose restringe as interfaces web ao endereço de loopback, mas publica a porta MQTT sem essa restrição. Não expor o broker à internet; limitar o acesso à rede e aos dispositivos necessários.

O fluxo inicial do Node-RED é copiado para o volume apenas quando `/data/flows.json` ainda não existe. Alterar o arquivo de origem não substitui automaticamente um fluxo já persistido. Configurações de inicialização do InfluxDB também não devem ser tratadas como redefinição automática de um volume existente.

## 9. Preparação do firmware

O código inclui `secrets.h`, que deve ser criado localmente em `iot/include/` e permanecer fora do versionamento:

```cpp
#pragma once
#define FZ_WIFI_SSID "SUA_REDE"
#define FZ_WIFI_PASSWORD "SUA_SENHA"
#define FZ_MQTT_HOST "IP_LOCAL_DO_COMPUTADOR_COM_O_BROKER"
```

Utilizar o IP acessível ao ESP32, não `localhost`. A porta configurada no firmware é 1884.

O código define DHT22 no GPIO 15, TRIG no GPIO 5, ECHO no GPIO 18 e LED RGB nos GPIOs 25, 26 e 27. Esses números descrevem o firmware, não um guia elétrico completo: conferir montagem, alimentação, resistores e adequação dos níveis de sinal antes de energizar.

A partir da pasta `iot`, com PlatformIO instalado:

```bash
pio run
pio run --target upload
pio device monitor --baud 115200
```

A porta de gravação pode precisar ser selecionada no ambiente local. Esses comandos descrevem o procedimento de execução; não representam uma nova execução de testes realizada durante a redação deste README.

## 10. Demonstração e evidências

1. Identificar lote e dispositivo.
2. Demonstrar leituras, conectividade e sinalização local.
3. Verificar recebimento e armazenamento na infraestrutura.
4. Exibir séries históricas no Grafana.
5. Mostrar ocorrência térmica e, quando disponível, sua normalização.
6. Importar os registros históricos no aplicativo e manter essa origem identificada.

Preservar capturas e logs efetivamente produzidos. Não substituir ausência de dados por zero nem apresentar imagens ilustrativas como resultados medidos. O ensaio do sensor não deve colocar alimentos destinados à doação em condições inadequadas.

## 11. Relação com as demais disciplinas

- **Front End e Mobile Development:** importa e apresenta registros históricos associados ao lote.
- **AI for Robotic Process Automation:** considera informações e alertas no fluxo de processamento e revisão de entregas, conforme os contratos do projeto.
- **Cluster Computing, Computação Neuromórfica e Supercomputadores:** investiga memória resistiva em simulação independente, sem conexão física ao módulo.
- **Governança em IA e Business Analytics:** utiliza a origem e os limites dos registros para discutir rastreabilidade, revisão humana e interpretação responsável.
- **Computação Quântica e IA:** analisa roteamento futuro; a IoT atual não fornece GPS ou rotas reais.
- **Generative AI e Advanced Nets, Processamento de Linguagem Natural, Chatbots e Virtual Agents e Visão Computacional:** contribuem para outros recursos do FOMEZERO, sem integração direta com os sensores afirmada neste README.

## 12. Privacidade, segurança e limites

- Utilizar dados fictícios nos cadastros e confirmações da demonstração.
- Não publicar senhas, tokens, arquivos `.env` ou `secrets.h`.
- Revisar logs e capturas para evitar exposição de informações de rede ou pessoas.
- O firmware atual conecta ao broker sem credenciais MQTT e não utiliza TLS nesse cliente. A configuração é demonstrativa e local, não arquitetura de produção.
- Nenhum sensor confirma aproveitamento do alimento, identidade do receptor, consumo ou segurança sanitária.
- O módulo não mede emissões evitadas, não calcula refeições e não fornece localização por GPS.
- O projeto não declara conformidade integral com a LGPD nem disponibilidade contínua.

## 13. Vídeos

- Demonstração específica da disciplina: inserir URL após a publicação, caso exista vídeo próprio.
- Integração para Governança em IA e Business Analytics: inserir URL após a publicação.

Os links devem apontar para os vídeos correspondentes. O vídeo integrado não deve ser identificado como entrega específica de outra disciplina sem conferir o respectivo enunciado.

## 14. Base desta documentação

README redigido com base nos arquivos `src/main.cpp`, `platformio.ini` e `infra/compose.yaml` consultados no repositório e nas decisões de integração informadas pela equipe. A leitura do código não substitui validação elétrica, execução local ou teste de todos os serviços.
