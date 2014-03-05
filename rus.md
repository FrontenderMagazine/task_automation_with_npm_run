Есть несколько [модных инструментов][1] для автоматизации сборки 
в javascript-проектах, которые я никогда находил привлекательными, потому как
знаком с менее извесной командой `npm run`, которой вполне достаточно для всего,
что мне необходимо, при этом сохраняя достоточно маленький конфигурационный файл. (?)

Вот несколько трюков, которые я использую чтобы получить максимаьную отдачу от 
`npm run` и полей `script` в `package.json`.

## Поле «script»

Еслы вы не видели этого раньше, [npm][2] содержит поле под названием `scripts`
в файле `package.json` проекта для того, чтобы делать такие штуки, как `npm test`,
выполняющее содержимое поля `scripts.test`, и `npm start`, вызывающий команды
из поля `scripts.start`.

`npm test` и `npm start` — это всего лишь удобные ссылки для `npm run test` и
`npm run start`, и вы можете с помощью `npm run` выполнить совершенно любое
содержимое любого поля внутри `scripts`.

Кроме того, `npm run` великолепен еще и потому, что npm автоматически добавляет
в `$PATH` директорию `node_modules/.bin`, так что вы можете просто запускать
команды из `dependencies` или `devDependencies` напрямую, без необходимости
устанавливать это модули глобально. npm-пакеты, которые вы хотели бы включить
в свой воркфлоу, должны иметь всего лишь простой интерфейс командной строки, и
вы сможете написать простую автоматисацию самостоятельно.

## сборка javascript

Я пишу клиентский код, используя для его организации, принятые в commonjs 
`module.exports` и `require()` и подключая модули, опубликованные в npm. 
[browserify][3] может разрешить все вызовы `require()` статически, на этапе
сборки, создав единый склееный бандл-файл, который можно загрузить, используя
тег `script`. Для использования browserify я просто держу в package.json поле
`scripts['build-js']`, которыое выглядит так:

    "build-js": "browserify browser/main.js > static/bundle.js"

Если я хочу собрать javascript для продакшна, я также выполняю минификацию —
подключаю `uglify-js` как devDependency и дописываю его через пайп:

    "build-js": "browserify browser/main.js | uglifyjs -mc > static/bundle.js"

## watching javascript

Для автоматической перекомпиляции клиентского javascript при любых изменениях
файлов, я просто заменяю команду `browserify` на [watchify][4] и добавляю ключи
`-d` и `-v` для дебага и более подробного вывода.

    "watch-js": "watchify browser/main.js -o static/bundle.js -dv"

## сборка CSS

Я обнаружил, что `cat` обычно полностью удовлетворяет мои потребности, так что
я просто держу для сборки что-то вроде этого:

    "build-css": "cat static/pages/*.css tabs/*/*.css > static/bundle.css"

## watching css

Так же как и с `watchify, я пересобираю css при изменениях с помощью замены `cat`
на [catw][5]:

    "watch-css": "catw static/pages/*.css tabs/*/*.css -o static/bundle.css -v"

## Последовательности задач

Если у вас есть две задачи, которые вы хотели бы запускать последовательно, то
вы можете записать их через `npm run` и сгруппировать с помощью `&&`:

    "build": "npm run build-js && npm run build-css"

## Параллельные задачи

Если вам нужно запустить несколько задач параллельно, просто разделите их
с помощью `&`:

    "watch": "npm run watch-js & npm run watch-css"

## the complete package.json

Соединив все, о чем я говорил, мы получим примерно такой `package.json`

Altogether, the package.json I've just described might look like:

    {
      "name": "my-silly-app",
      "version": "1.2.3",
      "private": true,
      "dependencies": {
        "browserify": "~2.35.2",
        "uglifyjs": "~2.3.6"
      },
      "devDependencies": {
        "watchify": "~0.1.0",
        "catw": "~0.0.1",
        "tap": "~0.4.4"
      },
      "scripts": {
        "build-js": "browserify browser/main.js | uglifyjs -mc > static/bundle.js",
        "build-css": "cat static/pages/*.css tabs/*/*.css",
        "build": "npm run build-js && npm run build-css",
        "watch-js": "watchify browser/main.js -o static/bundle.js -dv",
        "watch-css": "catw static/pages/*.css tabs/*/*.css -o static/bundle.css -v",
        "watch": "npm run watch-js & npm run watch-css",
        "start": "node server.js",
        "test": "tap test/*.js"
      }
    }

Если мне нужно выполнить сборку для продакшна, я просто сделаю `npm run build`.
Для локальной разработки я запущу `npm run watch`.

Вы можете расширять базовое приближение как хотите! Например, вам может
понадобиться выполнить `build` до запуска `start`, в этом случае вы просто
напишете:

    "start": "npm run build && node server.js"

Или, возможно, вы захотите создато команду `npm run start-dev`, которая также
запустит вотчеры:

    "start-dev": "npm run watch & npm start"

Вы можете реорганизовать все части так, как хотите!

## Когда становится действительно сложно...

Если вы поняли, что набили слишком много команд в одно поле `scripts`, советую
вам подумать над тем, чтобы вынести некоторые из этих команд в отдельное место,
такое как `bin/`.

Эти скрипты можно написать на bash, на node, на perl, да на чем угодно! Просто
добавьте свойство `#!` в начало файла, выполните `chmod +x`, получится так:

    #!/bin/bash
    (cd site/main; browserify browser/main.js | uglifyjs -mc > static/bundle.js)
    (cd site/xyz; browserify browser.js > static/bundle.js)

    "build-js": "bin/build.sh"

Если вам совершенно точно нужно собирать ваш проект на windows, просто
удостоверьтесь, что у разработчиков, которые пользуются windows, есть копия
[msysgit][6], которая поставляется вместе с bash, cygwin или чем-то похожим.
Или предложите им перейти на UNIX.

I have [some experiments][7] in the works to help with this windows-can't-run-
bash problem, but the job control and subshell sections aren't finished yet.

## Вывод

Я надеюсь, что те примеры использования `npm run`, которые я описал здесь, 
помогут тем из вас, кто не был вдохновлен текущим состоянием дел в инструментах
для сборки фронтенда, и особенно тем, кто, как и я, не проникся призывами
этих инструментов. Я предпочитаю инструменты, пропитанные наследием UNIX,
такие, как git, или как npm, о котором я говорил здесь. Эти инструменты
предоставляют быстрый и минималистичный интерфейс, с которым можно
взаимодействовать через bash. Некоторые из этих штук не требуют долгих церемоний,
или обсуждений. Можно зайти очень далеко, используя очень простые инструменты,
которые делают очень обычные вещи.

Если вам не нравится стиль `npm run`, о котором я рассказал здесь, вы можете
присмотреться к makefile, как к простой и крепкой альтернативе пришедшей из
весьма далеких дней(??).

(to some of the more baroque approaches to task automation making the rounds these days).


 [1]: http://gruntjs.com/
 [2]: https://npmjs.org
 [3]: http://browserify.org
 [4]: https://npmjs.org/package/watchify
 [5]: https://npmjs.org/package/catw
 [6]: https://github.com/msysgit/msysgit#the-build-environment
 [7]: https://npmjs.org/package/bashful