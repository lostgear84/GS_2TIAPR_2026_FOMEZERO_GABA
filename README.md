# Dashboard de Governança e Impacto

O dashboard é uma proposta extra do FOMEZERO. Ele complementa a demonstração da integração, mas não substitui o vídeo, o repositório GitHub ou o documento ABNT/FIAP.

## Indicadores previstos

- Lotes cadastrados, entregues, processados, em revisão humana e com falha.
- Massa recebida validada, em kg.
- CO₂ evitado estimado, em kg CO₂eq.
- Pontos acumulados por doador.
- Selo Verde ESG: Bronze, Prata e Ouro.
- Ranking ESG dos parceiros.
- Lotes monitorados e lotes com alerta térmico.
- Percentual de lotes em revisão humana.
- Aviso de que os dados são sintéticos e exigem validação humana.

## Fonte dos indicadores

Os indicadores devem ser alimentados pelos arquivos gerados pela AIPA:

- `saida/ct05/indicadores.json`
- `saida/ct05/historico_entregas.csv`

Valores não calculados devem ser exibidos como `null`, e não como zero.