In this file I will document my findings as I look into the details of CrossCode and the modding thereof.

# The Engine

[According to its developers](https://steamcommunity.com/app/368340/discussions/0/371918937280889458/) CrossCode is written using a heavily customized version of [Impact Engine](https://impactjs.com/), which is not known for robust modding support (even if it was, the customized version almost certainly would not have the same kind of support). All the code is written in vanilla JavaScript.

# The Directory Structure

Here's a basic mockup of the structure of the game's files:

```
steamapps/common/CrossCode
+ assets
| + js
|   + game.minified.js
+ CrossCode
```

The Steam or Itch download includes a basic launcher, which I have not introspected (because it is compiled), but presumably it instantiates an HTML canvas for the JavaScript to render to. The game's code is included in the assets/js directory as a 3.7MB minified JS file (when pretty-printed, it is 184826 lines long) and, seems not to be obfuscated at all.

# Networking

Slapping on a socket to communicate with the server should be trivial. [archipelago.js](https://github.com/ThePhar/archipelago.js) will be the interface, we just need to hook CrossCode up to that. (Question: can we distribute this without requiring everyone to install NPM?)

# Code Structure

I've begun to dig through the source code (`game.minified.js`) but pretty-printed. I used `prettier` with no additional configuration; line numbers will be based on that. With no prior knowledge of ImpactJS, here are my reads:

Impact works through the use of modules (not JS modules or node modules; this is an ImpactJS-specific concept). They function about as you'd expect. They contain a list of modules they depend on, a list of defined functions. Modules are pretty deeply namespaced. If you need to get a list of modules, run the following on the prettified code:

```sh
$ grep ig.module\( crosscode-maxified.js | perl -ne '/"(.+)"/; print "$1\n"'
```

An abridged list of modules follows:

```
impact.base.worker
impact.base.loader
impact.base.image
impact.base.font
impact.base.system
impact.base.system.web-audio
...
impact.feature.base.action-steps
impact.feature.base.event-steps
impact.feature.base.entities.marker
impact.feature.base.entities.object-layer-view
impact.feature.base.entities.touch-trigger
impact.feature.base.entities.sound-entities
...
game.feature.version.version
game.feature.control.control
game.feature.combat.stat-change
game.feature.font.font-system
game.feature.gui.base.text
game.feature.interact.button-group
game.feature.gui.base.button
game.feature.gui.base.boxes
game.feature.gui.base.numbers
game.feature.menu.gui.menu-misc
game.feature.version.gui.changelog-gui
game.feature.version.gui.dlc-gui
game.feature.version.plug-in
...
game.beta
game.main
```

Rather conveniently, this allows us to easily discriminate between engine code and game code (the top-level namespace will be either `impact` or `game`); not only that, but it also makes finding the relevant part to modify pretty easy. Need to change something about the "tesla coil" puzzle element? Look in `game.feature.puzzle.entities.tesla-coil`. Need to edit the map GUI? Look in the `game.feature.menu.gui.map.*` modules. Need to add your own GUI (as we probably will want to)? Add a new namespace under `game.feature.menu.gui`. The fact that code is organized like this gives me a lot of hope.

Also, modules seem to be lazily evaluated. All the code that they provide is stored in a function that is presumably only called when a module that depends on it tries to load it, meaning we should be able to insert our own modules wherever we want as long as it's between the definition of the `ig` object and the main function.
