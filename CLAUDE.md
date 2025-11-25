# Purpose of this repo 

For getting speech to text to work on Wayland on Linux, `ydotool` is often the stumbling block. 

For dictation tools that intend to support real time text entry into any text position where the cursor is. a virtual keyboard device is needed to provide a text input stream. Several tools that should work well on Linux fail. This is very unfortunate as it breaks the chain. The local speech to text might work flawlessly but be thwarted at this last step of the process. 

Two motivations for this process:

1) the challenges that Wayland poses are frustrating but ultimately it's a widely supported project in Linux. I have been tempted to go back to X11 many times, but believe it's better to work with the flow than against it.

2) ydotool hasn't been updated in quite some time. Many approaches advocate building that package from source. However, the idea of building something on a dependency that isn't maintained doesn't seem like a good approach to me.


## Deepgram ... figured something out!

Deepgram released a starter for a Linux keyboard. It's virtually the only speech to text tool on Linux that I've seen work on Wayland basically out of the box. 

The project is open source and available here. From what I saw from a first scrutiny of the code base, it relies upon privilege escalation and creating its own virtual keyboard directly at the kernel level rather than leveraging existing dependency. 

The purpose of this repository is purely to examine the Deepgram project in order to scrutinize exactly how it works.  The objective is not to attempt to reverse engineer the exact process. But rather to look to it as instructive in how Reliable. Virtual keyboard input can be used in other speech to text apps.

Your outputs therefore can be an analysis of the code base with code snippets enclosed. 


https://github.com/deepgram/voice-keyboard-linux/