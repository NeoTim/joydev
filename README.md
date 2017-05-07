### Aim

In certain NES games, execution of special attack moves requires pressing
multiple buttons in a particular order.

For example, Teenage Mutant Ninja Turtles 3 for NES requires pressing A + B
combination to execute the special attack.

Is it possible to use a single spare button (like X or Y) press to simulate
pressing of the A + B combination?

![gamepad](snes-gamepad.jpg)

### Details

When executing the special attack (by actually executing the A + B
combination), we can obtain the following trace from `jstest` program,

```
$ jstest /dev/input/js0
Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:off  3:off
Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:on   3:off
Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:off  3:off
Axes:  0:     0  1:     0 Buttons:  0:off  1:off  2:off  3:off
```

I have modified the `joydev` kernel driver to implement these "state
transitions" when the Y button is pressed.

```
From a12297c000216e81ab2394c246a5888369ab246a Mon Sep 17 00:00:00 2001
From: Dhiru Kholia <dhiru.kholia@gmail.com>
Date: Sat, 22 Apr 2017 20:37:33 +0530
Subject: [PATCH] USB gamepad hack to trigger "special attack" by a single
 keypress

$ jstest /dev/input/js0

For special attack in "Teenage Mutant Ninja Turtles III: The Manhattan
Project" we require,

Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:off  3:off
Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:on   3:off
Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:off  3:off
Axes:  0:     0  1:     0 Buttons:  0:off  1:off  2:off  3:off
---
 drivers/input/joydev.c | 41 +++++++++++++++++++++++++++++++++++++++--
 1 file changed, 39 insertions(+), 2 deletions(-)

diff --git a/drivers/input/joydev.c b/drivers/input/joydev.c
index 29d677c..7b8f816 100644
--- a/drivers/input/joydev.c
+++ b/drivers/input/joydev.c
@@ -148,8 +148,45 @@ static void joydev_event(struct input_handle *handle,
 	event.time = jiffies_to_msecs(jiffies);

 	rcu_read_lock();
-	list_for_each_entry_rcu(client, &joydev->client_list, node)
-		joydev_pass_event(client, &event);
+	// B = 2, A = 1, Y = 3 (on a USB SNES gamepad)
+	list_for_each_entry_rcu(client, &joydev->client_list, node) {
+		// When Y is pressed, simulate special attack (A + B), which
+		// requires the following transitions.
+		//
+		// Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:off  3:off
+		// Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:on   3:off
+		// Axes:  0:     0  1:     0 Buttons:  0:off  1:on   2:off  3:off
+		// Axes:  0:     0  1:     0 Buttons:  0:off  1:off  2:off  3:off
+		if (event.number == 3) { // something to do with Y button
+			if (event.value == 1) { // Y press
+				// A on
+				event.type = JS_EVENT_BUTTON;
+				event.number = 1;
+				event.value = 1;
+				joydev_pass_event(client, &event);
+				// B on
+				event.type = JS_EVENT_BUTTON;
+				event.time = event.time + 80;
+				event.number = 2;
+				event.value = 1;
+				joydev_pass_event(client, &event);
+				// B off
+				event.type = JS_EVENT_BUTTON;
+				event.time = event.time + 160;
+				event.number = 2;
+				event.value = 0;
+				joydev_pass_event(client, &event);
+				// A off
+				event.time = event.time + 30;
+				event.number = 1;
+				event.value = 0;
+				joydev_pass_event(client, &event);
+			} else { // do nothing on Y unpress
+			}
+		} else {
+			joydev_pass_event(client, &event);
+		}
+	}
 	rcu_read_unlock();

 	wake_up_interruptible(&joydev->wait);
--
2.7.4

```

### Problem

With the patched `joydev` driver, `jstest` seems to work OK (use `jstest
--event /dev/input/js0` to verify this). When I actually press the `Y` button,
it seems to emulate A + B pressing combination.

The kernel modifications seem to be OK. See the `puNES-joystick-debug.patch`
puNES patch for verify that the modified joystick events are successfully
reaching puNES.

However, special moves in the games do not get triggered in emulation softwares
like FCEUX or puNES.

I am wondering what/where the problem is. Do I need to introduce some artifical
delay between the four `joydev_pass_event` calls in my patch?

Update: I added the required delays using the `mdelay` function but still no
joy. The actual delay doesn't seem to happen between the `joydev_pass_event`
calls for some unknown reason.

### References

* http://elixir.free-electrons.com/linux/latest/source/drivers/input/joydev.c

* https://github.com/flosse/linuxconsole/blob/master/utils/jstest.c (this uses the event interface)

* https://github.com/punesemu/puNES/blob/master/src/core/input.c (this doesn't seem to be using joydev's event interface like jstest)

* https://github.com/punesemu/puNES/blob/master/src/gui/linux/jstick.c

* https://www.kernel.org/doc/Documentation/input/joystick-api.txt
