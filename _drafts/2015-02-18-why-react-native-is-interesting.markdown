---
layout: post
title: "Why React Native is Interesting"
date: 2015-02-18 15:10:00
tags: [UI, iOS, OS X, React Native]
---
[React Native](http://TODO) is a new library that lets you use JavaScript to build native UIs for iOS apps. That's probably great if you're a web developer, but what interests me about this library is not the JavaScript part. 

The two ideas in React Native that get me excited are:

1. You define the UI for your app in a declarative manner, as opposed to the imperative approach used in most other UI toolkits.
2. Whenever something changes in the state of your app, React generates a completely new UI to replace the old one.

This may seem a little odd -- and slow -- but React is pretty smart about it.

Even though UIKit is a fairly decent framework, user interface programming is still notoriously tricky and labor intensive. Any innovation in this area is welcome.

### What's wrong with UI programming?

With regular UIKit programming, you create a hierarchy (or tree structure) of `UIView` widgets and subclasses -- `UIButton`, `UILabel`, `UITableView`, and so on.

These objects send messages to each other and to a controlling object, the `UIViewController`.

This is a very imperative way of programming, where you give the computer step-by-step instructions on how to perform certain tasks.

> The button was tapped? OK, send a message to the view controller, which then tells the label to change its text. But only if this switch is in the "off" position, otherwise the label should say something else.
>
> The switch is toggled to "on"? OK, now change the label text again, but take into account whether that button was tapped before. And so on...

It quickly becomes hard to see which part of the UI state is changed where -- and when. If you're not careful, it's easy for parts of the UI to go out-of-sync with each other.

Here's a somewhat contrived example:

![UI of the example app](/images/ReactNative/example-app.png)

The top half of this app lets you calculate an area. The bottom portion adds a third dimension to that area to turn it into a volume. There are three actions that can change what is displayed on the screen:

1) Tapping the **Calculate Area** button takes the contents of the Width and Height text fields, performs the calculation, and puts the result into a label. How this result is presented depends on the position of the Metric Units switch.

In code that looks like this:

	@IBAction func areaButtonTapped(sender: AnyObject) {
	  updateAreaLabel(area())
	}
	
	private func area() -> Float {
	  let width = (widthTextField.text as NSString).floatValue
	  let height = (heightTextField.text as NSString).floatValue
	  return width * height
	}

	private func updateAreaLabel(area: Float) {
	  areaLabel.text = String(format: "Area: %g %@", area, metricUnitsSwitch.on ? "m2" : "square feet")
	}

2) Tapping the **Calculate Volume** button does the same thing but also uses the contents of the Depth text field, and puts the result into the second label.

	@IBAction func volumeButtonTapped(sender: AnyObject) {
	  updateVolumeLabel(volume())
	}
	
	private func volume() -> Float {
	  let depth = (depthTextField.text as NSString).floatValue
	  return area() * depth
	}

	private func updateVolumeLabel(volume: Float) {
	  volumeLabel.text = String(format: "Volume: %g %@", volume, metricUnitsSwitch.on ? "m3" : "cubic feet")
	}

3) Toggling the **Metric Units** switch changes the results from square meters to square feet. But it will only change the text in the result labels if a calculation was already performed previously.

	@IBAction func switchToggled(sender: AnyObject) {
	  if areaLabel.text != "" {
        updateAreaLabel(area())
	  }
	  if volumeLabel.text != "" {
        updateVolumeLabel(volume())
	  }
	}

In addition, when this screen is first displayed, it should set up the initial state of the views. That happens in `viewDidLoad()` (or if you prefer, `viewWillAppear()`).

	override func viewDidLoad() {
	  super.viewDidLoad()
	  areaLabel.text = ""
	  volumeLabel.text = ""
	}

Because this is such a simple app, it's not too hard to follow what's going on, but you can see that the logic for changing the UI is spread out over several different methods. 

With a more complex UI, this becomes complicated really fast.

### A primitive solution

One solution is to make a big `updateUI()` method that looks at your application data and correspondingly sets the properties for each of your views. This keeps the entire UI update logic in one central place, making it easy to understand.

	private func updateUI() {
	  let widthText = widthTextField.text
	  let heightText = heightTextField.text

	  if widthText == "" || heightText == "" {
        areaLabel.text = ""
        volumeLabel.text = ""
        return
	  }

	  let width = (widthText as NSString).floatValue
	  let height = (heightText as NSString).floatValue
	  let area = width * height
	  areaLabel.text = String(format: "Area: %g %@", area, metricUnitsSwitch.on ? "m2" : "square feet")

	  let depthText = depthTextField.text
	  if depthText == "" {
        volumeLabel.text = ""
        return
	  }

	  let depth = (depthText as NSString).floatValue
	  let volume = area * depth
	  volumeLabel.text = String(format: "Volume: %g %@", volume, metricUnitsSwitch.on ? "m3" : "cubic feet")
	}

Whenever there is a change in application state -- for example, the user toggles a switch or new data is received from the server -- you simply call this big `updateUI()` method and it reloads the entire UI. 

	override func viewDidLoad() {
	  super.viewDidLoad()
	  updateUI()
	}

	@IBAction func switchToggled(sender: AnyObject) {
	  updateUI()
	}

	@IBAction func areaButtonTapped(sender: AnyObject) {
	  updateUI()
	}

	@IBAction func volumeButtonTapped(sender: AnyObject) {
	  updateUI()
	}

Sounds simple enough, but there's a problem with this approach. You lose things like the current selection in an active text field or the scroll position of a table view, because you're overwriting the contents of these views with new data every time. That makes a really jarring experience for the user.

### Where React Native comes in

What React provides is essentially that big `updateUI()` method, except that it is smart and only updates those views that actually have changes. It leaves the other views alone. 

The trick is that React does not directly update the UIKit view hierarchy when there is new state. When I said that React generates a completely new UI, what I meant was that it generates a *virtual* representation of the UI.

[image: state --> react --> virtual UI]

This virtual UI is just a list of all the views and their attributes that represent the state of your app at a specific time. Examples of these attributes are:

1. content such as text for a label or image for an image view
2. styling such as colors and fonts
3. layout attributes that describe where the view should be positioned

This list does not contain actual `UIView` objects, only descriptions of what such `UIView`s would be like. (It's very similar to what is inside a nib file or a storyboard.)

Every time you tell React that your application's state has changed, it goes through this process and outputs a new virtual UI object.

Given this new virtual UI object that represents the app's current state, React compares it to the previous one. It only pushes the changes between the two to the actual UIKit view hierarchy. 

Examples of such changes are:

- disable the **Calculate Area** button
- change the text of that label over there to `"Area: 10 square feet"`
- add a new subview with a cool animation
- change the frame of view X to `((10, 20), (100, 50))`
- and so on...

Often you'll end up with only a couple of changes, making the view hierarchy update really fast. Views that don't have changes are left alone.

If you're familiar with the MVVM design pattern, you can think of this virtual UI as the *view-model*. It's nothing more than a description of the attributes of your UI views. The render and diffing pass is what applies these attributes to the actual UIKit view objects. So React Native is even easier to use than MVVM because you no longer have to worry about how the data from your view-model ends up in your views. And you get all the same benefits.

### Declarative, not imperative UI

With React Native you describe the UI for a particular screen of your app as follows:

[example]

This creates a hierarchy of so-called *components*. Each of these components corresponds to a `UIView` or a subclass. 

The `render()` method describes the data that this component needs to draw itself. If the view has subviews, then you specify its sub-components here. In the example, TODO.

Data is passed down from higher-level components to lower ones. Each component can then render itself with whatever data it needs. For example, a `Text` component (corresponds to a `UIlabel`) will typically have a `text` property but also `color`, `font`, `alignment`, and so on.

Layout attributes are also part of this (RN does not use Auto layout but a variant of flexbox, a CSS layout mechanism). 

Think of these components as a tree of transforms that take input data (the application state) and convert it into a particular representation of the UI. 

[diagram]

The output of this process is an object that describes the virtual UI.

### Events

Of course, views do more than just passively display data, they also provide interactivity. React allows for that by letting you pass event handlers (closures) down to the components. These event handlers typically change application state, which then triggers a re-render in return. 

### Advantages

The advantage is that describing what happens in your UI is now just a matter of creating these components and passing data into them. Updating the actual UIKit view hierarchy is done for you by the React framework.

You only have to care about the "what" of your UI -- what do you want to show on the screen in what circumstances -- rather than the "how". 

All the work to build these virtual UI objects can be done on a background thread. The changes are then placed in a queue and applied to the UIKit view hierarchy on the main thread. (This is very similar to what [AsyncDisplayKit](http://asyncdisplaykit.org) does.)

It uses the basic idea from functional programming that these render methods are pure functions: the same inputs will always result in the same output, without any side effects. Virtual DOM objects are immutable value types, and therefore easy to unit test. 

UIKit now becomes this black box, an implementation detail. You no longer need to know anything about UIViews (unless you're building your own custom views).

So that's what I like about RN. It's an interesting model for describing what goes on in your UI that should make UI code simpler to write and easier to understand. 

### But... JavaScript?!

Does this mean you now have to use JavaScript to write your apps if you want to take advantage of this new model? Not really. 

JavaScript does have some cool benefits. Because it's an interpreted language, you can edit your code without having to recompile it. It's like using Swift in a playground, but now you can do this kind of "live coding" on a running app. Pretty sweet.

However, I'd rather code in Swift, so I'm porting these ideas to a new framework that is purely Swift. It may turn out that Swift is better at this than JavaScript, or it might be worse. But it'll be a fun project in any case. :-)
