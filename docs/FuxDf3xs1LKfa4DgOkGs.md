---
title: Is Resource Editable ?🤔
---

<div align="center">    <br/>    <div>DOC</div>    <h1>Is Resource Editable ?🤔</h1>    <br/>  </div>

### Files Used:
📄 src/app/common/mixins/storeModulesWrapper.js

📄 src/app/adapters/local_state_GW.ts


<br/>

## *To Edit Or Not To Edit? This is the question.*

<img src="https://media.giphy.com/media/l4hLAnH7GC9XaQkAo/giphy.gif" width="480" height="336" ></img>

In our app, it's a really good question. 

Before allowing the user to preform CRUD actions, we need to validate 2 main things:
 
### Permissions

* is the user allowed to edit the resource?
* On our own Swimm Public Repos, we do not permit any swimmmer to edit the content, lifeguards only.
* Unfinished units cannot be edited by swimmers. It will ruin the surprise to see the solution before finishing the unit :)
* On the other hand, walkthroughs can be edited even if not done.

 ### Local Workspace State

Since we work with local files, we need to check that we can access the local FS and edit the files.
* We check that we are on the "active" repository ~= in the right local directory. If the current directory is of another repository OR just a random folder, we cannot save and commit the changes to the files there.
* We check the current branch - if the branch is our beloved `swimm-active-branch`, then we don't want the user to save the changes and then to lose them with running `swimm reset` or `swimm done` 

The validations logic is kept in 2 separated mixins 


<br/>

In this step we can see how we validate if the user is allowed to edit a resource. The validation is different by the resource type (playlist/unit) and the containing repository kind.
Most of the times, we use `isResourceEditable` to determine if a user should even see a create/edit option

<div>    📄 src/app/common/mixins/storeModulesWrapper.js  </div>

```js
🟩 370    export const isResourceEditable = {
🟩 371      computed: {
🟩 372        ...mapState('auth', ['user']),
🟩 373        ...mapState('database', ['repositories', 'domainSettings']),
🟩 374        ...mapGetters('database', ['isRepoLifeguard', 'isPublicRepo', 'getSwimmStatus', 'getPlaylistStatus', 'getSwimm', 'getPlaylist']),
🟩 375      },
🟩 376      mixins: [unitPlayMode],
🟩 377      methods: {
🟩 378        isResourceEditable(repoId, resourceId, resourceType) {
🟩 379          const resource = this.getResourceByType(repoId, resourceId, resourceType);
🟩 380          if (!resource || resource.is_example) {
🟩 381            return false;
🟩 382          }
🟩 383          if (this.isRepoLifeguard(repoId, this.user.uid) || this.isCurrentUserCreatorOfResource(resource)) {
🟩 384            return true;
🟩 385          }
🟩 386          if (!this.isPublicRepo(repoId)) {
🟩 387            return resourceType === 'unit' || (resourceType === 'playlist' && this.getPlaylistStatus(repoId, this.user.uid, resourceId));
🟩 388          }
🟩 389          return false;
🟩 390        },
🟩 391        isResourceDeletable(repoId, resourceId, resourceType) {
🟩 392          const resource = this.getResourceByType(repoId, resourceId, resourceType);
🟩 393          if (!resource || resource.is_example) {
🟩 394            return false;
🟩 395          }
🟩 396          return this.isRepoLifeguard(repoId, this.user.uid) || this.isCurrentUserCreatorOfResource(resource);
🟩 397        },
🟩 398        isCurrentUserCreatorOfResource(resource) {
🟩 399          return this.user.uid === resource.creator;
🟩 400        },
🟩 401        getResourceByType(repoId, resourceId, resourceType) {
🟩 402          if (resourceType === 'playlist') {
🟩 403            return this.getPlaylist(repoId, resourceId);
🟩 404          }
🟩 405          return this.getSwimm(repoId, resourceId);
🟩 406        },
🟩 407        isAllowedToCreateWorkspace() {
🟩 408          return !this.domainSettings.isWorkspaceCreationProhibited;
🟩 409        },
🟩 410      },
🟩 411    };
```
<br/>

Behind the scenes, we use the `localStateGW` adapter to get the metadata on the local repo in current context

<div>    📄 src/app/adapters/local_state_GW.ts  </div>

```ts
🟩 126    const getLocalRepoMetadata = () => {
🟩 127      let repoId = 'NONE';
🟩 128      let activeBranchName = 'NONE';
🟩 129      const activeUnit = state.isUnitStarted() ? state.isUnitStarted() : 'NONE';
🟩 130      try {
🟩 131        activeBranchName = gitwrapper.getActiveBranchName();
🟩 132        repoId = state.getCurrentActiveRepoId();
🟩 133        return { code: config.SUCCESS_RETURN_CODE, repoMetadata: { repoId: repoId, activeBranchName: activeBranchName, cwd: state.get('cwd'), activeUnit: activeUnit } };
🟩 134      } catch (error) {
🟩 135        logger.error(`Could not get some local metadata ${error.message}`, { service: 'adapter-state-GW' });
🟩 136        return { code: config.ERROR_RETURN_CODE, repoMetadata: {}, error: 'Error occurred while trying to fetch local metadata' };
🟩 137      }
🟩 138    };
```
<br/>

In this step we can see how we check the status of the local workspace.
Note that if required to, this method will also alert the user with the relevant error

<div>    📄 src/app/common/mixins/storeModulesWrapper.js  </div>

```js
⬜ 560        /**
⬜ 561         * Validates that the local workspace is ready for CRUD changes
⬜ 562         * @param repoId - the repoId currently shown to the user
⬜ 563         * @param repoName - the name of the current repository
⬜ 564         * @param repoCwd - the path to the repo locally
⬜ 565         * @param shouldAlert - if true, also alerts the user for the error
⬜ 566         * @param resourceName - the name of the current resource in context
⬜ 567         * @param action - the action that's currently being validated for
⬜ 568         * @return {Promise<{code: number, status: string}>} - returns whether CRUD can be preformed locally and the relevant state of the local state
⬜ 569         */
🟩 570        async isLocalWorkspaceValidForEdit({ repoId, shouldAlert = false, resourceName, action, repoName, repoCwd }) {
🟩 571          try {
🟩 572            if (!(await this.isRepoCurrentlyActive(repoId)).isActive) {
🟩 573              if (shouldAlert) {
🟩 574                await this.alertInactiveLocalRepo({ action: action, anotherCwd: repoCwd, currentRepo: repoName });
🟩 575              }
🟩 576              return { code: ERROR_RETURN_CODE, status: localWorkspaceStatuses.LOCAL_REPO_IS_INACTIVE };
🟩 577            }
🟩 578            if (await this.isSwimmActiveBranch()) {
🟩 579              if (shouldAlert) {
🟩 580                await this.alertActiveSwimm({ resourceName: resourceName, action: action });
🟩 581              }
🟩 582              return { code: ERROR_RETURN_CODE, status: localWorkspaceStatuses.SWIMM_BRANCH_ACTIVE };
🟩 583            }
🟩 584            return { code: SUCCESS_RETURN_CODE, status: localWorkspaceStatuses.LOCAL_WORKSPACE_VALID };
🟩 585          } catch (error) {
🟩 586            return { code: ERROR_RETURN_CODE, status: localWorkspaceStatuses.UNKNOWN_ERROR, errorMessage: error };
🟩 587          }
🟩 588        },
🟩 589      },
```
<br/>

### Some Notes
* For online resources (like plans) we also validate that the user has the required permissions before showing the "add" button. 
* Same goes for `ellipsis` edit/delete options for resources that a swimmer cannot edit
* It is recommended to read about the `firestore security rules` in order to learn more about how we secure our database from leaks and unwanted resource CRUDing.

<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRkZ1eERmM3hzMUxLZmE0RGdPa0dz). Timestamp: 2021-04-24T18:52:02.136Z
