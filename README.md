Download Link: https://assignmentchef.com/product/solved-csci611-project-4-goldchase-two-tin-cans-and-a-string
<br>
This project is an enhancement of project 3, where we add support for sockets so players on other computers can connect to a running game.

<h1>Summary</h1>

A game instance may be running on either a server computer or a client computer. More than one game instance may be running on both the client and the server. There may only be one server computer but there may be multiple client computers.

The first game instance to start on a server computer will also start a daemon on that server. Likewise, the first game instance to start on a client computer will also start a daemon on the client computer. The daemons manage the communications between computers. It will also now be the responsibility of the the daemons to clean up/remove shared memory once all players have left the game.

<h1>Initial game play enhancements</h1>

Make three minor modifications to goldchase:

<ol>

 <li>Add an additional member to the gameBoard structure:</li>

</ol>

int daemonID;

<ol start="2">

 <li>As in the previous project, assign the game’s process ID to the gameBoard structure in shared memory.</li>

 <li>Check the contents of the daemonID variable you added above. If it is not equal to zero, then send the daemon process a SIGHUP</li>

 <li>When the game is exiting, after the game’s process ID has been set to zero and the player removed from the map, send the daemon process another SIGHUP In the previous implementation of Goldchase, the last player would unlink the shared memory and semaphore.  The daemon will now be responsible for removing the shared memory and semaphore.</li>

</ol>

<h1>Signal handlers in daemons</h1>

<u>SIGHUP<strong> (player has joined or left game)</strong></u>

<ol>

 <li>Socket write: <strong>Socket Player</strong></li>

 <li>After the socket write, check the value of the byte you just wrote to the socket. If the byte equals G_SOCKPLR (that is, all of the player bits are off), then there are no more players playing the game.  Close and unlink the shared memory and semaphore, the exit the program.</li>

</ol>

<u>SIGUSR1<strong> (map has changed)</strong></u>

<ol>

 <li>Create vector pairs</li>

</ol>

[pair.first=(short)offset, pair.second=(unsigned char)mapsquare]

of each map square which is different between local copy and shared memory.  Update local copy as necessary.

<ol start="2">

 <li>If length of vector is &gt; 0, socket write: <strong>Socket Map</strong></li>

</ol>

<u>SIGUSR2<strong> (a message queue message is waiting)</strong></u>

<ol>

 <li>Loop through each message queue. When a message is found in queue, socket write:</li>

</ol>

<strong>Socket Message </strong>protocol.

<h1>Socket communications protocol</h1>

<h2>Socket Message (a message is being sent to a player)</h2>

<ul>

 <li>1 byte: G_SOCKMSG or’d with G_PLR# of message recipient</li>

 <li>1 short: n, # of bytes in message</li>

 <li>n bytes: the message itself</li>

</ul>

<strong>Socket Player (a player is joining or leaving the game)</strong>

(1) 1 byte: G_SOCKPLR or’d with all active players G_PLR#

<h2>Socket Map (a map refresh is required)</h2>

<ul>

 <li>1 byte: 0</li>

 <li>1 short: n, # of changed map squares</li>

 <li>for 1 to n:</li>

</ul>

1 short: offset into map memory

1 byte: new byte value

<h2>Socket Client Initialize (the client daemon has connected)</h2>

<ul>

 <li>1 int: the number of map rows</li>

 <li>1 int: the number of map cols</li>

 <li>n bytes: (where n=rows X cols) the map</li>

</ul>

<h1>Daemon behaviors</h1>

<h2>Server start up</h2>

<ol>

 <li>Connect to shared memory &amp; map via mmap()</li>

 <li>Call getpid() to retrieve the daemon’s process id for the daemonID field in the gameBoard structure in shared memory.</li>

 <li>Allocate memory (rows * columns) and initialize it with the same values that are in the shared memory map. The server maintains a local copy of the map so that when it receives a SIGUSR1, it can compare the two maps to determine which bytes in the shared memory map were changed by a game process.</li>

 <li>Start trapping SIGHUP, SIGUSR1, and SIGUSR2</li>

 <li>Listen on socket for a client to connect</li>

</ol>

<h2>Server receives client connect</h2>

<ol>

 <li>Socket write: <strong>Socket Client Initialize </strong>(see Socket communications protocol)</li>

 <li>Socket write: <strong>Socket Player </strong>(see Socket communications protocol)</li>

 <li>Enter an infinite loop. The loop should block on the read() system call (actually, READ), trying to read a single byte. The contents of that byte will determine which communication is being started (note that all types of messages under <strong>Socket communications protocol</strong> begin with a single byte).</li>

</ol>

<h2>Client startup</h2>

<ol>

 <li>Read the rows and columns from the socket (see <strong>Socket Client Initialize</strong> protocol).</li>

 <li>Allocate space (rows * cols) for a local copy of map (not the gameBoard, just the map)</li>

 <li>Read <em>n</em> bytes (where <em>n</em>=rows*cols) from the socket into the local copy of map (see <strong>Socket Client Initialize</strong> protocol). The client maintains a local copy of the map so that when it receives a SIGUSR1, it can compare this local copy with the shared memory map to determine which bytes in the shared memory map were changed by a game process.</li>

 <li>Create a semaphore</li>

 <li>Create shared memory (shm_open, ftruncate) just as a new game instance would.</li>

 <li>Initialize shared memory from your local copy of map, and rows &amp; cols.</li>

 <li>Call getpid() to retrieve the daemon’s process id for the daemonID field in the gameBoard structure in shared memory.</li>

 <li>Start trapping SIGHUP, SIGUSR1, and SIGUSR2</li>

 <li>Read <strong>Socket Player</strong> from Socket (handle as per SIGHUP, step 2).</li>

 <li>Enter an infinite loop. The loop should block on the read() system call (actually, READ), trying to read a single byte. The contents of that byte will determine which communication is being started (note that all types of messages under <strong>Socket communications protocol</strong> begin with a single byte).</li>

</ol>

<strong>The three headings below correspond to the three possible bytes which may read from the socket as per the infinite loop #9 above</strong>

<h2>On socket read “Socket Player”</h2>

Loop through player bits. The player bits are contained in that single byte you read from the socket protocol: <em>Socket Player (a player is joining or leaving the game)</em>:

<ol>

 <li>If player bit is on and shared memory ID is zero, a player (from other computer) has joined:

  <ul>

   <li>Create and open player’s message queue for read (daemon will receive messages on behalf of player, forwarding them across socket)</li>

   <li>Set shared memory ID to pid of daemon</li>

  </ul></li>

 <li>If player bit is off and shared memory ID is not zero, remote player has quit:

  <ul>

   <li>Close and unlink player’s message queue</li>

   <li>Set shared memory ID to zero</li>

  </ul></li>

</ol>

<em>After</em> looping through the player bits, check the value of that single byte you read in.  If that single byte equals G_SOCKPLR (that is, all of the player bits are off), then no players are left in the game.  Close and unlink the shared memory.  Close and unlink the semaphore.  Then exit the program.

<h2>On socket read “Socket Message”</h2>

After reading this first byte, you need to process the remaining bytes from the socket protocol: <em>Socket Message (a message is being sent to a player)</em><strong>.  </strong>

<ol>

 <li>Determine message recipient</li>

 <li>Determine message</li>

 <li>Open, write, and close to message queue of recipient</li>

</ol>

<h2>On socket read “Socket Map” (A 0 which is map update)</h2>

After reading this first byte, you need to process the remaining bytes from the socket protocol:

<em>Socket Map (which means a map refresh is required)</em>

<ol>

 <li>Determine number of changed map squares</li>

 <li>Change local copy and shared memory copy of map</li>

 <li>Send SIGUSR1 to each local player</li>

</ol>

<h1>Final game play enhancements</h1>

Don’t integrate these additional enhancements until after you have all other aspects of the application working:

<ol>

 <li>If a hostname <em>is</em> provided on the command line, the Goldchase process is assumed to be a <strong>remote process</strong> which interacts with a <strong>client daemon</strong>.</li>

 <li>If a hostname is <em>not</em> provided on the command line, the Goldchase process is assumed to be a <strong>local process</strong> which interacts with a <strong>server daemon</strong>.</li>

 <li><em>After</em> a Goldchase <strong>local process</strong> has registered its process ID with shared memory, check the daemonID.

  <ol>

   <li>If the daemonID is not zero, send a SIGHUP signal to the daemon process (this was completed as part of the initial enhancements).</li>

   <li>If the daemonID is zero, fork and exec the server daemon.</li>

  </ol></li>

 <li><em>Before</em> a Goldchase <strong>remote process</strong> initializes (immediately after startup),

  <ol>

   <li>if the shm_open() command fails, then the <strong>client daemon</strong> isn’t running yet. Fork and exec the <strong>client daemon</strong>. Be sure to provide the client daemon with the process ID of the parent process (the game is the parent, the new client daemon is the child). Once the client daemon is initialized and ready, it should signal its parent that it may continue initializing The parent should proceed as in #2 below.</li>

   <li>If the shm_open() command succeeds, the the <strong>client daemon</strong> is already running.</li>

  </ol></li>

</ol>

After the remote process has registered its process ID with shared memory, send a SIGHUP signal to the <strong>client daemon</strong> (this was completed as part of the initial enhancements).

<ol start="5">

 <li>When a Goldchase <strong>local process</strong> or <strong>remote process</strong> quits, send a SIGHUP signal to the daemon process (this was completed as part of the initial enhancements).</li>

</ol>