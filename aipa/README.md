# FOMEZERO · Robô pós-entrega (AI for Robotic Process Automation)

Global Solution 2026.2 · FIAP · Tecnólogo em Inteligência Artificial · tema CitizenTech.
Repositório do grupo (pasta `aipa/`): https://github.com/lostgear84/GS_2TIAPR_2026_FOMEZERO_GABA/tree/main/aipa

> **Protótipo acadêmico. Dados SIMULADOS (sintéticos) e portal do parceiro SIMULADO.** Nenhum módulo declara que um alimento é seguro para consumo. CO₂ evitado é **estimativa**; pontos e Selo Verde ESG são **regra de demonstração**, não certificação.

## Conteúdo desta entrega
| Entregável | Onde |
|---|---|
| Código do protótipo | `codigo/` (ponto de entrada: `codigo/executar_robo.py`; serviço para o portal: `codigo/servico_aipa.py`) |
| README | este arquivo |
| Quadro de temas | `QUADRO_DE_TEMAS.md` |
| Evidências de validação | `EVIDENCIAS_DE_VALIDACAO.md` |
| Proteção de dados, ética e ODS | `ETICA_E_ODS.md` |

## Integrantes
| Nome | RM |
|---|---|
| Erick Miguel | 564896 |
| Leonardo Melo | 567097 |
| Luna Oliveira | 564215 |

## Problema
O FOMEZERO liga quem tem excedente de alimento (o **doador**: mercado, padaria, restaurante) a quem precisa (o **beneficiário**: ONG, cozinha comunitária, cidadão), para reduzir o desperdício e a insegurança alimentar. Depois que o **lote de doação** é entregue, alguém precisa conferir o que o beneficiário confirmou, registrar o impacto, emitir o **Termo de Doação** e lançar o reconhecimento do doador (pontos e **Selo Verde ESG**) no sistema da prefeitura ou da empresa parceira. Feito à mão, isso é repetitivo, sujeito a erro e pode premiar um lote com problema (avaria, divergência de quantidade, quebra da cadeia de frio).

## O que o robô faz (entrada → IA → saída)
`python executar_robo.py` executa, numa execução contínua, o processo modelado em BPMN (T1 a T12):
1. confere disco, memória e pastas (psutil) e sobe o portal SIMULADO se ele não estiver no ar;
2. lê os lotes de doação entregues (`entrada/lotes_entregues.json`) e as mensagens da IoT (`entrada/iot/*.log`; os dois logs de 10 e 11/11/2026 são **dado físico real** do ESP32 do grupo);
3. valida e manda o inválido para a quarentena; descarta duplicados (SQLite);
4. **IA:** spaCy extrai o kg recebido do texto do beneficiário e um classificador (TF-IDF + regressão logística, scikit-learn) diz se houve ocorrência (avaria, divergência, temperatura, texto ilegível), com uma confiança;
5. **regras inteligentes (R1 a R11):** ocorrência, baixa confiança, kg que não bate, alerta `quebra_cadeia_frio` da IoT ou problema marcado pelo beneficiário → **revisão humana**, sem pontos;
6. lote validado: calcula o **CO₂ evitado (kg CO₂eq)** = kg recebido validado × 2,54 (estimativa; FAO 2013), os **pontos** (10 por kg) e o **Selo Verde ESG** do doador pelo acumulado (Bronze 50 kg, Prata 200 kg, Ouro 500 kg);
7. gera o **Termo de Doação** em PDF e confere o PDF;
8. lança no portal SIMULADO com **Selenium** e confirma pela **API REST**;
9. grava as saídas, a fila de revisão, o log e o relatório de execução com tempo de ciclo e taxa de erros por tarefa.

## Como executar
Pré-requisitos: **Python 3.14** (testado com 3.14.4; também executado com 3.13.5) e **Microsoft Edge** (ou Chrome, com `--param NAVEGADOR=chrome`). Na primeira execução o Selenium Manager baixa o driver do navegador (precisa de internet).

```powershell
cd codigo

# 1. ambiente
py -3.14 -m venv .venv              # Linux/macOS: python3.14 -m venv .venv
.\.venv\Scripts\activate             # Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt

# 2. chaves (ver a seção abaixo)
copy .env.exemplo .env               # Linux/macOS: cp .env.exemplo .env

# 3. dados sintéticos, modelo e robô
python gerar_dados_sinteticos.py     # textos rotulados, 12 lotes de demonstração, prepara entrada/
python treinar_modelo.py             # treina e mede a IA (cerca de 1 min)
python executar_robo.py              # PONTO DE ENTRADA: roda o robô do início ao fim
```

Saídas em `codigo/saida/`: `termos/` (PDF), `revisao/fila_revisao.csv`, `relatorios/` (MD e JSON), `logs/`, `prints/` (telas do Selenium), `quarentena/`. O portal SIMULADO sobe sozinho em `http://127.0.0.1:5050` e é encerrado no fim. No Windows, se o console mostrar acentos trocados, rode antes `$env:PYTHONIOENCODING="utf-8"`.

| Comando (dentro de `codigo/`) | O que faz |
|---|---|
| `python executar_robo.py --limpar` | execução limpa (apaga saídas e banco do portal antes) |
| `python executar_robo.py --headful` | mostra o navegador |
| `python executar_robo.py --cenario portal_fora` | demonstra uma falha tratada (outros: `arquivo_ausente`, `iot_ausente`, `modelo_ausente`, `iot_invalido`, `portal_lento`, `erro_api`, `json_invalido`, `credencial_invalida`, `navegador_indisponivel`, `disco_cheio`) |
| `python executar_robo.py --agendar 10` | roda agora e a cada 10 minutos (Ctrl+C para parar) |

**Integração com o portal FOMEZERO (pasta `femd/` do repositório do grupo).** `python servico_aipa.py` (dentro de `codigo/`) sobe um serviço local em `http://127.0.0.1:5070`: quando o beneficiário confirma a entrega no portal, o lote vai para `POST /api/lotes` e o serviço executa **o mesmo robô** e devolve o resultado. Pelo serviço também passam a fila de revisão humana (`GET /api/revisao`, `POST /api/revisao/<lote_id>`), o Termo de Doação (`GET /api/termos/<lote_id>`) e o certificado do Selo Verde ESG em PDF (`GET /api/certificados/<doador_id>`). Aceita chamadas só deste computador.

**Revisão humana.** Lotes retidos vão para `saida/revisao/fila_revisao.csv`. O operador preenche `decisao` (`aprovado` ou `reprovado`), `kg_aprovado` e `revisor` (código, não nome) e salva como `entrada/decisoes_revisao.csv`; a próxima execução lê o arquivo. O robô nunca diz se o alimento é "seguro" ou "impróprio": quem decide é a pessoa.

## Como configurar as chaves
O robô não usa chave de serviço externo. As únicas credenciais são as do **portal SIMULADO** local (usuário e senha do login, token da API e chave de sessão), lidas de um arquivo `.env` que não vai para o repositório:
1. dentro de `codigo/`, copie `.env.exemplo` para `.env`;
2. troque os valores de exemplo (`PORTAL_USUARIO`, `PORTAL_SENHA`, `PORTAL_API_TOKEN`, `PORTAL_CHAVE_SESSAO`) por valores seus. O robô e o portal simulado leem o mesmo `.env`, então qualquer valor funciona desde que esteja nele.

Nenhuma senha ou token aparece no código ou no log. Sem o `.env`, o robô avisa no log e não lança nada.

## Ferramentas (todas gratuitas)
Python e bibliotecas de código aberto: selenium, flask, spacy, scikit-learn, pandas, openpyxl, pypdf, fpdf2, requests, schedule, psutil, python-dotenv, joblib (versões em `codigo/requirements.txt`). Navegador Microsoft Edge. Nenhum serviço pago e nenhuma API externa.
