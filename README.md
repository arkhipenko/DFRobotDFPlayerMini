# DFPlayer - A Mini MP3 Player For Arduino


DFPlayer - A Mini MP3 Player For Arduino
https://www.dfrobot.com/index.php?route=product/product&product_id=1121

This example shows the all the function of library for DFPlayer.

Created 2016-12-07
By [Angelo qiao](Angelo.qiao@dfrobot.com)

GNU Lesser General Public License.
See <http://www.gnu.org/licenses/> for details.
All above must be included in any redistribution

Notice and Trouble shooting

1.Connection and Diagram can be found here
https://www.dfrobot.com/wiki/index.php/DFPlayer_Mini_SKU:DFR0299#Connection_Diagram
2.This code is tested on Arduino Uno, Leonardo, Mega boards.

## Updates by Anatoli Arkhipenko
Created an asynchronous version of this library to avoid use of blocking `delay` methods 
in the original library, and make it friendly for cooperative multitasking frameworks, 
like TaskScheduler.
Even a 10 ms delay could be problematic for processes that require polling of data at 500Hz
for instance (IMUs).

The way the MP3 board works is every method ends up sending a 10 bytes command via 9600 serial pipe. 
Depending on a microcontroller, this can take up to 12-15 milliseconds. The original library also adds
explicit delays to wait for serial operations completion, which are blocking and are strongly discouraged
in a cooperative multitasking environment. 

The new approach is to utilize `availableForWrite()` method available in some of the `Serial` implementations
to guarantee a very fast return from the `DFPlayerMini` methods involving serail communications. 

Definition of `availableForWrite` is: Get the number of bytes (characters) available for writing in the serial 
buffer without blocking the write operation.

Any time a `DFPlayerMini` method is called, it returns quickly having written only as many bytes as the target
platform can transmit without delay. (E.g., Teensy 3.5 board has a 8 byte long FIFO buffer on `Serial1` and `Serial2` 
interfaces). The implementation should call `bool DFPlayerMini::commandCompleted()` periodically (once every 1 to 10 ms)
until it returns `true`. If `commandCompleted()` returns `false`, there is still unsent data in the buffer.

For implementations using `TaskScheduler` cooperatinve multitasking, an apprach would be to call appropriate player
method, and then restart a periodic `Task` that calls `commandCompleted()` until that returns `true`.

If delay is not an issue, OR you need to play a sound in the `setup()` method when scheduling is not yet active, 
a new method `void DFPlayerMini::completeCommand()` could be used to finish transmission synchronously. 

#### Examples:
##### Player setup:
```
void setupPlayer() {
  Serial.begin(9600);
  delay(100);
  while (!Serial) ;;

  player.begin(Serial, false, true);  // No ACK, reset

  player.setTimeOut(500);
  player.volume(volume);  			// Set volume value (0~30).
  player.completeCommand();         // Complete synchronously 

  //----Set different EQ----
  player.EQ(DFPLAYER_EQ_NORMAL);

  //----Set device we use SD as default----
  player.outputDevice(DFPLAYER_DEVICE_SD);
  player.completeCommand();
}
```

##### Asyncronous control with TaskScheduler
```
// Task definition:
Task  tPlayerIterator(10 * TASK_MILLISECOND, TASK_FOREVER, &iterateCallback, &ts, false);

...

  if ( b.cmd == C_PLAY_FILE_ONCE ) {  // play once
    player.playFolder(b.folder, b.file);
    tPlayerIterator.restart();
  }
  
  
void iterateCallback() {
  if ( player.commandCompleted() ) tPlayerIterator.disable();
}
```


