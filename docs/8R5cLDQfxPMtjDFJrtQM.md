---
title: Electron ⚡ Main Process
---

<div align="center">    <br/>    <div>DOC</div>    <h1>Electron ⚡ Main Process</h1>    <br/>  </div>

### Files Used:
📄 src/app/electron-main.js


<br/>

# Electron Main process

## The DOD of this walkthrough
In this walkthrough, we will go over some of the key features of electron's main process in Swimm.

## What is it about?
Electron's main process (sometimes referred to as *background.js*) is responsible for managing all the processes and tasks of the Electron app.

The main process is responsible for responding to applications life cycle events, starting and quitting the application. When we launch an Electron application, the framework runs the script specified in `vue.config.js` (*electron-main.js*) . The script creates the application window (renderer process).

## Important concepts:
### Events
In general, Electron uses lots of events to handle different flows of the application. In the main process, we subscribe to an event and implement a callback function that will run whenever the related event triggers. 

### Single Instance
Swimm.app is running as a singleton. That means that if the user has already a running Swimm app instance they won't be able to run another one in parallel. We use [SingleInstanceLock](https://www.electronjs.org/docs/api/app#event-second-instance) to make sure that the app instance is a singleton.
*asking for the lock actually generates a *`second-instance`* event that is handled by the main process.

### CLI <-> App
The CLI communicates with our app by using `openElectronInstance()` that runs a designated command on the user's machine. The designated command tries to open the app (what may trigger a `second-instance` event by trying to get the lock). Sometimes the cli also redirects the view to another page, like with `swimm done`. We will look into how we call this from the CLI in another walkthrough.

##### side notes: 
* There are some platform-specific handling methods that take care of several use cases or bugs with Electron. These cases are well documented within the code in `electron-main.js`

* package.json's **main** field is set for "background.js", not "electron-main.js" as expected. `background.js` is actually the name of the bundled main electron process file. You can find it inside the `dist/` directory:

![image.png](https://firebasestorage.googleapis.com/v0/b/swimmio-content/o/repositories%2FveezvxCuzpPrRLLXWD2E%2Fimg%2F9bc9aa79-995b-4c3a-85f7-a9a1e06d3c24.png?alt=media&token=847c17c5-b21b-42e9-b440-425a7ed8b3cc)

<br/>

When the application is initialized we create a new chromium window with the  Swimm app.
If there is already a running app - we cancel the current request for opening the app.
Asking for the instance lock with another instance on the background will make the already running instance focused on the user's machine.

<div>    📄 src/app/electron-main.js  </div>

```js
🟩 265    function handleDirectAppOpen() {
🟩 266      setAppLinkAsRedirectionUrl();
🟩 267      // If there is an already running app - second-instance will be triggered.
🟩 268      const gotTheLock = app.requestSingleInstanceLock();
🟩 269      if (gotTheLock) {
🟩 270        const swimmVersion = state.get('swimm_version');
🟩 271        if (!swimmVersion || getFriendlyVersion(swimmVersion) !== getFriendlyVersion(PJSON_VERSION)) {
🟩 272          state.set('should_show_version_modal', true);
🟩 273        }
🟩 274        state.set('swimm_version', PJSON_VERSION);
🟩 275        createWindow();
🟩 276      } else {
🟩 277        app.quit();
🟩 278      }
🟩 279    }
```
<br/>

This method creates a focused  visual window of our app.
If needed, this function will handle redirections settings (for example going to the solution view page)
we register to several `webContents` events that are being triggered by the renderer process (for example - asking to open a new native browser window)

<div>    📄 src/app/electron-main.js  </div>

```js
🟩 142    function createWindow(windowMode = 'app') {
🟩 143      // Create the browser window.
🟩 144      const windowConfig = generateBrowserWindowConfigs(windowMode);
🟩 145      win = new BrowserWindow(windowConfig);
🟩 146      win.webContents.on('did-finish-load', () => {
🟩 147        win.setTitle('Swimm');
🟩 148      });
🟩 149    
🟩 150      win.webContents.on('new-window', handleNavigation);
🟩 151      win.webContents.on('will-navigate', handleNavigation);
🟩 152    
🟩 153      if (process.env.WEBPACK_DEV_SERVER_URL) {
🟩 154        // Load the url of the dev server if in development mode
🟩 155        win.loadURL(process.env.WEBPACK_DEV_SERVER_URL);
🟩 156        if (!isTest) {
🟩 157          win.webContents.openDevTools();
🟩 158        }
🟩 159      } else {
🟩 160        createProtocol('app');
🟩 161        const redirectData = state.get('redirection_data');
🟩 162        let url = redirectData ? redirectData.url : 'app://./index.html';
🟩 163    
🟩 164        // opened by app link (swimm://...)
🟩 165        const lastArg = process.argv[process.argv.length - 1];
🟩 166        if (isAppLink(lastArg)) {
🟩 167          url = updateAppLinkToAppInternalUrl(lastArg);
🟩 168        }
🟩 169        win.loadURL(url);
🟩 170        state.deleteKey('redirection_data');
🟩 171      }
🟩 172      win.once('ready-to-show', () => {
🟩 173        if (windowMode === 'cli') {
🟩 174          win.hide();
🟩 175        } else {
🟩 176          win.show();
🟩 177          win.maximize();
🟩 178          app.focus({ steal: true });
🟩 179          if (!isStaging && !isDevelopment && localConfig.get('check_updates_on_startup', true)) {
🟩 180            setTimeout(() => checkForUpdates(true), 1000);
🟩 181          }
🟩 182        }
🟩 183      });
🟩 184    
🟩 185      // setting spellcheck
🟩 186      win.webContents.session.setSpellCheckerLanguages(['en-US']);
🟩 187      win.webContents.on('context-menu', initContextMenu);
🟩 188    
🟩 189      win.on('closed', () => {
🟩 190        win = null;
🟩 191      });
🟩 192    }
```
<br/>

When the CLI "calls" the app and tries to open it, we would want to only tell the app to redirect or to relaunch.
If the app was closed what results in the  CLI success in getting the instance lock of the app, we will initiate an **app** instance of Swimm and release the lock. 

<div>    📄 src/app/electron-main.js  </div>

```js
🟩 248    async function handleCliCall() {
🟩 249      const redirectData = state.get('redirection_data');
🟩 250      if (redirectData) {
🟩 251        const gotTheLock = app.requestSingleInstanceLock();
🟩 252        // The CLI process should never hold the lock.
🟩 253        // If it got the lock, that means no app is open -> CLI process should release the lock and open the app in a new process.
🟩 254        if (gotTheLock) {
🟩 255          app.releaseSingleInstanceLock();
🟩 256          spawn(process.execPath, [__filename, 'spawn_instance_from_cli'], {
🟩 257            cwd: process.cwd(),
🟩 258            detached: true,
🟩 259          });
🟩 260        }
🟩 261      }
🟩 262      app.quit();
🟩 263    }
```
<br/>

When detecting a second instance event, this subscription will focus the already-running app on the user's machine
Also, this step the process will check if it needs to handle redirections to new pages (will happen when the second-instance was a result of a CLI related call)

<div>    📄 src/app/electron-main.js  </div>

```js
🟩 95     app.on('second-instance', (event, commandLine, workingDirectory) => {
🟩 96       // Someone tried to run a second instance, we should focus our window.
🟩 97       const redirectData = state.get('redirection_data');
🟩 98       if (redirectData) {
🟩 99         let redirectionUrl = redirectData.url;
🟩 100        redirectionUrl = updateAppUrlToDevServerUrl(redirectionUrl);
🟩 101        if (win) {
🟩 102          win.loadURL(redirectionUrl);
🟩 103        }
🟩 104        state.deleteKey('redirection_data');
🟩 105      }
🟩 106    
🟩 107      if (win) {
🟩 108        if (win.isMinimized()) {
🟩 109          win.restore();
🟩 110        }
🟩 111        // required for windows
🟩 112        win.focus();
🟩 113      }
🟩 114      // required for OSX
🟩 115      app.focus({ steal: true });
🟩 116    });
```
<br/>

If there are any more questions about this part of the code, talk to @Eden 

<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRjhSNWNMRFFmeFBNdGpERkpydFFN). Timestamp: 2021-04-24T18:52:02.072Z
