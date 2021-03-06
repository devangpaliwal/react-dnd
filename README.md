react-dnd
=========

HTML5 drag-and-drop mixin for React with full DOM control.  

*Note: both API and docs are in a bit of a flux now (haha). I'll try to get them stable and more easily understandable during the course of the following weeks, but meanwhile I appreciate feedback and corrections to the docs.*

## Prior Work

Check these first and see if they fit your use case.

* [react-draggable](https://github.com/mzabriskie/react-draggable) by [Matt Zabriskie](https://github.com/mzabriskie)
* [react-sortable](https://github.com/danielstocks/react-sortable) by [Daniel Stocks](https://github.com/danielstocks)

If they don't, read on.

## Installation

```
npm install --save react-dnd
```

Dependencies: Flux and a couple of functions from lodash-node;  
Peer Dependencies: React >= 0.11.0.

Note: [I'm using ES6 features in this library](https://github.com/gaearon/react-dnd/issues/2), so you may want to enable Harmony transforms in JSX build step. This library has to be used with a bundler such as Webpack (or Browserify).


## Rationale

Existing drag-and-drop libraries didn't fit my use case so I wrote my own. It's similar to the code we've been running for about a year on Stampsy.com, but rewritten to take advantage of React and Flux.

Key requirements:

* Emit zero DOM or CSS of its own, leaving it to the consuming components;
* Impose as little structure as possible on consuming components;
* Use HTML5 drag and drop as primary backend but make it possible to add different backends in the future;
* Like original HTML5 API, emphasize dragging data and not just “draggable views”;
* Hide [HTML5 API quirks](http://www.quirksmode.org/blog/archives/2009/09/the_html5_drag.html) from the consuming code;
* Different components may be “drag sources” or “drop targets” for different kinds of data;
* Allow one component to contain several drag sources and drop targets when needed;
* Make it easy for drop targets to change their appearance if compatible data is being dragged or hovered;
* Make it easy to use images for drag thumbnails instead of element screenshots, circumventing [browser quirks](http://stackoverflow.com/questions/7340898/html5-setdragimage-only-works-sometimes).

Hopefully the resulting API reflects that.

## Examples

>**While I'm working on better examples, [check out this fiddle](https://github.com/gaearon/react-dnd/issues/5#issuecomment-59856094).** It shows the most basic usage.  
>If you have specific uses of drag and drop in mind, [let me know so I can include them in examples](https://github.com/gaearon/react-dnd/issues/3).


### Simple Drag Source

First, declare types of data that can be dragged.  

These are used to check “compatibility” of drag sources and drop targets:

```javascript
// ItemTypes.js
module.exports = {
  BLOCK: 'block',
  IMAGE: 'image'
};
```

(If you don't have multiple data types, this libary may not be for you.)

Then, let's make a very simple draggable component that, when dragged, represents `IMAGE`:

```javascript
var { DragDropMixin } = require('react-dnd'),
    ItemTypes = require('./ItemTypes');

var Image = React.createClass({
  mixins: [DragDropMixin],
  
  configureDragDrop(registerType) {

    // Specify all supported types by calling registerType(type, { dragSource?, dropTarget? })
    registerType(ItemTypes.IMAGE, {

      // dragSource, when specified, is { beginDrag(), canDrag()?, endDrag(didDrop)? }
      dragSource: {

        // beginDrag should return { item, dragAnchors?, dragPreview?, dragEffect? }
        beginDrag() {
          return {
            item: this.props.image
          };
        }
      }
    });
  },
  
  render() {

    // {...this.dragSourceFor(ItemTypes.IMAGE)} will expand into
    // { draggable: true, onDragStart: (handled by mixin), onDragEnd: (handled by mixin) }.

    return (
      <img src={this.props.image.url}
           {...this.dragSourceFor(ItemTypes.IMAGE)} />
    );
  }
);
```

By specifying `configureDragDrop`, we tell `DragDropMixin` the drag-drop behavior of this component. Both draggable and droppable components use the same mixin.

Inside `configureDragDrop`, we need to call `registerType` for each of our custom `ItemTypes` that component supports. For example, there might be several representations of images in your app, and each would provide a `dragSource` for `ItemTypes.IMAGE`.

A `dragSource` is just an object specifying how the drag source works. You must implement `beginDrag` to return `item` that represents the data you're dragging and, optionally, a few options that adjust the dragging UI. You can optionally  `canDrag` to forbid dragging, or `endDrag(didDrop)` to execute some logic when the drop has (or has not) occured. And you can share this logic between components by letting a shared mixins generate `dragSource` for them.

Finally, you must use `{...this.dragSourceFor(itemType)}` on some (one or more) elements in `render` to attach drag handlers. This means you can have several “drag handles” in one element, and they may even correspond to different item types. (If you're not familiar with [JSX Spread Attributes syntax](https://gist.github.com/sebmarkbage/07bbe37bc42b6d4aef81), check it out).

### Simple Drop Target

Let's say we want `ImageBlock` to be a drop target for `IMAGE`s. It's pretty much the same, except that we need to give `registerType` a `dropTarget` implementation:

```javascript
var { DragDropMixin } = require('react-dnd'),
    ItemTypes = require('./ItemTypes');

var ImageBlock = React.createClass({
  mixins: [DragDropMixin],
  
  configureDragDrop(registerType) {

    registerType(ItemTypes.IMAGE, {

      // dropTarget, when specified, is { acceptDrop(item)?, canDrop(item)? enter(item)?, over(item)?, leave(item)? }
      dropTarget: {
        acceptDrop(image) {
          // Do something with image! for example,
          DocumentActionCreators.setImage(this.props.blockId, image);
        }
      }
    });
  },
  
  render() {

    // {...this.dropTargetFor(ItemTypes.IMAGE)} will expand into
    // { onDragEnter: (handled by mixin), onDragOver: (handled by mixin), onDragLeave: (handled by mixin), onDrop: (handled by mixin) }.

    return (
      <div {...this.dropTargetFor(ItemTypes.IMAGE)}>
        {this.props.image &&
          <img src={this.props.image.url} />
        }
      </div>
    );
  }
);
```

### Drag Source + Drop Target In One Component

Say we now want the user to be able to *drag out* an image out of `ImageBlock`. We just need to add appropriate `dragSource` to it and a few handlers:

```javascript
var { DragDropMixin } = require('react-dnd'),
    ItemTypes = require('./ItemTypes');

var ImageBlock = React.createClass({
  mixins: [DragDropMixin],
  
  configureDragDrop(registerType) {

    registerType(ItemTypes.IMAGE, {

      // Add a drag source that only works when ImageBlock has an image:
      dragSource: {
        canDrag() {
          return !!this.props.image;
        },
        
        beginDrag() {
          return {
            item: this.props.image
          };
        }
      }
  
      dropTarget: {
        acceptDrop(image) {
          DocumentActionCreators.setImage(this.props.blockId, image);
        }
      }
    });
  },
  
  render() {

    return (
      <div {...this.dropTargetFor(ItemTypes.IMAGE)}>

        {/* Add {...this.dragSourceFor} handlers to a nested node */}
        {this.props.image &&
          <img src={this.props.image.url}
               {...this.dragSourceFor(ItemTypes.IMAGE)} />
        }
      </div>
    );
  }
);
```

### What Else Is Possible?

I have not covered everything but it's possible to use this API in a few more ways:

* Use `getDragState(type)` and `getDropState(type)` to learn if dragging is active and use it to toggle CSS classes or attributes;
* Specify `dragPreview` to be `Image` to use images as drag placeholders (use `ImagePreloaderMixin` to load them);
* Say, we want to make `ImageBlock`s reorderable. We only need them to implement `dropTarget` and `dragSource` for `ItemTypes.BLOCK`.
* Suppose we add other kinds of blocks. We can reuse their reordering logic by placing it in a mixin.
* `dropTargetFor(...types)` allows to specify several types at once, so one drop zone can catch many different types.
* When you need more fine-grained control, most methods are passed drag event that caused them as the last parameter.

## API

### `require('react-dnd').DragDropMixin`

`configureDragDrop(registerType)`

Gives you a chance to configure drag and drop on your component.  
Components with `DragDropMixin` will have this method.

`registerType(type, { dragSource?, dropTarget? })`

Call this method to specify component behavior as drag source or drop target for given type.
This method is passed as a parameter to `configureDragDrop`.

`getDragState(type)`

Returns `{ isDragging: bool }` describing whether a particular type is being dragged from this component's drag source. You may want to call this method from `render`, e.g. to hide an element that is being dragged.

`getDropState(type)`

Returns `{ isDragging: bool, isHovering: bool }` describing whether a particular type is being dragged or hovered, when it is compatible with this component's drop source. You may want to call this method from `render`, e.g. to highlight drop targets when they are comparible and when they are hovered.

===================

### Drag Source API

Implement to specify drag behavior of a component.

* `beginDrag()` — return value must contain `item` with an object representing your data and may also contain `dragPreview: Image`, `dragAnchors`.

* `canDrag()` — optionally decide whether to allow dragging.

* `endDrag(didDrop)` — optionally handle end of dragging operation. `didDrop` is `false` if item was dropped outside compatible drop targets, or if drop target returned `false` from `acceptDrop`.

===================

### Drop Target API

Implement to specify drop behavior of a component.

* `enter(item)`, `leave(item)`, `over(item)` — optionally implement these to perform side effects (e.g. might use `over` for reordering items when they overlap). If you need to render different states when drop target is active or hovered, it is easier to use `this.getDropState(type)` in `render` method.

* `canDrop(item)` — optionally implement this method to reject some of the items.

* `acceptDrop(item)` — optionally implement this method to perform some action when drop occurs. If you explicitly return `false`, drop source will receive `false` as `didDrop` argument.

===================

### `require('react-dnd').ImagePreloaderMixin`

Used to preload drag thumbnails images. In your component do this -

```javascript
mixins : [DragDropMixin,ImagePreloaderMixin],
// This method should return array of image urls for preloading
getImageUrlsToPreload(){
    return ['some-img-url1','some-img-url2'];
},

```

Above code will load the images after componentDidMount is executed.

===================


TODO: describe how to use it for preloading drag thumbnails


## Thanks

This library is a React port of an API, parts of which were originally written by [Andrew Kuznetsov](http://github.com/cavinsmith/).
