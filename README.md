# Fun with [funretro.io](https://funretro.io)

Just a few functions whipped up while bending the rules on https://funretro.io.

```js
function parseUrl() {
  const match = window.location.href.match(/board[/](?<teamId>[^/]+)[/](?<boardId>[^/?]+)/);
  if (match) {
    return match.groups;
  }
  throw new Error();
}

function getBoardId() {
  return parseUrl().boardId;
}

/** @returns [string] ID of the closest message/board/vote ID */
function getMessageId(element) {
  const match = closest(element, '[messageid]');
  return match ? match.getAttribute('messageid') : match;
}

/** @returns [null|Element] Closest match of selector from element to document root */
function closest(element, selector) {
  if (element === null) {
    return element;
  }

  const match = element.querySelector(selector);
  return match
    ? match
    : closest(element.parentElement, selector);
}

function getVoteCount(id) {
  const element = document.querySelector(`[messageid="${id}"] .show-vote-count`);
  return Number( element.innerText.trim() );
}

function getUserStorageKey() {
  const keys =
    Object.keys(localStorage)
    .filter(k => !k.startsWith('firebase'));

  if (keys.length === 0) {
    return getBoardId();
  }

  if (keys.length !== 1) {
    throw new Error();
  }

  return keys[0];
}

/** @returns { key: string, item: any } */
function getUserStorage() {
  const key = getUserStorageKey();
  return {
    key,
    item: JSON.parse( localStorage.getItem( key ) ) || {},
  };
}

function setUserStorage(key, obj) {
  localStorage.setItem(key, JSON.stringify(obj));
}

function setAvailableVotes(id, value) {
  const { key, item } = getUserStorage();
  item[id] = value;
  setUserStorage(key, item);
}

//------------------------------------------------------------- firebase -----

/** @returns [Promise<Message>] */
async function getMessages(boardId) {
  const messages =
    await firebase
      .database()
      .ref(`messages/${boardId}`)
      .once('value');

  return messages.toJSON();
}

/** @returns [String|null] */
function getCurrentUserId() {
  const user = firebase.auth().currentUser;
  return user ? user.uid : user;
}

/** @returns [Promise<String[]>] */
async function getTeamIds() {
  const userId = getCurrentUserId();
  const teams =
    await firebase
      .database()
      .ref(`/users/${userId}/teams`)
      .once('value');

  return Object.keys(teams.toJSON() || {});
}

/** @returns [Promise<Object>] */
async function getTeamMembers(teamId) {
  const team =
    await firebase
      .database()
      .ref(`/teams/${teamId}`)
      .once('value');

  const { members, teamName, user_id: managerId } = team.toJSON();
  return {
    ...members,
    [managerId]: `Manager of "${teamName}"`,
  };
}

/** @returns [Promise<Object>] */
async function getAllTeamMembers() {
  const teamIds = await getTeamIds();
  const members = await Promise.all(teamIds.map(getTeamMembers));
  return members.reduce(
    function merge(result, teamMembers) {
      for (const [ userId, emailAddress ] of Object.entries(teamMembers)) {
        if (typeof emailAddress !== 'string') throw new Error();

        result[userId]
          ? result[userId].add(emailAddress)
          : result[userId] = new Set([emailAddress])
      }
      return result;
    },
    {},
  );
}

/** @returns [Void] **/
async function addAuthors() {
  const boardId = getBoardId();
  const messages = await getMessages(boardId);
  const emailsByUserId = await getAllTeamMembers();

  Object.keys(messages)
  .forEach(id => {
    const messageUserId = messages[id].user_id;
    const emailAddresses = emailsByUserId[messageUserId];

    if (!emailAddresses) {
      debugger;
    }
    const author = emailAddresses
      ? Array.from(emailAddresses)[0]
      : 'Unknown';

    const front = document.querySelector(`[messageid="${id}"] .front`);
    let a = front.querySelector('.author');
    if (!a) {
      a = document.createElement('div');
      a.style.cssText = 'bottom: 0; font-size: 0.8em; left: 0; position: absolute;';
      a.className = 'author';
    }
    a.innerHTML = `> ${author}`;

    front.appendChild(a);
  })
}
```


With the functions above, you can tell **funretro.io** that you've made the votes on an inspected element:
```js
var id = getMessageId($0);
setAvailableVotes( id, getVoteCount(id) );

// The refresh is required to update Angular's data model
window.location.reload(true);
```

Add the authors to the messages on a board:
```
addAuthors()
``
