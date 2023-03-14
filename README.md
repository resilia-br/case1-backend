
# Passo a passo de construção

1. `npm init -y` para inicar o npm
2. `npm install cors express sqlite3`
3. `npm install --save-dev nodemon` Para instalar como dependência de desenvolvimento
4. Adicionar no package.json o seguinte script:

```
  "scripts": {
    "start": "nodemon ./src/server.js",
  },
```

5. Criar arquivo `app.js` dentro de `src` com as configurações básicas do projeto:

```js
  import express from 'express'
  import cors from 'cors'

  // instanciando o servidor
  const app = express()

  // configurando o servidor para receber requisições com o corpo no formato JSON
  app.use(express.json())

  // configurando o servidor para receber requisições qualquer origem
  app.use(cors())

  export default app
```

6. Criar arquivo `server.js` para iniciar o servidor usando as configurações do `app.js`:

```js
  import app from './app.js'

  // escolhendo a porta em que o servidor será aberto
  const port =  3000

  // abrindo o servidor na porta escolhida
  app.listen(port, ()=>{
    console.log(`Server rodando em http://localhost:${port}/`)
  })
```

7. Criar arquivo `db.js` dentro de uma pasta com nome de `infra` para fazer a configuração do sqlite:

```js
  import sqlite3 from 'sqlite3'

  sqlite3.verbose()
  const db = new sqlite3.Database('./src/infra/database.db')

  export default db;
```

8. Criar arquivo `contents.js` dentro de infra para criar a tabela de conteúdos:

```js
    /*
  Esse arquivo deve ser executado apenas uma vez para que a tabela do banco seja criada
  */
  import db from "./db.js";

  //==== Conteúdos
  const CONTENTS_SCHEMA = `
  CREATE TABLE IF NOT EXISTS "conteudos" (
      "ID" INTEGER PRIMARY KEY AUTOINCREMENT,
      "TITULO" varchar(64),
      "DESCRICAO" varchar(64),
      "PORCENTAGEM" INTEGER
    );`;

  function createTableContents() {
      db.run(CONTENTS_SCHEMA, (error)=> {
        if (error) console.log("Erro ao criar tabela de conteúdos");
      });
  }

  db.serialize( ()=> {
      createTableContents();
  });
```

9. Agora para rodar esse script vamos importá-lo no arquivo `server.js` logo antes do nosso `app.listen`:

```js
  import './infra/contents.js'
```

10. Dentro de uma pasta com o nome de `controllers` vamos criar o arquivo `contentsController.js` com uma estrutura básica:

```js
  class contentsController {
    static rotas(app){
      app.get('/conteudo', contentsController.listar)
      app.post('/conteudo', contentsController.inserir)
    }

    static async listar(req, res){
      console.log('a')
    }

    static async inserir(req, res){
      console.log('b')
    }
  }

  export default contentsController
```

11. Dentro do arquivo `app.js` vamos importar nosso controller chamando o método `rotas` para registrar as rotas iniciais da nossa aplicação:

```js
  import contentsController from './controllers/contentsController.js'

  contentsController.rotas(app)
```

12. Agora precisamos fazer as ações desses métodos no banco. Vamos criar um arquivo chamadao `ContentsDAO` dentro de uma pasta chamada DAO, com as ações de listar e criar conteúdo:

```js
  import db from '../infra/db.js'

  class ContentsDAO {
      static listar() {
          const query = 'SELECT * FROM conteudos';
          return new Promise((resolve, reject) => {
              db.all(query, (err, rows) => {
                  if (err) {
                      reject(err);
                  }

                  resolve(rows)
              });
          });
      }

      static inserir(conteudo) {
        const query = 'INSERT INTO conteudos (titulo, descricao, porcentagem) VALUES (?, ?, ?)';
        return new Promise((resolve, reject) => {
            db.run(query, [conteudo.titulo, conteudo.descricao, conteudo.porcentagem], function (err) {
                if (err) {
                    reject({
                        mensagem: 'Erro ao inserir o conteúdo',
                        erro: err
                    })
                }

                resolve({
                    mensagem: 'Conteúdo criado com sucesso',
                    data: this.lastID
                 })
            });
        });
    }
  }

  export default ContentsDAO;
```

13. Agora com esses métodos de ação no banco só precisamos chamar eles no controller, e vamos fazer desta maneira:

```js
  static async listar(req, res){
    const conteudos = await ContentsDAO.listar()

    res.send(conteudos)
  }

  static async inserir(req, res){
    const conteudo = {
      titulo: req.body.titulo,
      descricao: req.body.descricao,
      porcentagem: req.body.porcentagem
    }

    const result = await ContentsDAO.inserir(conteudo)

    if(result.erro) {
      res.status(500).send(result)
    }

    res.send(result)
  }
```

14. Agora vamos fazer a deleção e atualização no nosso `ContentsDAO.js`:

```js
  static deletar(id) {
    const query = 'DELETE FROM conteudos WHERE id = ?';
    return new Promise((resolve, reject) => {
        db.run(query, [id], (err) => {
            if (err) {
                reject({
                    mensagem: 'Erro ao deletar o conteúdo',
                    erro: err
                })
            }

            resolve({ mensagem: 'Conteúdo deletado com sucesso' })
        });
    });
  }

  static atualizar(id, conteudo) {
    const query = 'UPDATE conteudos SET titulo = ?, descricao = ?, porcentagem = ? WHERE id = ?';
    return new Promise((resolve, reject) => {
        db.run(query, [conteudo.titulo, conteudo.descricao, conteudo.porcentagem, id], (err) => {
            if (err) {
                reject({
                    mensagem: 'Erro ao atualizar o conteúdo',
                    erro: err
                })
            }

            resolve({ mensagem: 'Conteúdo atualizado com sucesso' })
        });
    });
  }
```

15. Com esses métodos que se comunicam com o banco já feitos, precisamos chamar eles no controller:

```js
  static async deletar(req, res){
    const conteudo = await ContentsDAO.deletar(req.params.id)

    if(conteudo.erro){
        res.status(500).send('Erro ao deletar o conteúdo')
    }

    res.send({mensagem: 'Conteúdo removido com sucesso'})
  }

  static async atualizar(req, res){
    const conteudo = {
      titulo: req.body.titulo,
      descricao: req.body.descricao,
      porcentagem: req.body.porcentagem
    }

    const result = await ContentsDAO.atualizar(req.params.id, conteudo)

    if(result.erro){
        res.status(500).send('Erro ao atualizar o conteúdo')
    }

    res.send({mensagem: 'Conteúdo alterado com sucesso'})
  }
```

16. Agora para conseguirmos usar esses métodos precisamos adicionar essas rotas lá no método `rotas` do nosso controller:

```js
  app.delete('/conteudo/:id', contentsController.deletar)
  app.put('/conteudo/:id', contentsController.atualizar)
```