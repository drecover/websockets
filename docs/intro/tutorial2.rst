Part 2 - Route & broadcast
==========================

.. currentmodule:: websockets

.. admonition:: This is the second part of the tutorial.

    * In the :doc:`first part <tutorial1>`, you created a server and connected
      one browser; you could play if you shared the same browser.
    * In this :doc:`second part <tutorial2>`, you'll connect a second browser;
      you can play from different browsers on a local network.
    * In the :doc:`third part <tutorial3>`, you'll deploy the game to the web;
      you can play from any browser connected to the Internet.

Share game state
----------------

In the first part of the tutorial, you opened a WebSocket connection from a
browser to a server and exchanged moves over the connection. The state of the
game was maintained in the connection handler coroutine.

Now you want to open two WebSocket connections from two separate browsers, one
for each player, and to have them interact with the same game. This requires
moving the state of the game where both connections can share it.

Since you're running a single server process, you can share state by storing
it in a global variable.

.. admonition:: What if you need to scale to multiple server processes?
    :class: hint

    You must design a way for the process that manages a given connection to
    be aware of events that must be sent over that connection. This is often
    achieved with a publish / subscribe mechanism.

How do you do this? When a player starts a game, you give it an identifier.
Then, they communicate the identifier to the other player. When they join the
game, you look it up with the identifier.

In addition to the game itself, you need to keep track of the WebSocket
connections of the two players. In the first part, we designed a set of
events sent from the server to the browser. They're identical for both
players, so you can store both connections together.

Let's sketch this in code.

At the module level, in ``app.py``::

    JOIN = {}

When the first player starts the game::

    import secrets

    async def handler(websocket):
        ...

        # Initialize a new game.
        game = Connect4()
        # Register ourselves to be notified of moves.
        connected = {websocket}

        # Generate an identifier for the game.
        join_key = secrets.token_urlsafe(12)
        # Store the game and the set of players in global state.
        JOIN[join_key] = game, connected

        ...

When the second player joins the game::

    async def handler(websocket):
        ...

        join_key = ...  # TODO

        # Locate the game we're joining.
        game, connected = JOIN[join_key]
        # Register ourselves to be notified of moves.
        connected.add(websocket)

        ...

Perhaps you noticed one big piece missing from the puzzle here. How can the
second player obtain and send ``join_key``? Let's design new events to carry
this information.

To start a game, the first player sends:

.. code-block:: js

    {type: "init"}

The server creates a game as shown above and sends:

.. code-block:: js

    {type: "init", join: "<join_key>"}

With this information, the user interface of the first player can create a
link to http://localhost:8000/?join=<join_key>. For the sake of simplicity,
we'll assume that the first player shares this link with the second player
outside of the application, for example via an instant messaging service.

To join the game, the second player sends:

.. code-block:: js

    {type: "init", join: "<join_key>"}

Start a game
------------

Let's start by implementing the initialization sequence for the first player.

In ``main.js``, define this function to send the ``"init"`` event when the
WebSocket connection is open:

.. code-block:: js

    function initGame(websocket) {
      websocket.addEventListener("open", () => {
        const event = {type: "init"};
        websocket.send(JSON.stringify(event));
      });
    }

Modify the initialization to call ``initGame()``:

.. literalinclude:: ../../example/tutorial/step2/main.js
    :language: js
    :start-at: window.addEventListener

In ``app.py``, define a new ``handler`` coroutine — you can keep the previous
one elsewhere to reuse it later:

.. code-block:: python

    import secrets

    JOIN = {}

    async def start(websocket):
        game = Connect4()
        connected = {websocket}

        join_key = secrets.token_urlsafe(12)
        JOIN[join_key] = game, connected

        event = {
            "type": "init",
            "join": join_key,
        }
        await websocket.send(json.dumps(event))

        print("started game")

    async def handler(websocket):
        # Receive and parse the "init" event from the UI.
        message = await websocket.recv()
        event = json.loads(message)
        assert event["type"] == "init"

        await start(websocket)

In ``index.html``, add an ``<a>`` element to receive the link to share with
the other player:

.. code-block:: html

    <body>
        <div class="actions">
            <a class="action new" href="/">New game</a>
            <a class="action join" href="">Join</a>
        </div>
        <!-- ... -->
    </body>

In ``main.js``, modify ``receiveMoves()`` to handle the ``"init"`` message and
set the target of that link:

.. code-block:: js

        switch (event.type) {
          case "init":
            document.querySelector(".join").href = "?join=" + event.join;
            break;
          // ...
        }

Restart the WebSocket server and reload http://localhost:8000/ in the browser.
There's a link labeled JOIN below the board with a target that looks like:
http://localhost:8000/?join=95ftAaU5DJVP1zvb

Join a game
-----------

Let's update the initialization sequence to account for the second player.

In ``main.js``, update ``initGame()`` to send the join key in the ``"init"``
message when it's present in the URL:

.. code-block:: js
    function initGame(websocket) {
      websocket.addEventListener("open", () => {
        const params = new URLSearchParams(window.location.search);
        let event = {type: "init"};
        if (params.has("join")) {
            event["join"] = params.get("join");
        }
        websocket.send(JSON.stringify(event));
      });
    }

In ``app.py``, update the ``handler`` coroutine to look for the join key in
the ``"init"`` message:

    async def error(websocket, message):
        event = {
            "type": "error",
            "message": message,
        }
        await websocket.send(json.dumps(event))


    async def join(websocket, join_key):
        try:
            game, connected = JOIN[join_key]
        except KeyError:
            await error(websocket, "Game not found.")
            return

        print("joined game")


    async def handler(websocket):
        # Receive and parse the "init" event from the UI.
        message = await websocket.recv()
        event = json.loads(message)
        assert event["type"] == "init"

        if "join" in event:
            await join(websocket, event["join"])
        else:
            await start(websocket)

Restart the WebSocket server and reload http://localhost:8000/ in the browser.
Copy the link labeled JOIN and open it in another browser. You see ``joined
game`` in the server logs.

At this point, the ``start()`` and ``join()`` coroutines both have a reference
to ``game``, a :class:`~connect4.Connect4` instance, and ``connected``, a set
containing the two WebSocket connections.

Let's restore the logic for playing moves and we should have a fully
functional two-player game.

Reconnect the game logic
------------------------

Modify the ``start()`` and ``join()`` coroutines to invoke a
``play()`` coroutine once the initialization is done::

    async def play(game, player, connected):
        ...

    async def start(websocket):
        game = Connect4()
        connected = {websocket}

        join_key = secrets.token_urlsafe(12)
        JOIN[join_key] = game, connected

        try:
            event = {
                "type": "init",
                "join": join_key,
            }
            await websocket.send(json.dumps(event))
            await play(game, PLAYER1, connected)
        finally:
            del JOIN[join_key]

    async def join(websocket, join_key):
        try:
            game, connected = JOIN[join_key]
        except KeyError:
            await error(websocket, "Game not found.")
            return

        connected.add(websocket)
        try:
            await play(game, PLAYER2, connected)
        finally:
            connected.remove(websocket)

In this version, we're carefully using ``try: ... except: ...`` blocks to
remove objects from the global state once they're stale. Else, we would leak
memory.

Now you can recover the code from part 1 of the tutorial and implement the
``play()`` coroutine. Try to write it by yourself!

Keep in mind that restarting the WebSocket server resets the ``JOIN``
variable. When you make changes, you must reload http://localhost:8000/ in
the first browser, copy the JOIN link, and open it in another tab or
another browser.

Once ``play()`` works, you can play the game from two separate browsers,
possibly running on separate computers on the same local network.

Here's a solution.

::

    async def play(websocket, game, player, connected):
        while True:
            # Receive and parse the next "play" event from the UI.
            message = await websocket.recv()
            event = json.loads(message)
            assert event["type"] == "play"
            column = event["column"]

            # Play the move and send an "error" event if it's invalid.
            try:
                row = game.play(player, column)
            except RuntimeError as exc:
                await error(websocket, str(exc))
                continue

            # Send a "play" event to update the UI.
            event = {
                "type": "play",
                "player": player,
                "column": column,
                "row": row,
            }
            for connection in connected:
                await connection.send(json.dumps(event))

            # If move is winning, send a "win" event and exit.
            if game.winner is not None:
                event = {
                    "type": "win",
                    "player": game.winner,
                }
                for connection in connected:
                    await connection.send(json.dumps(event))
                break

Broadcast the game
------------------

Let's add a feature allowing spectators to watch the game.

Duplicate all the logic for joining a game to support watching a game.

Then, watching a game is as simple as registering the WebSocket connection in
the ``connected`` set::

    async def watch(websocket, watch_key):
        try:
            game, connected = WATCH[watch_key]
        except KeyError:
            await error(websocket, "Game not found.")
            return

        connected.add(websocket)
        try:
            await websocket.wait_closed()
        finally:
            connected.remove(websocket)

``watch()`` exits when the WebSocket connection is closed, either because the
browser closed it or because the network failed.

Here's a complete solution.

**app.py**

.. literalinclude:: ../../example/tutorial/step2/app.py

**index.html**

.. literalinclude:: ../../example/tutorial/step2/index.html

**main.js**

.. literalinclude:: ../../example/tutorial/step2/main.js
