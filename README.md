# Covert channels in the SOP

In computer security, a covert channel is a mechanism that allows an attacker to transfer information between objects or processes that were not supposed to be allowed to communicate. In the context of browsers, I specifically refer to the procedures that can be used to transmit information across windows on different domains. Below there are several examples of this technique, that in some case can arrrive to be exploited for leaking information via side channel attacks.

I have created some PoCs to send information across `window`s (iframes or tabs) with different `origin`. Online version can be found: http://vwzq.net/lab/covert/

In the online page you can find further details and a couple more of test cases that have been fixed. I also added a bonus covert-channel by using timing measurements.

Sources are divided into "senders" and "receivers", since each set should be in a different domain.

## Test cases

#### `window.name`
It was originally designed for setting targets for hyperlinks and forms, but it has been used in some frameworks for providing cross-domain communication. For that reason is not really "covert", but it serves as historical example.
This channel can be used in both ways, just sharing a window between sender and receiver, and redirecting the page for sending a message. By using `history.back()` and `history.forward()` the network requests can be avoided and the speed increased.

#### `location.hash`
Location hash or fragment identifier was thought to allow navigation towards a subresource in a document. The information in the URI fragment is not sent to the server, and after changing the page won't be reloaded.
This is used in one direction, though it can be duplicated in order to have a bidirectional channel.
The event onhashchange will be triggered every time a new message is received, bringing an asynchronous messaging system, reliable and fast.

#### `history.length`
`history.length` read-only property returns the number of elements in the session history, including the currently loaded page. It is maintained during navigation (and shared between iframes) and therefore shared across domains. By modifying this value and mapping it to a byte of information it is possible to create a communication channel.
I have implemented a communication protocol between frames as demonstration, but we could use it in other ways:

1. The sender emits one character increasing the `history.length` by `N`
2. The receiver is polling the object until it stops growing and reads the value `N`
3. The receiver resets the `history.length` (going back and pushing a new location)
4. The sender detects the reset and go to `1` to send the next character or finishes the transmision.

#### Iframe's scrollbar positiona
Modifying `location.hash` does not only trigger the `onhashchange` event, but also sets the focus of an element with an equal identifier and potentially moves the scrollbar to its position. In this way, we could stablish a channel by navigating to different elements of a cross-domain document.
The example is pretty simple and probably uninteresting, but this behaviour has been abused in the real world, in combination with a CSS scroll bar detection trick, in order to probe elements of a cross-domain page and, for example, detect logged users.
In summary, some browsers allow to set the scroll bar brackground via CSS, the trick consists on requesting an image from the attacker domain which will set a cookie. After that we can inspect the document.cookie to check if the scroll bar was requested.

#### `window.frames.length`
Similar to the history object, the window.frames is shared cross-domain and allows us to inspect its length (shortlink via `window.length`), as well as navigate across iframes. This channel creates N frames in a web page, allowing the receiver to read that value.
Apparently, the maximum number of iframes in Chrome is 1000. Firefox does not seem to limit it, but bigger numbers degredate the performance too much. In any case using a dictionary again with values arround 50 works fine enought.
The protocol uses an iframe as `referee` to detect when a message has been send/read:

1. Receiver loads and stores `sender.length` as `init` value
2. Sender waits the referee, creates `N` iframes and switch the `referee`
3. Receiver waits for `referee`, reads `sender.length` and gets `N` as `sender.length-init`
4. If not `EOF`, go to `2`

## Side channel example
As an example, the `history.length` test case can also be abused to leak the user's navigation history.

When navigating in the same tab, the `history` object keeps adding each page into its stack. By performing some redirection and comparing previos/posterior lengths of that object, it is possible to determine whether or not the last page in the history has been visited.

Let's suppose the following history state:

`   | null | full_url_1 | full_url_2 | full_url_3 | attacker.com | // history.length == 5`

Now, if we want to check which was the last visited url we need to open a new window referencing the current one (`opener`) and perform this algorithm with a dictionary of urls:

1. `N = opener.history.length` (save initial history length)
2. `opener.history.go(-1)` (set navigation pointer to `full_url_3`)
3. `opener.location = test_url`
4. `opener.location = ""` (come back to same SOP)
5. `N' = opener.history.length` (new history length)
6. compare `N` and `N'`

If `test_url` corresponds to the last item in the history, the new history state will be:

`   | null | full_url_1 | full_url_2 | full_url_3 | attacker.com | // history.length == 5`

Because when redirecting to a same URL no new entry is add to the history. Otherwise, our state will be:

`   | null | full_url_1 | full_url_2 | full_url_3 | test_url | attacker.com | // history.length == 6`

Now do `history.go(-2)` (or -3) and keep iterating over the length of the history object in order to extract all the visited pages.

## References
...
