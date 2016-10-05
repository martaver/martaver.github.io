---
title: "Running the 'Debugger for Chrome' VS Code extension alongside other Chrome tabs"
modified: 2016-10-05T16:38:02-03:00
categories:  
  - technology  
tags:  
  - vscode
  - chrome
  - debugging
  
---

I'm making a simple [cycle.js](http://cycle.js.org/) app for a personal project. I want to use [Typescript](https://www.typescriptlang.org/) (TS) and [Webpack](https://webpack.github.io/), because I like strong typing and hot module replacement (HMR), so I went looking and found this handy [starter](https://github.com/cyclejs-community/typescript-starter-cycle) and corresponding [guide](https://journal.artfuldev.com/cycle-js-quick-start-with-typescript-and-webpack-in-visual-studio-code-e562a009e9d6#.qpn2b7vkl) by [@artfuldev](https://journal.artfuldev.com/). 

It tells us about the VS Code extension 'Debugger for Chrome' which lets you use VS Code to debug sites viewed in Chrome. Chrome DevTools is awesome, but it's handier to be able to debug TS directly in the IDE and edit inline - so I went to [set it up](https://blogs.msdn.microsoft.com/jtarquino/2016/01/24/debugging-typescript-in-visual-studio-code-and-chrome/). 

And... got this error after 10 seconds when I tried to debug (`F5`): 

```
Cannot connect to runtime process, timeout after 10000 ms - (reason: Cannot connect to the target: 404 - Not Found. The requested location could not be found.).
```

The remote debugger wasn't running on Chrome. You can easily tell this by browsing to `http://localhost:9222/json` in the browser and seeing the 404 yourself. If it was running you should see a list of debuggable tabs.

**The problem is** that Chrome needs to be started with the argument `--remote-debugging-port=9222` and if there are any other instances already running (read as: if you are a normal Chrome user), they will share the original configuration, and ignore the request to start remote debugging. 

Yes, even background processes that keep things like Inbox alive will prevent remote debugging from starting. If you `ctrl-shift-esc` and close **every** Chrome process, then when you `F5`, 'Debugger for Chrome' works fine.

*But that sucks*. I'm a junkie and keep open many Chrome tabs & background apps during development - having to close them every time you want to debug is a major pain in the ass. 

**So here's a solution**:

Set up your `launch.json` to run Chrome with another (fake) user's profile. Add the following line to your `Launch Chrome against localhost, with sourcemaps` configuration:

```
"userDataDir": "C:\\temp\\chrome-dev-configuration"
```

This has the side-effect of running Chrome, with remote debugging enabled, in an *isolated* configuration, leaving all of your other tabs and processes alone.

*There's one drawback...* Your usual array of Chrome extensions won't be loaded in this instance - since they're installed under your own user configuration. But... 

**You can re-use this configuration**. Change `C:\\temp\\chrome-dev-configuration` to a more permanent folder, and you can install all those fancy add-ons again, keeping them isolated to your dev setup of Chrome.

This is actually kinda cool, because you can have different Chrome setups for different projects!

It's the little wins in life.