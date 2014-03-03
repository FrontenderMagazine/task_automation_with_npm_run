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

To recompile my browser javascript automatically whenever I change a file, I
can just substitude the`browserify` command for [watchify][4] and add `-d` and
`-v` for debugging and more verbose output:

    "watch-js": "watchify browser/main.js -o static/bundle.js -dv"

## building css

I find that `cat` is usually adequate so I just have a script that looks
something like:

    "build-css": "cat static/pages/*.css tabs/*/*.css > static/bundle.css"

## watching css

Similarly to my watchify build, I can recompile css as it changes by
substituting`cat` with [catw][5]:

    "watch-css": "catw static/pages/*.css tabs/*/*.css -o static/bundle.css -v"

## sequential sub-tasks

If you have 2 tasks you want to run in series, you can just `npm run` each task
separated by a`&&`: 

    "build": "npm run build-js && npm run build-css"

## parallel sub-tasks

If you want to run some tasks in parallel, just use `&` as the separator!

    "watch": "npm run watch-js & npm run watch-css"

## the complete package.json

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

If I want to build for production I can just do `npm run build` and for local
development I can just do`npm run watch`!

You can extend this basic approach however you like! For instance you might
want to run the`build` step before running `start`, so you could just do:

    "start": "npm run build && node server.js"

or perhaps you want an `npm run start-dev` command that also starts the
watchers:

    "start-dev": "npm run watch & npm start"

You can reorganize the pieces however you want!

## when things get really complicated...

If you find yourself stuffing a lot of commands into a single `scripts` field
entry, consider factoring some of those commands out into someplace like`bin/`

You can write those scripts in bash or node or perl or whatever. Just put the
proper`#!` line at the top of the file, `chmod +x`, and you're good to go:

    #!/bin/bash
    (cd site/main; browserify browser/main.js | uglifyjs -mc > static/bundle.js)
    (cd site/xyz; browserify browser.js > static/bundle.js)

    "build-js": "bin/build.sh"

If you absolutely need your project to build on windows, just make sure that
your windows devs have a copy of[msysgit][6] which ships with bash or cygwin or
something similar. Or tell them to get a UNIX.

I have [some experiments][7] in the works to help with this windows-can't-run-
bash problem, but the job control and subshell sections aren't finished yet.

## conclusion

I hope that this `npm run` approach I've documented here will appeal to some of
you who may be unimpressed with the current state of frontend task automation 
tooling, particularly those of you like me who just don't "get" the appeal of 
some of these things. I tend to prefer tools that are more steeped in the unix 
heritage like git or here with npm just providing a rather minimal interface on 
top of bash. Some things really don't require a lot of ceremony or coordination 
and you can often get a lot of mileage out of very simple tools that do very 
ordinary things.

If you don't like the `npm run` style I've elaborated upon here you might also
consider Makefiles as a solid and simple alternative to some of the more baroque
approaches to task automation making the rounds these days.

 [1]: http://gruntjs.com/
 [2]: https://npmjs.org
 [3]: http://browserify.org
 [4]: https://npmjs.org/package/watchify
 [5]: https://npmjs.org/package/catw
 [6]: https://github.com/msysgit/msysgit#the-build-environment
 [7]: https://npmjs.org/package/bashful