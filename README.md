# js8net
### _Jeff Francis <jeff@gritch.org>_

js8net is a python3 package for interacting with the JS8Call API. It works exclusively in TCP mode.

The JS8Call API is extremely painful to use directly, for several reasons. First, it's essentially undocumented. While there are some partial docs floating around, mostly, you have to read the JS8Call source code to find the calls and understand how to use them. Second, the API is completely asynchronous. You send commands to JS8Call whenever you wish, and it sends whatever data it has ready at any given time, which may or may not include a reply to a query or command you've sent, whenever it darned well pleases. As a result, and API library has to keep track of what queries were sent, then attempt to match up pseudo-random replies with the original queries.

This library is an attempt to hide that complexity behind a much more traditional query/reply paradigm.

Before you start, note that there are a series of bugs in JS8Call where the GUI does not properly reflect changes made to the state or configuration of the system. For example, if you change the grid square using the API, the GUI will still show the old grid square. Or if you change your transmission speed using the API, the GUI will still show the old transmission speed. Everything will work just fine, but it will look very strange on screen. There are bugs open against JS8Call to fix this. Jordan has also repeatedly promised a new, better API at some point in the future.

While JS8Call by and large does work well, it's been 18+ months since the last release, and there are numerous anticipated bug fixes supposedly in the works that should make JS8Call a much better piece of software.

To get started, import the library, then tell it to connect to your JS8Call instance:

````python3
from js8net import *
start_net("10.1.1.141",2442)
````

At this point, there are two threads running. One thread receives requests from your code, and delivers them to JS8Call. The second thread receives the random data sent my JS8Call, processes that data, and provides it back to you, the user.

Generally speaking, you do not interact directly with the first queue. You make function calls using the js8net library, and those function calls are internally translated to the proper queries, and pushed into the queue for delivery. For example, to ask JS8Call for the currently configured Maidenhead grid square, you can simple do the following:

````python3
grid=get_grid()
````

Behind the scenes, the library creates the proper JSON for the query, delivers it to JS8Call, then watches the stream of traffic returning from JS8Call until it finds the grid information, then returns it as a result of the function call.

There are approximately a dozen or so of these function calls, listed below. The majority of these calls return a single value, however the one exception is the call to return or set the radio frequency. Because this function returns three values (radio dial frequency, the offset within the audio passband, and the effective transmit frequency), that one single call actually returns a JSON blob, rather than a single value. It's up to you to extract the values you need from the returned JSON:

````python3
>>> get_freq()
{'dial': 7078000, 'freq': 7080000, 'offset': 2000}
>>>
````

````python3
get_freq()
````
Ask JS8Call to get the radio's frequency. Returns the dial
frequency (in hz), the offset in the audio passband (in hz), and the actual effective transmit frequency (basically the two values added together) as a JSON blob.

````python3
set_freq(dial,offset)
````
Set the radio's dial freq (in hz) and the offset within the
passband (also in hz).

````python3
get_callsign()
````
Ask JS8Call for the configured callsign.

````python3
get_grid()
````
Ask JS8Call for the configured grid square.

````python3
set_grid(grid)
````
Set the grid square.

````python3
send_aprs_grid(grid)
````
Send the supplied grid info to APRS (use
send_aprs_grid(get_grid()) to send your configured

````python3
send_sms(phone,message)
````
Send an SMS message via JS8.

````python3
send_email(address,message)
````
Send an email message via JS8.

````python3
get_info()
````
Ask JS8Call for the configured info field.

````python3
set_info(info)
````
Set the info field.

````python3
get_call_activity()
````
Get the contents of the right white box.

````python3
get_call_selected()
````
Never quite figured out what this does...

````python3
get_band_activity()
````
Get the contents of the left white box.

````python3
get_rx_text()
````
Get the contents of the yellow window.

````python3
get_tx_text()
````
Get the contents of the box below yellow window.

````python3
get_speed()
````
Ask JS8Call what speed it's currently configured for.
slow==4, normal==0, fast==1, turbo==2

````python3
set_speed(speed)
````
Set the JS8Call transmission speed.
slow==4, normal==0, fast==1, turbo==2

````python3
raise_window()
````
Raise the JS8Call window to the top.

````python3
send_message(message)
````
Send 'message' in the next transmit cycle.

Sending of data, querying of status, and setting configuration are handled by the function calls above. Receiving data, however, is handled more directly by your own code.

Incoming messages from JS8Call are intercepted and parsed by the js8net library. The bulk of these are quietly handled, and various internal tables and states are automatically updated. Actual text sent by other users, however, are passed along to the rx_queue for your own processing. Note that the rx_queue is protected by a mutex called rx_lock. Use of this lock is necessary to prevent simultaneous reading and writing to the queue.

There are three types of messages that will come in at random from JS8Call, and three more types that will occur as the result of queries you make with the functions detailed above. The three types of messages that come from other JS8Call users are:

* RX.SPOT - A spot message.
* RX.ACTIVITY - Received data (typically, a single, incomplete frame; a fragment of a larger message).
* RX.DIRECTED - A complete, reassembled message with each of the available frames properly concatenated together into a single string.

Unless you are doing something particularly interesting or different, it's likely that RX.DIRECTED is what you'll be interested in.

These messages are kept in a python queue. The following documention will be helpful in understanding how to properly deal with queued data:

https://docs.python.org/3/library/queue.html

In the simplest case, pulling an entry from the queue will look something like this:

````python3
>>> with rx_lock:
      rx_queue.get()
{'params': {'DIAL': 7078000, 'FREQ': 7080748, 'OFFSET': 2748, 'SNR': 4, 'SPEED': 1, 'TDRIFT': 3.59999990\
46325684, 'UTC': 1637172368463, '_ID': -1}, 'type': 'RX.ACTIVITY', 'value': 'ZDXB/R/U00 RP72 ', 'time': \
1637172368.9904108}                                         
>>>
````

As this is a simple python dictionary, you can check the type of this entry, then extract the text as follows:

````python3
>>> with rx_lock:
      message=rx_queue.get()
>>> message['type']
RX.ACTIVITY
>>> message['value']
ZDXB/R/U00 RP72
>>>
````

You can combine this into a loop with something like the following:

````python3
while(True):
    if(not(rx_queue.empty())):
        with rx_lock:
            message=rx_queue.get()
            if(message['type']=="RX.DIRECTED"):
                print(message)
        time.sleep(0.1)
````

You can, of course, do far more than simply print the received JSON blob.

The three additional types of messages that will show up in the queue are:

* RX.CALL_ACTIVITY - The result of the function call get_call_activity()
* RX.GET_BAND_ACTIVITY - The result of the function call get_band_activity()
* RX.TEXT - The result of the function call get_rx_text()

See above for documentation on what these calls do.
