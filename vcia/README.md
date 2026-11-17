# FOMEZERO · Triagem inteligente por foto (Visão Computacional)

Global Solution 2026.2 · FIAP · Tecnólogo em Inteligência Artificial · turma 2TIAPR-2026 · tema CitizenTech.
Repositório do grupo (pasta `vcia/`): https://github.com/lostgear84/GS_2TIAPR_2026_FOMEZERO_GABA/tree/main/vcia

> **Apoio à triagem visual. A imagem não atesta que o alimento é seguro para consumo; a decisão é do doador e da vigilância sanitária.**

## Conteúdo desta entrega ([§4] do enunciado)
| Entrega | Onde |
|---|---|
| (a) Notebook ou código executável comentado | `codigo/notebook/` (scripts `01_` a `07_` + `comum.py`: preparação dos dados, modelo, treino e avaliação) e `codigo/app/app.py` |
| (b) Vídeo de demonstração | https://youtu.be/1kg761jR7pA (também em `video/link do vídeo.txt` e `video/VIDEO.md`) e `video/bancada_sequencia_efficientnet_b0.mp4` (sequência de quadros) |
| (c) Relatório técnico breve | `RELATORIO_TECNICO_VCIA.pdf` |

## Integrantes
| Nome | RM |
|---|---|
| Erick Miguel | 564896 |
| Leonardo Melo | 567097 |
| Luna Oliveira | 564215 |

## Nicho e solução
**Nicho [§1.b, interesse social]: triagem visual de hortifrúti excedente antes do cadastro do lote de doação.** O doador fotografa **um item de hortifrúti por vez, de cima, sobre fundo claro**. Uma CNN (EfficientNet-B0, transferência de aprendizado) **sugere** a condição visual e o produto, com a confiança. No portal do FOMEZERO (tela "Triagem inteligente por foto (Visão Computacional)"), a sugestão pré-preenche o campo "Condição informada pelo doador" do lote de doação, e o doador pode editá-la; se ele usar um resultado de deterioração, o lote vai para **compostagem** e não aparece para os beneficiários.

| Resultado (teste, n = 786 fotos originais) | EfficientNet-B0 |
|---|---|
| F1 macro da condição (3 classes) | 0,940 |
| F1 macro do produto (8 classes) | 0,993 |
| Inferência por imagem, 224×224 (RTX 3050 6 GB Laptop / i5-13420H) | 17,1 ms GPU · 44,1 ms CPU |

Métricas completas, latência, justificativa da arquitetura e análise dos erros: `RELATORIO_TECNICO_VCIA.pdf`.

## Como executar (Windows, Python 3.13; só ferramentas gratuitas)
```powershell
cd codigo
python -m venv .venv
.venv\Scripts\pip install torch torchvision --index-url https://download.pytorch.org/whl/cu130
.venv\Scripts\pip install scikit-learn gradio opencv-python-headless matplotlib
```
Sem placa NVIDIA, instale o PyTorch para CPU (`pip install torch torchvision`).

**Só o app** (usa `modelo/efficientnet_b0.pt` e os JSON de `evidencias/`; não precisa da base):
```powershell
.venv\Scripts\python app\app.py
```
Abre em http://127.0.0.1:7860 (abas Imagem e Sequência; endpoint `/triagem`, usado pelo portal). Com `--share`, gera um link público temporário do Gradio. Sem GPU, roda na CPU (cerca de 44 ms por imagem).

**No portal do FOMEZERO** (pasta `femd/` do repositório do grupo): com o app no ar, entre como doador e abra "Triagem inteligente por foto (Visão Computacional)".

**Refazer tudo:** baixe a AgriFreshNET (DOI abaixo) e extraia `Processed Data`. Dentro de `codigo/notebook/`, rode nesta ordem (`D` = pasta `Processed Data`, `E` = `../evidencias`, `M` = `../modelo`):

| # | Comando | Gera |
|---|---|---|
| 1 | `python 01_preparar_dados.py D E` | divisão por grupo de origem sem vazamento (`divisao*.{json,csv}`) |
| 2 | `python 02_treinar.py --modelo efficientnet_b0 --dados D --divisao E --saida M` (repetir com `mobilenet_v3_large`) | pesos `M/*.pt`, `treino_*.json` |
| 3 | `python 03_avaliar.py --modelo efficientnet_b0 --dados D --divisao E --pesos M/efficientnet_b0.pt` | métricas do teste, τ, matrizes, galeria |
| 4 | `python 04_interferencias.py …` (mesmos argumentos do passo 3) | interferências sintéticas |
| 5 | `python 05_desempenho.py … --video ../../video/bancada_sequencia_efficientnet_b0.mp4` (`--so_video` refaz só o vídeo) | latência, FPS, hardware, vídeo da sequência |
| 6 | `python 06_nitidez.py …` | limiar de nitidez (validação) e efeito no teste |
| 7 | `python 07_gradcam.py …` | Grad-CAM de acertos e erros (qualitativo) |

`notebook/preliminar/` tem os scripts de apoio: inspeção da base, mosaico de amostra, agrupamento por origem, comparação das variantes da medida de nitidez e tempo da nitidez.

## Fontes e licenças
- **Base:** AgriFreshNET Freshness and Shelf-Life Image Dataset — Risdin, F.; Salauddin, H.; Tufa, T. A.; Hossain, M. J. (2026). Mendeley Data, v1. DOI 10.17632/42m5tb7yv9.1. **CC BY 4.0**. As fotos em `codigo/app/exemplos/` e nas figuras são dessa base (atribuição em `codigo/app/exemplos/LEIAME.md`). A base em si não vai junto.
- **Modelos e bibliotecas:** pesos ImageNet da torchvision (EfficientNet-B0, MobileNetV3-Large), que podem ter termos próprios (uso acadêmico); PyTorch (licença estilo BSD), torchvision e scikit-learn (BSD-3-Clause), Gradio e OpenCV (Apache-2.0); NumPy, Pillow e Matplotlib.
- Documentos de trabalho (rastreabilidade, contrato CT-08, decisões, pesquisa de bases e fontes): repositório de desenvolvimento, https://github.com/meloleonardo-dev/GS_2TIAPR_2026_FOMEZERO_VISAO_COMPUTACIONAL
