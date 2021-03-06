
# Миграция на съществуващ NodeJS проект към StartApp.bg

**Важно:** За да работят примерите по-долу:

- Регистрирай се в [StartApp.bg](https://www.startapp.bg/#contacts)
- Инсталирай [StartApp Client Tools (app)](http://docs.startapp.bg/getting-started/app-client-tools-install.html)


#### 1. Клонирай проекта си на лоаклния компютър

Това, може да е всеки един товй проект. В нашия случай това е много простичък проект, който
изписва `Hello World`, когато заредите `/` пътя.

```bash
git clone https://github.com/StartappTemplates/existing_expressjs.git
```

#### 2. Инсталирай си пакетите дефинирани в `package.json`

```bash
npm install
```

#### 3. Стартирай приложението за да се увериш, че работи коректно.

```bash
node server.js
```

Горния ред стартира сървър, който ще работи на `localhost` и ще слуша на порт `3000`. За да видиш дали работи коректно, отвори
`http://localhost:3000/` в браузъра си и ако видиш `Hello World`, значи всичко е нормално :).

#### 4. Създай NodeJS приложение в StartApp.bg

Преди малко подакара `NodeJS` сървъра на локалната си машина. Сега е на ред да го накарш да заработи на StartApp.bg
За целта създай едно обикновенно `NodeJS` приложение, което в този случай е версия `0.10` и се казва `mynodejs`

Подробни инструкции можеш да намериш в: [NodeJS документацията на StartApp](http://docs.startapp.bg/getting-started/startapp-with-nodejs.html#env-vars)

```bash
app create mynodejs nodejs-0.10 --no-git
```

Аргументът `--no-git` казва на StartApp да не клонира новосъздаденото приложение на компютъра след като го създаде.

Като резултат, трябва да видиш, нещо подобно:

```bash
Application Options
-------------------
Domain:     demos
Cartridges: nodejs-0.10
Gear Size:  default
Scaling:    no

Creating application 'mynodejs' ... done


Waiting for your DNS name to be available ... done

Your application 'mynodejs' is now available.

  URL:        http://mynodejs-demos.sapp.io/
  SSH to:     54f160d13bba31e20ed000103@mynodejs-demos.sapp.io
  Git remote: ssh://54f160d13bba31e20ed000103@mynodejs-demos.sapp.io/~/git/mynodejs.git/

Run 'app show-app mynodejs' for more details about your app.
```

#### 5. Добавяне на задължителни директории и файлове за да работи на сървърите на StartApp.bg

```bash
sh <(curl -s http://install.opensource.sh/sappio/dot) -a mynodejs
```

На кратко този скрипт създава една скрита директория `.openshift`, която се изпозлва служебно от StartApp. Директорията `.openshift` също така се използва и за различни автоматизации, които можеш да си направиш благодарение на [Action Hooks](http://docs.startapp.bg/getting-started/startapp-with-nodejs.html#action-hooks). Ако си запознат с Git тогава знай, че `.openshift` директорията за StartApp е като `.git` директорията за Git.


#### 6.Промени `server.js` файла

Както вече видя, след като старитра, `server.js` той тръгва на `localhost` и на порт `3000`. Понеже на сървърите на StartApp порт `3000` на `localhost` най-вероятно е зает от друго приложение, затова StartApp се грижи да осигурява служебно уникално `IP` и `PORT` за всяко едно приложение. Информацията за `IP` и `PORT` се записват в  Environment променливи на сървъра със следните имена:

- `OPENSHIFT_NODEJS_IP`
- `OPENSHIFT_NODEJS_PORT`

Повече информация за Environment променливите на NodeJS приложения в StartApp, орвори тук: [NodeJS Environemnt променливи в StartApp](http://docs.startapp.bg/getting-started/startapp-with-nodejs.html#env-vars)


###### Редактирай файла `server.js` от:

```js
var express = require('express')
var app = express()

app.get('/', function (req, res) {
  res.send('Hello World!')
})

var server = app.listen(3000, function () {

  var host = server.address().address
  var port = server.address().port

  console.log('Example app listening at http://%s:%s', host, port)

})
```

###### Да стане така:

```js
var express = require('express')
var app = express()

// Така дефинирани, означава, че
// на локалния компютър ще работи на localhost:3000
// а на сървъра, ще работи служебно_ip:служебжен_port
var server_ip    = process.env.OPENSHIFT_NODEJS_IP || 'localhost',
    server_port  = parseInt(process.env.OPENSHIFT_NODEJS_PORT) || 3000;

app.get('/', function (req, res) {
  res.send('Hello World!')
})

var server = app.listen(server_port, server_ip, function () {

  var host = server.address().address
  var port = server.address().port

  console.log('Example app listening at http://%s:%s', host, port)

})
```

#### 7. Качи на сървъра

```bash
git add .
git commit -m "Chnages for StartApp"
git push startapp master -f
```

#### 7. Свържи за база данни

Както вече си се досетил, конфигурацията на база данни в StartApp става също с Environment променливи.

###### Пример с MongoDB:

- `OPENSHIFT_APP_NAME` - име на базата данни
- `OPENSHIFT_MONGODB_DB_USERNAME` - потребителско име
- `OPENSHIFT_MONGODB_DB_PASSWORD` - парола
- `OPENSHIFT_MONGODB_DB_HOST`- hostname
- `OPENSHIFT_MONGODB_DB_PORT`- порт

Съветваме те да използваш тези Environment променливи, вместо да hardcode-ваш `hostname`, `username`, `password` и тнт, защото за всяко едно приложение те са различни и така, ще си спестиш много главоболия!

Ето как изглеждат в кода тези Environment променливи:

```js
var MongoClient = require('mongodb').MongoClient,

    host     = process.env.OPENSHIFT_MONGODB_DB_HOST,
    username = process.env.OPENSHIFT_MONGODB_DB_USERNAME,
    password = process.env.OPENSHIFT_MONGODB_DB_PASSWORD,
    port     = process.env.OPENSHIFT_MONGODB_DB_PORT,
    db_name  = process.env.OPENSHIFT_APP_NAME,
    mongodb_uri = "mongodb://" + username + ":" + password + "@" + host + ":" + port + "/" + db_name;

// Свързване с базата данни
MongoClient.connect(mongodb_uri, function(err, db) {
  if(err) {
    return console.dir(err);
  }

  db.collection('test', function(err, collection) {});
  db.collection('test', { w:1 }, function(err, collection) {});
  db.createCollection('test', function(err, collection) {});
  db.createCollection('test', { w:1 }, function(err, collection) {});
});
```

Виж още:
- [MongoDB](http://docs.startapp.bg/getting-started/startapp-with-nodejs.html#add-mongo-to-app)
- [MySQL](http://docs.startapp.bg/getting-started/startapp-with-nodejs.html#add-mysql-to-app)
- [PostgreSQL](http://docs.startapp.bg/getting-started/startapp-with-nodejs.html#add-postgresql-to-app)

Това е :) Честито.
