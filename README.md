# Derby Synthetica API

Backend do portal **Derby Synthetica**, entrega de Framework Application do
challenge FIAP "O Mundo de Synthetica" (2026).

API REST em **FastAPI** com os quatro verbos do CRUD sobre conteúdos
editoriais, mais a leitura das categorias. Os dados ficam em memória e são
recarregados do `dados_iniciais.json` a cada startup, então o portal nunca
aparece vazio.

O frontend que consome esta API é um projeto Next separado.

## Como rodar

Requer Python 3.10 ou superior (testado no 3.14).

```bash
# 1. ambiente virtual
python -m venv .venv

# Windows
.venv\Scripts\activate
# Linux ou macOS
source .venv/bin/activate

# 2. dependências
pip install -r requirements.txt

# 3. servidor
uvicorn main:app --reload
```

A API sobe em **http://127.0.0.1:8000** e a documentação interativa fica em
**http://127.0.0.1:8000/docs**, que é por onde dá para testar tudo sem escrever
uma linha de código.

## Endpoints

Base local: `http://127.0.0.1:8000`

| Método | Rota | Função |
|---|---|---|
| GET | `/categorias` | Lista as categorias. Filtro: `trilha` |
| GET | `/conteudos` | Lista conteúdos. Filtros: `trilha`, `categoria_id`, `busca` |
| GET | `/conteudos/{id}` | Detalha pelo id |
| GET | `/conteudos/slug/{slug}` | Detalha pelo slug, que é como o portal monta a URL |
| POST | `/conteudos` | Cadastra. Devolve 201 |
| PUT | `/conteudos/{id}` | Atualiza por inteiro |
| DELETE | `/conteudos/{id}` | Remove. Devolve 204 |
| GET | `/` | Checagem de saúde, usada pelo deploy |

### Códigos de erro

| Código | Quando acontece |
|---|---|
| 404 | Id ou slug que não existe |
| 409 | Slug já usado por outro conteúdo |
| 422 | Payload inválido ou `categoria_id` inexistente |

### Exemplos

```bash
# listar tudo
curl http://127.0.0.1:8000/conteudos

# filtrar por trilha
curl "http://127.0.0.1:8000/conteudos?trilha=velocidade"

# buscar por texto
curl "http://127.0.0.1:8000/conteudos?busca=jammer"

# detalhar pelo slug
curl http://127.0.0.1:8000/conteudos/slug/o-que-e-roller-derby
```

## Modelo de dados

Os nomes de campo estão em português porque seguem o MER da disciplina de
Database Application. O frontend traduz para inglês na própria camada de
services, de propósito, para a API não ficar amarrada ao vocabulário da
interface.

```
Categoria                      Conteudo
  id               int           id                  int
  slug             str           slug                str  único
  nome             str           titulo              str
  descricao_curta  str           subtitulo           str
  trilha           enum          categoria_id        int  FK
                                 resumo              str
                                 tempo_leitura       str
                                 data                str
                                 autor               str
                                 autor_cargo         str
                                 destaque            bool
                                 selo_editorial      str  opcional
                                 arquetipo_grafico   str
                                 aprendizados        list[str]
                                 secoes              list[Secao]
                                 slugs_relacionados  list[str]
```

`Secao`: `texto` obrigatório, mais `titulo`, `citacao` e `destaque` opcionais.

`trilha`: `velocidade` ou `expressao`.

### A trilha pertence à categoria

`categoria_slug`, `categoria_nome` e `trilha` **são derivados na leitura e
nunca gravados no conteúdo**. Uma função privada do `database.py` acrescenta
os três a partir da categoria antes de devolver a resposta.

Essa é a razão de existir da regra: se a trilha fosse copiada para dentro do
conteúdo, editar a categoria depois faria as duas divergirem em silêncio.
Derivando na leitura, trocar a `categoria_id` de um conteúdo faz a trilha dele
acompanhar sozinha, e mandar `trilha` no corpo de um POST não tem efeito
nenhum, porque o schema de entrada simplesmente não tem esse campo.

## Arquitetura

| Arquivo | Responsabilidade |
|---|---|
| `main.py` | Aplicação FastAPI, CORS e rotas |
| `models.py` | Schemas Pydantic e validação |
| `database.py` | Base em memória e todas as regras de acesso |
| `dados_iniciais.json` | Seed: 7 categorias e 9 conteúdos |

Nenhuma rota lê as listas de dados diretamente: tudo passa pelas funções do
`database.py`. A intenção é que trocar a memória por um banco real signifique
reescrever só esse arquivo, sem encostar no `main.py`.

## CORS

O middleware libera, por padrão, `http://localhost:3000` e
`http://127.0.0.1:3000`, que é onde o Next roda em desenvolvimento.

No deploy, defina a variável de ambiente **`FRONTEND_URL`** com a URL pública
do frontend. Ela é acrescentada à lista de origens no startup:

```bash
FRONTEND_URL=https://[URL PÚBLICA DO FRONTEND] uvicorn main:app
```

Sem isso o navegador bloqueia as respostas e a tela fica vazia sem nenhum erro
aparecer no log do servidor, que é a falha mais comum nesse tipo de entrega.

## Pendências da entrega

- [ ] `FRONTEND_URL` preenchida com a URL real do deploy
- [ ] Link do repositório público
- [ ] Link da API publicada
