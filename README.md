# Arena Multiplayer

Jogo de arena 2D multiplayer em tempo real que roda no navegador. O servidor em **Flask + Socket.IO** é a autoridade do jogo (movimento, colisões, dano, pickups, mortes e vitória); o front-end desenha tudo em um **HTML5 canvas** de 1280x720.

Mova-se com **WASD**, mire com o mouse, pegue armas e bônus espalhados pelo mapa e seja o primeiro a chegar a **20 kills**.

---

## Funcionalidades

- **Multiplayer em tempo real** via WebSocket, com todos os jogadores na mesma arena
- **Mapas com paredes**, sorteados a cada partida
- **Três tipos de ataque**: tiro, golpe corpo a corpo e Kamehameha
- **Pickups** que aparecem periodicamente em posições aleatórias livres de paredes
- **Dash** com clique direito (depois de coletar o bônus)
- **Kill streak**: 5 kills seguidas concedem o Kamehameha
- **Respawn automático** 5 segundos após a morte
- **Kill feed** e placar com kills e mortes
- **Troca de nickname** durante o jogo
- **Efeitos sonoros** gerados com Web Audio API, sem arquivos de áudio
- Fim de partida ao atingir **20 kills**

---

## Controles

| Ação | Comando |
|---|---|
| Mover | `W` `A` `S` `D` |
| Mirar | Mouse |
| Atacar (conforme o item equipado) | Clique esquerdo |
| Dash (após pegar o bônus) | Clique direito |
| Iniciar partida | Botão **Start Game** |
| Voltar ao jogo após morrer | Botão **Respawn** (ou aguarde o respawn automático) |
| Trocar nome | Campo de texto + **Change Name** |

---

## Itens (pickups)

| Tipo | Efeito |
|---|---|
| 1 | **Arma de fogo**: +10 munições, ataque à distância |
| 2 | **Vida**: recupera 10 de HP (máximo 20) |
| 3 | **Dano dobrado** nos tiros |
| 4 | **Ataque corpo a corpo**: golpe em arco de 180° (dano 10, cooldown de 5 s) |
| 5 | **Velocidade**: +20% a cada coleta |
| 6 | **Dash**: libera o dash com o clique direito (160 px) |
| Kill streak (5 kills) | **Kamehameha**: raio que atravessa a arena por 5 s e elimina quem estiver na linha |

---

## Regras

- Cada jogador começa com **20 de HP**
- Tiro causa 1 de dano (multiplicado pelo bônus de dano), o golpe corpo a corpo causa 10 e o Kamehameha mata instantaneamente
- Ao morrer, o jogador volta sozinho após 5 segundos, em uma posição livre
- Vence quem chegar primeiro a **20 kills**; ao fim da partida o placar é zerado
- Quando todos saem, o servidor reinicia o estado do jogo

---

## Como funciona

```
Navegador (index.html)  ◄── Socket.IO ──►  Flask-SocketIO (app.py)
  canvas, entrada (WASD/mouse),            estado dos jogadores, mapa,
  render, sons                             colisões, dano, pickups
                                                  │
                                    tarefas em segundo plano (~60 Hz)
                                    balas · Kamehameha · vitória ·
                                    kill streak · spawn de pickups
```

O cliente envia **intenções** (mover, atacar, mirar) e o servidor decide o resultado e retransmite o estado atualizado para todos.

---

## Requisitos

- Python 3.9+
- Navegador moderno

```bash
pip install flask flask-socketio
```

O front-end carrega o **Socket.IO client** por CDN, então é preciso acesso à internet (ou hospedar o script localmente).

---

## Instalação e execução

```bash
git clone https://github.com/<seu-usuario>/arena-multiplayer.git
cd arena-multiplayer

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python app.py
```

O servidor sobe em `http://0.0.0.0:5080`. Para jogar em rede local, os outros jogadores acessam `http://<ip-do-servidor>:5080`.

---

## Estrutura do projeto

```
.
├── app.py
├── templates/
│   └── index.html
├── requirements.txt
└── README.md
```

---

## Configuração

| Item | Onde | Descrição |
|---|---|---|
| Porta | `socketio.run(...)` | Padrão `5080` |
| `debug=True` | `socketio.run(...)` | Desative em produção |
| `cors_allowed_origins="*"` | `SocketIO(...)` | Restrinja aos domínios usados em produção |
| `TODOS_OS_MAPAS` | `app.py` | Lista de mapas sorteados a cada partida |
| `KAMEHAMEHA_DURATION_S` | `app.py` | Duração do Kamehameha (padrão 5 s) |
| `MELEE_RANGE` | `app.py` | Alcance do golpe corpo a corpo |
| `MELEE_COOLDOWN_MS` | `index.html` | Cooldown do golpe corpo a corpo |

### Criando mapas

Cada mapa é uma matriz de **18 linhas x 32 colunas** (tiles de 40 px), em que `0` é caminho livre e `1` é parede. Para adicionar um novo layout, crie a matriz e inclua em `TODOS_OS_MAPAS`.

---

## Eventos Socket.IO

### Cliente → Servidor

| Evento | Descrição |
|---|---|
| `start_game` | Inicia a partida e sorteia o mapa |
| `respawn` | Reaparece no mapa |
| `change_name` | Altera o nickname |
| `move_up` / `move_down` / `move_left` / `move_right` | Movimento (`{speed}`) |
| `send_mouse` | Posição do mouse |
| `attack` | Disparo de projétil |
| `melee` | Golpe corpo a corpo |
| `kamehameha` | Dispara o Kamehameha |
| `dash` | Ativa o dash |
| `end_game` | Reinicia o estado ao fim da partida |

### Servidor → Cliente

| Evento | Descrição |
|---|---|
| `update_player_pos` | Estado dos jogadores (posição, HP, kills, mortes...) |
| `update_game_state` | Se a partida está em andamento |
| `recebe_mapa` | Paredes do mapa atual |
| `spawn_pickup` / `pickup_dispawn` | Pickups no mapa |
| `spawn_bullets` | Projéteis ativos |
| `melee_swing` / `kamehameha_fire` | Animações de ataque |
| `kill_feed` | `{killer_name, victim_name}` |
| `equip_weapon` | Jogador equipou um item |
| `end_game` | Fim de partida |

---

## Limitações conhecidas

- Todo o estado fica **em memória** e há **uma única arena global**, sem salas.
- O servidor recalcula e retransmite o estado completo dos jogadores a cada movimento, o que não escala bem para muitos jogadores.
- A lógica de movimento e coleta de itens está repetida nas quatro direções.
- Não há autenticação nem proteção contra clientes modificados.
- O mapa 3 está definido no código, mas ainda não entra no sorteio.

---

## Ideias futuras

- Salas e partidas separadas
- Refatorar o movimento em uma única função
- Enviar apenas as mudanças de estado (delta) em vez do estado completo
- Mais mapas, armas e modos de jogo
- Placar final e histórico de partidas

---

