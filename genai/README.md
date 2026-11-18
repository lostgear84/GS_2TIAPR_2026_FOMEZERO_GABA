# FOMEZERO · Assistente de consulta para doadores (Generative AI & Advanced Nets)

Global Solution 2026.2 · FIAP · Tecnólogo em Inteligência Artificial · turma 2TIAPR-2026 · tema CitizenTech.
Repositório do grupo (pasta `genai/`): https://github.com/lostgear84/GS_2TIAPR_2026_FOMEZERO_GABA/tree/main/genai

O FOMEZERO liga quem tem comida sobrando (o **doador**) a quem precisa (o **beneficiário**), acompanhando cada **lote de doação**. Esta parte é o **assistente de consulta para doadores**: responde dúvidas sobre o que doar, como conservar e como transportar, com **RAG** (busca os trechos certos em documentos oficiais e gera uma resposta curta, citando a fonte). Os documentos são os mesmos da matéria de PLN (corpus do contrato CT-01).

> **O assistente nunca diz que um alimento é seguro para consumo e nunca decide sobre um lote.** Quando a pergunta pede essa decisão, ele mostra o que os documentos dizem e encaminha à vigilância sanitária do município. Quando os documentos não respondem, ele diz que não encontrou a informação.

## Conteúdo desta entrega ([§4] do enunciado)
| Entregável | Onde |
|---|---|
| 1. Aplicação funcional | interface Gradio aberta pela seção N10 de `codigo/assistente_fomezero.ipynb` (abas Consulta, Comparação S × P × R e Avaliação) e a tela "Tirar dúvidas com o assistente" do portal FOMEZERO (pasta `femd/` do repositório do grupo) |
| 2. Código-fonte | `codigo/assistente_fomezero.ipynb` (seções N0 a N11) e `codigo/assistente/` |
| 3. Documentação técnica (com capa: nomes e RM) | `DOCUMENTACAO_TECNICA.pdf` |
| 4. Evidências de funcionamento e avaliação | `codigo/evidencias/` (índice em `EVIDENCIAS.md`, prints da interface em `prints/`, notebook executado no Colab com as saídas, testes locais) e `codigo/avaliacao/` (perguntas de teste, respostas S, P e R, métricas e avaliação humana); resumo e análise na documentação técnica, seções 6 e 7 |
| 5. Demonstração em vídeo | https://youtu.be/k2YepWTsgKo (também em `video/link do vídeo.txt`) |

## Integrantes
| Nome | RM |
|---|---|
| Erick Miguel | 564896 |
| Leonardo Melo | 567097 |
| Luna Oliveira | 564215 |

## Como executar no Google Colab (GPU)
O corpus **não** é copiado para a GENAI: ele é lido da entrega da PLN (pasta `pln/` do repositório do grupo), e a seção N1 confere o SHA-256 dos três arquivos antes de começar.

1. No Google Drive, crie `MyDrive/GS_FOMEZERO/` e coloque lá, lado a lado:
   - o conteúdo de `codigo/` desta entrega, numa pasta chamada `GENAI`;
   - a pasta da PLN (com `corpus/`, `saidas/`, `revisao/`, `documentos/` e `fabrica/`), numa pasta chamada `PLN`.
2. Abra `GENAI/assistente_fomezero.ipynb` com o Google Colaboratory e troque o ambiente para **GPU T4** (*Ambiente de execução > Alterar o tipo de ambiente de execução*).
3. **Executar tudo.** A N0 monta o Drive e instala o `requirements.txt` (se aparecer erro de importação do Pillow, use *Reiniciar sessão* e rode de novo). Com `REGERAR = False` (padrão), o notebook valida o corpus, monta o índice, carrega o Qwen2.5-7B, mostra exemplos, recalcula as métricas a partir de `avaliacao/` e abre a interface Gradio com um link público temporário (saída da N10).
4. Para gerar de novo as 126 respostas da avaliação S × P × R, mude `REGERAR = True` na N0 e rode a N9.

| Seção | O que faz |
|---|---|
| N0 | monta o Drive, instala as bibliotecas e define as chaves `REGERAR`, `USAR_LLM` e `LANCAR_GRADIO` |
| N1 | carrega e valida o corpus da PLN (CT-01) pelo hash e pelas contagens |
| N2 | monta a base de conhecimento (149 trechos com documento, URL e página ou artigo) |
| N3 | embeddings `multilingual-e5-small` e índice FAISS |
| N4 | recuperador (6 trechos mais próximos) |
| N5 | carrega o Qwen2.5-7B-Instruct em 4 bits |
| N6 | prompts das configurações R, P e S |
| N7 | política de resposta antes do modelo (responder, encaminhar, fora do tema, manipulação) e limiar de abstenção τ |
| N8 | `consultar()`: rota → recuperação → geração → verificação das citações e das frases proibidas → saída no contrato CT-02 |
| N9 | avaliação S × P × R (42 perguntas de teste) e da recuperação (30 perguntas), com a avaliação humana |
| N10 | interface Gradio e endpoint `/consultar` (usado pelo portal) |
| N11 | grava as respostas reais do modo simulado do portal (`contrato/respostas_SIMULADO.json`) |

## Como executar localmente (sem GPU, testes)
Dentro de `codigo/`. O código acha sozinho a pasta da PLN (no repositório do grupo, `../../pln`); em outro lugar, defina `GENAI_PASTA_PLN`.
```powershell
python -m venv .venv
.venv\Scripts\python -m pip install torch --index-url https://download.pytorch.org/whl/cpu
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\python -I testes_locais.py
```
Sem GPU, o Qwen não é carregado: as respostas "geradas" vêm de um **stub rotulado SIMULADO**, só para testar o caminho. A busca (e5 + FAISS), o limiar τ, as rotas e os filtros rodam de verdade em CPU.

## No portal FOMEZERO
Com o notebook no ar, entre como doador, abra **Tirar dúvidas com o assistente**, escolha o modo **Integrado** e cole o link do Gradio (N10). O portal envia só a pergunta (contrato CT-02, sem dado do lote). Sem o Colab, o portal usa o modo **Simulado**, com 10 respostas reais gravadas pela N11.

## Ferramentas (todas gratuitas)
`intfloat/multilingual-e5-small` (MIT), `Qwen/Qwen2.5-7B-Instruct` (Apache-2.0), sentence-transformers, faiss-cpu, transformers, langchain-core, gradio, accelerate e bitsandbytes, torch, Google Colab com GPU T4. Versões e revisões em `codigo/requirements.txt` e na documentação técnica (seção 3). Nenhuma chave de API é usada.

## Dados pessoais
Nenhum dado pessoal real. CPF, e-mail e telefone digitados na pergunta são mascarados antes de qualquer registro ou chamada ao modelo. O link público do Gradio só existe enquanto a sessão do Colab está aberta.
