Using plugins as controllers for Ext.Components
===============================================

ExtJs with MVC
==============

Thanks Dave! Do you think you could kick off with a brief sentence that sets a little context? Like, "Here at Rally we've been using MVC for ....." Or, "When we're building X, one of the things we like about Sencha is ...."

First off, "MVC" in this post refers to unopinionated, generic MVC. View == UI, Model == application logic, Controller == glue between View and Model. "Component" refers to a class that extends [Ext.Component](http://docs.sencha.com/extjs/4.1.3/#!/api/Ext.Component).

There are 2 popular ways to implement MVC with ExtJs applications. Sencha provides an [MVC implementation](http://docs.sencha.com/extjs/4.1.3/#/guide/application_architecture) distributed with ExtJs and [DeftJs](https://github.com/deftjs/DeftJS/) is a third-party  library. I have experimented with both of these MVC implementations, and found them lacking.

Sencha's built-in MVC implementation requires a specific directory structure and configuration with Ext.application. These restrictions make it difficult to adapt the pattern to existing code bases.

Sencha's controllers also encourage code that listens for events fired from a Container's child Components. This is a code smell. Controller code should never "know" about child Components, because refactorring the UI structure will break the controller. Replacing a text input with a slider shouldn't force you to change controller code.

DeftJs is easier to adapt to existing code, but there are some odd object life-cycle issues that make it difficult to work with. Components must mixin Deft's `Controllable` class and DeftJs controllers must be configured when a Component is defined, rather than when it is instantiated. If you find the need to re-use a Component with two different controllers, you will need to create two separate classes, each defined with a different DeftJS controller.

A Deft controller's `init` method is not called until after the Component has been rendered, which makes things rather challenging if the controller needs to hook into any pre-render events such as `staterestore` or `render`.  

Plugins as Controllers
======================

I've found that standard Ext [Plugins](http://docs.sencha.com/extjs/4.1.3/#!/api/Ext.AbstractPlugin) are a great alternative to Sencha and DeftJs controllers. Plugins fit nicely into the Component life cycle. They can be configured at instantiation time, so reusing the same UI with different behavior is trivial.

Multiple Plugins can be configured for a single Component. This may seem like an odd property for a controller, but it's actually quite nice to split code into multiple controllers for Components that have many different behaviors.

A Plugin's `init` method is called during the Component's `constructor` call, so it can respond to any Component events fired during or after the Component's `initComponent` call. A Plugin's `destroy` method is called during the Component's `destroy` call, which allows for easy cleanup of listeners and models orchestrated by the Plugin.

So how does it Work?
====================

Wire up controller Plugins just like any other Plugin.

```javascript
	/**
	 * Controller Plugin
	 */
	Ext.define('MyViewController', {
		extend: 'Ext.AbstractPlugin',
		alias: 'plugin.myviewcontroller',

		/**
		 * @cfg {Ext.data.Store}
		 */
		store: null,

		init: function(cmp) {
			// add listeners to respond to UI events
			cmp.on('save', this.onSave);
		},

		onSave: function(cmp, data) {
			// respond to UI events by calling public view and model methods
			cmp.setLoading(true);
			this.store.add(new this.store.model(data))
			this.store.sync({
				success: function() {
					cmp.setLoading(false);
					cmp.showNotification('Record Saved');
				}
			});
		}	
	});
```

```javascript
	/**
	 * View Component
	 */
	Ext.define('MyView', {
		extend: 'Ext.Component',
		alias: 'widget.myview',

		items: [{
			xtype: 'button',
			label: 'Save'
		}],

		initComponent: function() {
			this.callParent(arguments);

			this.addEvents(['save']);

			// button click is abstracted into semantic event to avoid
			// controller having to know UI implementation details
			this.down('button').on('click', function() {
				this.fireEvent('save', {...});
			});
		},

		showNotification: ...
	});
```

```javascript
	// instantiate view with controller configuration
	var cmp = container.add('myview', {
		plugins: [{
			ptype: 'myviewcontroller',
			store: myStore
		}]
	});
```
Similarly, it'd be great to have a sentence or two at the end here to wrap things up -- like, "Going forward, ...." or "In the future, ...." Something like that.
