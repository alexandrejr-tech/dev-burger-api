# Dev Burguer API

API REST de uma hamburgueria: cadastro e login de usuários, catálogo de produtos e categorias (com upload de imagem) e pedidos.

Construída com **Node.js + Express 5**, usando dois bancos ao mesmo tempo:

- **PostgreSQL** (via Sequelize) — usuários, produtos e categorias;
- **MongoDB** (via Mongoose) — pedidos.

---

## Stack

| Camada | Ferramenta |
| --- | --- |
| Servidor HTTP | Express 5 |
| Banco relacional | PostgreSQL + Sequelize 6 (`sequelize-cli` para migrations) |
| Banco de documentos | MongoDB + Mongoose 9 |
| Autenticação | JWT (`jsonwebtoken`) + `bcrypt` |
| Validação | Yup |
| Upload de arquivos | Multer (disco local, nome gerado com `uuid`) |
| Lint / format | Biome |
| Gerenciador de pacotes | pnpm |

---

## Pré-requisitos

- Node.js 18+ (o script `dev` usa `node --watch`)
- pnpm
- PostgreSQL rodando em `localhost:5432`
- MongoDB rodando em `localhost:27017`

As configurações de conexão ficam em:

- Postgres — [src/config/database.cjs](src/config/database.cjs)
- Mongo — [src/database/index.js](src/database/index.js)
- JWT — [src/config/auth.js](src/config/auth.js)

Ajuste esses arquivos conforme o seu ambiente antes de subir a aplicação.

---

## Instalação e execução

```bash
pnpm install

# cria as tabelas do Postgres
pnpm exec sequelize db:migrate

# sobe a API em modo watch
pnpm dev
```

A aplicação sobe em `http://localhost:3001` ([src/server.js](src/server.js)).

### Migrations

Os caminhos do `sequelize-cli` são definidos em [.sequelizerc](.sequelizerc).

```bash
pnpm exec sequelize db:migrate            # aplica as migrations pendentes
pnpm exec sequelize db:migrate:undo       # desfaz a última
pnpm exec sequelize migration:generate --name nome-da-migration
```

---

## Estrutura

```
src/
├── app.js                  # instancia o Express, CORS, JSON e rotas estáticas de imagens
├── server.js               # sobe o servidor na porta 3001
├── routes.js               # todas as rotas e onde os middlewares entram
├── app/
│   ├── controllers/        # User, Session, Product, Category, Order
│   ├── middlewares/        # auth (JWT) e admin (checa a flag admin)
│   ├── models/             # models Sequelize: User, Product, Category
│   └── schemas/            # schema Mongoose: Order
├── config/
│   ├── auth.js             # segredo e expiração do JWT (7d)
│   ├── database.cjs        # conexão Postgres (também lida pelo sequelize-cli)
│   ├── multer.cjs          # storage em disco, arquivos vão para uploads/
│   └── fileRoutes.cjs      # express.static apontando para uploads/
└── database/
    ├── index.js            # conecta Postgres + Mongo e inicializa os models
    └── migrations/         # migrations do sequelize-cli
```

---

## Autenticação

Em [src/routes.js](src/routes.js), `POST /users` e `POST /sessions` são públicas. Tudo abaixo de `routes.use(authMiddleware)` exige o header:

```
Authorization: Bearer <token>
```

O [middleware de auth](src/app/middlewares/auth.js) valida o token e injeta `request.userId`, `request.userName` e `request.userIsAdmin`. As rotas de escrita de produtos/categorias e as de listagem/atualização de pedidos passam ainda pelo [middleware de admin](src/app/middlewares/admin.js), que devolve `401` se o usuário não for admin.

---

## Endpoints

### Usuários e sessão

#### `POST /users` — público

```json
{
  "name": "Nome do Usuário",
  "email": "usuario@email.com",
  "password": "suaSenhaSegura",
  "admin": true
}
```

`admin` é opcional (default `false`). Retorna `201` com o usuário criado (sem o hash da senha); `400` se o e-mail já existir.

#### `POST /sessions` — público

```json
{
  "email": "usuario@email.com",
  "password": "suaSenhaSegura"
}
```

Retorna `200` com `id`, `name`, `email`, `admin` e o `token` JWT (válido por 7 dias).

### Produtos

#### `GET /products` — autenticado

Lista todos os produtos com a categoria associada (`id` e `name`). Cada produto traz um campo virtual `url` com o link público da imagem.

#### `POST /products` — admin · `multipart/form-data`

| Campo | Tipo | Obrigatório |
| --- | --- | --- |
| `name` | string | sim |
| `price` | number (centavos) | sim |
| `category_id` | number | sim |
| `offer` | boolean | não |
| `file` | arquivo (imagem) | sim |

#### `PUT /products/:id` — admin · `multipart/form-data`

Mesmos campos, todos opcionais. Se `file` não for enviado, a imagem atual é mantida.

### Categorias

#### `GET /categories` — autenticado

Lista as categorias, cada uma com a `url` pública da imagem.

#### `POST /categories` — admin · `multipart/form-data`

`name` (obrigatório) + `file`. Retorna `400` se já existir categoria com o mesmo nome.

#### `PUT /categories/:id` — admin · `multipart/form-data`

`name` e `file` opcionais.

### Pedidos

#### `POST /orders` — autenticado

```json
{
  "products": [
    { "id": 1, "quantity": 2 },
    { "id": 4, "quantity": 1 }
  ]
}
```

A API busca os produtos no Postgres, monta o pedido com os dados do usuário (extraídos do token) e grava no MongoDB com status inicial `"Pedido Realizado"`.

#### `GET /orders` — admin

Lista todos os pedidos.

#### `PUT /orders/:id` — admin

```json
{ "status": "Em preparação" }
```

O `:id` é o `_id` do documento no MongoDB.

---

## Upload de imagens

Arquivos enviados via `file` vão para a pasta `uploads/` na raiz, renomeados para `<uuid>-<nome-original>` ([src/config/multer.cjs](src/config/multer.cjs)).

São servidos estaticamente em dois prefixos ([src/app.js](src/app.js)):

- `GET /product-file/:filename`
- `GET /category-file/:filename`

Os models [Product](src/app/models/Product.js) e [Category](src/app/models/Category.js) montam a `url` completa em um campo virtual, hoje com o host `http://localhost:3001` fixo no código.

---

## Lint e formatação

Configuração em [biome.json](biome.json) — indentação com tab e aspas simples.

```bash
pnpm exec biome check src      # verifica
pnpm exec biome check --write src   # corrige
```

---

## Pontos de atenção

Itens conhecidos do estado atual do projeto, caso alguém vá levá-lo adiante:

- **Configuração hardcoded.** Dados de conexão e a URL base das imagens estão no código. Migrar para variáveis de ambiente é o primeiro passo antes de qualquer deploy.
- **`node_modules/` está commitado** no repositório, apesar de constar no [.gitignore](.gitignore) (foi adicionado antes do ignore existir). Para limpar: `git rm -r --cached node_modules`.
- **`SessionController`** chama `emailOrPasswordIncorrect()` sem `return`, então a execução continua após uma credencial inválida em vez de parar ali.
- Não há testes automatizados nem rotas de `DELETE` para produtos e categorias.
