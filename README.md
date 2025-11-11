# Habla! — Juego de combinadas de fútbol con “Lukas”

Este repositorio es la **fuente de verdad** del front web y del backend lógico de Habla!, una webapp donde los jugadores compran Lukas (moneda in-app) para entrar a torneos de predicciones de fútbol. El pozo se forma con las entradas y se liquida a los primeros puestos; las Lukas pueden canjearse en la tienda de premios.
La app es navegable como visitante (sin registro), pero participar en torneos, hacer top-ups y canjear premios requiere iniciar sesión.

## Mecánica del juego (resumen)

- Cada torneo corresponde a un solo partido y pasa por estados: open → en_juego → terminado (o cancelado).
- Cada ticket es una combinada de 5 predicciones:
1. Local/Empate/Visita (3 pts)
2. Ambos anotan (2 pts)
3. Más de 2.5 goles (2 pts)
4. Habrá tarjeta roja (6 pts)
5. Marcador exacto (8 pts)
- Un jugador puede enviar múltiples combinadas no idénticas para el mismo partido.
- Al kickoff se revelan todas las combinadas (transparencia).
- Al finalizar, se califican tickets, se publica Leaderboard y se abonan premios en la Wallet.
- En el perfil se muestra el historial de 30 días (tickets, puntos, posiciones y movimientos de wallet).

## Front-end

- HTML único (SPA con enrutamiento por hash) + TailwindCSS.
- Vistas clave: Matches, Match Detail (Transparencia/Leaderboard), Create Ticket, Wallet/Movements, Prize Store, How to Play, FAQ.
- Sin conexión a backend (mock in-memory), preparado para reemplazar llamadas por APIs reales.
- Visitantes pueden navegar todo; acciones sensibles (crear ticket, top-up, canjear) piden login.
- Formulario de Registro (UI); Campos visibles y obligatorios:
  -  Correo electrónico
  -  Contraseña
  -  Nombres
  -  Apellidos
  -  Nickname /Nombre visible
  -  Fecha de Nacimiento
  -  País de Residencia
  -  Aceptaciones (checkboxes): Términos y Condiciones, Política de Privacidad, Política de Juego Responsable, Declaración "Soy mayor de 18 años"

## Diagrama (Mermaid) — Microservicios y flujos
>**Nota**: Este diagrama refleja solamente los microservicios definidos para Habla! y sus conexiones.

```mermaid
flowchart LR

%% === Frontend ===
subgraph FE["Frontend - Habla Webapp (SPA)"]
  direction TB
  FE_Router["Router SPA / Navegacion"]
  FE_Login["Login / Signup (modal)"]
  FE_TopUp["Top Up Lukas (modal)"]
  FE_Create["Crear Ticket (5 predicciones)"]
  FE_Match["Match Detail / Transparency / Leaderboard"]
  FE_Store["Prize Store"]
  FE_Wallet["Wallet / Movements"]
  FE_Toast["Toasts / Notifs"]
end

EB["Event Bus (Kafka o NATS)"]

subgraph AUTH["Auth"]
  direction TB
  A_Signup["POST /auth/signup"]
  A_Login["POST /auth/login"]
  A_Token["Emitir JWT / Session"]
  A_Evt["Evento: User.Registered"]
end

subgraph PERFIL["Perfil"]
  direction TB
  P_Create["Crear perfil base"]
  P_History["GET /me/history?days=30"]
end

subgraph WALLET["Wallet"]
  direction TB
  W_Create["Crear Wallet"]
  W_Credit["POST /wallet/credit"]
  W_Debit["POST /wallet/debit (con idempotency_key)"]
  W_Mov["GET /wallet/movements"]
  W_evtC["Evento: Wallet.CreditPosted"]
  W_evtD["Evento: Wallet.DebitPosted"]
end

subgraph PAY["Pagos (On-Ramp)"]
  direction TB
  Pay_Start["POST /payments (inicia checkout PSP)"]
  Pay_Webhook["Webhook PSP (validar firma/estado)"]
  Pay_Settle["Asentar Lukas netas"]
  Pay_evtI["Evento: Payment.Initiated"]
  Pay_evtS["Evento: Payment.Settled"]
end

subgraph TP["Torneos y Partidos"]
  direction TB
  T_Create["Crear partido/torneo (lock_time = kickoff)"]
  T_Lock["Match.StateChanged(en_juego)"]
  T_End["Match.StateChanged(terminado)"]
  T_Cancel["Match.StateChanged(cancelado)"]
end

subgraph RULES["Reglas y Puntajes"]
  direction TB
  R_Ruleset["Ruleset: 1X2=3, BTTS=2, O2.5=2, Roja=6, Exacto=8"]
end

subgraph TICKETS["Combinadas (Tickets)"]
  direction TB
  C_Validate["Validar picks y dominio"]
  C_Uniq["Validar unicidad (user, match, hash)"]
  C_Debit["Solicitar debito fee a Wallet"]
  C_Save["Persistir ticket + hash"]
  C_Log["Log inmutable (hash/merkle)"]
  C_evt["Evento: Ticket.Submitted"]
end

subgraph ORACLE["Oraculos de Datos"]
  direction TB
  O_Live["Ingesta live (goles, tarjetas)"]
  O_Final["MatchFacts.Finalized (FT score, BTTS, O2.5, Red, 1X2)"]
  O_Correct["MatchFacts.Corrected (v+1)"]
end

subgraph TRANS["Transparencia"]
  direction TB
  TR_Append["Append tickets pre-kickoff (WORM)"]
  TR_Reveal["Revelar combinadas al kickoff"]
  TR_Merkle["Publicar Merkle Root"]
end

subgraph SCORE["Scoring"]
  direction TB
  S_Eval["Evaluar tickets vs hechos del partido"]
  S_evt["Evento: Scoring.Completed"]
end

subgraph LDB["Leaderboard"]
  direction TB
  L_Rank["Ranking y desempates (exacto -> timestamp -> prorrateo)"]
  L_evt["Evento: Leaderboard.Finalized"]
end

subgraph SETTLE["Liquidacion"]
  direction TB
  LI_Pool["Calcular Pozo (sum fees - rake)"]
  LI_Payouts["Abonar ganadores (Wallet credit)"]
  LI_Adjust["Ajustes por correcciones"]
  LI_evt["Evento: Payouts.Posted"]
end

subgraph STORE["Tienda de Premios"]
  direction TB
  ST_List["GET /store (stock, condiciones)"]
  ST_Redeem["POST /redeem (validar saldo/politicas)"]
  ST_Block["Reservar stock"]
  ST_evt["Evento: Reward.Redeemed"]
end

subgraph FULL["Fulfillment"]
  direction TB
  F_Digital["Emitir codigo (digital)"]
  F_Ship["Courier fisico: requested -> shipped -> delivered"]
  F_evt["Evento: Fulfillment.Completed"]
end

subgraph RISK["Riesgo / Fraude"]
  direction TB
  RISK_Checks["Velocity, IP/device, multi-cuenta, patrones"]
  RF_evt["Evento: Risk.Flagged"]
end

subgraph NOTIF["Notificaciones"]
  direction TB
  N_Send["Enviar plantilla email/push/in-app"]
end

subgraph AUDIT["Auditoria y Logs"]
  direction TB
  AU_Log["Append-only de eventos"]
end

%% Registro/Login
FE_Login --> A_Login
A_Login --> A_Token
A_Login -.-> A_Evt
A_Evt -.-> EB
EB -.-> P_Create
EB -.-> W_Create
EB -.-> AU_Log
A_Token --> FE_Router

%% Top Up
FE_TopUp --> Pay_Start
Pay_Start -.-> Pay_evtI
Pay_evtI -.-> EB
EB -.-> AU_Log
Pay_Webhook --> Pay_Settle
Pay_Settle --> W_Credit
W_Credit -.-> W_evtC
W_evtC -.-> EB
EB -.-> N_Send
N_Send --> FE_Toast

%% Crear Torneo
T_Create --> R_Ruleset
T_Create -.-> EB
EB -.-> AU_Log

%% Tickets
FE_Create --> C_Validate
C_Validate --> C_Uniq
C_Uniq --> C_Debit
C_Debit --> W_Debit
W_Debit -.-> W_evtD
W_evtD -.-> EB
C_Debit --> C_Save
C_Save --> C_Log
C_Log --> TR_Append
C_Save -.-> C_evt
C_evt -.-> EB
EB -.-> AU_Log
EB -.-> RISK_Checks
RISK_Checks -.-> RF_evt
RF_evt -.-> EB

%% Kickoff y Transparencia
T_Lock -.-> EB
EB -.-> TR_Reveal
TR_Reveal --> FE_Match
TR_Merkle --> FE_Match
EB -.-> AU_Log

%% Final, Scoring, Leaderboard, Liquidacion
O_Final -.-> EB
EB -.-> S_Eval
S_Eval -.-> S_evt
S_evt -.-> EB
EB -.-> L_Rank
L_Rank -.-> L_evt
L_evt -.-> EB
EB -.-> LI_Pool
LI_Pool --> LI_Payouts
LI_Payouts --> W_Credit
W_Credit -.-> W_evtC
W_evtC -.-> EB
EB -.-> LI_evt
EB -.-> N_Send
N_Send --> FE_Toast
EB -.-> AU_Log

%% Correcciones (v+1)
O_Correct -.-> EB
EB -.-> S_Eval
S_Eval -.-> S_evt
S_evt -.-> EB
EB -.-> L_Rank
L_Rank -.-> L_evt
L_evt -.-> EB
EB -.-> LI_Adjust
LI_Adjust --> W_Credit
LI_Adjust --> W_Debit
W_Credit -.-> W_evtC
W_Debit -.-> W_evtD
W_evtC -.-> EB
W_evtD -.-> EB
EB -.-> N_Send
EB -.-> AU_Log

%% Cancelacion
T_Cancel -.-> EB
EB -.-> LI_Adjust
LI_Adjust --> W_Credit
W_Credit -.-> W_evtC
EB -.-> N_Send
EB -.-> AU_Log

%% Store -> Fulfillment
FE_Store --> ST_List
FE_Store --> ST_Redeem
ST_Redeem --> ST_Block
ST_Redeem --> W_Debit
W_Debit -.-> W_evtD
ST_Redeem -.-> ST_evt
ST_evt -.-> EB
EB -.-> F_Digital
EB -.-> F_Ship
F_Digital -.-> F_evt
F_Ship -.-> F_evt
F_evt -.-> EB
EB -.-> N_Send
N_Send --> FE_Toast
EB -.-> AU_Log

%% Lecturas (GET)
FE_Router --> P_History
FE_Wallet --> W_Mov
FE_Match --> TR_Reveal
FE_Match --> L_Rank

classDef fe fill:#f8fafc,stroke:#94a3b8,stroke-width:1px,color:#111827;
classDef bus fill:#eef2ff,stroke:#6366f1,stroke-width:2px,color:#111827;
classDef evt fill:#ecfeff,stroke:#06b6d4,stroke-width:1px,color:#0e7490;
class FE_Router,FE_Login,FE_TopUp,FE_Create,FE_Match,FE_Store,FE_Wallet,FE_Toast fe;
class EB bus;
class A_Evt,W_evtC,W_evtD,Pay_evtI,Pay_evtS,C_evt,S_evt,L_evt,LI_evt,ST_evt,F_evt,RF_evt evt;
```
## Descripción de Flujos  del Usuario en el app
>**Nota**: En esta sección se describe las acciones que el jugador va a realizar dentro del app como parte de su experiencia de juego.

1) **Registro de Cuenta**

   *Objetivo*: crear la cuenta y el perfil mínimo para permitir compras de Lukas y participación en torneos, dejando pendiente la verificación de correo y de mayoría de edad para el momento del canje.

   - Campos de registro (obligatorios): Correo electrónico, constraseña, Nombres, Apellidos, Nickname / Nombre visible, Fecha de Nacimiento, País de Residencia (código ISO-2), Aceptaciones (Términos y Condiciones, Política de Privacidad, Política de Juego Responsable, Declaración de ser mayor de 18 años
   

## Microservicios (responsabilidades, interfaces y eventos)

Formato por servicio: Responsabilidades · Endpoints (ejemplos) · Publica · Suscribe · Datos

1. **Auth**
- Resp.: identidad, sesiones (JWT), alta/login.
- Endpoints: POST /auth/signup, POST /auth/login.
- Publica: User.Registered.
- Suscribe: —
- Datos: usuarios (id, email, estado verificación).

2. **Perfil**
- Resp.: perfil básico, alias, avatar; historial agregado 30 días.
- Endpoints: GET /me/history?days=30.
- Publica: —
- Suscribe: User.Registered (para crear perfil).
- Datos: perfil y vistas materializadas de historial.

3. **Wallet**
- Resp.: saldo de Lukas, asientos contables, idempotencia.
- Endpoints: POST /wallet/credit, POST /wallet/debit, GET /wallet/movements.
- Publica: Wallet.CreditPosted, Wallet.DebitPosted.
- Suscribe: pagos liquidados, liquidación de premios, ajustes.
- Datos: ledger (append-only) y saldos.

4. **Pagos (On-Ramp)**
- Resp.: checkout con PSP, conciliación, conversión a Lukas.
- Endpoints: POST /payments, POST /webhooks/psp.
- Publica: Payment.Initiated, Payment.Settled.
- Suscribe: —
- Datos: órdenes de pago, estados PSP, trazas.

5. **Torneos & Partidos**
- Resp.: alta de partido/torneo; estados open/en_juego/terminado/cancelado; lock_time.
- Endpoints: (backoffice) POST /tournaments, PATCH /match/state.
- Publica: Match.StateChanged(en_juego|terminado|cancelado).
- Suscribe: —
- Datos: partidos/torneos y metadatos.

6. **Reglas & Puntajes**
- Resp.: catálogo versionado de reglas (puntajes fijos).
- Endpoints: GET /rulesets/:id.
- Publica: —
- Suscribe: —
- Datos: ruleset_id referenciado por torneos.

7. **Combinadas (Tickets)**
- Resp.: validar picks, unicidad por (user, match, hash), registrar tickets.
- Endpoints: POST /tickets.
- Publica: Ticket.Submitted.
- Suscribe: Match.StateChanged(en_juego) (lock de venta).
- Datos: tickets con hash/merkletree ref.

8. **Oráculos de Datos**
- Resp.: hechos oficiales (goles, BTTS, O2.5, roja, FT).
- Endpoints: webhooks/ingesta proveedor.
- Publica: MatchFacts.Finalized, MatchFacts.Corrected(v+1).
- Suscribe: —
- Datos: snapshot por partido (versionado).

9. **Transparencia**
- Resp.: WORM/append-only; Reveal al kickoff; Merkle root.
- Endpoints: GET /transparency/:matchId.
- Publica: —
- Suscribe: Ticket.Submitted, Match.StateChanged(en_juego).
- Datos: set de tickets por partido + pruebas de no-mutación.

10. **Scoring**
- Resp.: evaluar cada ticket vs hechos; sumar puntos; desglose.
- Endpoints: (interno) POST /score/:matchId.
- Publica: Scoring.Completed.
- Suscribe: Match.StateChanged(terminado), MatchFacts.Finalized|Corrected.
- Datos: resultados por ticket.

11. **Leaderboard**
- Resp.: ranking por torneo; desempates (exacto → timestamp → prorrateo).
- Endpoints: GET /leaderboard/:matchId.
- Publica: Leaderboard.Finalized.
- Suscribe: Scoring.Completed.
- Datos: snapshot inmutable por versión.

12. **Liquidación**
- Resp.: cálculo de pozo (Σ fees − rake); abono a ganadores; ajustes por correcciones/cancelaciones.
- Endpoints: (interno) POST /settle/:matchId.
- Publica: Payouts.Posted.
- Suscribe: Leaderboard.Finalized, Match.StateChanged(cancelado).
- Datos: órdenes de pago de premio y ajustes.

13. **Tienda de Premios**
- Resp.: catálogo, reglas, canje de Lukas y reserva de stock.
- Endpoints: GET /store, POST /redeem.
- Publica: Reward.Redeemed.
- Suscribe: —
- Datos: productos, stock y órdenes de canje.

14. **Fulfillment**
- Resp.: entrega digital (códigos) o física (courier).
- Endpoints: (interno) POST /fulfill/:orderId.
- Publica: Fulfillment.Completed.
- Suscribe: Reward.Redeemed.
- Datos: estados de entrega.

15. **Riesgo / Fraude**
- Resp.: velocity/IP/device, multi-cuenta, spikes, patrones.
- Endpoints: (interno) POST /risk/check.
- Publica: Risk.Flagged.
- Suscribe: Ticket.Submitted, Reward.Redeemed, Payment.Settled.
- Datos: señales y decisiones.

16. **Notificaciones**
- Resp.: plantillas y envío a email/push/in-app.
- Endpoints: POST /notify.
- Publica: —
- Suscribe: eventos clave (pagos, tickets, leaderboard, payouts, fulfillment).
- Datos: colas de notificación y logs de entrega.

17. **Auditoría & Logs**
- Resp.: append-only de todos los eventos para trazabilidad.
- Endpoints: (interno) POST /audit/append.
- Publica: —
- Suscribe: todos los eventos del bus.
- Datos: almacén WORM.

## Catálogo de eventos canónicos (resumen)

- **Usuarios/Wallet/Pagos**: User.Registered, Wallet.CreditPosted, Wallet.DebitPosted, Payment.Initiated, Payment.Settled.
- **Torneos/Partidos**: Match.StateChanged(en_juego|terminado|cancelado), Ticket.Submitted.
- **Datos/Resolución**: MatchFacts.Finalized, MatchFacts.Corrected, Scoring.Completed, Leaderboard.Finalized, Payouts.Posted.
- **Tienda/Entrega**: Reward.Redeemed, Fulfillment.Completed.
- **Riesgo**: Risk.Flagged.
> **Event Bus (Kafka o NATS)**: backbone de mensajería asíncrona para publicar/suscribir eventos, escalar consumidores y lograr consistencia eventual con bajo acoplamiento.

## Licencia
© Habla! 2025. Todos los derechos reservados.
