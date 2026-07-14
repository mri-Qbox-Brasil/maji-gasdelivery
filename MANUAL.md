# maji-gasdelivery — Manual

Emprego de caminhoneiro: alugue caminhão e tanque, encha o tanque no depósito e abasteça os postos de gasolina espalhados por Los Santos.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Fluxo do trabalho](#fluxo-do-trabalho)
5. [Menu do NPC](#menu-do-npc)
6. [Integrações](#integrações)
7. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
8. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core` | Sim | Declarado em `dependencies`. Framework, dinheiro, notificações, progressbar e spawn de veículos |
| `qb-target` | Sim | Declarado em `dependencies`. Único target suportado (`Config.Target = 'qb'`) |
| `ox_lib` | Sim | Menu de contexto do NPC e `lib.callback` |
| `PolyZone` | Sim | Carregado via `@PolyZone/client.lua` e `@PolyZone/BoxZone.lua`. Delimita a área do depósito |
| `rep-talkNPC` | Sim | O NPC "Seu Wilson" é criado por `exports['rep-talkNPC']:CreateNPC`. Sem ele não há como iniciar o trabalho |
| `cdn-fuel` | Sim | Valor padrão de `Config.FuelScript`. Usado em `exports[Config.FuelScript]:SetFuel` ao spawnar o caminhão |
| `qb-vehiclekeys` ou `mk_vehiclekeys` | Sim | Entrega as chaves do caminhão alugado. Escolhido em `Config.VehicleKeys` |
| `qb-phone` | Não | Envia o e-mail com o próximo destino (`qb-phone:server:sendNewMail`). Sem ele o blip ainda é criado |
| `InteractSound` | Não | Efeitos sonoros de bico e abastecimento (`InteractSound_SV:PlayOnSource`) |

---

## Instalação

1. Copie a pasta `maji-gasdelivery` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure maji-gasdelivery
   ```
3. Copie os `.ogg` de `sounds/` para a pasta de sons do seu recurso de InteractSound. Os nomes usados no código são: `pickupnozzle`, `putbacknozzle`, `refuel`, `fuelstop`.
4. Não há SQL nem itens de inventário a cadastrar.

---

## Configuração

Arquivo: `config.lua`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Config.Debug` | bool | Sim | Ativa prints de diagnóstico no console (IDs de veículo, zona, cooldown) |
| `Config.Blip` | array | Sim | Blips do ponto de início. Cada item usa `title`, `color`, `id` (sprite) e `x`/`y`/`z` |
| `Config.Target` | string | Sim | Sistema de target. Só `'qb'` é suportado — o suporte a ox foi removido |
| `Config.UseMenu` | bool | Sim | Presente no config, mas **não é lido em nenhum lugar do código**. O menu de contexto é sempre usado |
| `Config.Menu` | string | Sim | Idem: declarado, mas não consumido pelo código |
| `Config.PedType` | string | Sim | Modelo do NPC do trabalho. Padrão: `a_m_m_ktown_01` |
| `Config.VehicleKeys` | string | Sim | `'qb-vehiclekeys'` ou `'mk_vehiclekeys'` |
| `Config.TruckToSpawn` | string | Sim | Modelo do caminhão. Precisa poder puxar um trailer grande. Padrão: `packer` |
| `Config.TrailerToSpawn` | string | Sim | Modelo do tanque. Deve ser uma variante de tanker, senão a corda do bico fica fora de posição. Padrão: `tanker2` |
| `Config.PayPerFueling` | number | Sim | Pagamento por posto abastecido. É um `math.random(1200, 2500)` avaliado **uma única vez** no carregamento do recurso — o valor fica fixo até o próximo restart, e o pagamento final é ele multiplicado pelo número de postos |
| `Config.FuelScript` | string | Sim | Recurso de combustível usado no `SetFuel` do caminhão: `cdn-fuel`, `ps-fuel` ou `LegacyFuel` |
| `Config.PayType` | string | Sim | Conta usada tanto para cobrar quanto para pagar: `'bank'` ou `'cash'` |
| `Config.TruckPrice` | number | Sim | Custo de alugar o caminhão + tanque. Padrão: `1000` |
| `Config.TankPrice` | number | Sim | Custo de alugar só o tanque, para quem usa caminhão próprio. Padrão: `2000` |
| `Config.MaxFuelDeliveries` | number | Sim | Quantos postos podem ser abastecidos antes de voltar ao depósito para reencher o tanque. Padrão: `1` |
| `Config.PumpLocations` | tabela | Sim | Postos sorteados como destino. Cada entrada tem apenas `coords` (`vector3`), que é **onde o prop de abastecimento será spawnado** |

As coordenadas do depósito, do NPC e do ponto de spawn dos veículos estão fixas em `client/main.lua`, não no config.

---

## Fluxo do trabalho

1. Fale com o NPC "Seu Wilson" (blip "Caminhoneiro", em `1721.87, -1557.67`) e abra o menu.
2. Escolha alugar um caminhão (cobra `Config.TruckPrice`) ou trabalhar com caminhão próprio (cobra só `Config.TankPrice`). Em ambos os casos o tanque é spawnado e um blip aponta para ele.
3. Engate o tanque no caminhão. Um novo blip aponta para o depósito.
4. No depósito, use o target no prop `prop_storagetank_02b` para pegar o bico, leve-o até o tanque e use o target no trailer para encher (progressbar de 15s).
5. Um posto aleatório de `Config.PumpLocations` é sorteado, um e-mail é enviado pelo `qb-phone` e um blip com rota aparece.
6. No posto, pegue o bico no trailer e use o target no prop de abastecimento para descarregar (progressbar de 30s).
7. Repita até atingir `Config.MaxFuelDeliveries`; então volte ao depósito para reencher o tanque, ou volte ao NPC para receber.

O bico se solta e o combustível "quebra a linha" se o jogador se afastar mais de 10 metros do ponto onde o pegou.

---

## Menu do NPC

| Opção | Descrição |
|---|---|
| Reabastecer Posto | Só fica habilitada quando há um pedido pendente do `cdn-fuel`. Dispara `cdn-fuel:station:client:initiatefuelpickup` e consome o pedido |
| Alugue um caminhão e comece a trabalhar | Cobra `Config.TruckPrice` e spawna caminhão + tanque |
| Comece a trabalhar com seu próprio caminhão | Cobra `Config.TankPrice` e spawna só o tanque |
| Obtenha salário | Paga `postos abastecidos × Config.PayPerFueling`, remove os veículos e zera o progresso |
| Reiniciar o trabalho | Cancela o trabalho, remove os veículos e zera o progresso. Se houver postos abastecidos ainda não pagos, avisa uma vez antes de deixar cancelar |

---

## Integrações

### cdn-fuel

Além do `SetFuel` no caminhão alugado, o recurso escuta os pedidos urgentes de reabastecimento de posto do `cdn-fuel`. Um outro recurso registra o pedido via `md-refuelcdn:server:set`; a partir daí a opção "Reabastecer Posto" do menu fica habilitada e o primeiro jogador que aceitar dispara o fluxo do próprio `cdn-fuel` e consome o pedido para todos.

### qb-phone

Ao terminar de encher o tanque no depósito e a cada posto concluído, o jogador recebe um e-mail de "O chefe" com o aviso do próximo destino, via `qb-phone:server:sendNewMail`. Sem o `qb-phone` o e-mail simplesmente não chega — o blip com rota continua sendo criado.

### rep-talkNPC

O NPC do trabalho é criado com `exports['rep-talkNPC']:CreateNPC`, com diálogo de apresentação e as opções "Como funciona esse trabalho?" / "Trabalhar". Ambas levam ao menu de contexto do emprego.

---

## Entrypoints para outros recursos

### Export `Refuelcdn_status` (servidor)

Retorna `true` se existe um pedido urgente de reabastecimento pendente.

```lua
local pendente = exports['maji-gasdelivery']:Refuelcdn_status()
```

### Registrar um pedido urgente

```lua
-- Servidor: marca que há um pedido pendente
TriggerServerEvent('md-refuelcdn:server:set')

-- Servidor: limpa o pedido e avisa todos os clientes
TriggerServerEvent('md-refuelcdn:server:delete')

-- Cliente: envia os dados do pedido (amount, newAmount, location) para o menu
TriggerClientEvent('md-refuelcdn:client:set', src, amount, newAmount, location)
```

### Callback de status

```lua
local pendente = lib.callback.await('md-refuelcdn:server:status', false)
```

### Abrir o menu do trabalho

```lua
TriggerEvent('md-opentruckermenu')
```

---

## Estrutura de arquivos

```
maji-gasdelivery/
├── client/
│   └── main.lua      — NPC, menu, spawn de veículos, bicos e cordas, blips, targets e progressbars
├── server/
│   └── main.lua      — cobrança do aluguel, pagamento, estado do pedido do cdn-fuel e export Refuelcdn_status
├── config.lua        — veículos, preços, contas, target, fuel script e lista de postos
├── sounds/           — .ogg de bico e abastecimento (copiar para o recurso de InteractSound)
└── fxmanifest.lua
```
