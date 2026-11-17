# Evidências de validação: robô pós-entrega do FOMEZERO (AIPA)

Tudo abaixo saiu de execução real do robô (13/11/2026; repetida em 17/11/2026 a partir do código desta entrega, com o mesmo resultado). Dados de lote **SINTÉTICOS (SIMULADO)**; portal **SIMULADO**; logs da IoT **físicos reais** (10 e 11/11/2026). Os arquivos completos (logs, PDFs, prints e as 11 falhas simuladas) estão no repositório, pasta `evidencias/`: https://github.com/meloleonardo-dev/GS_2TIAPR_2026_FOMEZERO_AIPA

## 1. Execução com dados de teste
Comando: `python executar_robo.py --limpar` (dentro de `codigo/`).

```
== EXEC-20261113-215131 | tratados 11 | processado 3 | revisão humana 6 | falha 2 | duplicados 1 | não entregues 1 | taxa de erros 0.1818 | 13.31 s
   gabarito: 12 de 13 registros conferem
```

- 13 registros (12 lotes de demonstração + 1 em transporte) em 13,3 s.
- 3 processados: Termo de Doação em PDF, lançamento pelo Selenium e confirmação pela API. O DOADOR-001 chega a 57 kg e ganha o Selo Verde ESG Bronze.
- 6 em revisão humana, 2 inválidos em quarentena, 1 duplicado ignorado.
- Tempo de ciclo por lote: média 484 ms, máximo 2,9 s. A tarefa mais lenta é o lançamento no portal (T9, média 2,8 s).
- O LOTE-2026-0001, cruzado com o alerta **físico real** `quebra_cadeia_frio` do log de 11/11/2026, foi para revisão humana, como esperado.

## 2. Resultado medido (acertos e erros)

**Processo, contra o gabarito:** 12 de 13 registros corretos (11 de 12 lotes de demonstração). O erro: o LOTE-2026-0006 ("recebemos 6 kg de refeição pronta, muito obrigado") foi classificado como avaria, com confiança 0,63, e foi para revisão humana em vez de seguir sozinho. É um erro conservador: nenhum lote com problema passou sem revisão.

**IA de ocorrência** (modelo `ocorrencia-v2`, TF-IDF + regressão logística), em três conjuntos:

| conjunto | n | acurácia v1 / v2 / palavras-chave | F1 macro v1 / v2 / palavras-chave | falso automático v1 / v2 / palavras-chave |
|---|---|---|---|---|
| teste sintético | 75 | 0,987 / 1,000 / 0,920 | 0,987 / 1,000 / 0,921 | 0 / 0 / 1 |
| desenvolvimento ("fora do molde") | 30 | 0,467 / 1,000 / 0,400 | 0,461 / 1,000 / 0,252 | 0 / 0 / 10 |
| **cego v2 (medição única)** | 60 | **0,650 / 0,733 / 0,483** | **0,640 / 0,734 / 0,487** | **1 / 0 / 13** |

"Falso automático" é um texto com ocorrência real que seguiria sem revisão humana (a meta é 0). O conjunto que mede generalização é o **cego v2**: 60 textos novos, congelados com SHA-256 antes de qualquer ajuste no modelo e medidos uma única vez. Nele a v2 acerta 44 de 60 (a v1, 39; as palavras-chave, 29), e nenhum texto com ocorrência seguiria sozinho.

Acertos da v2 por classe no cego:

| classe | acertos |
|---|---|
| sem_ocorrencia | 8 de 12 |
| avaria | 9 de 12 |
| divergência | 7 de 12 |
| temperatura | 9 de 12 |
| ilegível | 11 de 12 |

Extração do kg: 51 de 60, sem nenhum kg inventado em texto que não tinha kg.

O que ainda erra:
- **negações**, como "nao faltou nada", que vira divergência;
- **divergência sem a palavra "falta"**, como "veio menor, 7 kg em vez de 10";
- **temperatura dita pela consequência**, como "chegou abafado".

Os textos com ocorrência que a v2 previu como `sem_ocorrencia` tiveram confiança abaixo do limiar de 0,70, então iriam para revisão humana.

**Limites:** n = 60 é pequeno, e os textos são sintéticos e não passaram por revisão humana. Com linguagem real, o robô funciona mais como triagem do que como automação completa.

**Testes automáticos:** `python -m pytest testes -v`, com 68 testes aprovados em 17/11/2026, cobrindo extração, regras, cálculos, adaptadores, idempotência, contrato JSON, Selenium, Termo e certificado em PDF.

## 3. Caso de falha tratado: portal fora do ar (F6)
Comando: `python executar_robo.py --cenario portal_fora`. O robô terminou com código de saída 0.

```
| T1 | health tentativa 1/3 falhou: ConnectionError
| T1 | health tentativa 2/3 falhou: ConnectionError
| T1 | health tentativa 3/3 falhou: ConnectionError
| T1 | portal SIMULADO fora do ar após as tentativas; lotes válidos ficam em falha portal_indisponivel e são retomados na próxima execução
| T9 | LOTE-2026-0002 | falha tratada: portal_indisponivel (health falhou no T1)
| T9 | LOTE-2026-0003 | falha tratada: portal_indisponivel (health falhou no T1)
```

O robô não travou nem lançou nada pela metade. Os dois lotes ficaram em `falha` com o motivo registrado.

**Retomada** com o portal normal, na mesma pasta:

| lote_id | status | protocolo |
|---|---|---|
| LOTE-2026-0002 | processado | SIM-000001 |
| LOTE-2026-0003 | processado | SIM-000002 |

```
| T4 | LOTE-2026-0002 | retomando lote pendente da etapa T9 (motivo anterior ['portal_indisponivel'])
| T9 | LOTE-2026-0002 | lançamento confirmado pela API REST (protocolo SIM-000001, meio selenium)
```

## 4. Idempotência (rodar duas vezes não lança de novo)

| execução | lançamentos no portal (total) | processados nesta execução | duplicados ignorados |
|---|---|---|---|
| 1ª EXEC-20261113-215131 | 3 | 3 | 1 |
| 2ª EXEC-20261113-215150 | 3 | 0 | 12 |

Outros cenários de falha demonstrados no repositório:

| código | falha | o que o robô fez |
|---|---|---|
| F1 | arquivo de lotes ausente | parou sem lançar nada |
| F2 | IoT ausente | mandou os lotes para revisão humana |
| F3 | modelo de IA ausente | mandou os lotes para revisão humana |
| F4 | IoT com dado inválido | mandou os lotes para revisão humana |
| F7 | portal lento (timeout) | marcou falha |
| F8 | erro HTTP 500 na API | marcou falha |
| F8b | JSON inválido na API | marcou falha |
| F9 | senha errada no login | marcou falha |
| F10 | navegador não abre | lançou pela API REST, como contingência |
| F11 | disco cheio | abortou com segurança, sem lançar nada |
