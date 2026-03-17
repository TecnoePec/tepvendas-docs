# Mapeamento: Dados Comerciais → Banco de Dados

## Projeto: tep_vendas_services
- **Tecnologia:** .NET Core + Entity Framework Core + PostgreSQL
- **Localização:** /Users/julianomenezes/Documents/Tecnoepec/tep_vendas_services
- **DB Docker:** Host=localhost, Port=5432, User=tep_sales, Password=tep_sales_dev, DB=tep_sales_db

---

## ESQUEMA RELACIONAL

```
product_groups (Família: Ração, Premium, Núcleo, Mineral, Proteico, Silos)
  └── products (Produtos individuais)
       └── product_especifications (Key-Value: Sacaria, Bag, etc.)

price_tables (Tabela principal: GO ou Outras UFs)
  └── payment_price_tables (1 por condição de pagamento)
       └── price_table_items (1 por produto, com o valor R$)

payment_conditions (À vista, 7 dias, 14 dias, 28 dias, etc.)

vehicle_types (14 TON, 18 TON, 28 TON, 32 TON, 38 TON, 50 TON)

freight_tables (Frete por: distância + veículo + condição pagamento + operação)

discount_weights (Desconto por volume: tonelagem → percentual)

discount_rules (Desconto especial: por produto/grupo/linha)

price_table_unloadings (Descarga: por grupo de produto + condição pagamento)
```

---

## ENTIDADES E CAMPOS DETALHADOS

### price_tables
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| Name | varchar(255) | Ex: "Tabela GO 24/07/2025" |
| OperationType | int | 1=EXTERNAL_STATE(Outras UFs), 2=SAME_STATE(GO) |
| Description | text | Descrição livre |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |
| CompanyId | uuid | FK para Company |

### payment_price_tables
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| PaymentConditionId | uuid | FK para payment_conditions |
| PriceTableId | uuid | FK para price_tables |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

### price_table_items
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| PaymentPriceTableId | uuid | FK para payment_price_tables |
| ProductId | uuid | FK para products |
| Value | double | Preço do produto nesta condição |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

### products
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| ProductGroupId | uuid | FK para product_groups (Família) |
| ProductLineId | uuid | FK para product_lines |
| Name | varchar(255) | Nome do produto |
| Description | varchar(1000) | Descrição |
| Weightkilograms | double | Peso em KG |
| UnitMeasurementType | int | 1=KG, 2=UN |
| Photo | varchar(500) | URL da foto |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

### product_groups
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| Name | varchar(255) | Ex: "Ração", "Premium", "Núcleo" |
| Description | text | Descrição |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

### payment_conditions
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| Name | varchar(255) | Ex: "À vista", "14 dias" |
| Sequence | int | Ordem de exibição |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

### vehicle_types
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| Name | varchar(255) | Ex: "14 TON", "50 TON" |
| Capacity | double | Capacidade padrão (ton) |
| MaxCapacity | double | Capacidade máxima (ton) |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

### freight_tables
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| PaymentConditionId | uuid | FK para payment_conditions |
| VehicleTypeId | uuid | FK para vehicle_types |
| InitialKilometer | double | KM inicial da faixa |
| FinalKilometer | double | KM final da faixa |
| Value | double | Valor do frete (R$/kg) |
| IsFractional | bool | Carga fracionada? |
| OperationType | int | 1=EXTERNAL_STATE, 2=SAME_STATE |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

### discount_weights
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| Quantity | double | Tonelagem mínima (ex: 4, 6, 8, 12, 18, 24) |
| Percent | double | Percentual de desconto |

### discount_rules
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| DiscountWeightType | int | 1=USE_KG, 2=USE_UNIT |
| MinQuantity | double | Quantidade mínima |
| DiscountType | int | 1=PERCENT, 2=VALUE |
| ReferenceType | int | 1=LINE, 2=GROUP, 3=PRODUCT |
| ReferenceId | uuid | FK para ProductLine/ProductGroup/Product |
| Discount | double | Valor ou percentual de desconto |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

### price_table_unloadings
| Campo | Tipo | Descrição |
|-------|------|-----------|
| Id | uuid | PK |
| ProductGroupId | uuid | FK para product_groups |
| PaymentConditionId | uuid | FK para payment_conditions |
| ValueOfKG | double | Valor R$/KG para descarga |
| ExternalCode | varchar(100) | Código externo |
| Status | int | Active/Inactive |

---

## ENUMS IMPORTANTES

```
TablePriceOperationTypeEnum:
  EXTERNAL_STATE = 1  (Outras UFs)
  SAME_STATE = 2      (GO)

DiscountTypeEnum:
  PERCENT = 1
  VALUE = 2

DiscountWeightTypeEnum:
  USE_KG = 1
  USE_UNIT = 2

ReferenceTypeEnum:
  LINE = 1      (Linha de Produto)
  GROUP = 2     (Grupo/Família)
  PRODUCT = 3   (Produto específico)

UnitMeasurementTypeEnum:
  KG = 1
  UN = 2
```

---

## STATUS DO BANCO (11/02/2026)
- Banco existe e está rodando via Docker
- **Todas as tabelas comerciais estão VAZIAS** (0 registros)
- Apenas `companies` (2), `users` (1) e `refresh_tokens` (3) possuem dados
- É necessário popular com os dados das tabelas de preço/frete/desconto
