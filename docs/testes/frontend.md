# Testes — Frontend (Next.js 14)

Suite de testes automatizada do `tep_vendas_backoffice`, executada localmente e no CI (CodeBuild).

## Estado atual

| Métrica | Valor |
|---|---|
| **Total** | 1.163 testes (130 test suites) |
| **Passando** | 1.113 |
| **Falhando** | 50 |
| **Suites com falha** | 10 |
| **Framework** | Jest 29 + React Testing Library |
| **Reporter CI** | Jest JUnit + Jest Coverage Report Action |

!!! warning "50 testes vermelhos"
    Todos concentrados em `__tests__/pages/registration/*/{index,detail,new}.test.tsx` + `utils/errors.test.ts`. São falhas de "should render the page" — provavelmente mocks de MUI/router quebrados após upgrade. Não bloqueiam deploy porque o CI hoje roda `npm test || true` (deveria ser removido — ver seção "Débitos técnicos" abaixo).

## Como rodar

**Toda a suite:**
```bash
cd tep_vendas_backoffice
npm test
```

**Watch mode:**
```bash
npm run test:watch
```

**Coverage:**
```bash
npm run test:coverage
```

**Só um arquivo:**
```bash
npm test -- __tests__/components/Copyright.test.tsx
```

**Só por nome:**
```bash
npm test -- -t "should render"
```

## Estrutura

```
__tests__/
├── components/         # Widgets isolados (Copyright, Logo, ...)
├── hooks/              # Custom hooks (useIsMountedRef, useAuth, ...)
├── pages/
│   ├── account/
│   ├── integration/
│   ├── management/
│   └── registration/   # ← 10/11 suites vermelhas aqui
├── services/           # API clients (Axios)
└── utils/              # Formatters, helpers
```

## No CI (CodeBuild)

Roda no stage BUILD do `buildspec.yml`:

```yaml
- npm ci
- npm run lint
- npm test -- --ci --coverage --maxWorkers=2 || true
- npm run build
```

O `|| true` mascara falhas — deveria sair depois que as 50 regressões forem corrigidas.

## Convenções

- **`describe('ComponentName', () => { … })`** — um bloco por componente/hook/service.
- **`it('should …')`** em inglês — herança do template Next.js.
- **`render` da React Testing Library** — nunca `enzyme` (aposentado).
- **Mocks de MUI:** ao mockar `Dialog`, garantir que `keepMounted` case com o que o componente sob teste espera.
- **Router:** `next/router` mockado via `jest.setup.js` — todos os testes têm `push`, `replace`, `query`, `pathname` disponíveis.

## Áreas com cobertura relevante

- **Auth flow:** login, session refresh, guards de rota.
- **Data grid components:** filtros, paginação, sort.
- **Services layer:** cliente Axios com interceptors de auth + retry.
- **Formatters:** `formatNumber`, `formatDate`, `formatCurrency`.

## Débitos técnicos

- **Remover `|| true` do CI** após corrigir os 50 testes de `pages/registration/*`.
- **Cobertura de código sem gate:** hoje não tem threshold; adicionar `jest.config.js > coverageThreshold` (ex: 60% lines).
- **`utils/errors.test.ts`** — suite falha ao carregar (module not found ou similar); rebuild da tipagem pode resolver.
- Muitos testes de página são "should render the page" — pouca asserção real. Considerar migrar pra testes de comportamento (ex: "clicar em X abre modal Y").

## Troubleshooting

**`Test suite failed to run — Cannot find module`** — geralmente path alias (`@/…`) não resolvendo. Confere `jest.config.js > moduleNameMapper`.

**`Warning: An update to X inside a test was not wrapped in act(...)`** — usa `await waitFor(() => …)` em torno da asserção que depende do state update.

**MUI `Portal` cria conteúdo fora do container do teste** — usa `screen.getByRole('dialog')` em vez de `container.querySelector`.
