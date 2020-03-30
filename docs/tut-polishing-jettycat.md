# Polishing the JettyCat

::: tip Hey,
This tutorial assumes that you have finished the tutorial [Making Games: Jetty Cat](tut-making-jettycat.html). You should complete it first.
:::

The game is complete mechanic-wise, but there is a lot of ways to improve it aesthetically and gameplay-wise! This section also highlights a number of new v1.3 features.

[[toc]]

## Transition between rooms

Ct.js has a module called `ct.transision`. It allows you to easily create nice transitions between levels. The idea is that you start a first half of a transition on button press or some other event, then switch to another room, and call the second half of a transition in its On Create code.

Enable the module `transition` in the Catmods tab. It signals that it depends on the `tween` catmod, so enable it as well.

Now, modify the `Button_Play` On Step code so that it shows a blue circled transition when clicked:

```js
if (ct.touch.collideUi(this)) {
    if (!this.pressed) {
        this.pressed = true;
        ct.transition.circleOut(1000, 0x446ADB)
        .then(() => {
            ct.rooms.switch('InGame');
        });
    }
}
```

`this.pressed` is our custom variable that remembers that a button was pressed. It will help us prevent occasional double clicking, that may have negative effects on game's logic.

The first argument in `ct.transition.circleOut(1000, 0x446ADB)` is the duration of the effect (1000 milliseconds = 1 second), and the second one is the color of the transition. It is like the hex color, but with `0x` instead of `#` in the beginning.

::: tip
There are much more methods and examples in the module's "Info" and "Reference" tabs.
:::

The transition itself is an asynchronous action! We use `.then(() => {…})` to switch to the next room right when the transition ends.

That was the first part of the transition. The second one will go to the `InGame` On Create code. Open the room, and put this line:

```
ct.transition.circleIn(500, 0x446ADB);
```

We can also show up our UI layers (the pause menu and the score screen) by making them transparent but slowly turning them opaque. We will use `ct.tween` there — that one catmod that is used by `ct.transition`.

Most entities in ct.js have the same parameters that allow you to tweak their look and feel. We've been using `this.scale.x` and `this.scale.y` to set a copy's scale, but we can also apply it to rooms, text labels, special effects, and so on. Besides scaling, there are parameters `this.rotation`, `this.alpha` and `this.tint` that rotate an object, set its opacity and color correspondingly.

We will change the property `this.alpha` through time. It is a number between 0 and 1. When set to 1 — its initial value —, a copy or a room will be fully opaque. When set to 0, it will be invisible. Any numbers in-between will bake an object partially transparent. The module `ct.tween` will help create a smooth transition of it.

So, to fade in a UI layer, we need to put this code in On Create of rooms `UI_OhNo` and `UI_Paused`:

```js
this.alpha = 0;

ct.tween.add({
    obj: this,
    fields: {
        alpha: 1
    },
    duration: 500
});
```

Firstly, we make a room fully transparent by setting its `alpha` to 0. Then, we call `ct.tween.add` to start a smooth transition. `obj` points to an object that should be animated, and `fields` lists all the properties and values we want to change. Finally, the `duration` key sets the length of the effect, in milliseconds.

We can fade out a UI layer, too. Let's gradually hide the pause menu when the player hits the "continue" button. Open the type `Button_Continue`, and modify its code:

```js
if (ct.touch.collideUi(this)) {
    if (!this.pressed) {
        this.pressed = true;
        ct.tween.add({
            obj: this.getRoom(),
            fields: {
                alpha: 0
            },
            duration: 1000
        })
        .then(() => {
            ct.pixiApp.ticker.speed = 1;
            ct.rooms.remove(this.getRoom());
        });
    }
}
```

We create a flag `this.pressed` to make sure that the code that runs the animation only once. Running it multiple times won't hurt, but keeps the debugger's log clean as `ct.tween` will warn about interrupted animations otherwise.

Then we start an animation for `this.getRoom()`, that will return the room `UI_Paused` that owns this button, and change its alpha value back to 0. After that, we can see that `ct.tween.add` creates an asynchronous event, and we remove the room and unpause the game inside the `.then(() => {…});` clause.

## Smoothly resuming the game after it has been paused

Though the "paused" menu fades out slowly, it is still hard for a player to catch up and prevent the cat from bumping into ground. To prevent that, we can use `ct.tween` to… animate time! `ct.pixiApp.ticker.speed = 1;` can be not just 0 and 1, but also anything in between, and even beyound 1. Large values will make the game run faster, while values close to 0 will slow the game. Thus, we can animate the value `ct.pixiApp.ticker.speed` to make the game transition from paused to fully runnning state.

Open the type `Button_Continue` again, and modify the script so it fires another `ct.tween.add` after it finishes the first one:

```js {12,13,14,15,16,17,18}
if (ct.touch.collideUi(this)) {
    if (!this.pressed) {
        this.pressed = true;
        ct.tween.add({
            obj: this.getRoom(),
            fields: {
                alpha: 0
            },
            duration: 1000
        })
        .then(() => {
            ct.tween.add({
                obj: ct.pixiApp.ticker,
                fields: {
                    speed: 1
                },
                duration: 1000
            });
            ct.rooms.remove(this.getRoom());
        });
    }
}
```

Now players can catch up with the game and save their cat from falling.

## Cat's jet smoke and star particles

From v1.3, ct.js allows you to visually design particle effects and play them in your game. And it's cool! Let's create two effects: one will be a jet smoke for the cat. The other will show a burst of smaller stars when you collect one.

### Making a star burst

TODO:

### Making a jet smoke

Open the "FX" tab at the top, and create a new particle emitter. Call it `Jet`.

As a start, press the button `Import default…` and load the texture called `Circle_08`. After that, feel free to tinker around the editor to make the effect you want. I made a jet of white bubbles of different size:

![A jet particle effect in ct.js](./images/tutJettyCat_Jet.gif)

Here are some hints:

* Change the background color in the top-right corner of the window to better see white bubbles;
* Start by changing the Direction tab » Starting direction fields so the particles flow downwards. A good range is between 90 and 110 degrees.
* The default texture's size will be way too big; tweak its scale in the graph under the folding section called "Scaling", so it is somewhere around `0.3`.
* Tweak the value Scaling » Minimum size to spawn particles of different size.
* Change the value Spawning » Time between bursts to change the density of a jet. Smaller values spawn larger amounts of particles.

To add the effect to the cat, open its type and put this code to the end of its On Create code:

```js
this.jet = ct.emitters.follow(this, 'Jet', {
    position: {
        x: -120,
        y: 100
    }
});
```

`ct.emitters.follow` tells to create a particle effect and make it follow a copy. It will look attached to the cat. The first argument is the copy we want to attach the effect to (`this` is our cat), the second one — the name of the effect (`'Jet'`), and an optional object of additional properties. `position` tells that the effect should be shifted; in our case, it is shifted 120 pixels to the left and 100 pixels downwards, so it is placed not in the middle of the cat but at the place where the jet smoke should go.

::: tip
Read [the docs for `ct.emitters`](ct.emitters.html) to learn more about other methods for creating effects and their options.
:::

The cat should now have a jet of smoke running from its jetpack. You may need to tweak the jet's particle size and its speed on the "FX" tab.

Let's add a bit of dynamics to this jet: we will spawn new particles only when the cat flies up.

TODO:

## Adding subtle animations to the cat and stars

## Adding a hint to start tapping

## Animating background in the main menu + parallax effect