# Custom Components

Custom components are a great way to reuse code / markup from within a HaxeUI application. They aren't strictly required for any HaxeUI application but they serve a great way to split up an application make it more modular and maintainable. With that in mind there are a fair few different methods that can be utilised to create custom components in HaxeUI.

## A note about `module.xml`

Its important to mention `module.xml` here and the role it plays in exposing HaxeUI component classes to the markup parser (eg: xml). If you are using HaxeUI _solely_ through code (ie, creating instances of components in haxe code manually) then `module.xml` need play no part. However, if you plan to use any of the markup parsing features (ie, creating instances of components and layouts in xml) then you need to be aware that these are exposed to HaxeUI via a `module.xml`. There are various ways to do this depending on the way that you are creating custom components but the general principle is that you create a `module.xml` file somewhere on your class path and let HaxeUI know about your custom components:

```xml
<module>
    <components>
        <component ... />
    </components>
</module>
```

The actual attributes used in the `<component/>` node vary depending on what you would like to expose and how, and these will be detailed below (when relevant). See "Modules" for more details about how modules work. 

## Using code

The simplest and most straight forward way to create a custom component in HaxeUI is to simply build it, and its entire hierarchy in plain haxe code. Although certainly the most straightforward method it is, by far, the most  laborious to maintain. Consider the following class:

```haxe
package custom;

class MyComponent extends HBox {
    public function new() {
        super();
        var textfield = new TextField();
        textfield.text = "0";
        addComponent(_textfield);
        
        var button = new Button();
        button.text = "-";
        button.onClick = function(e) {
            var n = Std.parseInt(textfield.text) - 1;
            textfield.text = Std.string(n);
        }
        addComponent(button);
        
        var button = new Button();
        button.text = "+";
        button.onClick = function(e) {
            var n = Std.parseInt(textfield.text) + 1;
            textfield.text = Std.string(n);
        }
        addComponent(button);
    }
}
```

This simple component will place a textfield with two buttons horizontally, clicking the buttons with increment / de-increment the value in the textfield. The purpose and functionality of this custom component is fairly simply to work out, however, with larger more complicated classes adding (or removing) from the custom component can be laborious and prone to errors.

#### `module.xml`

This custom component class will be fully available for use with haxe code. However, if you want to use it from markup you will need to let HaxeUI know about its existence.  This can be done by creating a `module.xml`and adding something similar to:

```xml
<module>
	<components>
    	<component class="custom.MyComponent" />
        <!-- alternative method for entire package
        <component package="custom" />
        -->
    </components>
</module>
```

This component will now be available to xml via `<mycomponent/>`.

trivia: when using xml in HaxeUI various operations are performed on node names to allow more flexibility and code style. For example, with the component above any of these node names would lead to the `custom.MyComponent`haxe class: `<mycomponent/>`, `<myComponent/>`, `<MyComponent/>`, `<my-component/>`

## Using a macro

A slight improvement on creating a custom component purely through code is to use a macro that would build the layout code for you. This will simplfy layout and component creation, consider the following xml file:

```xml
<hbox>
    <textfield id="textfield" text="0" />
    <button id="deinc" text="-" />
    <button id="inc" text="+" />
</hbox>
```

We can then use this xml in a haxe class similar to the following:

```haxe
class MyComponent extends HBox {
    public function new() {
        super();
        var ui = ComponentBuilder.fromFile("assets/my-component.xml");
        var textfield = ui.findComponent("textfield", TextField);
        ui.findComponent("deinc", Button).onClick = function(e) {
            var n = Std.parseInt(textfield.text) - 1;
            textfield.text = Std.string(n);
        }
        ui.findComponent("inc", Button).onClick = function(e) {
            var n = Std.parseInt(textfield.text) + 1;
            textfield.text = Std.string(n);
        }
        addComponent(ui);
    }
}
```

This new class has he exact same functionality except now the actual building of a component is delegate to a macro (which simply creates haxe code that adds components to a UI as if you had done it by hand). At first it may not seem much simpler (and indeed there are further refinements and simplifications that can be made below), but the important part here is that the UI layout itself is now coming from xml rather then written by hand. This means its extremely trivial (and easy to visualise) moving components around and creating new ones - something that becomes invaluable in larger applications / components. 

Note: exposing this classes using `module.xml`is exactly the same as if you created it using code in the first section above. 

## Using a build macro

A further refinement we can make to the `MyComponent`class is to use a build macro to build many parts of the class for us and eliminate alot of the boiler plate. Using the exact same xml file:

```xml
<hbox>
    <textfield id="textfield" text="0" />
    <button id="deinc" text="-" />
    <button id="inc" text="+" />
</hbox>
```

We can now create a custom component using a build macro similar to the following:

```haxe
@:build(haxe.ui.ComponentBuilder.build("assets/my-component.xml"))
class MyComponent extends HBox {
    public function new() {
        super();
        deinc.onClick = function(e) {
            var n = Std.parseInt(textfield.text) - 1;
            textfield.text = Std.string(n);
        }
        inc.onClick = function(e) {
            var n = Std.parseInt(textfield.text) + 1;
            textfield.text = Std.string(n);
        }
    }
}
```

The most important things to notice here is that we have now removed the need to add a UI to our custom component (`addComponent` in the previous example) as well as the need to perform any `findComponent` calls in order to access named components from the xml - any component with an `id` attribute will now be a correctly typed member variable of the class with that name. 

#### Using binding

Although not required a further refinement we can make here to remove boilerplate is the use of binding. By using binding metadata we can automatically link up values and events:

```haxe
@:build(haxe.ui.ComponentBuilder.build("assets/my-component.xml"))
class MyComponent extends HBox {
    @:bind(textfield.text)
    public var textfieldText:String = "10";
    
    @:bind(deinc, MouseEvent.CLICK)
    function onDeinc(e) {
        var n = Std.parseInt(textfieldText) - 1;
        textfieldText = Std.string(n);
    }    
    
    @:bind(inc, MouseEvent.CLICK)
    function onInc(e) {
        var n = Std.parseInt(textfieldText) + 1;
        textfieldText = Std.string(n);
    }    
}
```

#### Additional parameters to the build macro

There is an additional parameter that the build macro accepts which is an object representing parameters to use with this xml file, for example:

```xml
<hbox>
    <textfield id="textfield" text="${startValue}" />
    <button id="deinc" text="-" />
    <button id="inc" text="+" />
</hbox>
```

This variant of the xml file contains a `startValue` parameter, we can set that by using the following:

```haxe
@:build(haxe.ui.ComponentBuilder.build("assets/my-component.xml", {startValue: 10}))
class MyComponent extends HBox {
    ...
}
```

Though not hugely useful here its important to note that this can be used in various ways, for example as a way to create generic container layouts with different content (using the `<import/>` node, eg:

```xml
<vbox id="container">
    <label text="Container ${title}" />
    <import source="${content}" />
</vbox>
```

This is now a generic container that can have its title and content specified by different classes at compile time, for example:

```haxe
@:build(haxe.ui.ComponentBuilder.build("container.xml", {title: "container 1", content: "container1.xml"}))
class Container1 extends VBox {
	...       
}

@:build(haxe.ui.ComponentBuilder.build("container.xml", {title: "container 2", content: "container2.xml"}))
class Container2 extends VBox {
	...       
}

```

## Using xml metadata

Using xml metadata is very similar to using external xml files, with the exception that the xml source is specified using haxe metadata rather than an a file. This is useful for testing and small components but may become unwieldy when creating large custom components (separation of UI and logic is generally better). Using xml metadata our custom component would look like:

```haxe
@:xml('
<hbox>
    <textfield id="textfield" text="0" />
    <button id="deinc" text="-" />
    <button id="inc" text="+" />
</hbox>
')
class MyComponent extends HBox {
    ...
}
```

## Using an xml file

The final way to create custom components is to use _only_ and xml file, that is to say, no haxe code at all. This may or may not be an appropriate option, especially depending on how much logic the custom component performs (logic will be in the xml file as script and thus doesnt separate nicely from UI):

```xml
<hbox>
    <script>
        function inc(by) {
            var n = Std.parseInt(textfield.text) + by;
            textfield.text = Std.string(n);
        }
    </script>
   	<textfield id="textfield" text="0" />
    <button onClick="inc(-1)" text="-" />
    <button onClick="inc(1)" text="+" />
</hbox>

```

In order to use this xml file you must allow HaxeUI to know about it, this can be done via `module.xml´ similar to the following:

```xml
<module>
	<components>
    	<component file="custom/my-component.xml" />
        <!-- alternative method for entire package 
        <component folder="custom" />
        -->
    </components>
</module>
```

This component class will now be available using both code and markup, when using via code it would be `custom.MyComponent` and when using via markup with would simply be `<mycomponent />`

More information about this, and modules in general, can be found in the "Modules" sections



## Useful metadata

### @:composite

Sometimes you need to create a component that works both in native and composite.

In this case you use the `composite` metadata, to indicate which classes will be used only for composites



```haxe
@:composite(ButtonEvents, ButtonBuilder, ButtonLayout) 
class Button extends InteractiveComponent implements ICompositeInteractiveComponent {
    // this part will work both in native and composite backends
}

class ButtonEvents extends haxe.ui.events.Events {
    // this class won't be used in native components
    // where you register events
    // you can access it with component._internalEvents
}

class ButtonBuilder extends CompositeBuilder {
    // this class won't be used in native components
    // how you create the component, how you add/remove children components
    // you can access it with component._compositeBuilder
}

class ButtonLayout extends DefaultLayout {
    // this class won't be used in native components
    // how you reposition children
    // you can access it with component._defaultLayout
}
```

`@:composite(ButtonEvents, ButtonBuilder, ButtonLayout) `

Translates to 

```haxe
function registerComposite() {
	super.registerComposite();
	this._internalEventsClass = haxe.ui.components.ButtonEvents;
	this._compositeBuilderClass = haxe.ui.components.ButtonBuilder;
	this._defaultLayoutClass = haxe.ui.components.ButtonLayout;
}
```





### @:clonable

 When the component is cloned, it also clones the value of the property

It extremely useful if you use the component in an item renderer

###  @:value(...)  

Binds the value property to another property/variable

```haxe
@:clonable @:behaviour(SelectedBehaviour)               public var selected:Bool;
@:clonable @:value(selected)                            public var value:Dynamic;
```

translates to

```haxe
function get_value() {
    return this.get_selected();
}

function set_value(value:Dynamic) {
    {
        var _g = Type.typeof(value);
        switch ((enumIndex _g)) {
            case 1, 2: this.set_selected(value == 1);
            case 6: if ((_g[0] == String)) this.set_selected(value == "true" || value == "1") else this.set_selected(value);
            case 7: if ((_g[0] == haxe.ui.util.VariantType)) {
                var v = value;
                this.set_selected(@:implicitCast haxe.ui.util._Variant.Variant_Impl_.toBool(v));
            } else this.set_selected(value);
            default: this.set_selected(value)
        };
    };
    return value;
}
```



###  @:behaviour

Behaviour is a same a get/set propery, except that 

- it is used when the property has an influence on the visual of the component

  it handles the validation/invalidation of the component

- you can easily extend different type of behaviours

- the behaviour won't be used in the native backend except if you replace it with another

The behaviour of the class can be changed for a native backend

```xml
<native>
    <component id="haxe.ui.components.DropDown" class="hx.widgets.Choice" allowChildren="false" creator="none">
        <behaviour id="text" class="haxe.ui.backend.hxwidgets.behaviours.ChoiceSelectedLabel" />
    </component>
</native>
```



 ### @:call(...)  

Call meta is the same @:behaviour except that it works for functions

```haxe
@:call(ClearNodes) public function clearNodes():Void;

@:dox(hide) @:noCompletion
private class ClearNodes extends Behaviour {
    public override function call(param:Any = null):Variant {
        var treeview:TreeView = cast(_component, TreeView);
        treeview.selectedNode = null;
        var nodes = treeview.findComponents(TreeViewNode, 3);
        for (n in nodes) {
            treeview.removeComponent(n);
        }
        return null;
    }
}
```

It can be changed for a native backend

```xml
<component id="haxe.ui.containers.TreeView" class="hx.widgets.DataViewTreeCtrl" constructor="$style">
        <behaviour id="clearNodes" class="haxe.ui.backend.hxwidgets.behaviours.TreeViewClearNodes" />
</component>
```



### @:event

With `@:event` you can easily add an easy to set var, which works the same as `onChange`, `onClick`, etc

```haxe
@:event(ItemEvent.COMPONENT_EVENT)                                      public var onComponentEvent:ItemEvent->Void;
```

is transformed into

```haxe    
@:noCompletion
var _internal__onComponentEvent:haxe.ui.events.ItemEvent -> Void;

@:dox(group = "Event related properties and methods")
public var onComponentEvent(null,set):haxe.ui.events.ItemEvent -> Void;

function set_onComponentEvent(value:haxe.ui.events.ItemEvent -> Void) {
    if ((this._internal__onComponentEvent != null)) {
        this.unregisterEvent(haxe.ui.events.ItemEvent.COMPONENT_EVENT, this._internal__onComponentEvent);
        this._internal__onComponentEvent = null;
    };
    if ((value != null)) {
        this._internal__onComponentEvent = value;
        this.registerEvent(haxe.ui.events.ItemEvent.COMPONENT_EVENT, value);
    };
    return value;
}
```



### @:style

Using style enable to easily get and set styles. It will also manage invalidation.

`@:style`

`@:style(layout) `  will invalidate the layout

`@:style(layoutparent)` will invalidate the layoutparent



```haxe
@:style(layout)                                     public var textAlign:String;
```

will transform to

```haxe
	@:style @:keep @:dox(group = "Style properties")
	public var textAlign(get,set):String;

	function get_textAlign() {
		if ((this.get_customStyle().textAlign != null)) return this.get_customStyle().textAlign;
		if ((this.get_style() == null || this.get_style().textAlign == null)) return null;
		return this.get_style().textAlign;
	}

	function set_textAlign(value:String) {
		if ((this.get_customStyle().textAlign == value)) return value;
		if ((this._style == null)) this._style = new haxe.ui.styles.Style(null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null);
		if ((value == null)) this.get_customStyle().textAlign = null else this.get_customStyle().textAlign = value;
		this.invalidateComponent(cast "style", false);
        /** this part is added if you use @:style(layout) **/
		if ((! (this._layout == null || this._layoutLocked == true))) this.invalidateComponent(cast "layout", false);
        /** this part is added if you use @:style(parentLayout) **/
        if ((this.get_parentComponent() != null)) {
			var _this = this.get_parentComponent();
			if ((! (_this._layout == null || _this._layoutLocked == true))) _this.invalidateComponent(cast "layout", false);
		};
		return value;
	}
```
