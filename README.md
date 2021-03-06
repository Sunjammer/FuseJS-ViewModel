
# FuseJS ViewModel API (proposal)

This repo will contain documentation, implementation, tests and examples for a new ViewModel API that we propose to add to a future version of Fuse if it turns out well.

This is an open source effort, pull requests are welcome.

## Synopsis

The `ViewModel` API is a plain JavaScript wrapper on top of the `Observable` API that provides a familiar looking component infrastructure to people coming from Vue, Angular, React and similar frameworks. Instead of having the user juggle raw observables, with all their unfamiliar oddities, `ViewModel` wraps component internals in a well defined structure where there is less room for errors and unstructured improvisation on a component-by-component basis. 

## Motivation

* Plain Observables are somewhat hard to teach, hard to learn and hard to debug. In particular, newbies are often confused by the asynchronous nature of observables and the fact that no data flows unles they are subscribed to.
* Many JS developers are more familiar with a stricter component model (Vue, React, Angular).
* The current (0.27) vanilla Fuse pattern encourages non-strict view model code, where `this` has a defined meaning in the root scope  and non-standard symbols are injected, conflicting with certain JS transpiler assumptions. This `ViewModel` class intends to wrap that up so we get complete strict mode.
* `ViewModel` can be implemented as a plain JS layer that results in a tree of Observables, requiring no new protocol between JS and UX.

## Usage

`ViewModel` objects are created by calling the `ViewModel` function:

	var ViewModel = require("FuseJS/ViewModel");

	var vm = ViewModel(module, { /* descriptor */ });

A `ViewModel` is passed the current `module` to manage lifetimes of event listeners and observable subscriptions automatically. It also gives access to the root object `this` and enables strict mode access to injected symbols.

In a common and recommended UX markup scenario, each `ux:Class` has its own `ViewModel` in `<JavaScript>`, which makes up the entire `module.exports`.

## Todo-app example

	<Page ux:Class="TodoPage">
		<JavaScript>
			var ViewModel = require("FuseJS/ViewModel");

			module.exports = ViewModel(module, { 

				states: {
					tasks: [],
					newTask: ""
				},

				computed: {
					tasksRemainingLabel: function (){
						var count = this.tasks.filter(function(x) { return !x.isDone; }).length;
						return "There are " + count + " remaining tasks."
					}
				},

				methods: {
					addTask: function() {
						tasks.add({ label: this.newTask, isDone: false });
						this.newTask = "";
					}
				},

				onchanged: {
					Parameter: function(p) {
						console.log("The parameter to this page changed to " + JSON.stringify(p));
					}
				}
			}
		</JavaScript>
		<Each Items="{tasks}">
			<StackPanel Orientation="Horizontal">
				<Text Value="{label}" />
				<Switch Value="{isDone}" />
			</StackPanel>
		</Each>
	</Page>


## States

The `states` section of the descriptor holds plain data variables that may change over the lifetime of the component.

	states: {
		tasks: [],
		newTask: ""
	} 

The `states` should only be modified by `methods` and `events`. The `ViewModel` will create a hidden `Observable` and expose
a property on the context object.

## Computed values

The `computed` section of the descriptor holds functions that compute values derived from the `states`. When states change, the `ViewModel` automatically
detects what `computed` functions need to re-evaluate. 

	computed: {
		tasksRemainingLabel: function (){
			var count = this.tasks.filter(function(x) { return !x.isDone; }).length;
			return "There are " + count + " remaining tasks."
		}
	}

## Context objects

Inside functions of computed values (as well as for *methods*), the `this` parameter refers to the *context* object of the `ViewModel`. The context object contains:

* All the *states* as properties where `get`/`set` manipulates the hidden observable.
* All *computed* states as properties where `get` returnes an up to date value.
* All *methods* as functions that are called on the context object.

## Observable lists

Sometimes we want state lists where we can perform incremental changes (such as `add` and `remove`) without invalidating the entire list. If we use
a regular JavaScript array to back our list, `ViewModel` can't detect incremental changes.

To create an observable list, you can initialize a `states` variable to an explicit `Observable` instance. 

	states: {
		friends: Observable()
	}

In this case, the given observable will be used directly in context object instead of a hidden one. Accessing `this.friends` in a computed property or method will return the `Observable`. 

## The `created` method

The `created` method is called when the `ViewModel` is initialized. You can e.g. use this to populate the state.

	created: function() {
		for (var i = 0; i < 30; i++) {
			this.friends.add({ 
				name: "Generic friend " + i,
				avatar: "Assets/avatar" + ((Math.random()*4)+1) ".png"
			})
		}
	}

## Methods

The `methods` section hold functions that the view can call to make logical operations on the component.

	methods: {
		addTask: function() {
			tasks.add({ label: this.newTask, isDone: false });
			this.newTask = "";
		}
	},


## Property change handlers

The `onchanged` section holds functions that react to changes in UX properties (`ux:Property`) on the component.

	onchanged: {
		Parameter: function(p) {
			console.log("The parameter to this page changed to " + JSON.stringify(p));
		}
	}

> The `onchanged` feature depends on proposed Fuse changes not yet in production.
