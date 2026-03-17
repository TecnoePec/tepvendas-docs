# Tabela de Descontos para Vendas - Major Confina
## Vigência: A partir de 19/05/2023

---

## DESCONTO POR VOLUME (aplica-se a TODOS os produtos)

| Faixa de Peso | Desconto por Volume |
|---------------|---------------------|
| Acima de 4 ton | 1% |
| Acima de 6 ton | 1% |
| Acima de 8 ton | 2% |
| Acima de 12 ton | 3% |
| Acima de 18 ton | 3% |
| Acima de 24 ton | 4% |

> Mapeamento no banco: tabela `discount_weights`
> - Quantity = tonelagem mínima
> - Percent = percentual de desconto

---

## DESCONTO ESPECIAL (*) - Produtos específicos

| Faixa de Peso | Desconto Especial |
|---------------|-------------------|
| Acima de 4 ton | 9% |
| Acima de 6 ton | 10% |
| Acima de 8 ton | 10% |
| Acima de 12 ton | 11% |
| Acima de 18 ton | 12% |
| Acima de 24 ton | 12% |

### Condicionantes do desconto especial:
- **Produtos elegíveis:** Engorda 20.0 | Energético 5.0 | Top Pasto
- **Demais produtos:** Seguem apenas desconto por volume (1ª coluna)

> Mapeamento no banco: tabela `discount_rules`
> - ReferenceType = PRODUCT (3)
> - ReferenceId = ID do produto (Engorda 20.0, Energético 5.0, Top Pasto)
> - DiscountType = PERCENT (1)
> - MinQuantity = tonelagem mínima
> - Discount = percentual de desconto especial

---

## DESCONTO TOTAL (Volume + Especial) - Apenas para produtos elegíveis

| Faixa de Peso | Volume | + | Especial | = | Total |
|---------------|--------|---|----------|---|-------|
| Acima de 4 ton | 1% | + | 9% | = | **10%** |
| Acima de 6 ton | 1% | + | 10% | = | **11%** |
| Acima de 8 ton | 2% | + | 10% | = | **12%** |
| Acima de 12 ton | 3% | + | 11% | = | **14%** |
| Acima de 18 ton | 3% | + | 12% | = | **15%** |
| Acima de 24 ton | 4% | + | 12% | = | **16%** |

---

## Regra de Desconto para Silos
- Para vendas de Silos, pode-se aplicar desconto de **15%** de acordo ao volume de produtos de nutrição atrelados ao pedido
