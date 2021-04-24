---
title: Database store
---

<div align="center">    <br/>    <div>DOC</div>    <h1>Database store</h1>    <br/>  </div>

### Files Used:
📄 src/app/common/store/database.js

📄 src/app/common/components/organisms/RepoCard.vue


<br/>

The Swimm app has two main data stores:
1. `database` - stores data from the DB(Firebase).
2. `filesystem` - stores local files data (`./swm` folder).

The stores are used to share data across the app while fetching it once.
In this unit we focus on the `database` store, you will learn how to store data in it and how to use it.


<br/>

In this step we declare the base structure of the store state. we will keep the user workspaces and repositories in the state so we could fetch them once from the DB and use the data across the app.

<div>    📄 src/app/common/store/database.js  </div>

```js
⬜ 14     const emptyRepoUpvotes = () => ({ swimm: {}, playlist: {} });
⬜ 15     const emptyWorkspaceUpvotes = () => ({ plan: {} });
⬜ 16     
🟩 17     const getDefaultState = () => ({
🟩 18       repositories: {},
🟩 19       workspaces: {},
🟩 20       invitedWorkspaces: {},
🟩 21       upvotes: emptyUpvotes(),
🟩 22       hasFetchedUserRepos: false, // to avoid fetching all swimmer repos twice in the same session
🟩 23       hasFetchedUserWorkspaces: false,
🟩 24       hasFetchedWorkspacesRepos: false,
🟩 25       hasFetchedWorkspacesInvites: false,
🟩 26       hasFetchedOpenSourceRepos: false,
🟩 27       hasFetchedUserUpvotes: false,
🟩 28       domainSettings: {},
🟩 29     });
🟩 30     
🟩 31     export default {
🟩 32       namespaced: true,
🟩 33       state: getDefaultState(),
⬜ 34       mutations: {
⬜ 35         RESET_STATE(state) {
⬜ 36           Object.assign(state, getDefaultState());
```
<br/>

In this step we declare the store mutations, this is the code that actually update the state data.
Note that we use the Vue.set function to update the state data, this is needed to make reactive components that will update when the data is updated.

<div>    📄 src/app/common/store/database.js  </div>

```js
⬜ 31     export default {
⬜ 32       namespaced: true,
⬜ 33       state: getDefaultState(),
🟩 34       mutations: {
🟩 35         RESET_STATE(state) {
🟩 36           Object.assign(state, getDefaultState());
🟩 37         },
🟩 38         SET_REPO_METADATA(state, args) {
🟩 39           if (!(args.repoId in state.repositories)) {
🟩 40             Vue.set(state.repositories, args.repoId, emptyRepo());
🟩 41           }
🟩 42           Vue.set(state.repositories[args.repoId], 'metadata', { ...args.resource, id: args.repoId });
🟩 43         },
🟩 44         SET_REPO_RESOURCE(state, args) {
🟩 45           if (!(args.repoId in state.repositories)) {
🟩 46             Vue.set(state.repositories, args.repoId, emptyRepo());
🟩 47           }
🟩 48           // For backward compability
🟩 49           if (args.resourceName === 'swimms' && !('type' in args.resource)) {
🟩 50             args.resource.type = 'unit';
🟩 51           }
🟩 52           if ('id' in args.resource) {
🟩 53             Vue.set(state.repositories[args.repoId][args.resourceName], args.resource.id, args.resource);
🟩 54           } else {
🟩 55             Vue.set(state.repositories[args.repoId], args.resourceName, args.resource);
🟩 56           }
🟩 57         },
⬜ 58         SET_REPO_SWIMMER(state, args) {
⬜ 59           if (!(args.repoId in state.repositories)) {
⬜ 60             Vue.set(state.repositories, args.repoId, emptyRepo());
```
<br/>

In this step we declare an action that fetches the data from the DB(Firestore) and calls the mutation to update the state data.

<div>    📄 src/app/common/store/database.js  </div>

```js
⬜ 321            originalValue: !!originalValue,
⬜ 322          });
⬜ 323        },
🟩 324        fetchRepository({ commit, state }, args) {
🟩 325          const { repoId } = args;
🟩 326          return new Promise((resolve, reject) => {
🟩 327            if (!(repoId in state.repositories)) {
🟩 328              firebase
🟩 329                .firestore()
🟩 330                .collection('repositories')
🟩 331                .doc(repoId)
🟩 332                .get()
🟩 333                .then(function (resourceRef) {
🟩 334                  commit('SET_REPO_METADATA', { repoId: resourceRef.id, resource: resourceRef.data() });
🟩 335                  resolve();
🟩 336                })
🟩 337                .catch(function (error) {
🟩 338                  console.error('Error getting documents: ', error);
🟩 339                  reject();
🟩 340                });
🟩 341            } else {
🟩 342              resolve();
🟩 343            }
🟩 344          });
🟩 345        },
⬜ 346        subscribeToRepository({ commit, dispatch, state }, args) {
⬜ 347          let { repoId, updateChildren = [] } = args;
⬜ 348          if (!state.repositories[repoId] || !state.repositories[repoId].subscribed) {
```
<br/>

Here we created a getter function that allows components to access a specific state data.

<div>    📄 src/app/common/store/database.js  </div>

```js
⬜ 1049       getSwimm: (state) => (repoId, swimmId) => state.repositories[repoId].swimms[swimmId],
⬜ 1050       getPlaylist: (state) => (repoId, playlistId) => state.repositories[repoId].playlists[playlistId],
⬜ 1051       getSwimmer: (state) => (repoId, swimmerId) => state.repositories[repoId].swimmers[swimmerId],
🟩 1052       getRepository: (state) => (repoId) => state.repositories[repoId],
⬜ 1053       getSwimmsInPlaylist: (state, getters) => (repoId, playlistId, type) => {
⬜ 1054         let swimmsInPlaylist = getters
⬜ 1055           .getPlaylist(repoId, playlistId)
```
<br/>

and finally the repo card component fetches the data it needs by calling the "fetchRepository" action from the database store.

<div>    📄 src/app/common/components/organisms/RepoCard.vue  </div>

```js
⬜ 72           return this.repositories[this.repoId];
⬜ 73         },
⬜ 74       },
🟩 75       async created() {
🟩 76         try {
🟩 77           if (!(this.repoId in this.repositories)) {
🟩 78             await this.fetchRepository({ repoId: this.repoId });
🟩 79           }
🟩 80           const containingWorkspace = this.getWorkspaceByRepo(this.repoId);
🟩 81           this.workspaceLogo = containingWorkspace && containingWorkspace.logo;
🟩 82         } catch (error) {
🟩 83           console.error(`Could not fetch public repo (${this.repoId}).`);
🟩 84           this.unavailable = true;
🟩 85         }
🟩 86         this.loading = false;
🟩 87       },
🟩 88       methods: {
🟩 89         ...mapActions('database', ['fetchRepository']),
🟩 90       },
⬜ 91     };
⬜ 92     </script>
⬜ 93     
```
<br/>

<br/><br/>

This file was generated by Swimm. [Click here to view it in the app](https://swimm.io/link?l=c3dpbW0lM0ElMkYlMkZyZXBvcyUyRnZlZXp2eEN1enBQclJMTFhXRDJFJTJGZG9jcyUyRnVFcDFkSWEzNjlZcExnbUlOQXJ5). Timestamp: 2021-04-24T18:52:02.357Z
