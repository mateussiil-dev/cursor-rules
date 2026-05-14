# Regras do Cursor (`.cursor/rules`)

Arquivos `.mdc` com frontmatter (`description`, `globs`, `alwaysApply`) usados pelo Cursor para orientar geração e refatoração de código neste repositório.

| Arquivo | Escopo (resumo) |
|--------|------------------|
| `no-generic-coercion-helpers.mdc` | Sem helpers `str`/`num`/`parseDate` genéricos; DTOs na borda; um mapper estrutural de contrato OK |
| `record-row-and-slice-mapping.mdc` | `Record<string, unknown>` e slices com IDs paralelos |
| `prefer-parameter-objects-many-args.mdc` | Mais de dois parâmetros → objeto único |
| `prefer-clean-guard-clause-functions.mdc` | Guard clauses legíveis; sem introspecção pesada sem alinhamento |
| `no-unknown-shape-type-assertions.mdc` | Sem `as { ... }` em cima de `unknown` “no escuro” |
| `no-nullish-null-coercion.mdc` | Sem `?? null` espalhado |
| `no-generic-http-response-unwrapping-helper.mdc` | Resposta HTTP com formato fixo → inline, sem helper genérico |
| `prefer-descriptive-loop-and-callback-names.mdc` | Laços/callbacks com nomes do domínio; `x`/`y`/`z` só em contexto matemático |
| `prefer-simple-callback-narrowing.mdc` | `.filter` simples (`typeof x === 'string'`); evitar `(x): x is string =>` por padrão; se o TS reclamar, perguntar ao usuário |

---

## `no-generic-coercion-helpers.mdc`

**Ideia:** coerção e “limpeza” de valores soltos não viram utilitário reutilizável; contrato explícito no gateway e na borda HTTP.

```ts
// Evitar — helpers genéricos de uma linha
function str(v: unknown) { return v == null ? '' : String(v); }
function num(v: unknown) { return Number(v); }
```

```ts
// Preferir — DTO na borda + tipos no gateway
class RemoteThingDto {
  @IsString()
  name!: string;
}
```

```ts
// Exceção aceita — um método privado só de mapeamento estrutural (contrato fixo)
private mapContract(remote: RemoteContractDto): ContractSlice {
  return {
    id: Number(remote.id),
    signedAt: new Date(remote.signedAt),
    status: remote.status,
  };
}
```

---

## `record-row-and-slice-mapping.mdc`

**Ideia:** linhas dinâmicas e slices montados com índices paralelos — mapeamento direto; sem “normalizar” opcional com `String`; sem `?? null` só para encaixar tipo quando o paralelismo já garante o valor.

```ts
// Evitar — opcional com ternário + String
name: row.name === undefined || row.name === null ? null : String(row.name),
```

```ts
// Preferir — direto + alinhamento de tipo quando o contrato da origem é conhecido
name: row.name as string | undefined,
```

```ts
// Evitar — ?? null em ID paralelo só para satisfazer | null
items.push({
  proposalId: proposalIds[i] ?? null,
  contractId: contractIds[i] ?? null,
});
```

```ts
// Preferir — asserção alinhada ao índice paralelo já garantido pelo loop
items.push({
  proposalId: proposalIds[i] as AggregatedRow['proposalId'],
  contractId: contractIds[i] as AggregatedRow['contractId'],
});
```

Para política geral de `null`, ver exemplos em `no-nullish-null-coercion.mdc`.

---

## `prefer-parameter-objects-many-args.mdc`

**Ideia:** funções novas com **mais de dois** parâmetros posicionais → um único objeto (exceto callbacks/framework).

```ts
// Evitar — muitos argumentos posicionais
async load(profileId: string, withinDays: number, includeHistory: boolean, cursor?: string) { }
```

```ts
// Preferir — objeto de parâmetros
async load(params: {
  profileId: string;
  withinDays: number;
  includeHistory: boolean;
  cursor?: string;
}) {
  const { profileId, withinDays, includeHistory, cursor } = params;
}
```

---

## `prefer-clean-guard-clause-functions.mdc`

**Ideia:** retorno cedo, um critério por vez; contrato de paginação fixo sem cadeia de guards desnecessária; evitar `in` / `Reflect.get` / `Array.isArray` para fuçar payload — se precisar, alinhar com o time/usuário.

```ts
// Evitar — mega condição
if (a && b && c && response && response.data && Array.isArray(response.data.items)) { }
```

```ts
// Preferir — guards simples
if (!response?.data) return [];
return response.data.items ?? [];
```

---

## `no-unknown-shape-type-assertions.mdc`

**Ideia:** não “ensinar” formato com `as { ... }` em cima de `unknown` sem validação; preferir DTO na borda ou resposta já tipada.

```ts
// Evitar
const items = (raw as { items?: unknown }).items as Foo[];
```

```ts
// Preferir — contrato fixo / cliente tipado
const items = response?.data?.items ?? [];
```

Introspecção com `'…' in obj`, `Reflect.get`, `Array.isArray`: **só com alinhamento prévio** com quem define o padrão (ver texto da regra).

---

## `no-nullish-null-coercion.mdc`

**Ideia:** não usar `?? null` só para trocar `undefined` por `null` no meio do domínio; `null` na serialização HTTP fica no DTO/schema da borda.

```ts
// Evitar
return { title: dto.title ?? null, subtitle: dto.subtitle ?? null };
```

```ts
// Preferir — opcional real ou omitir campo
return { title: dto.title, ...(dto.subtitle !== undefined && { subtitle: dto.subtitle }) };
// ou ajustar o tipo de retorno para refletir undefined em vez de normalizar tudo para null
```

---

## `no-generic-http-response-unwrapping-helper.mdc`

**Ideia:** quando a API devolve sempre o mesmo formato (ex.: `data.items`), devolver **no próprio método**; não criar `unwrapItems(data: unknown)` genérico.

```ts
// Evitar
private static pickPagedItems<T>(data: unknown): T[] {
  // ... introspecção em unknown ...
}
```

```ts
// Preferir — no adapter, uma linha
return response?.data?.items ?? [];
```

Helper só faz sentido com **vários formatos documentados** ou **parse/validação real** (ver regra completa).

---

## `prefer-descriptive-loop-and-callback-names.mdc`

**Ideia:** em laços e callbacks, nomeie pelo domínio; evite `c`, `res`, `x` em lista de negócio. **`x` / `y` / `z`** só em contexto **matemático** explícito (coordenadas, grid). Prefira `index` / `page` quando o contador tiver uso no corpo.

```ts
// Evitar
for (const c of contractsPage) {
  const end = c.endDateIso ? new Date(c.endDateIso) : null;
}
rowsRaw.map(x => x.profileId);
```

```ts
// Preferir
for (const contract of contractsPage) {
  const endDate = contract.endDateIso ? new Date(contract.endDateIso) : null;
}
rowsRaw.map(row => row.profileId);
```

```ts
// OK — laço matemático / grid
for (let y = 0; y < height; y += 1) {
  for (let x = 0; x < width; x += 1) {
    matrix[y][x] = fn(x, y);
  }
}
```

---

## `prefer-simple-callback-narrowing.mdc`

**Ideia:** não carregar type predicate em todo `.filter` só para `string` — prefira callback simples; se o TypeScript reclamar, alinhar com o usuário.

```ts
// Evitar por padrão — predicado quando `typeof` basta
.filter((rowProfileId): rowProfileId is string => typeof rowProfileId === 'string');
```

```ts
// Preferir
.filter((rowProfileId) => typeof rowProfileId === 'string');
```

Se o compilador não estreitar e a alternativa for predicado rebuscado ou `as` em cadeia, **pergunte ao usuário** antes.

---

## Uso em outro repositório

Copie a pasta `.cursor/rules` (ou os `.mdc` desejados) para a raiz do outro projeto. Ajuste `globs` se quiser limitar a pastas específicas (ex.: só `src/**/*.ts`).
