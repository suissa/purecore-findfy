## System

Você é um desenvolvedor de software que possui mais de 2 décadas de dev web, usou 3/8 da vida Linux, 3/8  da vida MacOS e 2/8 da vida Windows, acha o SO uma bosta e você gosta de criar coisas que ajudem as pessoas a terem mais produtividade.

## Tecnologia

- Node.js/TS
- Electron
- React
- TailwindCSS
- Shadcn UI
- FAISS
- tipagem semântica + tipagem nominal (Atomic Behavior Types)

## Arquitetura

Crie os arquivos dentro da pasta TOOLS/purecore-findfy

Como será uma arquitetura ULTRA-SIMPLES que pensei agora em unir 2 conceitos que uso:

- AtomicBehaviorAgents
- AtomicBehaviorTypes

<span style="font-size: 24px;">Pensando o sistema utilizando o <span style="font-size: 30px; font-weight: bold; color: blue">Design by Intent</span></p>

- Domain já sabemos: filesystem - FORA!
- Entity: Folder & File
  - Properties: Atomic Behavior Types
- Intent: search, list, open file with, view, view details
- Behavior: left-click, search with (text, vector, graph, tree), show menu, show results, order results, filter results, choose option, open the file with chosen option

Vamos deixar simples, por hora.

E vamos aproveitar e transformar em um Sistema Orchestrated Mono-EntityAgent, sendo que cada Entity Agent é composto por Multi-Atomic Behavior Agents, que serão:

- SearchTextAgent: busca via textual (regex, fuzzy, soundex, etc) no `purecore.finder.treeflow` 
- SearchVectorAgent: busca via similaridade semântica no FAISS
- SearchGraphAgent: não implemente ainda, só deixe o Agent definido
- SearchTreeAgent: busca via Arvores com o `purecore.finder.treeflow` 
- ListThumbnailsAgent: gera a lista de resultados com suas thumbnails
- ListDetailsAgent: gera a lista de resultados com seus detalhes (nome, tipo, tamanha, data_modificação)
- DataVectorAgent: armazena as buscas e o mapeamento de arquivos no FAISS
- DataGraphAgent: armazena as buscas no arquivo `purecore.finds.graflow` 
- DataTreeAgent: armazena o mapeamento de arquivos em `purecore.finder.treeflow` 
- OpenWithAgent: abre o arquivo usando a opção escolhida
- MenuAgent: cria o menu no click do botão
- ViewDetailsAgent
- ViewAgent: cria a view com os resultados das buscas
  - Chama o ListThumbnailsAgent e o ListDetailsAgent
  - seleciona o arquivo
  - chama o MenuAgent
    - selecionado o Open with ...
      - chama o OpenWithAgent
    - selecionado o View details
      - chama o ViewDetailsAgent

### Atomic Behavior Types

Tipagem nominal semântica com comportamentos

#### Heurísticas de Inferência

##### A) Booleans

Detecção por **prefixos** e contexto de uso:

* `is*`, `has*`, `can*`, `should*`, `enable*`, `allow*`, `show*`
* Exemplos canônicos:

  * `isDone` → `project.task.isDone`
  * `hasAllergies` → `health.patient.hasAllergies`
  * `showWeekViewCalendar` → `ui.calendar.showWeekView`
    Valide: coação para boolean, proibir `null/undefined` (ou explicitar `Maybe`), e **combinadores** só via funções (`and`, `or`) — nunca `&&` direto entre AtomicTypes.

##### B) Numbers

Use nomes, unidades, e padrões:

* Sufixos/palavras-chave: `total`, `amount`, `price`, `quantity`, `count`, `size`, `capacity`, `rate`, `percentage`, `score`, `weight`, `length`, `height`, `width`, `radius`, `duration`.
* Padrões literais: `%`, `ms`, `s`, `min`, `h`, `kg`, `g`, `m`, `cm`, `km`, `px`, `rem`, `brl`, `usd`.
* Exemplos:

  * `totalAppointments` → `clinic.schedule.totalAppointments` (inteiro ≥ 0)
  * `revenueTotal` → `ecommerce.order.revenueTotal.brl` (moeda BRL)
  * `teethExtracted` → `dentistry.procedure.teethExtracted` (inteiro 0–32)
  * `percentage` → `metrics.kpi.percentage` (0–100 ou 0–1 — explicitar escala)
  * `durationMs` → `time.duration.ms`
    Valide: **faixas** (min/max), **inteireza** quando nome implicar contagem, e **unidade** no nome canônico (ex.: `.brl`, `.usd`, `.kg`, `.ms`).

##### C) Strings

Detecte **formatos**:

* `email`, `phone`, `url`, `slug`, `isoCode`, `cpf/cnpj`, `uuid`, `id` (se houver padrão).
* Regex úteis (documente no stub):

  * Email (robusto suficiente): `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`
  * URL: usar `new URL()` no validador (cair no catch se inválida)
  * UUID v4: `/^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i`
* Exemplos:

  * `Email` → `user.email`
  * `orderId` (uuid) → `ecommerce.order.id`
  * `countryIso2` → `geo.country.iso2`

##### D) Date/Tempo

Mapeie granularidade: `createdAt`, `updatedAt`, `startAt`, `endAt`, `birthday`, `scheduledFor`, `dueAt`.

* Exemplos:

  * `scheduledFor` → `calendar.event.scheduledAt` (Date)
  * `birthday` → `identity.person.birthDate` (Date sem hora — normalize)


#### Forge (Factory)

Crie **um** utilitário interno para os Atomic Behavior Types de Semânticos **sem runtime**:

```ts
// src/AtomicBehaviorTypes/forge.ts
declare const __AtomicType: unique symbol;

export type AtomicType<T, Name extends string> = T & { readonly [__AtomicType]: Name };

export function BehaviorType<Name extends string>() {
  return {
    of: <T>(v: T) => v as AtomicType<T, Name>,
    un: <T>(v: AtomicType<T, Name>) => v as unknown as T,
  };
}
```


## Facilitando a Instanciação com Funções 'Make'

Para tornar a criação de tipos semânticos mais intuitiva e menos verbosa, recomenda-se adicionar funções auxiliares simples, conhecidas como "makers" ou "factories", diretamente nos módulos de cada tipo. Essas funções permitem instanciações rápidas, como `makeEmail('sussu@gmail.com')`, evitando a necessidade de chamar métodos mais complexos como `Email.of(...)` em contextos cotidianos.

### Recomendações

- **Padronização:** Sempre inclua `make` em novos tipos para promover adoção rápida.
- **Validação:** A função `make` deve herdar as validações de `of`, garantindo consistência.
- **Evolução:** Se necessário, expanda `make` com overloads para aceitar múltiplos formatos (ex.: `make` para datas aceitando string ou timeBehaviorType).

Essa abordagem torna os tipos semânticos mais acessíveis, acelerando a migração e reduzindo erros em projetos reais.

### Como Implementar

Adicione uma função `make` em cada stub de tipo, que internamente chame a função `of` existente. Isso mantém a validação e o AtomicTyping, oferecendo uma interface mais amigável.

#### Exemplo para Boolean (ex.: `project.task.isDone`)

```ts
// types/project/task/is-done.ts
import { AtomicType, BehaviorType } from "../../src/AtomicBehaviorTypes/forge";
export type IsDone = AtomicType<boolean, "project.task.isDone">;

export const IsDone = (() => {
  const f = BehaviorType<"project.task.isDone">();
  return {
    of: (v: unknown): IsDone => f.of(Boolean(v)),
    un: (v: IsDone): boolean => f.un(v),
    and: (a: IsDone, b: IsDone): IsDone => f.of(f.un(a) && f.un(b)),
    make: (value: boolean): IsDone => f.of(value),
  };
})();
```
Uso: `const isCompleted = IsDone.make(true);` (equivalente a `IsDone.of(true)`).

#### Number com moeda (ex.: `ecommerce.order.revenueTotal.brl`)

```ts
// types/ecommerce/order/revenue-total.brl.ts
import { AtomicType, BehaviorType } from "../../src/AtomicBehaviorTypes/forge";
export type RevenueTotalBRL = AtomicType<number, "ecommerce.order.revenueTotal.brl">;

export const RevenueTotalBRL = (() => {
  const f = BehaviorType<"ecommerce.order.revenueTotal.brl">();
  return {
    of: (v: unknown): RevenueTotalBRL => {
      const n = Number(v);
      if (!Number.isFinite(n) || n < 0) throw new TypeError("revenue must be >= 0");
      return f.of(n);
    },
    un: (v: RevenueTotalBRL) => f.un(v),
    add: (a: RevenueTotalBRL, b: RevenueTotalBRL): RevenueTotalBRL => f.of(f.un(a) + f.un(b)),
    make: (value: number): RevenueTotalBRL => f.of(value),  // Nova função make
  };
})();
```

#### Date (ex.: `calendar.event.scheduledAt`)

```ts
// types/calendar/event/scheduled-at.ts
import { AtomicType, BehaviorType } from "../../src/AtomicBehaviorTypes/forge";
export type ScheduledAt = AtomicType<Date, "calendar.event.scheduledAt">;

export const ScheduledAt = (() => {
  const f = BehaviorType<"calendar.event.scheduledAt">();
  return {
    of: (v: unknown): ScheduledAt => {
      const d = v instanceof Date ? v : new Date(String(v));
      if (Number.isNaN(d.getTime())) throw new TypeError("invalid date");
      return f.of(d);
    },
    un: (v: ScheduledAt) => f.un(v),
    make: (value: string | Date): ScheduledAt => f.of(value),  // Nova função make
  };
})();
```

#### String formatada (ex.: `user.email`)

```ts
// types/universal/user/email.ts
import { AtomicType, BehaviorType } from "../../src/AtomicBehaviorTypes/forge";
export type Email = AtomicType<string, "user.email">;

export const Email = (() => {
  const f = BehaviorType<"user.email">();
  const emailRx = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return {
    of: (v: unknown): Email => {
      const s = String(v).trim();
      if (!emailRx.test(s)) throw new TypeError("invalid email");
      return f.of(s);
    },
    un: (v: Email) => f.un(v),
    make: (value: string): Email => f.of(value),  // Nova função make
  };
})();
```


  

No sistema deverá utilizar apenas:

```ts
types/
  universal/
    user/
      index.ts        # Exporta tudo
      user-id.ts      # Exporta UserId e UserId.make()
      name.ts         # Exporta Name e Name.make()
      email.ts        # Exporta Email e Email.make()
      is-active.ts    # Exporta IsActive e IsActive.make()
```

```ts
// types/universal/user/index.ts
export { UserId } from './user-id';
export { Name } from './name';
export { Email } from './email';
export { IsActive } from './is-active';

export interface User {
  id: UserId;
  name: Name;
  email: Email;
  isActive: IsActive;
}

// Opcional: exportar um objeto com funções de criação
export const UserForge = {
  id: UserId.make, 
  name: Name.make,
  email: Email.make,
  isActive: IsActive.make,
};
```

```ts
// FORMA 1: Importando tipos individualmente
import { UserId, Name, Email, IsActive } from './types/universal/user';

const novoUsuario = {
  id: UserId.make("550e8400-e29b"),
  name: Name.make("Suissa"),
  email: Email.make("suissa@dev.com"),
  isActive: IsActive.make(true)
};

// FORMA 2: Usando o UserForge (EU PREFIRO)
import { UserForge } from './types/universal/user';

const outroUsuario = {
  id: UserForge.id("550e8400-e29b"),
  name: UserForge.name("Suissa"),
  email: UserForge.email("suissa@dev.com"),
  isActive: UserForge.isActive(true)
};
```

É bem simples.


## Funcionalidade

crie em TOOLS/purecore-findfy um app em electron que mimetize o File Explorer do Windows que eu possa navergar nos diretórios, clicar 2x para abrir e no botao direito apenas:

- abrir com Powershell
- abrir com WSL
- abrir com File Explorer
- abrir com Antigravity
- abrir com Kiro
- abrir com Cursor
- abrir com VSCode
- abrir com Qoder
- abrir com Trae

Com botão no topo para Visualizar:

- lista
- lista com detalhes (data modificação, tipo e tamanho)
- ícones
- ícones grandes

Pane/Sheet lateral direito com 2 abas: 

- os detalhes do arquivo/pasta
- chat com a IA: que na primeira execução deve pedir ou OPENAI_API_KEY ou OPENROUTER_API_KEY ou GEMINI_API_KEY ou HF_API_KEY (HUGGINFACE), na próxima tela mostre um input de text para escrever o modelo padrão de cada uma, ao clicar no input deverá aparecer uma listagem com os modelos existentes para aquela chave, a cada digitada de tecla a lista deve ir se filtrando até que eu possa clicar no que eu quero. E em cada card deverá ter um ícone desabilitado de estrela e também um botão desabilitado escrito DEFAULT, no hover tanto o ícone como o botão devem se transicionar para 50% da sua cor, no clicado para 100%. Só pode haver 1 card DEFAULT, mas a sequencia dos próximos clicks irá definir a sequencia de fallback caso o provider DEFAULT não esteja respondendo. Implemente um timeout configurável de 60s. Isso tudo será mais para usar em outras funcionalidades fora o chat, no chat deve mostrar os providers que defini a key e até 20 dos models mais atuais no select para selecionar para conversar, como também um input de texto para eu poder adicionar o nome do model faltante, que deve ser salvo em um arquivo chamado purecore.finder.models.json

## Configurações

Deve haver uma tela de configuração que mostre todos os de configuração do finder e permita editar, adicionar e remover. Por exemplo:

- [ ] mostrar TODOS os arquivos sempre, mesmo os que iniciem com .
- [ ] mostrar TODAS partições
  - [ ] mostrar apenas partição/pasta X
- [ ] mostrar pastas no WSL
- [ ] salvar buscas recentes
  - [ ] salvar buscas em arquivo
  - [ ] salvar buscas em banco de dados
- [ ] mostrar seção de atalhos
- [ ] mostrar seção de favoritos
- [ ] mostrar seção de arquivos recentes
- [ ] gerenciar modelos de LLM
  - [ ] timeouts
  - [ ] max-tokens
  - [ ] top-p
  - [ ] top-k
  - [ ] frequency-penalty
  - [ ] presence-penalty
  - [ ] temperature
- [ ] gerenciar modelos de Embeddings
- [ ] gerenciar modelos de Buscas

Adicione mais configurações que você imaginar para eu largar essa bosta de File Explorer do Windows, como falei eu só quero mandar uma PASTA ou ARQUIVO abrir com algum programa.

## Busca

A busca tem que ser otimizada partindo da pasta atual, que estará mostrada no purecore-finder, para todas as pastas abaixo dela. Como estou no Windows quero que você execute o comando `wsl` antes e depois busque como Linux, os arquivos que preciso das partições do Windows estão em `/mnt/d`, `/mnt/c` e do WSL está em `/home/suissa`

Busca por:

- nome (LIKE/regex)
- extensão (LIKE/regex)
- conteúdo (LIKE/regex)
- data de modificação (exact, between, after, before)
- data de criação (exact, between, after, before)
- tamanho (between, size: [small, medium, large, extra-large])

Faça o sistema carregar o caminho de todos os arquivos EXCLUINDO SEMPRE PASTAS COM OS NOMES: 

- dist  (ou seja, dist/*)
- node_modules (ou seja, node_modules/*)
- iniciando em "." (ou seja, .git/*, .cache, .vscode, etc)

* Lista podendo ser gerenciada via configuração, tanto para pasta como para arquivos({nome, extensão e tamanho})

### Mapeamento

Para otimizarmos futuras buscas iremos mapear todos os arquivos e pastas existentes na partição D, para isso faça primeiro ele contar quantos arquivos tem sem essas pastas excluídas e aí mostrar em tempo real uma barra de LOADING, acima dela mostre a quantidade atual de arquivos mapeadas (atualizando a cada 1s) e o total existente na partição escolhida para iniciar, mostrando a velocidade de arquivos por segundo mapeados (atualizando a cada 1s). Salve como Arvores em um arquivo na pasta `{raiz_particao}/.purecore/findfy/purecore.finder.treeflow`

Vamos usar a notação de Leaf-Only Path Map ou File-Centric Index 
```treeflow
home>suissa>projetos>novos:[prompt.md, apy_key.json]
home>suissa>projetos>novos>pasta_proj1:[README.md, index.ts]
home>suissa>projetos>novos>pasta_proj2:[README.md, tsconfig.json]
home>suissa>projetos>novos>pasta_proj2>src:[index.ts]
```

Já fiz o indexador dos arquivos aqui `purecore-findfy/indexer.ts` 


### Otimização máxima

E toda vez que eu pedir para buscar algo você já monta a view usando a busca desse arquivo e em background faz uma busca completa pelo termo e deixa rodando até completar tudo para poder atualizar o arquivo `{raiz_particao}/.purecore/findfy/purecore.finder.treeflow` se surgir arquivos novos ou foram movidos para esse termo, salve em um arquivo `{raiz_particao}/.purecore/findfy/purecore.finds.graflow` com:

- (...busca.split(" "))->({particao})->({pasta_pai})->caminha_todas_pastas_filhas((pasta, i) => `({pasta[i]})->` )->({file}) 

deixe pronto pra eu salvar o grafo em um banco de vetor usando a busca inteira, como parte, como fuzzy, como soundex, como outras formas de busca não exatas, prefiro o FAISS e claro salvar todos seus vetores e tudo mais, PARA NUNCA ter que recriar nada do FAISS, crie ele dentro da pasta D:\FAISS ou /mnt/d/FAISS

## Entregáveis

- app electron executável
- criação da configuração inicial
- criação do mapeamento inicial
- busca por qualquer arquivo mapeado
- busca por qualquer arquivo não mapeado
- testes cobrindo todas as funcionalidades

## BONUS

Crie uma heurística que possa sugerir quais são os nomes semânticos que posso usar para aumentar a velocidade na busca e na atualização, precisa ser uma fórmula de busca em profundidade em grafos/árvores do tipo Rooted Tree e Forest com busca Vetorial via HNSW ou Product Quantization ou Inverted File Index.

Cada busca que eu fizer ele deve usar a busca que você definir padrão inicial, mas após mostrar os resultados ele deve me perguntar se desejo otimizar essa busca, se sim o sistema deve fazer buscas utilizando 1 ou mais heurísticas diferentes utilizando tanto um conjunto de busca em árvore, grafo, vetor e texto (regex, acha os arquivos no `purecore.finder.treeflow` e apenas vai pegando a pasta acima dela até achar a pasta que não possua nenhuma identação a esquerda). 

