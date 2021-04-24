---
title: Event Logger
---

<div align="center">    <br/>    <div>DOC</div>    <h1>Event Logger</h1>    <br/>  </div>

### Files Used:
📄 src/shared/event-logger/logger-schema.ts

📄 src/shared/event-logger/swimm-events.ts

📄 src/shared/event-logger/event-logger.ts

📄 cloud/firestore.rules

📄 src/app/common/mixins/event-logger.js

📄 src/app/common/components/organisms/PlaylistCard.vue

📄 src/app/common/mixins/storeModulesWrapper.js

📄 src/cli/utils/cli-utils.ts


<br/>

In our app (and in our CLI), we dispatch some interesting events to Firestore. For example, when a user is creating a new resource, inviting another user or joining a repo - we would like to get a notification for that, right? :)

The events are stored in Firestore under the `event_logs` collection and later displayed in our admin dashboard.

To avoid security errors on sending the events we use [Joi](https://joi.dev/) package that validates the structure and content of the event message by a pre-defined json schema ( `src/shared/event-logger/logger-schema.ts ✓` ) that reflects the related security rule.

If an event message fails on validation, we send an error to the local error log for later investigation

<br/>

The event message schema (validated by Joi) is defined in this file. Note that some of the fields are mandatory and some are optional.
Note that this schema should be aligned with the firebase security rule of the event-logger. It's not updated automatically for now, so whenever updating one of the values - remember to update also the related security rule.

<div>    📄 src/shared/event-logger/logger-schema.ts  </div>

```ts
🟩 1      import * as Joi from 'joi';
🟩 2      
🟩 3      export const eventSchema = Joi.object().keys({
🟩 4        // if a field is not `.required` - then it's optional
🟩 5        swimmEventCode: Joi.string().required(),
🟩 6        message: Joi.string(),
🟩 7        value: Joi.string(),
🟩 8        category: Joi.string().default('USER_ACTION'),
🟩 9        userId: Joi.string().required(),
🟩 10       userName: Joi.string().required(),
🟩 11       actionName: Joi.string().required(), // i.e. "clicked help", "created a unit" ,"initialized a repo"
🟩 12       app_version: Joi.string().required(),
🟩 13       workspaceId: Joi.string(),
🟩 14       workspaceName: Joi.string().allow(''),
🟩 15       repoId: Joi.string(),
🟩 16       repoName: Joi.string().allow(''),
🟩 17       playlistId: Joi.string(),
🟩 18       playlistName: Joi.string().allow(''),
🟩 19       planId: Joi.string(),
🟩 20       planName: Joi.string().allow(''),
🟩 21       // src and target: i.e added a unit to playlist unitId is srcId and playlist id is targetId
🟩 22       srcId: Joi.string(), // optional: the id of the resource in action
🟩 23       srcName: Joi.string(), // optional: the name of the resource in action
🟩 24       srcType: Joi.string(), // optional: the type of the resource in action
🟩 25       targetId: Joi.string(), // optional: the id of the resource the action targets
🟩 26       targetName: Joi.string(), // optional: the name of the resource the action targets
🟩 27     });
```
<br/>

These are the event codes that we support in Swimm. In order to add a new event you should add it here

<div>    📄 src/shared/event-logger/swimm-events.ts  </div>

```ts
🟩 1      export const SWIMM_EVENTS = {
🟩 2        // GENERAL ACTIONS
🟩 3        UNKNOWN: { code: 'UNKNOWN', description: 'Unknown Event' },
🟩 4        ERROR: { code: 'ERROR', description: 'Error' },
🟩 5      
🟩 6        // ACTIONS ON RESOURCES IN REPO
🟩 7        EXERCISE_CREATED: { code: 'EXERCISE_CREATED', description: 'Created an Exercise' },
🟩 8        EXERCISE_DELETED: { code: 'EXERCISE_DELETED', description: 'Deleted an Exercise' },
🟩 9        EXERCISE_UPDATED: { code: 'EXERCISE_UPDATED', description: 'Updated an Exercise' },
🟩 10       EXTERNAL_LINK_CREATED: { code: 'EXTERNAL_LINK_CREATED', description: 'Created an external link' },
🟩 11       EXTERNAL_LINK_DELETED: { code: 'EXTERNAL_LINK_DELETED', description: 'Deleted an external link' },
🟩 12       EXTERNAL_LINK_UPDATED: { code: 'EXTERNAL_LINK_UPDATED', description: 'Updated an external link' },
🟩 13       PLAYLIST_CREATED: { code: 'PLAYLIST_CREATED', description: 'Created a Playlist' },
🟩 14       PLAYLIST_DELETED: { code: 'PLAYLIST_DELETED', description: 'Deleted a Playlist' },
🟩 15       PLAYLIST_UPDATED: { code: 'PLAYLIST_UPDATED', description: 'Updated a Playlist' },
🟩 16       DOC_CREATED: { code: 'DOC_CREATED', description: 'Created a Doc' },
🟩 17       DOC_CREATED_FROM_SUGGESTION: { code: 'DOC_CREATED_FROM_SUGGESTION', description: 'Created a Doc out of a suggestion' },
🟩 18       DOC_DELETED: { code: 'DOC_DELETED', description: 'Deleted a Doc' },
🟩 19       DOC_UPDATED: { code: 'DOC_UPDATED', description: 'Updated a Doc' },
🟩 20     
🟩 21       // ACTIONS ON REPO
🟩 22       REPOSITORY_CREATED: { code: 'REPOSITORY_CREATED', description: 'Created Repository' },
🟩 23       REPOSITORY_UPDATED: { code: 'REPOSITORY_UPDATED', description: 'Updated Repository' },
🟩 24       REPOSITORY_DELETED: { code: 'REPOSITORY_DELETED', description: 'Deleted Repository' },
🟩 25       SWIMMER_SUBSCRIBED_TO_REPOSITORY: {
🟩 26         code: 'SWIMMER_SUBSCRIBED_TO_REPOSITORY',
🟩 27         description: 'a Swimmer joined the Repository',
🟩 28       },
🟩 29       SWIMMER_UNSUBSCRIBED_FROM_REPOSITORY: {
🟩 30         code: 'SWIMMER_UNSUBSCRIBED_FROM_REPOSITORY',
🟩 31         description: 'a Swimmer unsubscribed from the Repository',
🟩 32       },
🟩 33       LIFEGUARD_SUBSCRIBED_TO_REPOSITORY: {
🟩 34         code: 'LIFEGUARD_SUBSCRIBED_TO_REPOSITORY',
🟩 35         description: 'a Lifeguard joined the Repository',
🟩 36       },
🟩 37       LIFEGUARD_UNSUBSCRIBED_FROM_REPOSITORY: {
🟩 38         code: 'LIFEGUARD_UNSUBSCRIBED_FROM_REPOSITORY',
🟩 39         description: 'a Lifeguard unsubscribed from the Repository',
🟩 40       },
🟩 41       LIFEGUARD_DEMOTED_TO_SWIMMER: {
🟩 42         code: 'LIFEGUARD_DEMOTED_TO_SWIMMER',
🟩 43         description: 'A Lifeguard was demoted to Swimmer role by another Lifeguard',
🟩 44       },
🟩 45       SWIMMER_PROMOTED_TO_LIFEGUARD: {
🟩 46         code: 'SWIMMER_PROMOTED_TO_LIFEGUARD',
🟩 47         description: 'A Swimmer was promoted to Lifeguard role by another Lifeguard',
🟩 48       },
🟩 49       REPOSITORY_MD_EXPORT_ENABLED: { code: 'REPOSITORY_MD_EXPORT_ENABLED', description: 'Enable MD export for repository' },
🟩 50       REPOSITORY_MD_EXPORT_DISABLED: { code: 'REPOSITORY_MD_EXPORT_DISABLED', description: 'Disabled MD export for repository' },
🟩 51     
🟩 52       OPENED_REPOSITORY_INTEGRATIONS: { code: 'OPENED_REPOSITORY_INTEGRATIONS', description: 'Viewed integrations page' },
🟩 53     
🟩 54       // SWIMMER_STATUS (PROGRESS) CHANGES
🟩 55       PLAYLIST_SWIMMER_STATUS_CREATED: { code: 'PLAYLIST_SWIMMER_STATUS_CREATED', description: 'Playlist swimmer`s status created' },
🟩 56       PLAYLIST_SWIMMER_STATUS_DELETED: { code: 'PLAYLIST_SWIMMER_STATUS_DELETED', description: 'Playlist swimmer`s status deleted' },
🟩 57       PLAYLIST_SWIMMER_STATUS_UPDATED: { code: 'PLAYLIST_SWIMMER_STATUS_UPDATED', description: 'Playlist swimmer`s status updated' },
🟩 58       EXERCISE_SWIMMER_STATUS_UPDATED: { code: 'EXERCISE_SWIMMER_STATUS_UPDATED', description: 'Exercise swimmer`s status updated' },
🟩 59       DOC_SWIMMER_STATUS_UPDATED: { code: 'DOC_SWIMMER_STATUS_UPDATED', description: 'Doc swimmer`s status updated' },
🟩 60       EXTERNAL_LINK_SWIMMER_STATUS_CREATED: { code: 'EXTERNAL_LINK_SWIMMER_STATUS_CREATED', description: 'External link swimmer`s status created' },
🟩 61       EXTERNAL_LINK_SWIMMER_STATUS_DELETED: { code: 'EXTERNAL_LINK_SWIMMER_STATUS_DELETED', description: 'External link swimmer`s status deleted' },
🟩 62       EXTERNAL_LINK_SWIMMER_STATUS_UPDATED: { code: 'EXTERNAL_LINK_SWIMMER_STATUS_UPDATED', description: 'External link swimmer`s status updated' },
🟩 63     
🟩 64       // USER ACTIONS
🟩 65       USER_INVITED: { code: 'USER_INVITED', description: 'User invited a new person to the workspace' },
🟩 66       USER_SIGN_IN: { code: 'USER_SIGN_IN', description: 'User signed in' },
🟩 67       USER_SIGN_OUT: { code: 'USER_SIGN_OUT', description: 'User signed out' },
🟩 68       USER_RESET_PASSWORD: { code: 'USER_RESET_PASSWORD', description: 'User asked to reset password' },
🟩 69     
🟩 70       // GLOBAL ACTIONS
🟩 71       USER_RATED_US: { code: 'USER_RATED_US', description: 'User gave us rating' },
🟩 72     
🟩 73       // WORKSPACES ACTIONS
🟩 74       WORKSPACE_CREATED: { code: 'WORKSPACE_CREATED', description: 'Created a Workspace' },
🟩 75       WORKSPACE_DELETED: { code: 'WORKSPACE_DELETED', description: 'Deleted a Workspace' },
🟩 76       WORKSPACE_UPDATED: { code: 'WORKSPACE_UPDATED', description: 'Updated a Workspace' },
🟩 77       WORKSPACE_USER_JOINED_WORKSPACE: {
🟩 78         code: 'WORKSPACE_USER_JOINED_WORKSPACE',
🟩 79         description: 'a user joined the workspace',
🟩 80       },
🟩 81       WORKSPACE_USER_UNSUBSCRIBED_WORKSPACE: {
🟩 82         code: 'WORKSPACE_USER_UNSUBSCRIBED_WORKSPACE',
🟩 83         description: 'a workspace user has been unsubscribed from the workspace',
🟩 84       },
🟩 85       WORKSPACE_ADMIN_JOINED_WORKSPACE: {
🟩 86         code: 'WORKSPACE_ADMIN_JOINED_WORKSPACE',
🟩 87         description: 'a workspace admin joined the workspace',
🟩 88       },
🟩 89       WORKSPACE_ADMIN_UNSUBSCRIBED_WORKSPACE: {
🟩 90         code: 'WORKSPACE_ADMIN_UNSUBSCRIBED_WORKSPACE',
🟩 91         description: 'a  workspace admin has been unsubscribed from the workspace',
🟩 92       },
🟩 93     
🟩 94       WORKSPACE_ADMIN_DEMOTED_TO_WORKSPACE_USER: {
🟩 95         code: 'WORKSPACE_ADMIN_DEMOTED_TO_WORKSPACE_USER',
🟩 96         description: 'A Workspace Admin was demoted to Workspace User role by another Workspace Admon',
🟩 97       },
🟩 98       WORKSPACE_USER_PROMOTED_TO_WORKSPACE_ADMIN: {
🟩 99         code: 'WORKSPACE_USER_PROMOTED_TO_WORKSPACE_ADMIN',
🟩 100        description: 'A Workspace User was promoted to Workspace Admin role by another Workspace Admin',
🟩 101      },
🟩 102    
🟩 103      // ACTIONS ON RESOURCES IN WORKSPACE
🟩 104      PLAN_CREATED: { code: 'PLAN_CREATED', description: 'Created a Plan' },
🟩 105      PLAN_DELETED: { code: 'PLAN_DELETED', description: 'Deleted a Plan' },
🟩 106      PLAN_UPDATED: { code: 'PLAN_UPDATED', description: 'Updated a Plan' },
🟩 107    
🟩 108      // UPDATER ACTIONS
🟩 109      AUTO_UPDATE_REQUESTED: { code: 'AUTO_UPDATE_REQUESTED', description: 'user requested to update the app' },
🟩 110      AUTO_UPDATE_FAILED: { code: 'AUTO_UPDATE_FAILED', description: 'Autoupdate failed' },
🟩 111    
🟩 112      // FOLDER COMMENT
🟩 113      FOLDER_COMMENT_UPDATED: {
🟩 114        code: 'FOLDER_COMMENT_UPDATED',
🟩 115        description: 'A folder comment has been created / updated',
🟩 116      },
🟩 117      FOLDER_COMMENT_DELETED: { code: 'FOLDER_COMMENT_DELETED', description: 'A folder comment has been deleted' },
🟩 118    
🟩 119      // UPVOTE ACTIONS
🟩 120      USER_UPVOTED_RESOURCE: { code: 'USER_UPVOTED_RESOURCE', description: 'A user up-voted a resource' },
🟩 121      USER_DOWNVOTED_RESOURCE: { code: 'USER_DOWNVOTED_RESOURCE', description: 'A user down-voted a resource' },
🟩 122    
🟩 123      // HUNKS UPDATE ACTIONS
🟩 124      AUTOSYNC_HUNK_ACCEPTED: { code: 'AUTOSYNC_HUNK_ACCEPTED', description: 'Autosynced hunk was accepted by the user' },
🟩 125      AUTOSYNCED_HUNK_RESELECTED: { code: 'AUTOSYNCED_HUNK_RESELECTED', description: 'Autosynced hunk was reselected and replaced with another hunk by the user' },
🟩 126      OUTDATED_HUNK_DELETED: { code: 'OUTDATED_HUNK_DELETED', description: 'Outdated hunk was deleted by the user' },
🟩 127      OUTDATED_HUNK_RESELECTED: { code: 'OUTDATED_HUNK_RESELECTED', description: 'Outdated hunk was reselected and replaced with another hunk by the user' },
🟩 128    };
```
<br/>

This is how we prepare our event log message before sending it to firestore.
Note the validation stage (done by using Joi) after building the message

<div>    📄 src/shared/event-logger/event-logger.ts  </div>

```ts
⬜ 1      import { eventSchema } from './logger-schema';
⬜ 2      import { SWIMM_EVENTS } from './swimm-events';
⬜ 3      import { ERROR_RETURN_CODE, SUCCESS_RETURN_CODE } from '../config';
⬜ 4      
⬜ 5      /**
⬜ 6       Validate event log message by schema
⬜ 7       */
⬜ 8      export function validateEventMessage(eventMessage) {
⬜ 9        try {
⬜ 10         const validationResult = eventSchema.validate(eventMessage);
⬜ 11         if (!validationResult.error) {
⬜ 12           return {
⬜ 13             code: SUCCESS_RETURN_CODE,
⬜ 14             eventMessage: eventMessage,
⬜ 15           };
⬜ 16         }
⬜ 17         return {
⬜ 18           code: ERROR_RETURN_CODE,
⬜ 19           error: validationResult.error,
⬜ 20         };
⬜ 21       } catch (error) {
⬜ 22         return { code: ERROR_RETURN_CODE, error: error };
⬜ 23       }
⬜ 24     }
⬜ 25     
⬜ 26     /**
⬜ 27      Builds and validates the event message from basic user log message, app_version and user data
⬜ 28      */
🟩 29     export function prepareEventLogMessage({ logMessage, userData, version }) {
🟩 30       try {
🟩 31         const eventMessage = buildEventMessage({
🟩 32           userMessage: logMessage,
🟩 33           user: userData,
🟩 34           version: version,
🟩 35         });
🟩 36         return validateEventMessage(eventMessage);
🟩 37       } catch (error) {
🟩 38         return { code: ERROR_RETURN_CODE, error: error };
🟩 39       }
🟩 40     }
⬜ 41     
⬜ 42     /**
⬜ 43      Builds event log message
⬜ 44      */
⬜ 45     export function buildEventMessage({ userMessage, user, version }) {
⬜ 46       const event = SWIMM_EVENTS[userMessage.swimmEventCode] ? SWIMM_EVENTS[userMessage.swimmEventCode] : SWIMM_EVENTS.UNKNOWN;
⬜ 47       return {
⬜ 48         ...userMessage,
⬜ 49         swimmEventCode: event.code,
⬜ 50         actionName: event.description,
⬜ 51         app_version: version,
⬜ 52         userId: user.uid,
⬜ 53         userName: user.nickname,
⬜ 54       };
⬜ 55     }
```
<br/>

This is the security rule related to the event logger.
Since the eventlogger schema is not updated automatically for now, whenever updating one of the values here - remember to update also the related value in the eventLog schema.

<div>    📄 cloud/firestore.rules  </div>

```rules
⬜ 33     
🟩 34         match /event_logs/{event_log} {
🟩 35           // Read is allowed only for admins
🟩 36           allow create, update: if validateEventLogWhitelistFieldsForRequest() && (request.auth.uid == request.resource.data.userId);
🟩 37           // Deletion is not allowed from the client
🟩 38     
🟩 39           function validateEventLogWhitelistFieldsForRequest() {
🟩 40             return request.resource.data.keys().toSet().hasOnly(["swimmEventCode", "message", "value", "category", "userId", "userName", "actionName", "app_version", "workspaceId", "workspaceName", "planId", "planName", "repoId", "repoName", "playlistId", "playlistName", "srcId", "srcName", "srcType", "targetId", "targetName", "created"]);
🟩 41           }
🟩 42         }
⬜ 43     
```
<br/>

A good-to-know about this mixin is the usage of the `user` auth data. If this mixin is used by another mixin, then the `auth` store won't be accesible here by `mapState`. Therefor if called from a mixin, the caller should send the user object alongside the logMessage

<div>    📄 src/app/common/mixins/event-logger.js  </div>

```js
🟩 14       computed: {
🟩 15         ...mapState('auth', ['user']),
🟩 16       },
🟩 17       methods: {
🟩 18         async logEvent(logMessage, user) {
```
<br/>

This is the logEvent common `mixin` that we use through our FE code in order to send the events to the DB 
The event payload is being sent from the caller and being parsed to our "firestore friendly" event-log message structure. 
Note that for validation errors we send an error to the local logger

<div>    📄 src/app/common/mixins/event-logger.js  </div>

```js
⬜ 15         ...mapState('auth', ['user']),
⬜ 16       },
⬜ 17       methods: {
🟩 18         async logEvent(logMessage, user) {
🟩 19           // Reset password event happens without "auth". Therefore a special handling is needed
🟩 20           const eventCode = logMessage.swimmEventCode || '';
🟩 21           if (eventCode === SWIMM_EVENTS.USER_RESET_PASSWORD) {
🟩 22             return logForgotPassword(logMessage);
🟩 23           }
🟩 24           try {
🟩 25             const routeData = this.getRouteCollectionMetadata();
🟩 26             const logWithRoutedata = { ...routeData, ...logMessage };
🟩 27             const userData = user ? { ...user } : this.user; // When called from store modules, auth mapState will be undefined.
🟩 28             const createLogMessage = prepareEventLogMessage({ logMessage: logWithRoutedata, userData: userData, version: PJSON_VERSION });
🟩 29             if (createLogMessage.code === SUCCESS_RETURN_CODE) {
🟩 30               await firebase
🟩 31                 .firestore()
🟩 32                 .collection('event_logs')
🟩 33                 .add({ ...createLogMessage.eventMessage, created: firebase.firestore.FieldValue.serverTimestamp() });
🟩 34               return SUCCESS_RETURN_CODE;
🟩 35             }
🟩 36             logger.error(`could not prepare event message for sending event log ${eventCode}. Details: ${createLogMessage.error.toString()}`, { service: 'event-log' });
🟩 37             return ERROR_RETURN_CODE;
🟩 38           } catch (error) {
🟩 39             logger.error(`could not send event log ${eventCode}. Details: ${error.toString()}`, { service: 'event-log' });
🟩 40             return ERROR_RETURN_CODE;
🟩 41           }
🟩 42         },
🟩 43       },
⬜ 44     };
⬜ 45     
⬜ 46     const logForgotPassword = (userMessage) => {
```
<br/>

A usage example for the `event-logger` from a FE component.

<div>    📄 src/app/common/components/organisms/PlaylistCard.vue  </div>

```js
⬜ 124        async deletePlaylist() {
⬜ 125          this.deleting = true;
⬜ 126          const userResponse = await swal({
⬜ 127            title: 'Are you sure you want to delete this playlist?',
⬜ 128            buttons: {
⬜ 129              cancel: true,
⬜ 130              confirm: true,
⬜ 131            },
⬜ 132          });
⬜ 133          if (!userResponse) {
⬜ 134            this.deleting = false;
⬜ 135            return false;
⬜ 136          }
⬜ 137          try {
⬜ 138            await this.archiveResource({ resourceName: 'playlists', resourceId: this.playlist.id, containerDocId: this.repoId });
🟩 139            this.logEvent({
🟩 140              swimmEventCode: SWIMM_EVENTS.PLAYLIST_DELETED.code,
🟩 141              repoId: this.repoId,
🟩 142              repoName: this.getRepoMetadata(this.repoId).name || '',
🟩 143              srcId: this.playlist.id,
🟩 144              srcName: this.playlist.name,
⬜ 145            });
⬜ 146          } catch (error) {
⬜ 147            console.error(`could not delete the playlist: ${error}`);
⬜ 148          }
⬜ 149          this.deleting = false;
⬜ 150        },
```
<br/>

A usage example for the `event-logger` from a FE mixin (note the extra user argument)

<div>    📄 src/app/common/mixins/storeModulesWrapper.js  </div>

```js
🟩 66           await this.createRepoLifeguard(this.user, newRepoId);
🟩 67           const userName = this.user.nickname || this.user.name;
🟩 68           this.logEvent(
🟩 69             { swimmEventCode: SWIMM_EVENTS.LIFEGUARD_SUBSCRIBED_TO_REPOSITORY.code, repoId: newRepoId, repoName: newRepo.name, srcId: this.user.uid, srcName: userName },
🟩 70             this.user
🟩 71           );
```
<br/>

In the CLI, we user `axios` to send the event to firestore since we do not use the `firebase-tools` (firebaseAdmin) there. The logic is very similar to the one of the FE

<div>    📄 src/cli/utils/cli-utils.ts  </div>

```ts
🟩 291    export async function logEvent({ logMessage, user }) {
🟩 292      if (config.isTest) {
🟩 293        return config.SUCCESS_RETURN_CODE;
🟩 294      }
🟩 295    
🟩 296      try {
🟩 297        const createLogMessage = prepareEventLogMessage({ logMessage: logMessage, userData: user, version: config.PJSON_VERSION });
🟩 298        if (createLogMessage.code === config.SUCCESS_RETURN_CODE) {
🟩 299          return axios
🟩 300            .post(`${config.DB_REST_API_ADDRESS}/event_logs`, createFirestorePostMessage(createLogMessage.eventMessage), {
🟩 301              headers: { Authorization: `Bearer ${state.get('token')}` },
🟩 302            })
🟩 303            .then(() => {
🟩 304              return config.SUCCESS_RETURN_CODE;
🟩 305            })
🟩 306            .catch((err) => {
🟩 307              const errMessage = err.response.data && err.response.data.error ? err.response.data.error.message : 'UNKNOWN';
🟩 308              logger.error(`Writing log message to DB failed. status-code: ${err.response.status}; details: ${errMessage}`, { service: 'event-logger' });
🟩 309              return config.ERROR_RETURN_CODE;
🟩 310            });
🟩 311        }
🟩 312        logger.error(`Log event failed due to prepare message error: ${createLogMessage.error}`, { service: 'event-logger' });
🟩 313      } catch (error) {
🟩 314        logger.error(`Log event failed. details: "${error}"`, { service: 'event-logger' });
🟩 315      }
🟩 316      return config.ERROR_RETURN_CODE;
🟩 317    }
```
<br/>

If there are any question about the event logger, talk to @Daniel or @Eden to learn more.

and if you find another interesting example to document about - add it to this unit 😊

<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRk5sdnNjMXNhc0JrTmx4ZWZES28z). Timestamp: 2021-04-24T18:52:02.201Z
