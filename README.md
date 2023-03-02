# Propósito
Este repositório foi criado com o propósito de demonstrar uma alternativa rápida e extremamente fácil de ter uma API do estilo "data driven" e com zero codificação. Sendo assim uma ótima opção para quando há necessidade de uma prototipação rápida, ou construir uma POC com pouco esforço na parte de backend.
 
O arquivo `docker-compose.yml` possui dois serviços: Um banco de dados Postgres e a grande estrela: **pRest** <https://prestd.com>.
 
O pRest transforma seu banco de dados Postgres em um serviço Rest instantaneamente, com autenticação, JWT e sem a necessidade de escrever uma linha de código. 🤩🤩
 
Para entender como funciona as operações de 'CRUD' consulte <https://docs.prestd.com/prestd/docs/api-reference/>. Já aviso que a API rest do pRest é bastante poderosa, posibilitando fazer queries mais complexas inclusive com joins e muito mais, consulte <https://docs.prestd.com/prestd/docs/api-reference/advanced-queries/>

# Como fazer
## Requisitos
 - Ter o Docker instalado
 - Ter o Docker Compose instalado

## Mão na massa
### configuração
Abra um terminal na pasta em que você clonou esse repositório e digite:

```shell
docker-compose up
```
Um banco novo do postgres foi criado e está disponível para utilizarmos.
Como indicamos para o pRest que iremos trabalhar com uma API com autenticação (indicado no `docker-compose.yml` em `PREST_AUTH_ENABLED=true`) e como também nosso banco de dados é novo e não possui nenhuma tabela ainda, é necessário que uma estrutura mínima de autenticação seja criada. E o pRest tem um comando para isso. Abra outro terminal e digite:

```shell
docker-compose exec prest prestd migrate up auth
```
Agora com a estrutura criada, podemos criar um usuário para poder utilizar a API.

Vamos criar usuário com o nome 'pREST Full Name', username 'prest' e a senha também será 'prest'

```shell
docker-compose exec postgres psql -d prest -U prest -c "INSERT INTO prest_users (name, username, password) VALUES ('pREST Full Name', 'prest', MD5('prest'))"
```
Você pode verificar se o usuário realmente foi criado com um select simples

```shell
docker-compose exec postgres psql -d prest -U prest -c "select * from prest_users"
```
A saída deve ser algo parecido com isso
```
 id |      name       | username |             password             | metadata
----+-----------------+----------+----------------------------------+----------
  1 | pREST Full Name | prest    | 8c60e97b9cf709c643e4a583b1ae9cb1 |
(1 row)
```

E PRONTO!!!🤩🤩 Uma API segura em frente ao seu banco de dados, e agora pode consumi-lo por http, sem precisar de conectores ou escrever sql. 


### Autenticar
Você pode enviar as credenciais como json para o endpoint `/auth`. Você pode utilizar qualquer cliente http, como postman, insonmia e etc, ou o bom e velho curl:

```shell
curl -i -X POST http://127.0.0.1:3000/auth -H "Content-Type: application/json" -d '{"username": "prest", "password": "prest"}'
```

O retorno deve ser algo parecido com isso

```
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 02 Mar 2023 01:43:53 GMT
Content-Length: 329

{"user_info":{"id":1,"name":"pREST Full Name","username":"prest","metadata":null},"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJVc2VySW5mbyI6eyJpZCI6MSwibWV0YWRhdGEiOm51bGwsIm5hbWUiOiJwUkVTVCBGdWxsIE5hbWUiLCJ1c2VybmFtZSI6InByZXN0In0sImV4cCI6MTY3Nzc0MzAzMywibmJmIjoxNjc3NzQzMDMzfQ.wVKffo6oSuMiFj8JgQD6nXGQ1n-Xgj-zdXrH7zhGtBM"}
```
Onde o que nos interessa aqui é o conteúdo da resposta

```json
{
    "user_info":{
        "id":1,
        "name":"pREST Full Name",
        "username":"prest",
        "metadata":null
    },
    "token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJVc2VySW5mbyI6eyJpZCI6MSwibWV0YWRhdGEiOm51bGwsIm5hbWUiOiJwUkVTVCBGdWxsIE5hbWUiLCJ1c2VybmFtZSI6InByZXN0In0sImV4cCI6MTY3Nzc0MzAzMywibmJmIjoxNjc3NzQzMDMzfQ.wVKffo6oSuMiFj8JgQD6nXGQ1n-Xgj-zdXrH7zhGtBM"
}
```
Onde o token recebido deve ser utilizado em todas as requisições a serem feitas para a API

### Criar uma tabela
Para simplificação vamos criar uma tabela nova diretamente no banco de dados
```shell
docker-compose exec postgres psql -d prest -U prest -c "create table books (id serial PRIMARY KEY,title varchar(255),author varchar(255), pages integer,quantity integer)"
```
### Utilizando o pRest
A estrutura de path do pRest é bem simples sendo `/{database}/{schema}/{table}`
Filtros também são possíveis através de query params, como por exemplo:

```
curl -i -X GET http://127.0.0.1:3000/prest/public/books?id=1 -H "Accept: application/json" -H "Authorization: Bearer {token}"
```

Vamos enviar um novo registro para a nossa api

```shell
curl -i -X POST http://127.0.0.1:3000/prest/public/books -H "Accept: application/json" -H "Authorization: Bearer {token}" -d '{"title": "Dune", "author": "Frank Herbert", "pages":680, "quantity":100}'
```

O retorno será algo parecido com isso

```shell
HTTP/1.1 201 Created
Content-Type: application/json
Date: Thu, 02 Mar 2023 02:01:10 GMT
Content-Length: 75

{"id":1,"title":"Dune","author":"Frank Herbert","pages":680,"quantity":100}
```

Fazendo consultas na tabela books como `select * from books where title like %title%`

```shell
curl "http://127.0.0.1:3000/prest/public/books?title:tsquery=dune" -H 'Accept: application/json' -H 'Authorization: Bearer {token}'
```

Atualizar um registro
```shell
curl -i -X PUT http://127.0.0.1:3000/prest/public/books?id=1 -H "Accept: application/json" -H "Authorization: Bearer {token}" -d '{"title": "updated title", "author": "updated author"}'
```
O retorno será algo parecido com isso

```shell
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 02 Mar 2023 02:04:32 GMT
Content-Length: 19

{"rows_affected":1}
```

Remover um registro

```shell
curl -i -X DELETE http://127.0.0.1:3000/prest/public/books?id=1 -H "Accept: application/json" -H "Authorization: Bearer {token}"
```

Retorno

```
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 02 Mar 2023 02:17:14 GMT
Content-Length: 19

{"rows_affected":1}
```
Bom pessoal, esse é o básico para mais operações consulte a documentação oficial <https://docs.prestd.com/>


