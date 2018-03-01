# Fun with funretro.io

Just a few functions whipped up while playing with https://funretro.io.

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
```


With the functions above, you can tell **funretro.io** that you've made the votes on an inspected element:
```
var id = getMessageId($0);
setAvailableVotes( id, getVoteCount(id) );
window.location.reload(true);
```