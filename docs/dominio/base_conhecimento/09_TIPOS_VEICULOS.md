# Tipos de Veículos (vehicle_types)

## Veículos para Carga Fechada

| Nome | Capacidade (ton) | Usado em GO | Usado em Outras UFs |
|------|-----------------|-------------|---------------------|
| 14 TON | 14 | Sim | Sim |
| 18 TON | 18 | Sim | Não |
| 28 TON | 28 | Sim | Sim |
| 32 TON | 32 | Sim | Sim |
| 38 TON | 38 | Sim | Sim |
| 50 TON | 50 | Sim | Sim |

## Veículos para Carga Fracionada

| Nome | Capacidade (ton) | Observação |
|------|-----------------|------------|
| 14 TON (Fracionada) | 14 | Apenas GO, até 4 entregas |

## Observações
- Frete GO: faixas de distância de 1 a 600 KM (9 faixas)
- Frete Outras UFs: faixas de distância de 201 a 1500 KM (16 faixas)
- No banco: campo `IsFractional` diferencia carga fechada de fracionada
- No banco: campo `OperationType` diferencia GO (SAME_STATE=2) de Outras UFs (EXTERNAL_STATE=1)
- 18 TON existe apenas na tabela GO (não aparece na tabela Outras UFs)
- Cargas fracionadas CIF para Outras UFs devem ser SPOT (consultar custo individual)
