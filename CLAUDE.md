# MyApp: finanças pessoais gamificadas (app)

App de gestão financeira em formato de jogo, com um bichinho virtual estilo Tamagotchi.
A base de tudo é a **reserva de emergência**: só depois dela a pessoa passa a investir.

Este repositório é só o app. O projeto tem três repositórios, normalmente lado a lado em `~/Documents/funnycoins/`:
- `app/`: este, em `joaopedrolacerda/funnyCoins-app`.
- `backend/`: API em Rust, em `joaopedrolacerda/funnyCoins-backend`.
- `web/`: site em Svelte, em `joaopedrolacerda/funnycoins`.

- Plataformas: iOS, iPadOS, macOS e visionOS 27, num único target multiplataforma.
- Stack: SwiftUI + SwiftData. Sem Combine, use async/await.
- Isolamento padrão: `MainActor`, tanto no app quanto nos testes.
- Idioma do app: português do Brasil.

## Regras obrigatórias

1. **Toda mudança vem com testes, e todos os testes precisam passar** antes de dar a tarefa por concluída.
   - Use Swift Testing (`import Testing`, `@Test`, `#expect`). Não use XCTest em testes novos.
   - Regras de negócio ficam fora das views, em `Models/`, para poderem ser testadas.
     Se uma regra nascer dentro de uma view, mova-a para o motor antes de testar.
     Exemplo: `GameSnapshot.suggestedSplit`.
2. **Dinheiro sempre no formato brasileiro**, como `R$ 1.234,56`, independente do idioma do aparelho.
   - Para exibir, use `valor.brl`.
   - Para digitar, use `CurrencyField` ou `TextField(..., format: BRFormat.currency)`.
   - Nunca use `.currency(code:)` direto nem `Double` para valores monetários. Dinheiro é `Decimal`.
   - A raiz do app injeta `.environment(\.locale, BRFormat.locale)` para números e porcentagens.
3. Textos de interface em português, com tom amigável e de jogo.
4. Não force unwrap. Prefira `??`, `guard` e, nos testes, `try #require`.

## Como validar

- Compilar: `BuildProject`, a ferramenta de build do Xcode.
- Testar: `RunAllTests`. O esquema compartilhado `MyApp` inclui o target `MyAppTests`.
- Pelo terminal: `xcodebuild test -project funnycoins.xcodeproj -scheme MyApp -destination 'platform=macOS'`.
  O `xcode-select` aponta para as Command Line Tools, então use `DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer`.
- O target do app se chama `funnycoins` e gera `funnycoins.app`, mas o módulo Swift continua `MyApp`
  (`PRODUCT_MODULE_NAME`), por isso os testes usam `@testable import MyApp`. O `TEST_HOST` aponta para `funnycoins.app`.
- `MyAppTests` depende explicitamente do target do app, então o app sempre compila antes dos testes.
- Previews ajudam a verificar o visual com `RenderPreview`.

## Arquitetura

```
funnycoins/
  Models/
    FinanceModels.swift   modelos SwiftData, enums, BRFormat
    GameEngine.swift      GameSnapshot: níveis, fluxo, XP, patentes, conquistas
    PetEngine.swift       bichinho: necessidades, humor, evolução, falas
    SyncModels.swift      contrato da API: registros, centavos, conversão de modelos
    SyncEngine.swift      motor de sincronização e correção de identificadores
  Views/
    PetHomeView.swift     aba do bichinho (primeira aba)
    PetView.swift         desenho e animações do bichinho
    JourneyView.swift     slides: Nível 1, Nível 2, Conquistas
    GainWizardView.swift  passo a passo para registrar cada ganho
    CashFlowView.swift    entradas e saídas
    BillsView.swift       contas fixas mensais
    InvestmentsView.swift Nível 2: carteira e aportes
    Forms.swift           formulários de perfil, gasto, conta, reserva e investimento
    Components.swift      componentes reutilizáveis
    AppIconView.swift     arte do ícone do app
  Assets.xcassets/AppIcon.appiconset   PNG de 1024 para iOS e tamanhos do macOS
MyAppTests/               testes com Swift Testing
  Fixtures/rules-fixtures.json   casos das regras do jogo, compartilhados com o site
```

- `GameSnapshot` é calculado a cada renderização a partir das consultas do SwiftData em `MainTabView`.
  XP, conquistas e humor **não são salvos**: são derivados dos dados. Isso mantém tudo consistente.
- `GameSnapshot` recebe `now`, o que permite testar regras que dependem de data.
- Enums são salvos como `String` cru (`kindRaw`, `modalityRaw`...). Nunca renomeie um raw value existente.
- Campos novos em modelos SwiftData precisam de valor padrão para a migração automática funcionar.

### Regras compartilhadas com o site
- O site calcula as mesmas regras com um WASM gerado a partir do Rust (a pasta `rust/rules` sumiu em 13/11/2026; o site usa o WASM já gerado).
- Toda mudança nas regras do jogo precisa atualizar `MyAppTests/Fixtures/rules-fixtures.json`.
  O site tem uma **cópia** desse arquivo em `web/fixtures/rules-fixtures.json`: mantenha as duas iguais.
- O desenho do bichinho em `PetView.swift` é reproduzido no site (`web/src/components/Pet.svelte`) com as mesmas frações.
  Ao mudar a arte aqui, mude lá também.

### Sincronização
- **Plano:** o backend e o site estão prontos. No app, contrato e motor estão prontos e testados.
  Faltam o cliente HTTP, o login e a tela. O login principal do site é o código da conta (`POST /v1/auth/code`); ver a CLAUDE.md do backend.
- Todo modelo SwiftData tem `syncID`, `updatedAt` e `deletedAt`.
  - **Ao editar** um registro existente, chame `touch()`.
  - **Para excluir**, use `softDelete()`. Nunca use `context.delete`.
  - **Telas e jogo** usam os filtros `.active`, que escondem os excluídos.
- Conflitos: vence o `updatedAt` mais recente. Em empate, fica a versão local.
- Datas trafegam em milissegundos. `syncPrecision` trunca para evitar diferenças falsas.
- Dinheiro trafega em centavos (`Decimal.cents` e `Decimal(cents:)`).
- `SyncMaintenance.repairDuplicateIDs` roda ao abrir o app. Ele corrige UUIDs repetidos deixados pela migração de dados antigos.

### Ícone do app
- O ícone mostra o Filhote verde sobre fundo dourado, com uma moeda de "R$". A fonte é `AppIconView`.
- Ao mudar a arte, gere o PNG de novo. Para isso, renderize `AppIconView(size: 1024)` com `ImageRenderer`, na escala 1 e opaco.
  Depois substitua `AppIcon-1024.png` e gere os tamanhos do macOS com `sips -z`.
- O PNG não pode ter transparência.
- O target usa `ASSETCATALOG_COMPILER_APPICON_NAME = AppIcon`.
  A ferramenta de build settings não aceita esse nome, então ele foi definido direto no projeto.
- visionOS ainda não tem ícone próprio, porque exige um ícone em camadas.

## Regras do jogo (especificação atual)

### Nível 1: reserva de emergência
- Meta = custo de vida mensal × meses escolhidos. O padrão é 6, e o intervalo vai de 3 a 24.
- Depósitos e retiradas são `ReserveMovement`, com valor positivo ou negativo.
- Concluído quando saldo ≥ meta. Isso libera o Nível 2.

### Nível 2: investimentos
- Bloqueado até concluir o Nível 1.
- Meta mensal = `investPercent`% das entradas do mês. O padrão é 10%.
- Modalidades: Tesouro Selic, CDB, LCI/LCA, Tesouro IPCA+, FII, Ações, ETFs, Cripto, Previdência,
  **Cursos e educação** e Outros.
- Cursos e educação contam como investimento em você: risco 0, rótulo "Investimento em você".
  Contam para a diversificação.

### Registro de ganho (passo a passo)
Origem → valor → quanto separar → comemoração.
- No Nível 1, a parte separada vai para a reserva, limitada ao que falta para completar.
- No Nível 2, vai para um investimento na modalidade escolhida.

### XP e patentes
| Ação | XP |
|---|---|
| Lançamento | 5 |
| Conta paga, por mês | 15 |
| Depósito na reserva | 20 |
| Aporte | 30 |
| Conquista desbloqueada | 100 |

Patentes: Aprendiz (0), Poupador (200), Estrategista (600), Investidor (1500), Mestre das Finanças (4000).

### Bichinho
- Necessidades de 0 a 1:
  - **Fome** cai 20% por dia sem lançamentos.
  - **Proteção** acompanha a reserva.
  - **Alegria** começa em 0,5. Perde 0,25 por conta atrasada. Ganha 0,25 com todas as contas pagas e 0,25 se guardou ou investiu no mês.
- Humor, na ordem de prioridade: doente se houver conta atrasada, com fome se a fome estiver abaixo de 0,3. Pela média: feliz a partir de 0,7, tranquilo a partir de 0,4, senão triste.
- Evolução: Ovo, sem lançamentos → Filhote → Jovem, com 50% da reserva → Adulto, com o Nível 1 completo
  → Lendário, com o Nível 1 completo mais 3 modalidades ou 1500 XP.
- Formas: cada estágio tem uma forma própria, definida em `PetStage.appearance` e desenhada por `PetView`.
  Cada evolução ganha mais partes que a anterior. Isso é testado.

  | Estágio | Forma |
  |---|---|
  | Ovo | Ovo pintado que balança, com 7 espinhos simétricos e nenhum embaixo. Ver `EggDesign`. |
  | Filhote | Bolinha redonda sem partes extras. **Aprovada pelo usuário: não alterar.** |
  | Jovem | Mais alto, orelhinhas pontudas, bracinhos (um acena) e broto de folha |
  | Adulto | Orelhas longas, rabinho enrolado e escudo na barriga, pela reserva completa |
  | Lendário | Asas que batem, aura dourada com faíscas, coroa e estrela na barriga |

## Decisões em aberto

Itens para especificar melhor antes de implementar:
- Compra de curso **antes** de completar a reserva: hoje só dá para registrar como gasto da categoria Educação, porque o Nível 2 está bloqueado. Deveria ser liberado como exceção?
- Investimentos guardam só o valor aportado. Falta acompanhar o valor atual e o rendimento.
- Existe um Nível 3? Por exemplo, diversificação ou independência financeira.
- Contas fixas: permitir desfazer pagamento e escolher a data real do pagamento.
- Notificações: o bichinho avisar quando estiver com fome ou quando uma conta vencer.
- Datas ainda usam o formato do aparelho em alguns textos. Avaliar se devem ser fixadas em pt-BR.
- Dois aparelhos que fizerem o cadastro inicial antes de sincronizar vão criar dois perfis. Hoje o app usa o mais antigo. Definir se os perfis devem ser mesclados.
- Client IDs da Apple (Services ID para o site) e do Google.
