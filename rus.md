# Создание двухмерной браузерной игры на основе PhysicsJS

Я веб-разработчик и фанат физики. При каждой попытке реализовать симуляцию 
двухмерной физики на основе JavaScript мне неизменно казалось, что чего-то не 
хватает. Мне был нужен гибкий фреймворк, который не уступал бы другим 
современным JavaScript API. Я не хотел чтобы он предполагал по умолчанию, что 
мне нужна направленная вниз гравитация или что мне вообще нужна гравитация... 
Это побудило меня создать [PhysicsJS][1], который, надеюсь, оправдает своё 
описание:

> Модульный расширяемый и простой в использовании физический движок для 
JavaScript.

В последнее время появилось несколько новых физических движков для JavaScript и 
все они обладают своими достоинствами. Однако я хочу продемонстрировать вам 
некоторые действительно крутые возможности PhysicsJS, которые как мне кажется, 
позволяют ему выделяться на фоне остальных.

В этой статье я проведу вас через процесс разработки полноценной 2D-игры об 
астероидах, со взрывами и простенькой графикой. Вы попутно увидите примеры моего 
стиля разработки и сможете лучше понять что можно извлечь из PhysicsJS. Полный 
[код, использованный в качестве примера, можно скачать в репозитории PhysicsJS 
на GitHub][2] (посмотрите в examples/spaceship), [конечный результат можно 
увидеть на CodePen][3]. Однако перед тем, как приступить к делу, хочу сделать 
важное заявление:

> PhysicsJS является новой библиотекой и мы усиленно работаем над тем, чтобы 
представить её бета-версию в ближайшие месяцы. Этот API находится в состоянии 
корректирования и всё ещё содержит некоторые ошибки и недочёты. Имейте это ввиду. 
(Я буду использовать [PhysicsJS v0.5.2][4]). Я горячо приветствую ваше участие 
любого рода. Просто загляните в репозиторий [PhysicsJS на GitHub][5], где есть 
открытый доступ к коду. Я также стараюсь отвечать на вопросы о [PhysicsJS на 
StackOverflow][6]. Просто добавьте метку «physicsjs» к вашему вопросу. 

## Шаг 0: Пустота космоса

Начнём с начала. Нам нужно прописать HTML. Для рендеринга мы будем использовать 
HTML `canvas`, так что начнём со стандартного шаблона кода HTML5. Я буду 
исходить из того, что скрипты загружаются в конце тега `body`, чтобы не 
беспокоиться о готовности DOM. Кроме того, мы будем использовать [RequireJS][7] 
для загрузки всех скриптов. Если вы никогда раньше не использовали RequireJS и 
не имеете желания в него вникать, вам придётся избавиться от всех `define()` и 
`require()` и перегруппировать весь JavaScript в правильном порядке (однако я 
настоятельно рекомендую использовать RequireJS).

Вот дерево каталога, с которым мы будем работать:

    index.html
    images/
        ...
    js/
        require.js
        physicsjs/
        ...

Скачайте [RequireJS][8], [PhysicsJS-0.5.2][9], и [изображения][10] и разместите 
их в такое же дерево. 

Теперь добавим в `head` немного CSS для установки фонового изображения «космоса» 
и несколько игровых сообщений, привязанных к классу `body`:

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
        content: 'нажмите "z" чтобы начать игру';
    }
    body.lose-game:after {
        content: 'нажмите "z" чтобы попробовать ещё раз';
    }
    body.win-game:after {
        content: 'Победа! нажмите "z" чтобы сыграть ещё раз';
    }
    body {
        background: url(images/starfield.png) 0 0 repeat;
    }

Теперь для настройки RequireJS и запуска приложения добавим код в конец тега 
`body`:

    <script src="js/require.js"></script>
    <script src="js/main.js"></script>

Дальше нам нужно создать файл `main.js`. По ходу данного руководства этот файл 
будет изменяться. Обратите внимание, что я публикую живые примеры с CodePen, так 
что вам может потребоваться изменить настройки RequireJS для вашего локального 
образца. Нечто подобное должно сработать:

    // начало main.js
    require(
        {
            // для доступа к изображениям используем код высшего уровня
            baseUrl: './',
            packages: [{
                name: 'physicsjs',
                location: 'js/physicsjs',
                main: 'physicsjs'
            }]
        },
        ...

Итак, начнём с первой версии `main.js`.

<iframe id="cp_embed_78984a62f88d8a9cb31b6768db6890ac" src="//codepen.io/wellcaffeinated/embed/78984a62f88d8a9cb31b6768db6890ac?safe=true&amp;user=wellcaffeinated&amp;href=78984a62f88d8a9cb31b6768db6890ac&amp;type=js&amp;height=450&amp;slug-hash=78984a62f88d8a9cb31b6768db6890ac&amp;default-tab=js&amp;animations=run" allowtransparency="true" class="cp_embed_iframe" style="width: 100%; overflow: hidden;" frameborder="0" height="450" scrolling="no"></iframe>

Более скучной игры не придумаешь.

Так что у нас тут происходит? Во-первых, мы подгрузили библиотеку PhysicsJS, 
объявив её зависимость от RequireJS. Кроме того мы подгрузили несколько других 
зависимых компонентов. Это расширения PhysicsJS, подключённые для добавления 
функциональных возможностей вроде вращения тел, определения столкновений и 
других. Используя RequireJS (и модули AMD), нам нужно объявлять каждую 
зависимость такого рода. Это позволяет загружать только то, что будет 
впоследствии использоваться.

Внутри фабричной функции RequireJS мы прописываем настройки игры. Начинаем с 
добавления названия класса в тег `body` чтобы отображалось сообщение «нажмите 
"z" чтобы начать игру» и устанавливаем обработчик события нажатия клавиши «z», 
который вызывает вспомогательную функцию `newGame()`.

Затем настраиваем средство визуализации `canvas` для заполнения окна и 
устанавливаем обработчик события изменения размера окна, который выполняет 
изменение размера `canvas`. 

Немного дальше прописываем ядро игры в функции `init()`. Функция `init` будет 
передаваться конструктору `Physics()` для создания мира. Внутри этой функции 
прописываем некоторые нужные нам компоненты:

* «корабль» (который на данный момент является всего лишь телом круглой формы)
* планету (которая является кругом с большой массой и специальным изображением)
* обработчик события для запуска рендеринга на каждом этапе изменения мира

Разъяснения требует только код для планеты... по сути лишь эта часть:

    planet.view = new Image();
    planet.view.src = require.toUrl('images/planet.png');

И что ж это такое?

Мы создаём элемент изображения и храним его в свойстве `.view`. Изображение 
загружает картинку `planet.png` (и мы используем RequireJS чтобы определить путь 
к этому изображению). Но зачем это? Таким образом мы имеем возможность изменять 
отображение наших объектов. Средство рендеринга `canvas` проверяет все объекты, 
которые нужно визуализировать и если у какого-то из них отсутствует это свойство, 
отрисовывает его представление (круг, многогранник...) в `canvas` и кеширует его 
как изображение. Это изображение хранится в соответствующем свойстве `.view`. 
Однако, если в нём уже присутствует изображение, то использоваться будет оно. 

**Внимание:** Я знаю, что этот способ контроля на визуализацией далёк от 
идеального. Однако у меня для вас есть две хорошие новости:

1. Это средство визуализации можно легко расширить или заменить *чем хотите*... 
при желании можете создать своё собственное средство визуализации. В скором 
будущем будет добавлена [документация о том, как это можно сделать][11].
2. Через несколько недель в [ChallengePost][12] будет проходить конкурс на 
создание наиболее гибкого и могущественного средства визуализации для PhysicsJS. 
Вы должны принять участие! И даже если не примите, возьмите на вооружение его 
результаты. 

Я немного отвлёкся. Продолжаем... Давайте взглянем на последнее, что происходит 
в функции `init()`:

    // добавление объектов в мир игры
    world.add([
      ship,
      planet,
      Physics.behavior('newtonian', { strength: 1e-4 }),
      Physics.behavior('sweep-prune'),
      Physics.behavior('body-collision-detection'),
      Physics.behavior('body-impulse-response'),
      renderer
    ]);

Мы одним махом добавляем в мир всякие разности. В их числе тела, которые мы 
создали (корабль и планета), разные алгоритмы поведения и средство рендеринга. 
Давайте поочерёдно рассмотрим алгоритмы:

* `newtonian`: добавляет в мир [ньютоновскую гравитацию][13]. Тела будут 
притягиваться друг к другу согласно закону обратных квадратов `1e-4`.
* `sweep-prune`: это [алгоритм широкой фазы][14], который ускоряет обнаружение 
столкновений. Если ваша игра предусматривает столкновение тел, вероятно лучшего 
всего использовать его.
* `body-collision-detection`: узкая фаза определения столкновения, в которой 
установлен обработчик событий `sweep-prune` и используются алгоритмы (вроде 
[GJK][15], алгоритма Гилберта — Джонсона — Кёрти) для определения столкновения 
тел и того, как оно происходит. Примите во внимание что столкновение всего лишь 
*определяется*, в ответ на него ничего не происходит.
* `body-impulse-response`: непосредственно *выполняет* определённое действие при 
обнаружении столкновения. Этот алгоритм является обработчиком событий 
столкновения и отвечает на них, придавая столкнувшимся телам импульс.

Надеюсь на этом этапе вы уже начинаете улавливать модульную природу PhysicsJS. 
Ни у мира, ни у наполняющих его объектов нет встроенных алгоритмов в плане 
физических свойств. Если вы хотите чтобы физические свойства вашего мира не 
ограничивались свойствами вакуума, вам потребуется добавить алгоритмы, которые 
будут влиять на мир и тела в нём.

Следующим шагом будет добавление в наш скрипт `main.js` вспомогательной функции 
`newGame()`, которая производит зачистку и создаёт новый сеанс игры посредством 
вызова `Physics( init )`. Она также выполняет функции обработчика 
пользовательских событий в новосозданном мире, которые сигнализируют о 
необходимости закончить сеанс игры и начать новый. На данный момент ничто не 
порождает эти события, однако скоро они нам потребуются.

Наконец, мы присоединяем таймер (вспомогательный метод `requestAnimationFrame`) 
чтобы `world.step` вызывался для каждого кадра.

## Шаг 1: Приводим объекты в порядок

Всё станет намного интересней, когда мы сможем летать по нашему миру. Мы могли 
бы сделать с нашим кораблём нечто похожее на то, что сделали с планетой, однако 
нам не достаточно только применения подходящего внешнего облика. Нам нужны 
методы маневрирования кораблем. Лучшим планом действий будет применение одной из 
моих любимых возможностей PhysicsJS, *возможность практически неограниченного 
расширения*. В данном случае мы будет работать над расширением тел. 

Мы создадим новый файл с названием `player.js`. Он послужит для определения 
пользовательского объекта с названием `player`. Пусть у нашего корабля будет 
сферическая форма ([ха-ха][16]) и дополним тело `circle` как [описано в 
справочнике по PhysicsJS][17].

Мы настроим наше определение RequireJS для модуля...

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
        // здесь должен быть код...
    });

Теперь у нас есть доступ к круглому телу и мы можем его расширить, а также 
выпуклые многоугольные тела, которые обозначают космический мусор (я ведь обещал 
взрывы). На старт, внимание, расширяем!

    // расширение для круглого тела
    Physics.body('player', 'circle', function( parent ){
        // закрытые вспомогательные функции 
        // ...
        return {
            // определение расширения
        };
    });

Тело `player` должно быть подключено к PhysicsJS. Мы добавим закрытые 
вспомогательные функции и возвратим объект-литерал для расширения круглого тела. 

Вот наши закрытые вспомогательные функции:

    // закрытые вспомогательные функции
    var deg = Math.PI/180;
    var shipImg = new Image();
    var shipThrustImg = new Image();
    shipImg.src = require.toUrl('images/ship.png');
    shipThrustImg.src = require.toUrl('images/ship-thrust.png');

    var Pi2 = 2 * Math.PI;
    // ОЧЕНЬ грубое округление случайного числа по Гауссу... зато быстрое
    var gauss = function gauss( mean, stddev ){
        var r = 2 * (Math.random() + Math.random() + Math.random()) - 3;
        return r * stddev + mean;
    };
    // даёт случайный многоугольник, который скорее всего будет выпуклым
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

А вот наши расширения для круглого объекта (подробности в комментариях):

    return {
        // когда создано тело нам нужно его настроить
        // потому мы вызываем метод init родителя
        // для «this»
        init: function( options ){
            parent.init.call( this, options );
            // устанавливаем изображение для визуализации
            // с той картинкой которую я выбрал, нос корабля
            // будет повернут под тем же углом, согласно угловому положению тела
            this.view = shipImg;
        },
        // поворот корабля посредством изменения 
        // угловой скорости тела на + или - какую-то величину
        turn: function( amount ){
            // установка угловой скорости корабля
            this.state.angular.vel = 0.2 * amount * deg;
            return this;
        },
        // это увеличит скорость корабля вдоль направления
        // носа корабля
        thrust: function( amount ){
            var self = this;
            var world = this._world;
            if (!world){
                return self;
            }
            var angle = this.state.angular.pos;
            var scratch = Physics.scratchpad();
            // уменьшение величины к менее безумной
            amount *= 0.00001;
            // указание ускорения в направлении носа корабля
            var v = scratch.vector().set(
                amount * Math.cos( angle ), 
                amount * Math.sin( angle ) 
            );
            // самоускорение
            this.accelerate( v );
            scratch.done();

            // при ускорении изменение изображения на картинку с включёнными двигателями
            if ( amount ){
                this.view = shipThrustImg;
            } else {
                this.view = shipImg;
            }
            return self;
        },
        // создаём снаряд (маленький круг)
        // который отделяется от передней части корабля.
        // После определённого промежутка времени он удаляется
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
            // создание маленького круга у носа корабля,
            // который движется со скоростью 0.5 в направлении носа 
            // относительно текущей скорости корабля
            var laser = Physics.body('circle', {
                x: this.state.pos.get(0) + r * cos,
                y: this.state.pos.get(1) + r * sin,
                vx: (0.5 + this.state.vel.get(0)) * cos,
                vy: (0.5 + this.state.vel.get(1)) * sin,
                radius: 2
            });
            // установление пользовательского свойства для столкновений
            laser.gameType = 'laser';

            // удаление лазерного импульса через 600 мс
            setTimeout(function(){
                world.removeBody( laser );
                laser = undefined;
            }, 600);
            world.add( laser );
            return self;
        },
        // Бабах! Удаляем корабль
        // и заменяем его кучкой треугольников
        // для имитации взрыва!
        blowUp: function(){
            var self = this;
            var world = this._world;
            if (!world){
                return self;
            }
            var scratch = Physics.scratchpad();
            var rnd = scratch.vector();
            var pos = this.state.pos;
            var n = 40; // создание 40 элементов мусора
            var r = 2 * this.geometry.radius; // окружность
            var size = 8 * r / n; // примерный размер краев мусора
            var mass = this.mass / n; // масса мусора
            var verts;
            var d;
            var debris = [];

            // создание космического мусора
            while ( n-- ){
                verts = rndPolygon( size, 3, 1.5 ); // получение случайного многоугольника
                if ( Physics.geometry.isPolygonConvex( verts ) ){
                    // установка случайного расположения мусора (относительно игрока)
                    rnd.set( Math.random() - 0.5, Math.random() - 0.5 ).mult( r );
                    d = Physics.body('convex-polygon', {
                        x: pos.get(0) + rnd.get(0),
                        y: pos.get(1) + rnd.get(1),
                        // скорость мусора равна скорости игрока
                        vx: this.state.vel.get(0),
                        vy: this.state.vel.get(1),
                        // установка случайно угловой скорости для большего драматизма
                        angularVelocity: (Math.random()-0.5) * 0.06,
                        mass: mass,
                        vertices: verts,
                        // избавляемся от чрезмерного подёргивания
                        restitution: 0.8
                    });
                    d.gameType = 'debris';
                    debris.push( d );
                }
            }

            // добавляем мусор
            world.add( debris );
            // удаляем игрока
            world.removeBody( this );
            scratch.done();
            return self;
        }
    };

Вы возможно заметили, что мы используем `Physics.scratchpad`. Это ваш лучший 
помощник. Скретчпад помогает удалять временные объекты (векторы) для сокращение 
времени, требуемого на создание и удаление мусора. [Больше о скретчпадах можно 
почитать здесь][18]. 

Итак, теперь у нас есть игрок, однако он не привязан к какому-либо вводу данных 
со стороны пользователя. Сейчас нам нужно создать *сценарий поведения игрока* 
для реагирования на действия пользователя. Для этого мы создадим новый файл 
`player-behavior.js`, довольно похожий на предыдущие (подробности в комментариях):

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
                    // игрок добавляется через настройки options
                    // нам нужно его сохранить
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

                // Эта часть автоматически вызывается 
                // когда этот сценарий поведения добавляется в мир
                connect: function( world ){

                    // мы хотим отслеживать события в мире
                    world.subscribe('collisions:detected', this.checkPlayerCollision, this);
                    world.subscribe('integrate:positions', this.behave, this);
                },

                // Эта часть автоматически вызывается
                // когда сценарий поведения удаляется из мира
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
                        // и одним из тел является игрок...
                        if ( col.bodyA.gameType !== 'debris' && 
                            col.bodyB.gameType !== 'debris' && 
                            (col.bodyA === player || col.bodyB === player) 
                        ){
                            player.blowUp();
                            world.removeBehavior( this );
                            this.gameover = true;

                            // после столкновения мы порождаем событие 
                            // для которого можно установить обработчик, перезапускающий игру
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

Итак сейчас мы можем прописать зависимость между `js/player` и 
`js/player-behavior` и использовать их в файле `main.js` добавив в нашу функцию 
`init()` следующее:

    // в init()
    var ship = Physics.body('player', {
        x: 400,
        y: 100,
        vx: 0.08,
        radius: 30
    });

    var playerBehavior = Physics.behavior('player-behavior', { player: ship });

    // ...

    world.add([
        ship,
        playerBehavior,
        //...
    ]);

Последнее, что нам нужно перед тем, как представить вторую версию игры — это 
приготовить средство визуализации к отслеживанию движения пользователя. Этого 
можно достичь, добавив порцию кода в обработчик события `step` для изменения 
угла расположения средства визуализации перед вызовом `world.render()`. Выглядит 
это так:

    // внутри init()...
    // отрисовка каждого кадра
    world.subscribe('step', function(){
        // середина холста
        var middle = { 
            x: 0.5 * window.innerWidth, 
            y: 0.5 * window.innerHeight
        };
        // следование за игроком
        renderer.options.offset.clone( middle ).vsub( ship.state.pos );
        world.render();
    });

Наш второй вариант кода теперь больше напоминает игру!

<iframe id="cp_embed_7ef83de07c264d5f140b1c56027a9d29" src="//codepen.io/wellcaffeinated/embed/7ef83de07c264d5f140b1c56027a9d29?safe=true&amp;user=wellcaffeinated&amp;href=7ef83de07c264d5f140b1c56027a9d29&amp;type=js&amp;height=450&amp;slug-hash=7ef83de07c264d5f140b1c56027a9d29&amp;default-tab=js&amp;animations=run" allowtransparency="true" class="cp_embed_iframe" style="width: 100%; overflow: hidden;" frameborder="0" height="450" scrolling="no"></iframe>

## Шаг 2: Добавим немного остроты

На данный момент наша игра довольно скучна. Нам нужны враги. Давайте создадим их!

Мы будем делать почти то же, что и для игрока. Создадим новое тело (расширенный 
круг), но в этом случае сделаем всё проще. Нам нужно только чтобы враги 
взрывались, потому применим другой подход и не будем создавать сценарий 
поведения, поскольку их функциональность столь минимальна. Создадим новый файл с 
именем `ufo.js` и присвоим нашим НЛО упрощённый метод `blowUp()`:

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

                    // создание мусора
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

    // внутри init()...
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

    //...

    world.add( ufos );

`Math` в коде используется для случайного расположения тел так, чтобы они 
вращались на орбите планеты и медленно к ней подкрадывались. Однако на данный 
момент мы не можем их взорвать, потому добавим ещё немного кода в функцию 
`init()` для отслеживания количества убитых и порождения события `win-game` если 
убиты все. Также добавим обработчик для события `collisions:detected` и если в 
столкновении участвовал лазер, взорвём объект если его взрыв поддерживается.

    // внутри init()...
    // считаем количество уничтоженных НЛО 
    var killCount = 0;
    world.subscribe('blow-up', function( data ){

        killCount++;
        if ( killCount === ufos.length ){
            world.publish('win-game');
        }
    });

    // взрываем всё, чего касается лазерный импульс
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
управления НЛО, однако для них требуется настолько мало кода, что в этом нет 
необходимости. Я также хотел продемонстрировать насколько гибкой является 
PhysicsJS при использовании различных стилей написания кода; для одной и той же 
задачи можно придумать различные способы выполнения.

Давайте взглянем на третий вариант `main.js`:

<iframe id="cp_embed_ef2772ad82abfa8f04a349081afb0f19" src="//codepen.io/wellcaffeinated/embed/ef2772ad82abfa8f04a349081afb0f19?safe=true&amp;user=wellcaffeinated&amp;href=ef2772ad82abfa8f04a349081afb0f19&amp;type=js&amp;height=450&amp;slug-hash=ef2772ad82abfa8f04a349081afb0f19&amp;default-tab=js&amp;animations=run" allowtransparency="true" class="cp_embed_iframe" style="width: 100%; overflow: hidden;" frameborder="0" height="450" scrolling="no"></iframe>

Фантастика! Мы почти закончили. Есть лишь ещё одна вещь, которую я хотел бы вам 
показать...

## Шаг 3: Ориентация на местности

В космосе довольно трудно ориентироваться. Окружение плохо просматривается и 
иногда требуется помощь. Чтобы с этим справиться, создадим мини-карту с радаром. 
Как мы можем это сделать? 

Нам придётся определить позиции всех тел и нарисовать маленькие точки на 
небольшом участке `canvas` в верхнем правом углу. Хотя средство визуализации и 
не обладает настолько полным набором возможностей, как хотелось бы, в нашем 
распоряжении есть некоторые полезные вспомогательные методы. Мы привяжем событие 
`render`, вызов которого происходит после отрисовки объектов средством 
визуализации. Затем мы просто добавим изображение с мини-картой в рамку, 
используя эти вспомогательные методы. Вот код:

    // внутри init()...
    // отрисовка мини-карты
    world.subscribe('render', function( data ){
        // радиус мини-карты
        var r = 100;
        // отступы
        var shim = 15;
        // x,y центра
        var x = renderer.options.width - r - shim;
        var y = r + shim;
        // чрезвычайно полезный скретчпад для ускорения вычислений для векторов
        var scratch = Physics.scratchpad();
        var d = scratch.vector();
        var lightness;

        // отрисовка указателей радара
        renderer.drawCircle(x, y, r, { strokeStyle: '#090', fillStyle: '#010' });
        renderer.drawCircle(x, y, r * 2 / 3, { strokeStyle: '#090' });
        renderer.drawCircle(x, y, r / 3, { strokeStyle: '#090' });

        for (var i = 0, l = data.bodies.length, b = data.bodies[ i ]; b = data.bodies[ i ]; i++){

            // расчёт смещения тела от корабля и его масштабирование
            d.clone( ship.state.pos ).vsub( b.state.pos ).mult( -0.05 );
            // окрашивание точки в зависимости от массивности тела
            lightness = Math.max(Math.min(Math.sqrt(b.mass*10)|0, 100), 10);
            // если она внутри радиуса мини-карты
            if (d.norm() < r){
                // отрисовка точки
                renderer.drawCircle(x + d.get(0), y + d.get(1), 1, 'hsl(60, 100%, '+lightness+'%)');
            }
        }

        scratch.done();
    });

Хорошо, добавим этот код в наш `main.js` и взглянем на конечный продукт!

<iframe id="cp_embed_7c1f72c5c9c8e2159ee97dc6540e0b72" src="//codepen.io/wellcaffeinated/embed/7c1f72c5c9c8e2159ee97dc6540e0b72?safe=true&amp;user=wellcaffeinated&amp;href=7c1f72c5c9c8e2159ee97dc6540e0b72&amp;type=js&amp;height=450&amp;slug-hash=7c1f72c5c9c8e2159ee97dc6540e0b72&amp;default-tab=js&amp;animations=run" allowtransparency="true" class="cp_embed_iframe" style="width: 100%; overflow: hidden;" frameborder="0" height="450" scrolling="no"></iframe>

## В заключение

Вуаля! Вот и всё! Спасибо за терпение, которое потребовалось для прочтения 
такого длинного руководства. 

Сюжет игры, конечно, не самый увлекательный, и остаётся ещё много работы 
(например, минимизация скриптов с помощью инструмента RequireJS), однако моей 
целью было дать вам представление и возможностях PhysicsJS, а также о том, 
насколько полезной она может быть даже в текущей сырой версии. Надеюсь я дал вам 
достаточно материала для экспериментов. Не забывайте, если у вас есть вопросы, 
их можно задать в комментариях или на [StackOverflow][19].

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
[13]: http://en.wikipedia.org/wiki/Newton
[14]: http://en.wikipedia.org/wiki/Sweep_and_prune
[15]: http://en.wikipedia.org/wiki/GJK
[16]: http://en.wikipedia.org/wiki/Spherical_cow
[17]: https://github.com/wellcaffeinated/PhysicsJS/wiki/Fundamentals#extending-subtypes
[18]: https://github.com/wellcaffeinated/PhysicsJS/wiki/Scratchpads
[19]: http://stackoverflow.com/questions/tagged/physicsjs
[20]: http://millionthvector.blogspot.ca/p/free-sprites.html