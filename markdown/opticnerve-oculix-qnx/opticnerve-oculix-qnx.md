id: opticnerve-oculix-qnx
title: Test a QNX 8.0 Screen from Your Laptop with OpticNerve and OculiX
summary: Learn how to put a small native agent on a Raspberry Pi 5 running QNX 8.0 so that OculiX, an open-source GUI test tool, can see its screen, click and type, and check the result, from a Windows, macOS or Linux laptop.
categories: Testing, AI & ML
tags: intermediate
platforms: Raspberry Pi 5
published: 2026-11-18
featured: false
status: draft
authors: David Young
feedback_link: https://github.com/qnx/codelabs/issues

<!--
Draft for review, 2026-11-18. Before publishing: run every step on a fresh quick-start image exactly as written,
set `published` to the day it goes live, and point feedback_link at qnx/codelabs/issues if it is submitted there.
-->

# Test a QNX 8.0 Screen from Your Laptop with OpticNerve and OculiX

## Overview
Duration: 2:00

The HMI on an instrument cluster, an infusion pump or a factory panel is often tested by hand, tap by tap. GUI test tools that work this way on a desktop (finding a button by its picture, clicking it, reading the result) run on a desktop host and need a way to see and drive the target's screen.

In this codelab you give them that way in. You'll build **OpticNerve**, a small native agent for QNX 8.0. You'll put it on a Raspberry Pi 5 and drive the Pi's screen from your laptop with **OculiX**, the open-source successor of SikuliX.

The agent reads the display through QNX Screen and injects pointer and keyboard input through it. It serves both over VNC on port 5901. It also answers scripts on the target through a QNX resource manager, `/dev/opticnerve`, which can tell a test when the screen has settled.

### What you'll learn

- How to build a QNX Screen application and a resource manager with `qcc`
- How to install, start and remove the agent on a Pi 5 without changing anything else on the target
- How to run an OculiX test against a QNX screen, and write a check of your own
- How to make a test wait for the screen instead of sleeping for a guessed time
- Optionally, how a Claude Code session on the target can watch your test runs for errors that never reach the screen

### What you'll need

- A QNX Everywhere license and the QNX SDP 8.0 installed with the QNX Software Center (we used SDP 8.0.5 on Windows 11)
- A Raspberry Pi 5 running the QNX 8.0 quick-start image, on your network, with its `qnxuser` login, and an HDMI monitor
- A laptop with Git (on Windows, Git for Windows, which includes Git Bash), Python 3, Java 17 or newer, and `ssh`

---

## Get the tools
Duration: 5:00

1. Clone the repository:

    ```bash
    git clone https://github.com/davidyoung8196504567-sudo/OpticNerve.git
    cd OpticNerve
    ```

2. Install the Python packages the host tools use:

    ```bash
    python -m pip install cryptography Pillow
    ```

3. Download OculiX and the VNC client it needs. We tested with these two jars:

    - `oculixapi-4.0.0-<os>.jar` from the OculiX 4.0.0 release (`windows`, `linux` or `macos`)
    - `tigervnc-java-oculix-2.0.4.jar` from Maven Central (`io.github.oculix-org:tigervnc-java-oculix:2.0.4`)

    Put both in one folder and note their paths. The TigerVNC jar must come **first** on the classpath.

4. Set the environment for the rest of this codelab (Git Bash, Linux or macOS):

    ```bash
    export ON_HOST=192.168.1.50              # your Pi's address
    export ON_LOGIN_PASSWORD='...'           # the Pi's qnxuser password, also used for sudo
    export ON_VNC_PASSWORD='...'             # a new password for the agent's VNC port, at most 8 characters
    export CP="/path/to/tigervnc-java-oculix-2.0.4.jar;/path/to/oculixapi-4.0.0-windows.jar"
    ```

    On Linux and macOS, separate the jars with `:` instead of `;`.

> aside negative
>
> Type passwords into environment variables, not into commands. The OpticNerve tools pass them to the Pi on the SSH connection's input, never on a command line, so they don't show up in process lists or logs.

---

## Build the agent
Duration: 5:00

1. Open a terminal on your Windows build host and set the SDP environment by running `qnxsdp-env.bat` from your SDP folder. Check that `qcc` is on your `PATH`:

    ```bash
    qcc -V
    ```

2. Build for the Pi 5:

    ```bash
    cd agent
    make ARCH=aarch64le
    cd ..
    ```

    This gives you three programs in `agent/build/aarch64le/`:

    | Program | What it is |
    |---|---|
    | `opticnerve` | The agent and its subcommands |
    | `core_test` | The protocol and scheduling tests |
    | `demo_panel` | A small instrument panel to test against |

    Everything builds with `-std=c11 -Wall -Wextra -Werror`.

### What's QNX-specific in there

Take a minute to look at `agent/qnx/`:

- **`serve.c`:** one thread on a QNX dispatch loop (`dispatch_create()`). Network sockets arrive as select events (`select_attach()`). Screen's change notification, a timer and signals arrive as pulses (`pulse_attach()`). The display is read with `screen_read_display()`, and input is injected with `screen_inject_event()`.
- **`device.c`:** `/dev/opticnerve`, a resource manager (`resmgr_attach()`) on the same loop. It serves status, wait and capture requests alongside the VNC clients, with no locks.
- **`cli.c`:** the subcommands you'll use below. Each sends a `devctl()` message to the agent through `/dev/opticnerve` and gives up after a time limit.

---

## Install and start it on the Pi
Duration: 3:00

1. Install the agent. This copies it to `/data/opticnerve/` and sets its VNC password:

    ```bash
    python host/tools/opticnerve_deploy.py install
    ```

    The script prints the SHA-256 of the binary it installed. Keep it with your test results, so you always know which build was tested.

2. Start it. It runs as root, because reading the display and injecting input need it, and listens on port 5901:

    ```bash
    python host/tools/opticnerve_deploy.py start
    python host/tools/opticnerve_deploy.py status
    ```

    `status` shows the agent's version, process ID, display and clients.

> aside positive
>
> Everything lives in `/data/opticnerve/`. Nothing else on the target is changed, and the image's own services keep running as they were.

---

## See the screen
Duration: 2:00

Point any VNC viewer at `<Pi address>:5901` and log in with your VNC password. You should see the Pi's monitor.

Or grab one frame with the snapshot tool from the repository:

```bash
python host/tools/vnc_snapshot.py $ON_HOST:5901 "$ON_VNC_PASSWORD" frame.png
```

Open `frame.png`. The agent sends 32-bit pixels read straight from the display, so what you see is exactly what the monitor shows. That's what lets a test compare pixels.

---

## Run your first OculiX test
Duration: 6:00

1. Copy the demo panel to the Pi and start it over SSH:

    ```bash
    scp agent/build/aarch64le/demo_panel qnxuser@$ON_HOST:/data/home/qnxuser/
    ssh qnxuser@$ON_HOST 'cd /data/home/qnxuser; chmod +x demo_panel; nohup ./demo_panel 600 > demo.log 2>&1 < /dev/null &'
    ```

    A full-screen panel appears on the monitor:
    - three buttons, **+1**, **RESET** and **LAMP**;
    - a counter;
    - a lamp;
    - a text line.

    It closes itself after 600 seconds.

2. Run the demo test from the repository root:

    ```bash
    java -cp "$CP" host/demo/DemoTest.java $ON_HOST "$ON_VNC_PASSWORD" demo-out -slow
    ```

    Watch the monitor:
    - OculiX finds **RESET** by its picture and clicks it;
    - it clicks **+1** three times;
    - it checks the counter pixel by pixel against `host/demo/ref/count_3.png`;
    - it switches the lamp.

    Each step prints `PASS` or `FAIL`, and the screenshots go to `demo-out/`.

3. Run it again with `-fail`. One check now expects the wrong count, so you can see what a caught failure looks like.

Between steps, the test doesn't sleep. It asks the agent on the Pi to wait until the screen has settled, and prints how long that took:

```
     opticnerve wait: quiet 41 (0.3 s)
```

---

## Write a check of your own
Duration: 8:00

Create `MyCheck.java` in the repository root:

```java
import java.io.File;
import javax.imageio.ImageIO;
import org.sikuli.script.Match;
import org.sikuli.script.Pattern;
import org.sikuli.vnc.VNCScreen;

public class MyCheck {
    public static void main(String[] a) throws Exception {
        // Offer only VNC authentication (configuration used with OpticNerve)
        Object p = Class.forName("com.tigervnc.rfb.SecurityClient").getField("secTypes").get(null);
        p.getClass().getMethod("setParam", String.class).invoke(p, "VncAuth,None");

        VNCScreen s = VNCScreen.start(a[0], 5901, a[1], 10, 1000);
        try {
            // Cut the +1 button out of the live screen and use it as the pattern to find
            File plus = new File("plus.png").getAbsoluteFile();
            ImageIO.write(s.capture(160, 700, 420, 200).getImage(), "png", plus);

            var before = s.capture(160, 280, 700, 180).getImage();     // the counter
            Match m = s.exists(new Pattern(plus.getPath()).similar(0.9), 3);
            if (m == null) { System.out.println("FAIL +1 not found"); return; }
            System.out.println("PASS found +1, score " + m.getScore());
            s.click(m);
            Thread.sleep(700);
            var after = s.capture(160, 280, 700, 180).getImage();

            boolean changed = false;
            for (int y = 0; y < before.getHeight() && !changed; y++)
                for (int x = 0; x < before.getWidth() && !changed; x++)
                    changed = before.getRGB(x, y) != after.getRGB(x, y);
            System.out.println(changed ? "PASS the counter changed" : "FAIL the counter did not change");
        } finally {
            s.stop();
        }
    }
}
```

Run it while the demo panel is showing:

```bash
java -cp "$CP" MyCheck.java $ON_HOST "$ON_VNC_PASSWORD"
```

The coordinates are the demo panel's, on a 1920 x 1080 display. For your own app, take a `frame.png` first and read the positions off it.

> aside negative
>
> Always offer only VNC authentication (`VncAuth,None`) before the first `VNCScreen.start`, and send Enter as `type("\r")`. With the jar versions above, those are the settings we tested.

---

## Wait for the screen, not for the clock
Duration: 4:00

`Thread.sleep(700)` in your check is a guess. On a real device, a click can take effect later than you expect, and a guessed sleep makes a test either slow or flaky.

The agent knows when Screen has composed a new frame, and any script on the target can ask it through `/dev/opticnerve`:

```bash
ssh qnxuser@$ON_HOST
PATH=$PATH:/data/opticnerve/bin
opticnerve status                         # version, clients, captures, change count
opticnerve wait -timeout=5                # prints "change N" at the next screen change (exit 0), or "timeout N" (exit 2)
opticnerve wait -quiet=300 -timeout=5     # prints "quiet N" once the screen has been still for 300 ms
opticnerve capture 160 280 700 180 c.ppm  # writes that region to a file, as you
```

`N` is the agent's change count. Each subcommand is one `devctl()` message to the resource manager.

A host test can run `opticnerve wait -since=COUNT -quiet=300` over SSH after each click. That's what `DemoTest` does.
- `-since` makes it wait for a change *after* the one it already saw.
- `-quiet` waits for the screen to settle.

If the agent doesn't answer, the command says so and exits with status 3, so a test never hangs on it.

---

## Going further: a Claude session that watches your tests
Duration: 5:00

Some failures never reach the screen: a crash in a background process, an error in the system log, a process that quietly restarts. A GUI test sees only the consequences, often many steps later.

We closed that gap with an AI assistant running **on the Pi itself**. QNX's codelab [Use Claude Code on QNX Developer Desktop](https://qnx.github.io/codelabs/claude-code/index.html) (QNX Developer Relations, 2026) and QNX's [claude-code-qnx](https://github.com/qnx/claude-code-qnx) project (Apache-2.0) show how to run Claude Code on a QNX 8.0 target. That codelab is what alerted us that this was possible. We followed it on our Pi 5 with the image's Node.js, and used **Claude Code 2.1.226**: `node extract.js --version 2.1.226`. QNX's codelab pins `2.1.196`.

> aside negative
>
> Pin a version, don't use `--latest`. `claude-code-qnx` extracts the JavaScript from Claude Code's Linux package, and that package's layout changed in later releases: from 2.1.229 it needs an updated extractor (an open [pull request](https://github.com/qnx/claude-code-qnx/pull/1) covers up to 2.1.245), and its author reports that from 2.1.246 the package holds compiled bytecode with no source to extract. Use 2.1.226, the version we used, to reproduce our setup.

With Claude Code running on the target, we asked it to write and run a watcher for our test runs. As an ordinary user, the watcher follows:

- the system log (`slog2info -W`);
- the processes under test (`/proc`);
- the core files the dump utility writes when a process crashes.

It writes one line per incident to a file. Our host-side script `host/demo/run_watched.sh` marks the start and end of each run, and checks that file every 10 seconds. A new incident stops the test, saves the screen and prints the incident.

To prove it, we crashed a throwaway process during a run. The watcher recorded the core dump within a second, and the run stopped itself with the screen saved:

```
ALERT at 09:55:35 UTC: stopping the test
0007 09:55:35 UTC a core dump, sleep.core
screen saved: demo-out/stop-path-1_alert.png
end 09:55:39 UTC; STOPPED ON ALERT; test exit 143
```

To try it yourself, follow QNX's codelab on your Pi. Then describe to Claude what your tests run and what should never happen during a run, and ask it for a watcher that writes incidents to a file your test harness can read.

---

## Clean up
Duration: 1:00

Stop the demo panel and take the agent off the Pi:

```bash
ssh qnxuser@$ON_HOST 'slay -f demo_panel; rm -f /data/home/qnxuser/demo_panel /data/home/qnxuser/demo.log'
python host/tools/opticnerve_deploy.py undo
```

`undo` stops the agent and deletes `/data/opticnerve/`. The target is as it was before you started.

---

## Share your work
Duration: 1:00

You've given a desktop GUI test tool eyes and hands on a QNX 8.0 screen. Find buttons by their picture, check the result pixel by pixel, and let the target tell your test when it's ready.

- The source, more test suites (a GTK Demo crawl, the GTK 4 Widget Factory "Obstacle Course") and the full reference are in the [OpticNerve repository](https://github.com/davidyoung8196504567-sudo/OpticNerve).
- Tell us what you tested, on [Discord](https://discord.gg/Jj4EkkrFTT) or [Reddit](https://www.reddit.com/r/qnx).
