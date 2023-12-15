# Prova Sofia Pimazzoni

Esse arquivo estará dividido entre as vulnerabilidades encontradas no código e as melhorias que eu fiz nelas.

## Id não computado

O código para criar uma tabela nova no banco de dados estava assim:

``` js
db.run("CREATE TABLE cats (id INT, name TEXT, votes INT)");
```

Note que o "id"está definido como um inteiro, sendo que ele precisa ser autoincrementado toda vez ao criar um novo usuário, sendo ele ficará com o valor "null". Para isso, bastou trocar o tipo do valor, como está mostrado abaixo:

```js
db.run("CREATE TABLE cats (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, votes INT)");
```
**Observação:** essa modificação foi feita tanto para a tabela "cats" como para a "dogs"

## Corrigir lógica de votação

Inicialmente, o código de votação estava assim:

``` js
app.post('/vote/:animalType/:id', (req, res) => {
 
  db.run(`UPDATE ${animalType} SET votes = votes + 1 WHERE id = ${id}`);
  res.status(200).send("Voto computado");
});
```
Como é possível perceber, o tipo do animal e o id, que são identificadores, são passados como parâmetro, mas não estão definidos, então foi necessário adicionar a seguinte linha:

```js
const { animalType, id } = req.params;
```
Além disso, o código não tem verificação de usuário, nem mensagem de erro caso algo ocorra, ou seja, caso um usuário que não estivesse no banco de dados tentasse votar, ele iria receber uma mensagem dizendo que o voto foi computado, o que iria confundí-lo, pois a ação em si não iria ocorrer. Para isso, foi feito um tratamento de erro dentro da requisição de "UPDATE", além de uma verificação de usuáio. O código desse post completo pode ser visto abaixo:

```js
app.post('/vote/:animalType/:id', (req, res) => {
  const { animalType, id } = req.params;

  db.get(`SELECT * FROM ${animalType} WHERE id = ${id}`, (err, user) => {
    if (err) {
      return res.status(500).send("Erro ao consultar o banco de dados");
    }

    if (!user) {
      return res.status(404).send("Usuário não encontrado");
    }

    db.run(`UPDATE ${animalType} SET votes = votes + 1 WHERE id = ${id}`, function (err) {
      if (err) {
        return res.status(500).send("Erro ao atualizar os votos");
      }

      res.status(200).send("Voto computado");
    });
  });
});
```

## Implemetar e tratar erros sem vazar detalhes

Um erro grave que pude identificar, é que ao realizar uma requisição get para saber quantos votos tiveram em "cats" ou "dogs", eu obtive o nome e o id do usuário, que podem ser informações sigilosas dependendo do caso. Pensando nisso, eu alterei a consulta para retornar apenas os votos dos usuários, dessa forma: 

```js
db.all("SELECT votes FROM dogs", [], (err, rows) => {
    if (err) {
      res.status(500).send("Erro ao consultar o banco de dados");
    } else {
      res.json(rows);
    }
```
Essa implementação foi feita para ambas tabelas.

**Observação:** vale notar que essa não é uma falha de segurnaça se a pessoa que está realizando a consulta deveria ter acesso à essas informações


## Implementar todos os métodos

Infelizmente, no código original existiam 2 métodos incompletos, ambos envolvendo a tabela "dogs". Portanto, considerando que ela seguia a mesma lógica da tabela "cats", bastou copiar o mesmo código, alterando apenas o que era necessário. Os métodos completos podem ser vistos abaixo:

Post para criar um novo usuário na tabela 
```js
app.post('/dogs', (req, res) => {
  const name = req.body.name;
  db.run(`INSERT INTO dogs (name, votes) VALUES ('${name}', 0)`, function(err) {
    if (err) {
      res.status(500).send("Erro ao inserir no banco de dados");
    } else {
      res.status(201).json({ id: this.lastID, name, votes: 0 });
    }
  });
});
```

Get para consultar todos os usuários
```js
app.get('/dogs', (req, res) => {
  db.all("SELECT votes FROM dogs", [], (err, rows) => {
    if (err) {
      res.status(500).send("Erro ao consultar o banco de dados");
    } else {
      res.json(rows);
    }
  });
});
```
