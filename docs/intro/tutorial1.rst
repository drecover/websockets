Part 1 - Send & receive
=======================

.. currentmodule:: websockets

In this tutorial, you're going to build a web-based `Connect Four`_ game.

.. _Connect Four: https://en.wikipedia.org/wiki/Connect_Four

The web removes the constraint of being in the same room for playing a game.
Two players can connect over of the Internet, regardless of where they are,
and play in their browsers.

When a player makes a move, it should be reflected immediately on both sides.
This is difficult to implement in a traditional web application due to the
request-response style of the HTTP protocol.

Indeed, there is no good way to be notified when the other player makes a
move. Workarounds such as polling or long-polling introduce significant
overhead.

Enter `WebSocket <websocket>`_.

The WebSocket protocol provides two-way communication between a browser and a
server over a persistent connection. That's exactly what you need to exchange
moves between players.

.. admonition:: This is the first part of the tutorial.

    * In this :doc:`first part <tutorial1>`, you'll create a server and
      connect one browser; you can play if you share the same browser.
    * In the :doc:`second part <tutorial2>`, you'll connect a second browser;
      you can play from different browsers on a local network.
    * In the :doc:`third part <tutorial3>`, you'll deploy the game to the web;
      you can play from any browser connected to the Internet.

Prerequisites
-------------

This tutorial assumes basic knowledge of Python and JavaScript.

If you're comfortable with :doc:`virtual environments <python:tutorial/venv>`,
you can use one for this tutorial. Else, don't worry: websockets doesn't have
any dependencies; it shouldn't create trouble in the default environment.

If you haven't installed websockets yet, do it now:

.. code:: console

    $ pip install websockets

Confirm that websockets is installed:

.. code:: console

    $ python -m websockets --version

.. admonition:: This tutorial is written for websockets |version|.
    :class: tip

    If you installed another version, you should switch to the corresponding
    version of the documentation.

Download the starter kit
------------------------

Create a directory and download these three files:
:download:`connect4.css <../../example/tutorial/start/connect4.css>`,
:download:`connect4.js <../../example/tutorial/start/connect4.js>`,
and :download:`connect4.py <../../example/tutorial/start/connect4.py>`.

The JavaScript module, together with the CSS file, provides a web-based user
interface.

.. js:module:: connect4

.. js:data:: PLAYER1

    Color of the first player.

.. js:data:: PLAYER2

    Color of the second player.

.. js:function:: createBoard(board)

    Draw a board.

    :param board: DOM element containing the board; must be initially empty.

.. js:function:: playMove(board, player, column, row)

    Play a move.

    :param board: DOM element containing the board.
    :param player: :js:data:`PLAYER1` or :js:data:`PLAYER2`.
    :param column: between ``0`` and ``6``.
    :param row: between ``0`` and ``5``.

The Python module provides a class to record moves and tell when a player
wins.

.. module:: connect4

.. data:: PLAYER1
    :value: "red"

    Color of the first player.

.. data:: PLAYER2
    :value: "yellow"

    Color of the second player.

.. class:: Connect4

    .. method:: play(player, column)

        Play a move.

        :param player: :data:`~connect4.PLAYER1` or :data:`~connect4.PLAYER2`.
        :param column: between ``0`` and ``6``.
        :returns: Row where the checker lands, between ``0`` and ``5``.
        :raises RuntimeError: if the move is illegal.

    .. attribute:: moves

        List of moves played during this game, as ``(player, column, row)``
        tuples.

    .. attribute:: winner

        :data:`~connect4.PLAYER1` or :data:`~connect4.PLAYER2` if they
        won; :obj:`None` if the game is still ongoing.

.. currentmodule:: websockets

Bootstrap the web UI
--------------------

Create an ``index.html`` file next to ``connect4.css`` and ``connect4.js``
with this content:

.. literalinclude:: ../../example/tutorial/start/index.html
    :language: html

This HTML page contains an empty ``<div>`` element where you will draw the
Connect Four board. It loads a ``main.js`` script where you will write all
your JavaScript code.

Create a ``main.js`` script that calls :js:func:`~connect4.createBoard`:

.. literalinclude:: ../../example/tutorial/start/main.js
    :language: js

Open a shell, navigate to the directory containing these files, and start a
HTTP server:

.. code-block:: console

    $ python -m http.server

Open http://localhost:8000/ in a web browser. The page displays an empty board
with seven columns and six rows. You'll play move in this board later.

Bootstrap the server
--------------------

Create an ``app.py`` file next to ``connect4.py`` with this content:

.. literalinclude:: ../../example/tutorial/start/app.py

The entrypoint of this program is ``asyncio.run(main())``. It creates an
asyncio event loop, runs the ``main()`` coroutine, and shuts down the loop.

The ``main()`` coroutine calls :func:`~server.serve` to start a websockets
server. :func:`~server.serve` takes three positional arguments:

* ``handler`` is a coroutine that manages a connection. When a client
  connects, websockets calls ``handler`` with the connection in argument.
  When ``handler`` terminates, websockets closes the connection.
* The second argument defines the network interfaces where the server can be
  reached. Here, the server listens on all interfaces, so that other devices
  on the same local network can connect.
* The third argument is the port on which the server listens.

Invoking :func:`~server.serve` as an asynchronous context manager, in an
``async with`` block, ensures that the server shuts down properly when
exiting the program.

The ``handler()`` coroutine runs an infinite loop that receives messages from
the browser and prints them.

Open a shell, navigate to the directory containing these files, and start the
server:

.. code-block:: console

    $ python app.py

Hopefully the WebSocket server is running as expected. You cannot test it with
a web browser like you tested the HTTP server. However, you can test it with
websockets' interactive client.

Open another shell and run this command:

.. code-block:: console

    $ python -m websockets ws://localhost:8001/

You get a prompt. Type a message and press "Enter". Then check that the server
received and printed the message. Good!

Exit the interactive client with Ctrl-C or Ctrl-D.

At this point, you have a web-based interface and a WebSocket server. Let's
connect them.

Message from browser to server
------------------------------

In JavaScript, you can open a WebSocket connection as follows:

.. code-block:: js

    const websocket = new WebSocket("ws://localhost:8001/");

Before you exchange messages with the server, you need to decide their format.
There is no universal convention for this.

Let's use JSON objects with a ``type`` key identifying the type of the event
and the rest of the object containing properties of the event.

Here's an event describing a move in the middle slot of the board:

.. code-block:: js

    const event = {type: "play", column: 3};

Here's how to serialize this event to JSON and send it to the server:

.. code-block:: js

    websocket.send(JSON.stringify(event));

Now you have all the building blocks to send moves to the server.

Add this function to ``main.js``:

.. literalinclude:: ../../example/tutorial/step1/main.js
    :language: js
    :start-at: function sendMoves
    :end-before: window.addEventListener

``sendMoves()`` registers a listener for ``click`` events on the board. The
listener figures out which column was clicked, builds a  event of type
``"play"``, serializes it, and sends it to the server.

Modify the initialization to open the WebSocket connection and call the
``sendMoves()`` function:

.. code-block:: js

    window.addEventListener("DOMContentLoaded", () => {
      const board = document.querySelector(".board");
      createBoard(board);
      const websocket = new WebSocket("ws://localhost:8001/");
      sendMoves(board, websocket);
    });

Ensure the HTTP server and the WebSocket are still running. If you stopped
them, here are the commands to start them again:

.. code-block:: console

    $ python -m http.server

.. code-block:: console

    $ python app.py

Refresh http://localhost:8000/ in your web browser. Make the window running
the WebSocket server visible. Click various positions in the board. You can
see the server receiving messages with the expected column number.

Of course, there isn't any visual feedback in the board, because we haven't
implemented that yet. Let's do it now.

Message from server to browser
------------------------------

In JavaScript, you receive WebSocket messages by listening to ``message``
events. Here's how to receive a message from the server and deserialize it
from JSON:

.. code-block:: js

      websocket.addEventListener("message", ({ data }) => {
        const event = JSON.parse(data);
        // do something with event
      });

You're going to need three types of messages from the server to the browser:

.. code-block:: js

    {type: "play", player: "red", column: 3, row: 0}
    {type: "win", player: "red"}
    {type: "error", message: "This slot is full."}

The JavaScript code receiving these messages must dispatch events depending on
their type and take appropriate action. For example, it reacts to an event of
type ``"play"`` by displaying the move on the board with
the :js:func:`~connect4.playMove` function.

Add this function to ``main.js``:

.. literalinclude:: ../../example/tutorial/step1/main.js
    :language: js
    :start-at: function receiveMoves
    :end-before: function sendMoves

.. admonition:: Why ``window.setTimeout(() => window.alert(...), 50);``?
    :class: hint

    When :js:func:`playMove` modifies the state of the board, the browser
    renders changes asynchronously. Conversely, ``window.alert()`` runs
    synchronously and blocks rendering while the alert is visible.

    If you called ``window.alert()`` immediately after :js:func:`playMove`,
    the browser could display the alert before rendering the move. You could
    get a "Player red wins!" alert without seeing red's last move.

    We're using ``window.alert()`` for simplicity in this tutorial. A real
    application would display these messages in the user interface instead.

Modify the initialization to call the ``receiveMoves()`` function:

.. literalinclude:: ../../example/tutorial/step1/main.js
    :language: js
    :start-at: window.addEventListener

At this point, the user interface should receive events properly.

Let's test it by modifying the server to send some events.

Sending an event from Python is quite similar to JavaScript::

    event = {"type": "play", "player": "red", "column": 3, "row": 0}
    await websocket.send(json.dumps(event))

.. admonition:: Don't forget to serialize the event with :func:`json.dumps`.
    :class: tip

    Else, websockets raises ``TypeError: data is a dict-like object``.

Modify the ``handler()`` coroutine in ``app.js`` as follows::

    import json

    from connect4 import PLAYER1, PLAYER2

    async def handler(websocket):
        for player, column, row in [
            (PLAYER1, 3, 0),
            (PLAYER2, 3, 1),
            (PLAYER1, 4, 0),
            (PLAYER2, 4, 1),
            (PLAYER1, 2, 0),
            (PLAYER2, 1, 0),
            (PLAYER1, 5, 0),
        ]:
            event = {
                "type": "play",
                "player": player,
                "column": column,
                "row": row,
            }
            await websocket.send(json.dumps(event))
            await asyncio.sleep(0.5)
        event = {
            "type": "win",
            "player": PLAYER1,
        }
        await websocket.send(json.dumps(event))

Find the shell where the WebSocket server is running, stop it with Ctrl-C, and restart it.

.. admonition:: You must restart the WebSocket server when you make changes.
    :class: tip

    For each HTTP request, the HTTP server reads the corresponding file and
    sends it to the browser. This is why you can simply refresh the browser
    and get the latest version after making changes to ``main.js``.

    Conversely, the WebSocket server loads the Python code and then serves
    every WebSocket request with this version of the code. This is why your
    changes to ``app.py`` aren't visible until you restart the server.

    It is possible to :doc:`restart the WebSocket server automatically
    <../howto/autoreload>` but this isn't necessary for this tutorial.

Refresh http://localhost:8000/ in your web browser. Seven moves appear at 0.5
second intervals. Then an alert announces the winner.

Good! Now you know how to communicate both ways.

Let's plug the game engine to process moves and we should have a fully
functional game.

Add the game logic
------------------

In the connection handler, you're going to initialize a game::

    from connect4 import Connect4

    game = Connect4()

Then, you're going to take the following steps in a loop:

* receive an event of type ``"play"``, the only type of event that the user
  interface sends;
* play that move in the board;
* if the move is illegal, send an event of type ``"error"``;
* else, send an event of type ``"play"`` to tell the user interface where the
  checker landed;
* if the move won the game, send an event of type ``"win"`` and exit the loop.

Try to implement this by yourself!

Keep in mind that you must restart the WebSocket server and reload the page in
the browser when you make changes.

.. admonition:: Enable debug logs to see all messages sent and received.
    :class: tip

    Here's how to enable debug logs::

        import logging

        logging.basicConfig(format="%(message)s", level=logging.DEBUG)

Here's a complete solution.

.. literalinclude:: ../../example/tutorial/step1/app.py

Recap
-----

In this first part of the tutorial, you learnt how to:

* build and run a WebSocket server in Python with :func:`~server.serve`;
* receive messages in a connection handler
  with :meth:`~server.WebSocketServerProtocol.recv`;
* send messages in a connection handler
  with :meth:`~server.WebSocketServerProtocol.send`;
* open a WebSocket connection in JavaScript with the ``WebSocket`` API;
* send messages with ``WebSocket.send()``;
* receive messages by listening to ``message`` events;
* design a set of events to be exchanged between the browser and the server.

You can now play a Connect Four game!

However, the two players share a browser, so the constraint of being in the
same room still applies.

Move on to the :doc:`second part <tutorial2>` of the tutorial to break this
constraint and play from separate browsers.
