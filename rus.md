# Создание двухмерной браузерной игры на основе PhysicsJS

![Скриншот][Игра, которая у нас получится]

Я веб-разработчик и фанат физики, и очень давно пытался реализовать симуляцию 
двухмерной физики на основе JavaScript. Однако, мне постоянно казалось, что 
чего-то не хватает. Для качественной реализации этой идеи мне был необходим 
собственный фреймворк, который не уступал бы по гибкости другим современным 
JavaScript API. Мне не нужно было, чтобы он решал за меня, что мне нужна 
направленная вниз гравитация, или что мне она вообще нужна… Это побудило меня 
создать [PhysicsJS][1], который, как я надеюсь, оправдает своё собственное описание:

> Модульный, расширяемый и простой в использовании физический движок для 
JavaScript.

В последнее время появилось сразу несколько новых физических движков для 
JavaScript, и все они обладают определенными достоинствами. Однако, я хочу 
показать вам пару действительно интересных особенностей PhysicsJS, которые, 
как мне кажется, позволяют ему выигрышно выделиться на фоне остальных 
схожих библиотек.

В этой статье я продемонстрирую вам процесс разработки полноценной 2D-игры, об 
астероидах, со взрывами и простенькой графикой. Попутно вы сможете лучше понять 
мой стиль разработки, а также, узнаете, какие выгоды вы можете извлечь из 
практического использования PhysicsJS. Полную версию [исходного кода, 
использованного в качестве примера, можно скачать в репозитории PhysicsJS на 
GitHub][2] (ищите его в examples/spaceship), [конечный результат можно 
увидеть на CodePen][3]. Однако перед тем, как приступить к делу, я бы хотел 
сделать важное заявление:

> PhysicsJS является довольно новой библиотекой, и мы усиленно работаем над тем, 
чтобы представить её бета-версию в ближайшие месяцы. Это API пока еще находится 
в состоянии корректирования и может содержать некоторые ошибки и недочёты. 
Имейте это ввиду. Я буду использовать в данной статье [PhysicsJS v0.5.2][4] 
(*актуальная версия на данный момент 0.5.3, прим. ред.*). 
Я горячо приветствую ваше участие любого рода. Просто загляните в наш репозиторий 
[PhysicsJS на GitHub][5]. Я также стараюсь отвечать на вопросы о 
[PhysicsJS на StackOverflow][6]. Просто добавьте тэг «physicsjs» к вопросу. 

## Уровень 0: Бескрайняя пустота космоса

Начнём с самого начала. Нам нужно написать каркас на HTML. 
Для рендеринга мы будем использовать HTML `canvas`, так что начнём со 
стандартного шаблона кода HTML5. Я буду исходить из того, что скрипты 
загружаются в конце тега `body`, чтобы не беспокоиться о готовности DOM. 
Кроме того, мы будем использовать библиотеку [RequireJS][7] для загрузки всех 
скриптов. Если вы никогда раньше не использовали RequireJS и не имеете желания 
вникать в её работу, вам придётся избавиться от всех вызовов `define()` и 
`require()` и перегруппировать весь JavaScript в правильном порядке (однако, 
я настоятельно вам рекомендую использовать RequireJS).

Вот структура в виде дерева, с которой мы будем работать:

    index.html
    images/
        …
    js/
        require.js
        physicsjs/
        …

Скачайте [RequireJS][8], [PhysicsJS-0.5.2][9], и [изображения, необходимые для проекта][10] 
в соответствующие директории проекта. 

Теперь добавим в `head` немного CSS для установки «космического» бэкграунда и 
несколько геймплейных сообщений, привязанных к классу `body`:

    html, body {
        padding: 0;
        margin: 0;
        width: 100%;
        height: 100%;
        overflow: hidden;
    }
    body:after {
        font-size: 30px;
        line-height: 140px;
        text-align: center;
        color: rgb(60, 16, 11);
        background: rgba(255, 255, 255, 0.6);
        position: absolute;
        top: 50%;
        left: 50%;
        width: 400px;
        height: 140px;
        margin-left: -200px;
        margin-top: -70px;
        border-radius: 5px;
    }
    body.before-game:after {
        content: 'нажмите Z, чтобы начать игру';
    }
    body.lose-game:after {
        content: 'нажмите Z, чтобы попробовать ещё раз';
    }
    body.win-game:after {
        content: 'Победа! нажмите Z, чтобы сыграть ещё раз';
    }
    body {
        background: url(images/starfield.png) 0 0 repeat;
    }

А теперь, для настройки RequireJS и корректного запуска приложения, добавим 
следующий код в конец тега `body`:

    <script src="js/require.js"></script>
    <script src="js/main.js"></script>

Далее, нам нужно создать файл `main.js`. По ходу данного руководства, этот файл 
будет изменяться. Обратите внимание, что я публикую живые примеры с CodePen, 
так что вам может потребоваться изменить настройки RequireJS для локального 
запуска. Должно получится что то вроде этого:

    // начало main.js
    require(
        {
            // нам нужна вышестоящая директория для доступа к изображениям
            baseUrl: './',
            packages: [{
                name: 'physicsjs',
                location: 'js/physicsjs',
                main: 'physicsjs'
            }]
        },
        …

Окей, начнём с первой версии `main.js`.

<p data-height="450" data-theme-id="0" data-slug-hash="aJAtq" data-user="SilentImp" data-default-tab="result" class='codepen'>Посмотрите на CodePen: <a href='//codepen.io/SilentImp/pen/aJAtq'>Создание двухмерной браузерной игры на основе PhysicsJS (v1)</a></p>

Наискучнейшая игра во всем мире.

Давайте разберемся, что тут у нас происходит. 

Во-первых, мы подгрузили библиотеку PhysicsJS, объявив её зависимость от RequireJS. 
Кроме того мы подгрузили несколько других зависимых компонентов. Это модули 
PhysicsJS, которые мы подключили для добавления некоторых функциональных 
возможностей, вроде вращения тел, определения столкновений и прочего. Когда мы 
используем RequireJS (и модули AMD), нам нужно объявлять каждую зависимость 
такого рода. Подобный ход позволяет загружать только тот функционал, который 
будет впоследствии использоваться в проекте.

Внутри фабричной функции RequireJS мы прописываем основные настройки нашей игры. 
Мы начинаем с добавления названия класса в тег `body` для того, чтобы отображалось 
сообщение «нажмите Z чтобы начать игру», и устанавливаем обработчик нажатия 
клавиши Z, который будет вызывать вспомогательную функцию `newGame()`.

Затем мы настраиваем визуализацию `canvas` для заполнения окна, и устанавливаем 
обработчик изменения размера окна, который будет подгонять размер `canvas` под 
размер текущего окна пользователя. 

Немного дальше мы прописываем ядро игры в функцию `init()`. Функция `init` будет 
передаваться в конструктор `Physics()` для создания мира. Внутри этой функции 
мы устанавливаем некоторые необходимые нам компоненты:

* «Космический корабль» (который на данный момент является всего лишь объектом 
круглой формы);
* Планету (которая пока что является так же круглым объектом, но с большой массой 
и со специальным изображением);
* Обработчик события, который запускает рендеринг мира на каждом этапе его изменения.

Детального разъяснения, пока что, требует только код нашей планеты, а если точнее, 
вот эта его часть:

    planet.view = new Image();
    planet.view.src = require.toUrl('images/planet.png');

Так, и что же это за чертовщина?

Все что мы делаем — это создаём элемент изображения и сохраняем его в свойстве 
`.view`. Изображение подгружает картинку `planet.png` (мы так же используем 
RequireJS для определения пути к нашей картинке). Но зачем такие сложности? 
Поступив таким образом, мы имеем возможность кастомизировать отображение объектов. 
Рендеринг `canvas` проверяет все объекты, которые нужно визуализировать, и, если 
у какого-то из них отсутствует это свойство, обрисовывает его представление 
(круг, многогранник и т.п.) в `canvas`, а потом кэширует его как изображение. 
Это изображение и будет храниться в соответствующем для него свойстве `.view`. 
Однако, если в свойстве уже присутствует изображение, то использоваться будет оно. 

**Внимание:** Я знаю, что этот способ контроля над визуализацией далёк от 
идеального. Однако не унывайте, у меня для вас пара хороших новостей:

1. Это способ визуализации, можно легко расширить или даже, вовсе, заменить 
на *абсолютно другой*. При желании вы даже можете создать свой собственный 
способ визуализации. В скором времени мы планируем добавить в нашу документацию 
описание [методики, руководствуясь которой вы сможете это сделать][11].
2. Через несколько недель в [ChallengePost][12] будет проходить конкурс на 
создание наиболее гибкого и мощного средства визуализации для PhysicsJS. 
Вы должны принять участие! И даже если не примите, возьмите на вооружение 
результаты этого конкурса. 

Однако, мы немного отвлеклись. Давайте продолжим, и взглянем на то, 
что происходит сейчас в функции `init()`:

    // добавление объектов в игровой мир
    world.add([
      ship,
      planet,
      Physics.behavior('newtonian', { strength: 1e-4 }),
      Physics.behavior('sweep-prune'),
      Physics.behavior('body-collision-detection'),
      Physics.behavior('body-impulse-response'),
      renderer
    ]);

Мы одним махом добавляем в мир всякую всячину. Различные тела, которые мы 
создали (корабль и планета), алгоритмы их поведения и средство рендеринга. 
Давайте остановимся на алгоритмах поведения подробнее:

* `newtonian` — добавляет в мир [ньютоновскую гравитацию][13]. Тела будут 
притягиваться друг к другу согласно закону обратных квадратов `1e-4`.
* `sweep-prune`: это [алгоритм широкой фазы][14], который ускоряет обнаружение 
столкновений. Если ваша игра предусматривает столкновение тел, вероятно лучшего 
всего использовать его.
* `body-collision-detection` — узкая фаза определения столкновения, в которой 
установлен обработчик событий `sweep-prune`, и используются алгоритмы (вроде 
[GJK][15], он же алгоритм Гилберта — Джонсона — Кёрти) для определения 
столкновения тел, и того, как это столкновение происходит. Примите во внимание, 
что здесь всего лишь *определяется* столкновение, а не реакция на него.
* `body-impulse-response` — непосредственно *выполняет* определённое действие 
при обнаружении столкновения. Этот алгоритм является обработчиком событий 
столкновения и отвечает на них, придавая столкнувшимся телам импульс.

Надеюсь, на этом этапе вы уже начинаете понимать, почему PhysicsJS имеет 
модульную структуру. Ни у мира, ни у наполняющих его объектов нет никакой 
глобальной физической логики. Если вы хотите, чтобы физические свойства 
вашего мира не ограничивались свойствами вакуума, вам потребуется добавить 
алгоритмы, которые будут влиять и на мир, и на тела в нём.

Следующим шагом будет добавление в наш скрипт `main.js` вспомогательной функции 
`newGame()`, которая очищает и создаёт новый сеанс игры через вызов 
`Physics( init )`. А также, она служит обработчиком пользовательских событий 
в воссозданном нами мире, сигнализирующих о необходимости завершения текущей 
игровой сессии и начала новой. На данный момент ничто не порождает эти события, 
однако скоро они нам потребуются.

Наконец, мы присоединяем таймер (вспомогательный метод `requestAnimationFrame`) 
чтобы `world.step` вызывался для каждого кадра.

## Стадия 1: Придаем огранку

Всё станет намного интересней, когда мы сможем летать по нашему миру. Мы могли 
бы сделать с нашим кораблём нечто похожее на то, что сделали с планетой, однако, 
нам не достаточно только применения подходящего внешнего облика. Нам нужны 
методы симуляции маневрирования кораблем. Лучшее, что мы можем сделать — 
применить одну из моих любимых возможностей в PhysicsJS, *возможности 
неограниченного расширения практически всего на свете*. В данном случае, 
мы будем работать над расширением объектов, с которыми мы взаимодействуем 
в мире игры. 

Для начала мы создадим новый файл, назовем его `player.js`. Он послужит нам 
для определения кастомного объекта `player`. Пусть у нашего корабля будет 
сферическая форма ([ха-ха][16]), мы дополним тело `circle` так, как [описано в 
справочнике по PhysicsJS][17].

Определим зависимости модуля, используя RequireJS:

    define(
    [
        'require',
        'physicsjs',
        'physicsjs/bodies/circle',
        'physicsjs/bodies/convex-polygon'
    ],
    function(
        require,
        Physics
    ){
        // здесь должен быть ваш код…
    });

Теперь у нас есть доступ к круглому телу, и мы можем его расширить, а также 
создать выпуклые многоугольные тела, которые будут обозначать космический мусор 
(я ведь обещал вам взрывы в начале статьи). На старт, внимание, расширяем!

    // расширение круглого тела
    Physics.body('player', 'circle', function( parent ){
        // далее идут закрытые вспомогательные функции 
        // …
        return {
            // определение расширения
        };
    });

Тело `player` должно быть так же подключено к PhysicsJS. Мы добавим закрытые 
вспомогательные функции и возвратим объект-литерал для расширения круглого тела. 

Вот наши закрытые вспомогательные функции:

    // закрытые вспомогательные функции
    var deg = Math.PI/180;
    var shipImg = new Image();
    var shipThrustImg = new Image();
    shipImg.src = require.toUrl('images/ship.png');
    shipThrustImg.src = require.toUrl('images/ship-thrust.png');
    var Pi2 = 2 * Math.PI;
    // ОЧЕНЬ грубое округление случайного числа по Гауссу, зато быстрое
    var gauss = function gauss( mean, stddev ){
        var r = 2 * (Math.random() + Math.random() + Math.random()) - 3;
        return r * stddev + mean;
    };
    // даёт случайный многоугольник, который, скорее всего, будет выпуклым
    var rndPolygon = function rndPolygon( size, n, jitter ){
        var points = [{ x: 0, y: 0 }]
            ,ang = 0
            ,invN = 1 / n
            ,mean = Pi2 * invN
            ,stddev = jitter * (invN - 1/(n+1)) * Pi2
            ,i = 1
            ,last = points[ 0 ]
            ;
        while ( i < n ){
            ang += gauss( mean, stddev );
            points.push({
                x: size * Math.cos( ang ) + last.x,
                y: size * Math.sin( ang ) + last.y
            });
            last = points[ i++ ];
        }
        return points;
    };

Расширяем круглый объект (подробности в комментариях к коду):

    return {
        // после создания тела, нам нужно его настроить
        // потому мы вызываем метод init родителя
        // для «this»
        init: function( options ){
            parent.init.call( this, options );
            // устанавливаем изображение для визуализации,
            // с той картинкой, которую я выбрал, нос корабля
            // будет повернут под тем же углом, согласно угловому положению тела
            this.view = shipImg;
        },
        // поворот корабля посредством изменения 
        // угловой скорости тела на + или - какую-либо величину
        turn: function( amount ){
            // установка угловой скорости корабля
            this.state.angular.vel = 0.2 * amount * deg;
            return this;
        },
        // этот кусок увеличивает скорость корабля вдоль направления
        // его носовой части
        thrust: function( amount ){
            var self = this;
            var world = this._world;
            if (!world){
                return self;
            }
            var angle = this.state.angular.pos;
            var scratch = Physics.scratchpad();
            // приравниваем величину к менее сумасшедшему значению
            amount *= 0.00001;
            // указываем ускорение в направлении носовой части корабля
            var v = scratch.vector().set(
                amount * Math.cos( angle ), 
                amount * Math.sin( angle ) 
            );
            // самоускорение
            this.accelerate( v );
            scratch.done();
            // при ускорении изображение должно изменяться на картинку с включёнными двигателями
            if ( amount ){
                this.view = shipThrustImg;
            } else {
                this.view = shipImg;
            }
            return self;
        },
        // Далее нам нужно создать снаряд (маленький круг),
        // который отделяется от передней части корабля.
        // После определённого промежутка времени он должен исчезать
        shoot: function(){
            var self = this;
            var world = this._world;
            if (!world){
                return self;
            }
            var angle = this.state.angular.pos;
            var cos = Math.cos( angle );
            var sin = Math.sin( angle );
            var r = this.geometry.radius + 5;
            // Создаем маленький кружек у носа корабля,
            // который движется с ускорением в 0.5,
            // относительно текущей скорости корабля
            // по направлению носовой части, 
            var laser = Physics.body('circle', {
                x: this.state.pos.get(0) + r * cos,
                y: this.state.pos.get(1) + r * sin,
                vx: (0.5 + this.state.vel.get(0)) * cos,
                vy: (0.5 + this.state.vel.get(1)) * sin,
                radius: 2
            });
            // Устанавливаем пользовательские свойства для имитации столкновения
            laser.gameType = 'laser';
            // Убираем лазерный импульс через 600 мс
            setTimeout(function(){
                world.removeBody( laser );
                laser = undefined;
            }, 600);
            world.add( laser );
            return self;
        },
        // Бабах! Корабль взорвался,
        // заменяем его кучкой треугольников,
        // для имитации взрыва.
        blowUp: function(){
            var self = this;
            var world = this._world;
            if (!world){
                return self;
            }
            var scratch = Physics.scratchpad();
            var rnd = scratch.vector();
            var pos = this.state.pos;
            var n = 40; // создаем 40 осколков от взрыва
            var r = 2 * this.geometry.radius; // создаем окружность
            var size = 8 * r / n; // примерный размер краев осколков
            var mass = this.mass / n; // масса осколков
            var verts;
            var d;
            var debris = [];

            // создаем космический мусор
            while ( n-- ){
                verts = rndPolygon( size, 3, 1.5 ); // получаем случайный многоугольник
                if ( Physics.geometry.isPolygonConvex( verts ) ){
                    // устанавливаем мусор в случайной точке расположения, 
                    // относительно игрока.
                    rnd.set( Math.random() - 0.5, Math.random() - 0.5 ).mult( r );
                    d = Physics.body('convex-polygon', {
                        x: pos.get(0) + rnd.get(0),
                        y: pos.get(1) + rnd.get(1),
                        // скорость мусора равна скорости игрока
                        vx: this.state.vel.get(0),
                        vy: this.state.vel.get(1),
                        // установка случайной угловой скорости, 
                        // для придания ситуации большего драматизма
                        angularVelocity: (Math.random()-0.5) * 0.06,
                        mass: mass,
                        vertices: verts,
                        // избавляемся от излишней "прыгучести"
                        restitution: 0.8
                    });
                    d.gameType = 'debris';
                    debris.push( d );
                }
            }

            // добавляем мусор в игровой мир
            world.add( debris );
            // удаляем игрока из мира
            world.removeBody( this );
            scratch.done();
            return self;
        }
    };

Вы наверное заметили, что я постоянно использую `Physics.scratchpad`. 
Скретчпад — ваш лучший помощник в удалении временных объектов (векторов), 
он сэкономит вам кучу времени на создании и удалении мусора. 
[Более детальную информацию о скретчпадах можно прочесть тут][18]. 

Итак, теперь у нас есть персонаж игры, однако он пока еще не привязан к 
какому-либо вводу данных со стороны игрока. Сейчас нам нужно создать 
*пользовательский сценарий поведения*, для реакции на действия игрока. 
Для этого мы создадим новый файл `player-behavior.js`, он довольно похож 
на предыдущие файлы. Подробности снова в комментариях к коду:

    define(
    [
        'physicsjs'
    ],
    function(
        Physics
    ){

        return Physics.behavior('player-behavior', function( parent ){

            return {
                init: function( options ){
                    var self = this;
                    parent.init.call(this, options);
                    // Добавляем игрока через настройки options,
                    // и теперь нам нужно его сохранить
                    var player = self.player = options.player;
                    self.gameover = false;

                    // события
                    document.addEventListener('keydown', function( e ){
                        if (self.gameover){
                            return;
                        }
                        switch ( e.keyCode ){
                            case 38: // вверх
                                self.movePlayer();
                            break;
                            case 40: // вниз
                            break;
                            case 37: // влево
                                player.turn( -1 );
                            break;
                            case 39: // вправо
                                player.turn( 1 );
                            break;
                            case 90: // z
                                player.shoot();
                            break;
                        }
                        return false;
                    });
                    document.addEventListener('keyup', function( e ){
                        if (self.gameover){
                            return;
                        }
                        switch ( e.keyCode ){
                            case 38: // вверх
                                self.movePlayer( false );
                            break;
                            case 40: // вниз
                            break;
                            case 37: // влево
                                player.turn( 0 );
                            break;
                            case 39: // вправо
                                player.turn( 0 );
                            break;
                            case 32: // пробел
                            break;
                        }
                        return false;
                    });
                },

                // Эта часть автоматически вызывается, 
                // когда наш сценарий поведения добавляется в игровой мир
                connect: function( world ){

                    // Мы хотим отслеживать события в мире
                    world.subscribe('collisions:detected', this.checkPlayerCollision, this);
                    world.subscribe('integrate:positions', this.behave, this);
                },

                // Эта часть так же автоматически вызывается,
                // когда сценарий поведения удаляется из игрового мира
                disconnect: function( world ){

                    // Мы не хотим больше отслеживать события в мире
                    world.unsubscribe('collisions:detected', this.checkPlayerCollision);
                    world.unsubscribe('integrate:positions', this.behave);
                },

                // проверка на столкновение игрока с чем-либо
                checkPlayerCollision: function( data ){

                    var self = this
                        ,world = self._world
                        ,collisions = data.collisions
                        ,col
                        ,player = this.player
                        ;

                    for ( var i = 0, l = collisions.length; i < l; ++i ){
                        col = collisions[ i ];

                        // если мы не обращаем внимания на мусор
                        // и одним из столкнувшихся тел является игрок
                        if ( col.bodyA.gameType !== 'debris' && 
                            col.bodyB.gameType !== 'debris' && 
                            (col.bodyA === player || col.bodyB === player) 
                        ){
                            player.blowUp();
                            world.removeBehavior( this );
                            this.gameover = true;

                            // после того как мы разбились, запускается 
                            // событие, для которого нужно установить
                            // обработчик, перезапускающий игру.
                            world.publish('lose-game');
                            return;
                        }
                    }
                },

                // переключение состояния игрока
                movePlayer: function( active ){

                    if ( active === false ){
                        this.playerMove = false;
                        return;
                    }
                    this.playerMove = true;
                },

                behave: function( data ){

                    // активация двигателей если playerMove равно true
                    this.player.thrust( this.playerMove ? 1 : 0 );
                }
            };
        });
    });

На данном этапе мы можем прописать зависимость между `js/player` и 
`js/player-behavior` и использовать ее в файле `main.js`, добавив 
в нашу функцию `init()` следующее:

    // код внутри init()
    var ship = Physics.body('player', {
        x: 400,
        y: 100,
        vx: 0.08,
        radius: 30
    });

    var playerBehavior = Physics.behavior('player-behavior', { player: ship });

    // …

    world.add([
        ship,
        playerBehavior,
        //…
    ]);

Последнее, что нам нужно перед релизом второй версии нашей игры — это 
приготовить рендер к отслеживанию движения пользователя. Этого 
можно достичь, добавив немного кода в обработчик события `step` для изменения 
угла расположения рендера перед вызовом `world.render()`. 
Выглядит это так:

    // код внутри init()
    // отрисовка на каждом шаге
    world.subscribe('step', function(){
        // середина холста
        var middle = { 
            x: 0.5 * window.innerWidth, 
            y: 0.5 * window.innerHeight
        };
        // отслеживаем движения игрока
        renderer.options.offset.clone( middle ).vsub( ship.state.pos );
        world.render();
    });

Наша вторая итерация проекта теперь уже больше напоминает игру!

<p data-height="450" data-theme-id="0" data-slug-hash="jogle" data-user="SilentImp" data-default-tab="result" class='codepen'>Посмотрите на CodePen: <a href='//codepen.io/SilentImp/pen/jogle'>Создание двухмерной браузерной игры на основе PhysicsJS (v2)</a></p>

## Стадия 2: Ищем приключения на свою корму

На данный момент наша игра довольно скучна. Нам позарез нужны враги. 
Ну, давайте создадим парочку!

Последовательность действий почти та же, что и для корабля игрока. 
Создаём новое тело (расширяем круг), но в этом случае мы немного упростим себе 
задачу. Нам всего лишь нужно, чтобы враги взрывались, потому применим другой 
подход, и не будем создавать для них алгоритм поведения, ведь их функциональность 
столь минимальна. Создадим новый файл с именем `ufo.js` и присвоим нашим НЛО 
упрощённый метод `blowUp()`:

    define(
    [
        'require',
        'physicsjs',
        'physicsjs/bodies/circle'
    ],
    function(
        require,
        Physics
    ){

        Physics.body('ufo', 'circle', function( parent ){
            var ast1 = new Image();
            ast1.src = require.toUrl('images/ufo.png');

            return {
                init: function( options ){
                    parent.init.call(this, options);

                    this.view = ast1;
                },
                blowUp: function(){
                    var self = this;
                    var world = self._world;
                    if (!world){
                        return self;
                    }
                    var scratch = Physics.scratchpad();
                    var rnd = scratch.vector();
                    var pos = this.state.pos;
                    var n = 40;
                    var r = 2 * this.geometry.radius;
                    var size = r / n;
                    var mass = 0.001;
                    var d;
                    var debris = [];

                    // создание осколков после взрыва
                    while ( n-- ){
                        rnd.set( Math.random() - 0.5, Math.random() - 0.5 ).mult( r );
                        d = Physics.body('circle', {
                            x: pos.get(0) + rnd.get(0),
                            y: pos.get(1) + rnd.get(1),
                            vx: this.state.vel.get(0) + (Math.random() - 0.5),
                            vy: this.state.vel.get(1) + (Math.random() - 0.5),
                            angularVelocity: (Math.random()-0.5) * 0.06,
                            mass: mass,
                            radius: size,
                            restitution: 0.8
                        });
                        d.gameType = 'debris';
    
                        debris.push( d );
                    }

                    setTimeout(function(){
                        for ( var i = 0, l = debris.length; i < l; ++i ){
                            world.removeBody( debris[ i ] );
                        }
                        debris = undefined;
                    }, 1000);

                    world.add( debris );
                    world.removeBody( self );
                    scratch.done();
                    world.publish({
                        topic: 'blow-up', 
                        body: self
                    });
                    return self;
                }
            };
        });
    });

Теперь создадим несколько врагов в функции `init()` в файле `main.js`:

    // код внутри init()
    var ufos = [];
    for ( var i = 0, l = 30; i < l; ++i ){

        var ang = 4 * (Math.random() - 0.5) * Math.PI;
        var r = 700 + 100 * Math.random() + i * 10;

        ufos.push( Physics.body('ufo', {
            x: 400 + Math.cos( ang ) * r,
            y: 300 + Math.sin( ang ) * r,
            vx: 0.03 * Math.sin( ang ),
            vy: - 0.03 * Math.cos( ang ),
            angularVelocity: (Math.random() - 0.5) * 0.001,
            radius: 50,
            mass: 30,
            restitution: 0.6
        }));
    }

    //…

    world.add( ufos );

Мы используем `Math` в коде для случайного расположения тел на орбите планеты, 
так, чтобы они вращались и зловеще подплывали к ней. 
Однако, на данный момент мы не можем их взорвать, потому добавим ещё немного 
кода в функцию `init()` для отслеживания количества убитых и инициализации 
события `win-game`, если все противники уничтожены. Помимо этого, мы добавим 
обработчик для события `collisions:detected`. Если в событии «столкновение» 
участвовал объект «лазер», другой объект должен взорваться, если для него есть 
поддержка события «взрыв».

    // код внутри init()
    // счетчик уничтоженных НЛО 
    var killCount = 0;
    world.subscribe('blow-up', function( data ){

        killCount++;
        if ( killCount === ufos.length ){
            world.publish('win-game');
        }
    });

    // взрываем всё, к чему прикасается лазерный импульс
    world.subscribe('collisions:detected', function( data ){
        var collisions = data.collisions
            ,col
            ;

        for ( var i = 0, l = collisions.length; i < l; ++i ){
            col = collisions[ i ];

            if ( col.bodyA.gameType === 'laser' || col.bodyB.gameType === 'laser' ){
                if ( col.bodyA.blowUp ){
                    col.bodyA.blowUp();
                } else if ( col.bodyB.blowUp ){
                    col.bodyB.blowUp();
                }
                return;
            }
        }
    });

Обратите внимание, что мы могли бы создать новый сценарий поведения для 
управления НЛО, однако, для них требуется настолько мало кода, что в этом нет 
необходимости. Также я хотел вам продемонстрировать, насколько гибкой является 
библиотека PhysicsJS при использовании различных стилей написания кода. Одну и ту же задачу можно реализовать совершенно разными способами.

Давайте взглянем на третью итерацию `main.js`:

<p data-height="450" data-theme-id="0" data-slug-hash="wHslk" data-user="SilentImp" data-default-tab="result" class='codepen'>Посмотрите на CodePen: <a href='//codepen.io/SilentImp/pen/wHslk'>Создание двухмерной браузерной игры на основе PhysicsJS (v3)</a></p>


Фантастический результат! Мы почти закончили. Осталась лишь финальный штрих, 
который я хотел бы вам продемонстрировать.

## Стадия 3: Найти свой путь

В космосе довольно трудно ориентироваться. Окружение плохо просматривается и нам 
бы не помещало небольшое подспорье в этом деле. Что-то вроде мини-карты с радаром. 
Неплохая идея, но как нам ее реализовать? 

Нам придётся определить позиции всех тел в игровом мире и нарисовать маленькие 
точки на небольшом участке `canvas` в верхнем правом углу. И, хотя рендер и 
не обладает в полной мере теми возможностями, которые нам необходимы, в нашем 
распоряжении есть парочка полезных вспомогательных методов. Мы привяжем 
событие `render`, вызов которого происходит после обрисовки объектов рендером. 
А затем мы просто добавим изображение с мини-картой в рамку, используя эти 
вспомогательные методы. Вот и код:

    // код внутри init()
    // отрисовка мини-карты
    world.subscribe('render', function( data ){
        // радиус мини-карты
        var r = 100;
        // отступы
        var shim = 15;
        // x,y центра
        var x = renderer.options.width - r - shim;
        var y = r + shim;
        // чрезвычайно полезный скретчпад, 
        // для ускорения обработки векторов
        var scratch = Physics.scratchpad();
        var d = scratch.vector();
        var lightness;
        // отрисовка указателей радара
        renderer.drawCircle(x, y, r, { strokeStyle: '#090', fillStyle: '#010' });
        renderer.drawCircle(x, y, r * 2 / 3, { strokeStyle: '#090' });
        renderer.drawCircle(x, y, r / 3, { strokeStyle: '#090' });
        for (var i = 0, l = data.bodies.length, b = data.bodies[ i ]; b = data.bodies[ i ]; i++){
            // расчёт смещения тела относительно позиции корабля,
            // и его масштабирование
            d.clone( ship.state.pos ).vsub( b.state.pos ).mult( -0.05 );
            // окрашиваем точки в зависимости от массы тела
            lightness = Math.max(Math.min(Math.sqrt(b.mass*10)|0, 100), 10);
            // если точки внутри радиуса мини-карты
            if (d.norm() < r){
                // отрисовываем точки
                renderer.drawCircle(x + d.get(0), y + d.get(1), 1, 'hsl(60, 100%, '+lightness+'%)');
            }
        }

        scratch.done();
    });

Хорошо, добавим этот код в наш `main.js` и взглянем на финальный релиз!

<p data-height="450" data-theme-id="0" data-slug-hash="dDryl" data-user="SilentImp" data-default-tab="result" class='codepen'>Посмотрите на CodePen: <a href='//codepen.io/SilentImp/pen/dDryl'>Создание двухмерной браузерной игры на основе PhysicsJS (Законченный код)</a></p>

## Подводя итог

Вуаля! Вот и всё! Спасибо вам за то, что вы потратили свое драгоценное время на 
это длинное руководство. 

Сюжет игры, конечно, не самый увлекательный, и остаётся ещё много работы. 
Например, мы можем минимизировать скрипты с помощью инструментов RequireJS. 
Однако, я в большей степени хотел показать вам возможности PhysicsJS, а также то, 
насколько полезной может быть эта библиотека даже в текущей, сырой версии. 
Надеюсь, я дал вам достаточно материала для самостоятельных экспериментов.
Не забывайте, если у вас есть вопросы, их можно задать в комментариях или на 
[StackOverflow][19].

Спасибо [MillionthVector за спрайты][20].

[1]: http://wellcaffeinated.net/PhysicsJS/
[2]: https://github.com/wellcaffeinated/PhysicsJS
[3]: http://codepen.io/wellcaffeinated/full/7c1f72c5c9c8e2159ee97dc6540e0b72
[4]: https://github.com/wellcaffeinated/PhysicsJS/releases/tag/physicsjs-v0.5.2-alpha
[5]: https://github.com/wellcaffeinated/PhysicsJS
[6]: http://stackoverflow.com/questions/tagged/physicsjs
[7]: http://requirejs.org/
[8]: http://requirejs.org/
[9]: https://github.com/wellcaffeinated/PhysicsJS/releases/tag/physicsjs-v0.5.2-alpha
[10]: https://github.com/wellcaffeinated/PhysicsJS/tree/master/examples/spaceship/images
[11]: https://github.com/wellcaffeinated/PhysicsJS/wiki/Renderers
[12]: http://physicsjs.challengepost.com/
[13]: http://ru.wikipedia.org/wiki/%D0%9D%D1%8C%D1%8E%D1%82%D0%BE%D0%BD%D0%BE%D0%B2%D1%81%D0%BA%D0%B0%D1%8F_%D0%B3%D1%80%D0%B0%D0%B2%D0%B8%D1%82%D0%B0%D1%86%D0%B8%D1%8F
[14]: http://ru.wikipedia.org/wiki/Sweep_and_prune
[15]: http://ru.wikipedia.org/wiki/%D0%90%D0%BB%D0%B3%D0%BE%D1%80%D0%B8%D1%82%D0%BC_%D0%93%D0%B8%D0%BB%D0%B1%D0%B5%D1%80%D1%82%D0%B0_%E2%80%94_%D0%94%D0%B6%D0%BE%D0%BD%D1%81%D0%BE%D0%BD%D0%B0_%E2%80%94_%D0%9A%D1%91%D1%80%D1%82%D0%B8
[16]: http://ru.wikipedia.org/wiki/%D0%A1%D1%84%D0%B5%D1%80%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B8%D0%B9_%D0%BA%D0%BE%D0%BD%D1%8C_%D0%B2_%D0%B2%D0%B0%D0%BA%D1%83%D1%83%D0%BC%D0%B5#.D0.A1.D1.84.D0.B5.D1.80.D0.B8.D1.87.D0.B5.D1.81.D0.BA.D0.B8.D0.B9_.D0.BA.D0.BE.D0.BD.D1.8C_.D0.B2_.D0.B2.D0.B0.D0.BA.D1.83.D1.83.D0.BC.D0.B5
[17]: https://github.com/wellcaffeinated/PhysicsJS/wiki/Fundamentals#extending-subtypes
[18]: https://github.com/wellcaffeinated/PhysicsJS/wiki/Scratchpads
[19]: http://stackoverflow.com/questions/tagged/physicsjs
[20]: http://millionthvector.blogspot.ca/p/free-sprites.html


[Игра, которая у нас получится]: img/PhysicsJS_header_2.jpg "Игра, которая у нас получится"
