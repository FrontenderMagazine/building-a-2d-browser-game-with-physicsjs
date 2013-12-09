*By [Jasper Palfree][1]*

As a web developer and a physics geek, I always felt like something was lacking
whenever I tried to do 2D physics in JavaScript. I wanted an extensible 
framework that followed in the footsteps of other modern JavaScript APIs. I didn
’t want it to assume that I wanted gravity to point downwards, or that I wanted 
gravity at all… That’s why I was driven to create[PhysicsJS][2], which I hope
will live up to its tagline:

> A modular, extendable, and easy-to-use physics engine for JavaScript.

There are a few new JavaScript physics engines that have popped up, and they
have their merits. But I want to show you some things that I think are really 
cool about PhysicsJS that I believe help it stand above the others.

In this article, I’m going to walk you through the development of a full 2D
Asteroids-style game filled with explosions and cheesy graphics. Along the way 
you’ll see some of my development style show through, and it will give you some 
insight into the more advanced usage of PhysicsJS. You can download the full
[demo code on the PhysicsJS GitHub repository][3] (look in examples/spaceship)
and[see the final product on CodePen][4]. But before we get going, let me first
issue an important disclaimer:

> PhysicsJS is a new library, and we’re working hard to get it into beta in
> the coming months. The API is in flux and there are still bugs, and holes in the
> API. So keep that in mind. (We’re working with
>[PhysicsJS v0.5.2][5]). I’m looking for contributors of any commitment level
> . Just hop over to the
>[PhysicsJS GitHub][3] repo and hack away! I’m also trying to answer
> questions about
>[PhysicsJS on StackOverflow][6]. Just tag your question with a “physicsjs
> ” tag.
>

## Stage 0: The Emptiness of Space {#stage0theemptinessofspace}

First thing’s first. We need to set up our html. We’ll be using HTML canvas
for rendering, so just start with your standard HTML5 boilerplate code. I’m 
going to assume that scripts are loaded at the end of the body tag so that I don
’t need to worry about DOM readiness. We’ll also be using[RequireJS][7] to load
all of our scripts. If you haven’t used RequireJS before, and don’t feel like 
learning, then you’ll have to get rid of any`define()`s and `require()`s and
put together all of your JavaScript in the right order (but I highly recommend 
using RequireJS
).

Here’s the directory structure I’ll be working with:

    index.html
    images/
        ...
    js/
        require.js
        physicsjs/
        ...

Go ahead and download [RequireJS][7], [PhysicsJS-0.5.2][5], and the [images][8]

Now, let’s add some CSS in the head that will set up the “space”
background image and some gameplay messages depending on the classname of the 
body:

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
        content: 'press "z" to start';
    }
    body.lose-game:after {
        content: 'press "z" to try again';
    }
    body.win-game:after {
        content: 'Win! press "z" to play again';
    }
    body {
        background: url(images/starfield.png) 0 0 repeat;
    }

Now let’s add some code at the end of the `body` tag to setup RequireJS and
initialize our app:

    <script src="js/require.js"></script>
    <script src="js/main.js"></script>

Next, we need to create our `main.js` file. This file is going to change over
the course of this tutorial. Note, the versions I’m posting are live examples 
from CodePen, so you may need to change the RequireJS configuration paths to 
work in your local example. Something like this should work for you:

    // begin main.js
    require(
        {
            // use top level so we can access images
            baseUrl: './',
            packages: [{
                name: 'physicsjs',
                location: 'js/physicsjs',
                main: 'physicsjs'
            }]
        },
        ...

Okay, let’s start with our first version of `main.js`.

    [Check out this Pen!][9]

Most boring game… ever.

So what’s going on here? Firstly, we’re loading the PhysicsJS library by
declaring it as a dependency to RequireJS. We’re also loading in a bunch of 
other dependencies. These are PhysicsJS extensions that plug into PhysicsJS to 
add functionality like circle bodies, collision detection, and more. When using 
RequireJS (and AMD modules) we need to declare every dependency that we need 
like this. It allows us to only load what we’ll actually use.

Inside our RequireJS factory function, we’re setting up our game. We start by
adding a class name to the body tag so that the “press ‘z’ to start” message 
gets displayed, and we listen for the “z” key event to call our helper function
`newGame()`.

Next we set up the canvas renderer to fill the window and listen for the window
resize event to resize the canvas.

A bit farther down we set up the meat of our game within the `init()` function
. The`init` function will be passed to the `Physics()` constructor to create a
world. Inside this function we set up some things we need:

*   a “ship” (which right now is just a circle body)
*   a planet (which is a circle with a large mass and a custom image)
*   a listener to trigger rendering on each world step

The planet is the only part that really needs explanation… actually it’s
really just this part:

    planet.view = new Image();
    planet.view.src = require.toUrl('images/planet.png');

What the heck is that?

What we’re doing is creating an image and storing it in the `.view` property.
The image will load the`planet.png` image (and we use RequireJS to resolve the
path to that image). But why are we doing that? This is the way to customize how
our objects are displayed. The canvas renderer looks at all the bodies it needs 
to render and if any are missing this property it draws a representation of it (
circle, polygon…) to canvas and caches it as an image. That image gets stored in
the body’s`.view` property. However, if there’s already an image there, the
renderer will use it!

**Note:** This is far from an ideal way to customize the rendering process, I
know. But there are two bits of*great* news to cheer you up:

1.  The renderer can be easily extended, or replaced by *anything* you want…
    it’s just up to you to build a different renderer. There will be
   [documentation about how to do this][10] in the near future.
2.  In a few short weeks, there will be a contest at [ChallengePost][11] to
    build a more versatile and powerful renderer for PhysicsJS! You should enter! 
    But even if you don’t, you can still reap the benefits.
   

But I digress. Moving on… Let’s look at the last thing that’s going on in
the`init()` function:

    // add things to the world
    world.add([
      ship,
      planet,
      Physics.behavior('newtonian', { strength: 1e-4 }),
      Physics.behavior('sweep-prune'),
      Physics.behavior('body-collision-detection'),
      Physics.behavior('body-impulse-response'),
      renderer
    ]);

We’re adding all sorts of goodies to the world in one fell swoop. We’re
adding the bodies we created (ship and planet), some behaviors, and a renderer. 
Let’s look at those behaviors one by one:

*   `newtonian`: This adds [newtonian gravity][12] to the world. Bodies will be
    attracted to each other by an inverse square law of strength
   `1e-4`.
*   `sweep-prune`: This is a [broad phase algorithm][13] that speeds up
    collision detection. If you’re game includes colliding bodies, it’s probably 
    best to use this.
   
*   `body-collision-detection`: The narrow phase collision detection that
    listens to sweep-prune’s events and uses algorithms (like
   [GJK][14]) to figure out if and how bodies are colliding. Note that this
    only
   *detects* collisions, it doesn’t do anything in response to them.
*   `body-impulse-response`: This actually *does* something when collisions are
    detected. This behavior listens for collision events and responds by applying 
    impulses to the colliding bodies.
   

Hopefully, you are now starting to see the modular nature of PhysicsJS. Neither
the bodies, nor the world have any logic about the physics. If you want the 
physics of your world to be anything but a vacuum, you need to create a behavior
that plugs into the world and acts on the bodies in that world.

The next thing we do in our `main.js` script is setup a `newGame()` helper
function, which cleans up and creates a new game by calling`Physics( init )`.
It also listens for custom events on that newly created world to end the game 
and start again. Right now there’s nothing emitting those events, but we’ll need
them soon.

Lastly, we connect to the ticker (`requestAnimationFrame` helper) so that our
`world.step` gets called on every frame.

## Stage 1: Making Things Shipshape {#stage1makingthingsshipshape}

Things are going to get much more fun when we can start flying around our world
. We could do something similar to our ship as we did to our planet, but we’ll 
be needing more than just a custom skin. We need methods to maneuver the ship. 
Our best course of action is to use one of my favorite things about PhysicsJS,*
the ability to extend almost anything.*In this case we’ll be extending Bodies
.

We’re going to create a new file called `player.js`. It will serve to define a
custom body called`player`. We’ll assume a spherical ship ([haha][15]), and
extend the`circle` body as [described in the PhysicsJS wiki][16].

We’ll set up our RequireJS definition for a module…

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
        // code here...
    });

Now we’ve got access to the circle body so we can extend it, and the convex-
polygon body for debris (I promised explosions). Ready, set, extend!

    // extend the circle body
    Physics.body('player', 'circle', function( parent ){
        // private helpers
        // ...
        return {
            // extension definition
        };
    });

The `player` body will get plugged into PhysicsJS. We’ll put some private
helper functions in there and pass back an object literal to extend upon the 
circle body.

Here are our private helpers:

    // private helpers
    var deg = Math.PI/180;
    var shipImg = new Image();
    var shipThrustImg = new Image();
    shipImg.src = require.toUrl('images/ship.png');
    shipThrustImg.src = require.toUrl('images/ship-thrust.png');
    
    var Pi2 = 2 * Math.PI;
    // VERY crude approximation to a gaussian random number.. but fast
    var gauss = function gauss( mean, stddev ){
        var r = 2 * (Math.random() + Math.random() + Math.random()) - 3;
        return r * stddev + mean;
    };
    // will give a random polygon that, for small jitter, will likely be convex
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

And here are our extensions to the circle object (details in the comments):

    return {
        // we want to do some setup when the body is created
        // so we need to call the parent's init method
        // on "this"
        init: function( options ){
            parent.init.call( this, options );
            // set the rendering image
            // because of the image i've chosen, the nose of the ship
            // will point in the same angle as the body's rotational position
            this.view = shipImg;
        },
        // this will turn the ship by changing the
        // body's angular velocity to + or - some amount
        turn: function( amount ){
            // set the ship's rotational velocity
            this.state.angular.vel = 0.2 * amount * deg;
            return this;
        },
        // this will accelerate the ship along the direction
        // of the ship's nose
        thrust: function( amount ){
            var self = this;
            var world = this._world;
            if (!world){
                return self;
            }
            var angle = this.state.angular.pos;
            var scratch = Physics.scratchpad();
            // scale the amount to something not so crazy
            amount *= 0.00001;
            // point the acceleration in the direction of the ship's nose
            var v = scratch.vector().set(
                amount * Math.cos( angle ), 
                amount * Math.sin( angle ) 
            );
            // accelerate self
            this.accelerate( v );
            scratch.done();
    
            // if we're accelerating set the image to the one with the thrusters on
            if ( amount ){
                this.view = shipThrustImg;
            } else {
                this.view = shipImg;
            }
            return self;
        },
        // this will create a projectile (little circle)
        // that travels away from the ship's front.
        // It will get removed after a timeout
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
            // create a little circle at the nose of the ship
            // that is traveling at a velocity of 0.5 in the nose direction
            // relative to the ship's current velocity
            var laser = Physics.body('circle', {
                x: this.state.pos.get(0) + r * cos,
                y: this.state.pos.get(1) + r * sin,
                vx: (0.5 + this.state.vel.get(0)) * cos,
                vy: (0.5 + this.state.vel.get(1)) * sin,
                radius: 2
            });
            // set a custom property for collision purposes
            laser.gameType = 'laser';
    
            // remove the laser pulse in 600ms
            setTimeout(function(){
                world.removeBody( laser );
                laser = undefined;
            }, 600);
            world.add( laser );
            return self;
        },
        // 'splode! This will remove the ship
        // and replace it with a bunch of random
        // triangles for an explosive effect!
        blowUp: function(){
            var self = this;
            var world = this._world;
            if (!world){
                return self;
            }
            var scratch = Physics.scratchpad();
            var rnd = scratch.vector();
            var pos = this.state.pos;
            var n = 40; // create 40 pieces of debris
            var r = 2 * this.geometry.radius; // circumference
            var size = 8 * r / n; // rough size of debris edges
            var mass = this.mass / n; // mass of debris
            var verts;
            var d;
            var debris = [];
    
            // create debris
            while ( n-- ){
                verts = rndPolygon( size, 3, 1.5 ); // get a random polygon
                if ( Physics.geometry.isPolygonConvex( verts ) ){
                    // set a random position for the debris (relative to player)
                    rnd.set( Math.random() - 0.5, Math.random() - 0.5 ).mult( r );
                    d = Physics.body('convex-polygon', {
                        x: pos.get(0) + rnd.get(0),
                        y: pos.get(1) + rnd.get(1),
                        // velocity of debris is same as player
                        vx: this.state.vel.get(0),
                        vy: this.state.vel.get(1),
                        // set a random angular velocity for dramatic effect
                        angularVelocity: (Math.random()-0.5) * 0.06,
                        mass: mass,
                        vertices: verts,
                        // not tooo bouncy
                        restitution: 0.8
                    });
                    d.gameType = 'debris';
                    debris.push( d );
                }
            }
    
            // add debris
            world.add( debris );
            // remove player
            world.removeBody( this );
            scratch.done();
            return self;
        }
    };

You might have noticed that we’re using something called `Physics.scratchpad`.
This is your best friend. A scratchpad is a helper to recycle temporary objects
(vectors) to reduce creation and garbage collection time. You can
[read more about scratchpads here][17].

So now we have a player, but it’s not connected to any user input. What we
need to do is create a*player behavior* to accommodate user input. So we create
another file called`player-behavior.js` in a very similar fashion (details in
comments
):

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
                    // the player will be passed in via the config options
                    // so we need to store the player
                    var player = self.player = options.player;
                    self.gameover = false;
    
                    // events
                    document.addEventListener('keydown', function( e ){
                        if (self.gameover){
                            return;
                        }
                        switch ( e.keyCode ){
                            case 38: // up
                                self.movePlayer();
                            break;
                            case 40: // down
                            break;
                            case 37: // left
                                player.turn( -1 );
                            break;
                            case 39: // right
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
                            case 38: // up
                                self.movePlayer( false );
                            break;
                            case 40: // down
                            break;
                            case 37: // left
                                player.turn( 0 );
                            break;
                            case 39: // right
                                player.turn( 0 );
                            break;
                            case 32: // space
                            break;
                        }
                        return false;
                    });
                },
    
                // this is automatically called by the world
                // when this behavior is added to the world
                connect: function( world ){
    
                    // we want to subscribe to world events
                    world.subscribe('collisions:detected', this.checkPlayerCollision, this);
                    world.subscribe('integrate:positions', this.behave, this);
                },
    
                // this is automatically called by the world
                // when this behavior is removed from the world
                disconnect: function( world ){
    
                    // we want to unsubscribe from world events
                    world.unsubscribe('collisions:detected', this.checkPlayerCollision);
                    world.unsubscribe('integrate:positions', this.behave);
                },
    
                // check to see if the player has collided
                checkPlayerCollision: function( data ){
    
                    var self = this
                        ,world = self._world
                        ,collisions = data.collisions
                        ,col
                        ,player = this.player
                        ;
    
                    for ( var i = 0, l = collisions.length; i < l; ++i ){
                        col = collisions[ i ];
    
                        // if we aren't looking at debris
                        // and one of these bodies is the player...
                        if ( col.bodyA.gameType !== 'debris' && 
                            col.bodyB.gameType !== 'debris' && 
                            (col.bodyA === player || col.bodyB === player) 
                        ){
                            player.blowUp();
                            world.removeBehavior( this );
                            this.gameover = true;
    
                            // when we crash, we'll publish an event to the world
                            // that we can listen for to prompt to restart the game
                            world.publish('lose-game');
                            return;
                        }
                    }
                },
    
                // toggle player motion
                movePlayer: function( active ){
    
                    if ( active === false ){
                        this.playerMove = false;
                        return;
                    }
                    this.playerMove = true;
                },
    
                behave: function( data ){
    
                    // activate thrusters if playerMove is true
                    this.player.thrust( this.playerMove ? 1 : 0 );
                }
            };
        });
    });

So now we can declare `js/player` and `js/player-behavior` as dependencies and
use them in our`main.js` file by adding this to our `init()` function:

    // in init()
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

The last thing we need to add before we see our second version is to have the
renderer pan to track the user’s motion. This can be achieved by adding some 
code to the`step` event listener to change the renderer offset right before it
calls`world.render()`. It looks like this:

    // inside init()...
    // render on every step
    world.subscribe('step', function(){
        // middle of canvas
        var middle = { 
            x: 0.5 * window.innerWidth, 
            y: 0.5 * window.innerHeight
        };
        // follow player
        renderer.options.offset.clone( middle ).vsub( ship.state.pos );
        world.render();
    });

Our second iteration is now looking a lot more like a game!

    [Check out this Pen!][18]

## Stage 2: Looking for Trouble {#stage2lookingfortrouble}

At the moment, it’s a pretty tame game. We need some adversaries. So let’s
create some!

We’re going to do almost the same thing we did for our player. We’re going
to create a new body (extending the circle), but we’re going to make things much
simpler. We only need the adversaries to blow up, and we will take a different 
approach and not create a behavior because the functionality is so minimal. Let’
s create a new file called`ufo.js` and just give our UFO’s a simplified 
`blowUp()` method:

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
    
                    // create debris
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

Now let’s create a few adversaries in the `init()` function in our `main.js`
file:

    // inside init()...
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

The math in there is just to randomize them in such a way that they orbit the
planet, but slowly creep towards it. But at the moment, we can’t blow them up, 
so let’s add some more code to the`init()` function to keep track of how many
we kill and emit a`win-game` event if we kill all of them. We’ll also listen
to the world’s`collisions:detected` event and if any collisions have a laser in
them, we’ll blow that thing up, if it supports it.

    // inside init()...
    // count number of ufos destroyed
    var killCount = 0;
    world.subscribe('blow-up', function( data ){
    
        killCount++;
        if ( killCount === ufos.length ){
            world.publish('win-game');
        }
    });
    
    // blow up anything that touches a laser pulse
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

Notice that we could have created a new behavior here to manage these UFO’s,
but there’s so little code that it’s not really necessary. I also wanted to show
how versatile PhysicsJS is for accommodating different coding styles; there are 
many ways to do things.

Okay let’s look at our third iteration of `main.js`:

    [Check out this Pen!][19]

Fantastic! We’re almost done. There’s just one more thing I want to show you…

## Stage 3: Finding our way {#stage3findingourway}

Space can be a bit disorienting. It’s hard to see your surroundings, and
sometimes you need a little help. To solve this, let’s create a radar minimap! 
So how do we do this?

We’ll have to get the positions of all the bodies and draw little dots in a
small canvas area at the top righthand corner. Although the renderer isn’t as 
full-featured as I would like at this point, there are some useful helper 
methods we have at our disposal. What we’ll do is bind to the`render` event
which is called after the renderer has finished rendering the bodies. Then we’ll
just add our minimap drawing to the frame using these helper methods. Here’s the
code:

    // inside init()...
    // draw minimap
    world.subscribe('render', function( data ){
        // radius of minimap
        var r = 100;
        // padding
        var shim = 15;
        // x,y of center
        var x = renderer.options.width - r - shim;
        var y = r + shim;
        // the ever-useful scratchpad to speed up vector math
        var scratch = Physics.scratchpad();
        var d = scratch.vector();
        var lightness;
    
        // draw the radar guides
        renderer.drawCircle(x, y, r, { strokeStyle: '#090', fillStyle: '#010' });
        renderer.drawCircle(x, y, r * 2 / 3, { strokeStyle: '#090' });
        renderer.drawCircle(x, y, r / 3, { strokeStyle: '#090' });
    
        for (var i = 0, l = data.bodies.length, b = data.bodies[ i ]; b = data.bodies[ i ]; i++){
    
            // get the displacement of the body from the ship and scale it
            d.clone( ship.state.pos ).vsub( b.state.pos ).mult( -0.05 );
            // color the dot based on how massive the body is
            lightness = Math.max(Math.min(Math.sqrt(b.mass*10)|0, 100), 10);
            // if it's inside the minimap radius
            if (d.norm() < r){
                // draw the dot
                renderer.drawCircle(x + d.get(0), y + d.get(1), 1, 'hsl(60, 100%, '+lightness+'%)');
            }
        }
    
        scratch.done();
    });

Okay, let’s add that to our `main.js` and look at the finished product!

    [Check out this Pen!][20]

## Wrapping up {#wrappingup}

Voila! That’s it! Thanks for bearing with me through that long tutorial.

Obviously the gameplay experience isn’t the best, and there’s more to do (
like minifying our scripts with the RequireJS build tool), but my goal was to 
give you an idea of the capabilities of PhysicsJS, and how useful it can be even
in it’s early stages. Hopefully this has given you enough to tinker around with.
Remember, if you have any questions, feel free to post in the comments, or on
[StackOverflow][6].

Thanks to [MillionthVector for the sprites][21]

**Remember** there’s a 
[PhysicsJS contest you should enter at ChallengePost! Check it out!][11]

 [1]: http://flippinawesome.org/authors/jasper-palfree
 [2]: http://wellcaffeinated.net/PhysicsJS/
 [3]: https://github.com/wellcaffeinated/PhysicsJS
 [4]: http://codepen.io/wellcaffeinated/full/7c1f72c5c9c8e2159ee97dc6540e0b72

 [5]: https://github.com/wellcaffeinated/PhysicsJS/releases/tag/physicsjs-v0.5.2-alpha
 [6]: http://stackoverflow.com/questions/tagged/physicsjs
 [7]: http://requirejs.org/

 [8]: https://github.com/wellcaffeinated/PhysicsJS/tree/master/examples/spaceship/images
 [9]: http://codepen.io/wellcaffeinated/pen/78984a62f88d8a9cb31b6768db6890ac
 [10]: https://github.com/wellcaffeinated/PhysicsJS/wiki/Renderers
 [11]: http://physicsjs.challengepost.com/
 [12]: http://en.wikipedia.org/wiki/Newton
 [13]: http://en.wikipedia.org/wiki/Sweep_and_prune
 [14]: http://en.wikipedia.org/wiki/GJK
 [15]: http://en.wikipedia.org/wiki/Spherical_cow

 [16]: https://github.com/wellcaffeinated/PhysicsJS/wiki/Fundamentals#extending-subtypes
 [17]: https://github.com/wellcaffeinated/PhysicsJS/wiki/Scratchpads
 [18]: http://codepen.io/wellcaffeinated/pen/7ef83de07c264d5f140b1c56027a9d29
 [19]: http://codepen.io/wellcaffeinated/pen/ef2772ad82abfa8f04a349081afb0f19
 [20]: http://codepen.io/wellcaffeinated/pen/7c1f72c5c9c8e2159ee97dc6540e0b72
 [21]: http://millionthvector.blogspot.ca/p/free-sprites.html