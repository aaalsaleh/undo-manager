## UndoManager

An undo manager implementation in JavaScript. It follows the [W3C UndoManager and DOM Transaction Draft][1] and the undocumented and disabled [Mozilla Firefox's UndoManager][2] implementation.

### Loading

The undo manager is web browser, AMD, and CommonJS compatible.

#### Web Browser

```html
<script type="text/javascript" src="undo-manager.min.js"></script>
```

#### AMD

The module is named "undo-manager", but can be easily made an anonymous module by removing the name from the define statement.

```javascript
require(['undo-manager', ...], function(UndoManager, ...) {
	...
});
```

#### CommonJS

```javascript
var UndoManager = require('undo-manager');
```

### Usage

#### Initialization

The undo manager can be instantiated using:

```javascript
var undoManager = new UndoManager(limit, undoScopeHost)
```

It can also be attached to the document:

```javascript
document.undoManager = new UndoManager(limit, document)
```

Similarly, it can be attached to any DOM element:

```javascript
element.undoManager = new UndoManager(limit, element)
```

The `limit` argument is optional, and can be used to set a limit on the number of undo/redo items on the stack. However, if `limit` was not passed, or was `0`, then there will be no limit (not recommended).

The `undoScopeHost` argument is optional, and can be used to set the undo scope host which will be the source of the fired events. However, if `undoScopeHost` was not passed, or was `null`, then no events will be executed (see **Events** section).

#### Transactions

To push and execute a transaction, the method `undoManager.transact` must be called by passing a transaction object as the first argument, and a merge flag as the second argument. A transaction object has the following properties:

- `label`: a string describing the transaction. Can be useful to determine the type of the transaction, or for displaying it in UI (optional).
- `execute`: a function defining what the transaction executes.
- `undo`: a function defining what should be executed once an undo is requested.
- `redo`: a function defining what should be executed once a redo is requested.

If preferred, any transaction can be merged with the previous transaction(s) by setting the merge argument to `true`. Otherwise, passing `false`, it will be a separate transaction. Merged transactions are grouped as a single transaction when performing undo or redo.

Once a transaction is passed to the method `undoManager.transact`, the method `execute` of the transaction will be executed automatically by the undo manager. There is no need to call the method `execute` manually.

```javascript
var undoManager = new UndoManager(10);

undoManager.transact({
    label: 'Typing',
    execute: function() { ... },
    undo: function() { ... },
    // redo same as execute
    redo: function() { this.execute(); }
}, false);

// merge transaction
undoManager.transact({
    label: 'Typing',
    execute: function() { ... },
    undo: function() { ... },
    // redo same as execute
    redo: function() { this.execute(); }
}, true);
```

#### Undo and Redo

The method `undoManager.undo()` unapplies the last transaction, while the method `undoManager.redo()` reapplies the last transaction. Merged transactions are grouped as a single transaction, and will be unapplied and reapplied as a group once undo or redo is requested.

The property `undoManager.position` indicates the current position in the transactions stack. The property `undoManager.length` indicates the total number of transactions. These two properties should not be changed manually. The current positions always satisfies the condition `0 <= undoManager.position <= undoManager.length`. The condition `undoManager.position < undoManager.length` checks whether undo can be performed or not, while `undoManager.position > 0` checks whether redo can be performed or not. If `undoManager.positon == undoManager.length`, undo is not possible. If `undoManager.position == 0`, redo is not possible.

To access transactions, `undoManager.item(n)` gets the nth transaction on the stack. If `0 <= n < undoManager.length`, then the returned value is an array of one or more merged transactions. Otherwise, `null` will be returned. The returned array is always a new array, and modifying it will not have an effect on the original transactions array. However, modifying the transactions properties within the array will change their behaviours.

To clear undo transactions, `undoManager.clearUndo()` can be used. Similarly, to clear redo transactions, `undoManager.clearRedo()` can be used. To clear all transactions, both methods can be called.

```javascript
var undoManager = new UndoManager(10);

undoManager.transact(...);
undoManager.transact(...);

// can undo?
if(undoManager.position < undoManager.length) {
    // log current undo transaction's label
    console.log(undoManager.item(undoManager.position)[0].label);
    // perform undo
    undoManager.undo();
}

// can redo?
if(undoManager.position > 0) {
    // log current redo transaction's label
    console.log(undoManager.item(undoManager.position - 1)[0].label);
    // perform redo
    undoManager.redo();
}

// clear undo and redo transactions
undoManager.clearUndo();
undoManager.clearRedo();
```

#### Events

If `undoScopeHost` is specified, then the undo manager will fire three types of events from the specified `undoScopeHost`. No events will be fired if `undoScopeHost` was not specified, or event dispatching was not supported.

- `DOMTransaction` event: it will be fired whenever a transaction is pushed and executed using `undoManager.transact`.
- `undo` event: it will be fired whenever an undo is performed using `undoManager.undo()`. Note that it will be fired only once for merged transactions.
- `redo` event: it will be fired whenever a redo is performed using `undoManager.redo()`. Note that it will be fired only once for merged transactions.

All three events will `bubble`, and are not `cancelable`. Also, all three events will include the performed transactions under the event property `event.detail.transactions` as an array of one or more merged transactions.

```javascript
// document is specified as an undoScopeHost
document.undoManager = new UndoManager(10, document);

// register events listeners
document.addEventListener("DOMTransaction", function(event) {
    console.log("DOMTransaction", event.detail.transactions);
});
document.addEventListener("undo", function(event) {
    console.log("undo", event.detail.transactions);
});
document.addEventListener("redo", function(event) {
    console.log("redo", event.detail.transactions);
});

// fires a transaction event
document.transact(...);

// fires an undo event
document.undo();

// fires a redo event
document.redo();
```

### Examples

#### DOM Element

```javascript
// #editor is a div with contenteditable="true"
var editor = document.querySelector('#editor');

editor.undoManager = new UndoManager(10, editor);

function EditingTransaction(label, html) {
	this.label = label;
	this.oldHTML = editor.innerHTML;
	this.newHTML = html;
	this.execute = function () {
		editor.innerHTML = this.newHTML;
	};
	this.undo = function () {
		editor.innerHTML = this.oldHTML;
	};
	this.redo = this.execute;
}

editor.undoManager.transact(new EditingTransaction('Typing', 'Hello '), false);
editor.undoManager.transact(new EditingTransaction('Typing', 'Hello World!'), false);
editor.undoManager.transact(new EditingTransaction('Typing', 'Hello World! 1'), false);
editor.undoManager.transact(new EditingTransaction('Typing', 'Hello World! 12'), true);
editor.undoManager.transact(new EditingTransaction('Typing', 'Hello World! 123'), true);
editor.undoManager.transact(new EditingTransaction('Bold', 'Hello <b>World</b>! 123'), false);

// Hello <b>World</b>! 123
editor.undoManager.undo();
// Hello World! 123
editor.undoManager.undo();
// Hello World!
editor.undoManager.undo();
// Hello 
editor.undoManager.redo();
// Hello World!
editor.undoManager.redo();
// Hello World! 123
```

#### Custom Objects

```javascript
var undoManager = new UndoManager(5);

// shallow object mutation
function ObjectMutation(obj, mutation) {
	this.label = 'Property Change';

	this.oldObj = {};
	for (var p in obj)
		if (obj.hasOwnProperty(p))
			this.oldObj[p] = obj[p]

	this.execute = function() {
		for (var p in mutation)
			if (mutation.hasOwnProperty(p))
				obj[p] = mutation[p];
	};

	this.undo = function() {
		for (var p in this.oldObj)
			if (this.oldObj.hasOwnProperty(p))
				obj[p] = this.oldObj[p];
	}

	this.redo = this.execute;
}

var x = {color: 'red', size: 3};
var y = {color: 'blue', size: 5};
var z = {color: 'green', size: 10};

undoManager.transact(new ObjectMutation(x, {color: 'white'}), false);
// x: {color: 'white', size: 3}
undoManager.transact(new ObjectMutation(y, {color: 'black', size: 0}), false);
// y: {color: 'black', size: 0}
undoManager.transact(new ObjectMutation(z, {size: 15}), false);
// z: {color: 'green', size: 15}

undoManager.undo();
// z: {color: 'green', size: 10};
undoManager.undo();
// y: {color: 'blue', size: 5};
undoManager.redo();
// y: {color: 'black', size: 0}
```

#### Editor with Input and Undo/Redo Shortcuts

```javascript
// #editor is a div with contenteditable="true"
var editor = document.querySelector('#editor');

editor.undoManager = new UndoManager(10, editor);

var html = editor.innerHTML;
var merge = false;
var timer = null;

// warning: some browsers do not support input event
editor.addEventListener('input', function(e) {
	if(editor.innerHTML != html) {
		clearTimeout(timer);
		timer = setTimeout(function() { merge = false; }, 1000);

		editor.undoManager.transact({
			oldHTML: html,
			newHTML: editor.innerHTML,
			// nothing to execute (content already changed when input is fired)
			execute: function() { },
			undo: function() {
				editor.innerHTML = html = this.oldHTML;
			},
			redo: function() {
				editor.innerHTML = html = this.newHTML;
			}
		}, merge);

		html = editor.innerHTML;
		merge = true;
	}

});

// warning: some browsers do not allow overriding native shortcuts
editor.addEventListener('keydown', function(e) {
	if ((e.ctrlKey || e.metaKey) && e.keyCode === 90) {
		if (e.shiftKey)
			editor.undoManager.redo(); // Ctrl/Command + Shift + Z
		else
			editor.undoManager.undo(); // Ctrl/Command + Z

		e.preventDefault();
	}
});
```

### Limitations

Compared to the [W3C UndoManager and DOM Transaction Draft][1], the following limitations and inconsistencies are known:

- The undo manager must be instantiated and attached to the document and DOM elements manually if needed.
- The `limit` is not specified in the draft. It was added to limit the memory usage, and allow flexibility.
- The DOM element's attribute `undoScope` is not supported. The `undoScopeHost` must be specified explicitly, and any `undoScope` attribute will be ignored.
- The undo manager's properties `position` and `length` are not read-only. Although they can be made read-only, they were left to ensure highest browsers compatibility.
- The transaction method `executeAutomatic` is not supported. While it can be implemented, the implementation will not be efficient. It is advisable to use `execute`, `undo`, and `redo` instead.
- The fired events are not native `DOMTransactionEvent`, but `CustomEvent` with the appropriate types.
- The fired events have the event property `transaction` under `event.detail.transactions` instead of `event.transaction`. Notice the use of plural form `transactions` as implemented by Mozilla Firefox.
- Unlike what is specified in the draft, the event property `event.detail.transactions` is not a single transaction object. Instead, it is an array of one or more merged transactions as implemented by Mozilla Firefox.

### License

[The MIT License (MIT)][3].

[1]:https://dvcs.w3.org/hg/undomanager/raw-file/tip/undomanager.html
[2]:https://bugzil.la/617532
[3]:LICENSE
