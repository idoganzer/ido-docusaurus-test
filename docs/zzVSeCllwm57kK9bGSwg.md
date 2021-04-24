---
title: The Local Logger
---

<div align="center">    <br/>    <div>DOC</div>    <h1>The Local Logger</h1>    <br/>  </div>

### Files Used:
📄 src/shared/config.js

📄 src/shared/logger.js

📄 src/app/adapters/load.ts

📄 src/app/electron-utils/auto-updater.js

📄 src/cli/commands/log.ts


<br/>

### Logging about things that happen through the app is important.

In addition to the `EventLogger` service (which is described in another unit),
we have a **Local Logger**.

We are using the library [winston](https://github.com/winstonjs/winston) as a local-log solution. Whenever an error or an important event in the CLI or in an Adapter occurs, we can safely log that with our logger (instead of spamming the user with `console.log` / `console.error`).

The logger's config is defined in `Shared/logger.js`, and using it is super easy! 
We log events by `logLevel`:
- for errors, we use `logger.error`
- for info messages, we use `logger.info`

Yes. It's that simple.

To view the log, we can either open the local `.log` file with our favorite text editor, or use the CLI command `swimm log` (or `swimmdev log`) which automatically tails the log file (also for windows users :) ) and watch for changes:
![image.png](https://firebasestorage.googleapis.com/v0/b/swimmio-content/o/repositories%2FveezvxCuzpPrRLLXWD2E%2Fimg%2Fca282700-0416-4dbc-b2fe-faf804f7e67f.png?alt=media&token=4b7c5554-9042-4816-a681-2eb73786a321)


<br/>

Default properties for the local logger are defined here

<div>    📄 src/shared/config.js  </div>

```js
⬜ 118      STORE_DB_METADATA: 'STORE_DB_METADATA',
⬜ 119    };
⬜ 120    
🟩 121    const LOCAL_LOG_FILE_CONFIG = {
🟩 122      filename: locations.LOG_FILE,
🟩 123      maxLogSize: 10485760, // 10 MB - The maximum log file size in bytes
🟩 124      maxAdditionalLogFiles: 1, // The number of additional logfiles to create when log-rotating
🟩 125      isTailable: true, // Makes sure that the original log file will always get the latest logs
🟩 126      lastLinesCount: 10, // The default last log lines to show with swimmlog before tailing the file
🟩 127    };
⬜ 128    
⬜ 129    const LOCAL_CONFIGS_PATH = paths.config;
⬜ 130    const AUTOFIXED_FILE_PREFIX = 'autofixed_';
```
<br/>

In this step we can see how we actually create the winston logger instance by using the configuration properties from the previous snippet

<div>    📄 src/shared/logger.js  </div>

```js
🟩 1      import winston from 'winston';
🟩 2      
🟩 3      import { LOCAL_LOG_FILE_CONFIG } from './config';
🟩 4      
🟩 5      const logger = winston.createLogger({
🟩 6        level: 'info',
🟩 7        format: winston.format.combine(winston.format.timestamp({ format: 'DD-MM-YYYY HH:mm:ss' }), winston.format.json()),
🟩 8        defaultMeta: { service: 'user-service' },
🟩 9        transports: [
🟩 10         new winston.transports.File({
🟩 11           filename: LOCAL_LOG_FILE_CONFIG.filename,
🟩 12           maxsize: LOCAL_LOG_FILE_CONFIG.maxLogSize,
🟩 13           maxFiles: LOCAL_LOG_FILE_CONFIG.maxAdditionalLogFiles,
🟩 14           tailable: LOCAL_LOG_FILE_CONFIG.isTailable,
🟩 15         }),
🟩 16       ],
🟩 17     });
🟩 18     
🟩 19     export default logger;
```
<br/>

In this step we have a logger usage of logging an `error` event.
Note that the `service` metadata-option is set as `adapter-load`. Setting this property allows us to easily understand that this specific log-message is related to the "load" adapter.

<div>    📄 src/app/adapters/load.ts  </div>

```ts
🟩 72       } catch (error) {
🟩 73         logger.error(`failed to load unit: ${swmFileName}. Details: ${error.toString()}`, { service: 'adapter-load' });
```
<br/>

In this step we can see a usage for logging with `info` log level.

<div>    📄 src/app/electron-utils/auto-updater.js  </div>

```js
⬜ 16     autoUpdater.on('checking-for-update', () => {
🟩 17       logger.info('Checking for update...', { service: 'auto-updater' });
⬜ 18     });
```
<br/>

If you are interested on how we tail the log file for changes and pretty-print it, you can read the implementation here

<div>    📄 src/cli/commands/log.ts  </div>

```ts
⬜ 1      import * as fs from 'fs';
⬜ 2      import { Tail } from 'tail';
⬜ 3      
⬜ 4      import * as config from '../../shared/config';
⬜ 5      import * as pprint from '../../shared/pprint';
⬜ 6      
🟩 7      export async function app_log(linesNumber) {
🟩 8        try {
🟩 9          const logPath = config.LOCAL_LOG_FILE_CONFIG.filename;
🟩 10         const logLastLinesToPrint = linesNumber || config.LOCAL_LOG_FILE_CONFIG.lastLinesCount;
🟩 11     
🟩 12         pprint.out(pprint.colors.yellow('\n-=-=-=-=-=-=-=-=-'));
🟩 13         pprint.out(pprint.colors.green('*** Swimm Log ***'));
🟩 14         pprint.out(pprint.colors.yellow('-=-=-=-=-=-=-=-=-'));
🟩 15         if (!fs.existsSync(logPath)) {
🟩 16           pprint.out(pprint.colors.magenta('Log file does not exist. Run `swimm start` to start the app and create the log file before using `swimm log`'));
🟩 17           process.exit(config.SUCCESS_RETURN_CODE);
🟩 18         }
🟩 19         pprint.out(pprint.colors.magenta(`Log file path: ${logPath}\n`));
🟩 20     
🟩 21         const logFileLines = fs.readFileSync(logPath).toString().trimEnd().split('\n');
🟩 22         const lastLinesOfFile = logFileLines.slice(Math.max(logFileLines.length - logLastLinesToPrint, 0));
🟩 23     
🟩 24         lastLinesOfFile.forEach((line) => pprintLogMessage(line));
🟩 25     
🟩 26         // Quit logging if user asked only for specific number of lines
🟩 27         if (linesNumber) {
🟩 28           return config.SUCCESS_RETURN_CODE;
🟩 29         }
🟩 30     
🟩 31         await tailLogFileSync();
🟩 32       } catch (error) {
🟩 33         pprint.err(error);
🟩 34         return config.ERROR_RETURN_CODE;
🟩 35       }
🟩 36       return config.SUCCESS_RETURN_CODE;
🟩 37     }
⬜ 38     
⬜ 39     function pprintLogMessage(message) {
⬜ 40       try {
⬜ 41         const parsedMessage = JSON.parse(message);
⬜ 42         const logLevelOutputs = {
⬜ 43           info: pprint.styles.info,
⬜ 44           error: pprint.styles.error,
⬜ 45           warning: pprint.styles.warning,
⬜ 46         };
⬜ 47         const logLevel = (logLevelOutputs[parsedMessage.level] || logLevelOutputs.info)('');
⬜ 48         const timestamp = pprint.colors.blueBright(parsedMessage.timestamp || '');
⬜ 49         const service = parsedMessage.service || '';
⬜ 50         const prettyMessage = `${timestamp} [${service}] ${logLevel}${parsedMessage.message}`.trimEnd();
⬜ 51         pprint.out(prettyMessage);
⬜ 52       } catch (e) {
⬜ 53         // Print a normal string if the log message object could not be parsed for some reason
⬜ 54         pprint.out(message);
⬜ 55       }
⬜ 56     }
⬜ 57     
⬜ 58     const waitForLogs = (delay) => new Promise((resolve) => setTimeout(resolve, delay));
⬜ 59     
⬜ 60     async function tailLogFileSync() {
⬜ 61       const logPath = config.LOCAL_LOG_FILE_CONFIG.filename;
⬜ 62       const tail = new Tail(logPath, { useWatchFile: true, follow: true });
⬜ 63       tail.on('line', function (data) {
⬜ 64         pprintLogMessage(data);
⬜ 65       });
⬜ 66     
⬜ 67       tail.on('error', function (error) {
⬜ 68         pprint.out('Error tailing log file: ', error);
⬜ 69         return config.ERROR_RETURN_CODE;
⬜ 70       });
⬜ 71       tail.watch();
⬜ 72     
⬜ 73       // The next line is the best line in the codebase :D
⬜ 74       // It makes the log CLI command run endlessly (until terminated by user)
⬜ 75       // eslint-disable-next-line no-constant-condition
⬜ 76       while (true) {
⬜ 77         await waitForLogs(1000);
⬜ 78       }
⬜ 79     }
```
<br/>

Eventually, logging really helps us to debug and understand sad/important flows in Swimm.
The more logs we have, the less time we need to invest on "blind-debugging" errors.

If you feel like there is an interesting place to add a log about, don't hesitate and add a log entry right away! :) 

<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRnp6VlNlQ2xsd201N2tLOWJHU3dn). Timestamp: 2021-04-24T18:52:02.423Z
