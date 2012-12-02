Adding some sunctionality to Drupal's autocomplete.js
=====================================================

Well, for many developers <del>too lazy</del> with no time to climb the steep 
drupal's learning curve the solution is to simply implement a hook_menu and do 
the job with plain old PHP.

```php
function mymodule_menu(){
    $routing["my/route1"] = array(
        "page callback" => "mymodule_do_something",
        "access callback" => true,
    );
    return $routing;
}

function mymodule_do_something(){
    $something = "Look mom, I'm doing something!";
    if(/*Is this an ajax request?*/){
        echo $something;
    }else{
        return $something;
    }
}
```

But one usually takes advantage of many drupal functionalities, like the 
```node_load/save/delete, drupal_add_js/css``` functions or the form api with
```drupal_render```. In this particular case, one desires to create an autocomplete
control with a lot of complex capabilities (and the jquery.ui.autocomplete 
doesn't work for some unknown reason), but there is a small problem... the 
drupal's autocomplete library have some faults:

* Doesn't emit any events, neither when an option is picked (I still don't know why).
* It doesn't care about search results with complex data. It isn't something estrictly
necessary, but it would be useful... or at least emit an event when there are results!
but, again, autocomplete.js doesn't emit any event.

Ok, lets listen to the onChange event in my field, that will solve all my problems!....
sorry bro, but it's worthless because the onChange event occurs when the user types
something in the field and then it looses focus. That doesn't happen with
autocomplete.js because it does this: ```field.value = newvalue``` doing this doesn't ever
trigger such event, so, this solution won't work.

The only solution I could think is to simply override the functions that loads the search results
and the one which handles the pickup action.

<table>
    <tr>
        <td>
            <div>What do we want?</div>
            <img src="http://i1.kym-cdn.com/entries/icons/original/000/006/199/x-all-the-y.png" />
        </td>
        <td>
           <div>Maganage results with complex data and Have an event to hook at!</div>
            <img src="http://i1.kym-cdn.com/entries/icons/original/000/006/199/x-all-the-y.png" />
        </td>
    </tr>
</table>

Here is how:

The first thing I wanted to do was to be able to get complex data from search results
so I first hacked the Drupal.jsAC.prototype.found method from autocomplete.js.
This method is called when there is a search result and is the responsible for
creating the list of possible options. It receives a JSON object where keys are
considered the values to be put in the field and the values are the text to be
displayed in the options menu.

```javascript
//First of all, have a copy to the parent method.
var parentFound = Drupal.jsAC.prototype.found;
//Then override the prototype.
Drupal.jsAC.prototype.found = function (matches) {
    var plainMatches = {};
    /**
    * With this loop we just check if the values are objects(not strings), if so,
    * we use the same key as the value. This ensure that the parent's version of
    * this method will receive an array or object which values are all strings.
    */
    for(var key in matches){
        var obj = matches[key];
        if(obj != null && typeof obj == "object"){
            plainMatches[key] = key;
        }else{
            plainMatches[key] = obj;
        }
    }
    //Here we call the parent's version of this method.
    parentFound.call(this, plainMatches);
    //An then we allocate the complex data in each option, so when we pickup an option
    //we'll be able to retrieve the data.
    $(this.popup).find("ul > li").each(function () {
        this.dataEntity = matches[this.autocompleteValue];
    });
};
```

And now we can have complex data in our options, but how to retrieve it? by listening
the onChangeEvent of our field. But to do so, we need to trigger such event from the
field by overriding the Drupal.jsAC.prototype.hidePopup method:

```javascript
//As before, we need the original function.
var parentHidePopup = Drupal.jsAC.prototype.hidePopup;
Drupal.jsAC.prototype.hidePopup = function (keycode) {
    var entity;
    //First of all we check if an option was picked up exactly the same way the original method would do.
    var changed = this.selected && ((keycode && keycode != 46 && keycode != 8 && keycode != 27) || !keycode);
    if (changed) {
        //If so, we retrieve our entity from the selected value.
        entity = this.selected.dataEntity;
    }
    //now we call the original method as it is responsible for hiding the poup and
    //setting the value of the field.
    parentHidePopup.call(this, keycode);
    //And then we emit the "change" event. Note that we did this after calling the
    //original method because we need the field to be already changed.
    if(changed){
        var event = jQuery.Event("change");
        //Here we set an 'entity' property in our event so we can get it at our listener.
        event.entity = entity;
        $(this.input).trigger(event);
    }
};
```

And finally we can use a listener for our brand new triggered event:

```javascript
$(myFieldSelector).live("change", function (e) {
    //Now our entity is part of the event.
    var entity = e.entity;
    //As this event can be triggered by means different than picking up an option
    //we need to check if the entity is set, to avoid overriding previus correct
    //values with null
    if(entity != null && typeof entity == "object"){
        //And here is our neat entity! ^_^
        console.log(entity);
    }
})
```

And we now have a cool autocomplete functionality added to the drupal's version.
All we have to do is to put it into a js file and load such file with our module.

Here is the full thing:

drupal.enhanced_autocomplete.js

```javascript
jQuery(function($){
    if(Drupal.jsAC){
        var parentFound = Drupal.jsAC.prototype.found;
        Drupal.jsAC.prototype.found = function (matches) {
            var plainMatches = {};
            for(var key in matches){
                var obj = matches[key];
                if(obj != null && typeof obj == "object"){
                    plainMatches[key] = key;
                }else{
                    plainMatches[key] = obj;
                }
            }
            parentFound.call(this, plainMatches);
            $(this.popup).find("ul > li").each(function () {
                this.dataEntity = matches[this.autocompleteValue];
            });
        };
        var parentHidePopup = Drupal.jsAC.prototype.hidePopup;
        Drupal.jsAC.prototype.hidePopup = function (keycode) {
            var entity;
            var changed = this.selected && ((keycode && keycode != 46 && keycode != 8 && keycode != 27) || !keycode);
            if (changed) {
                entity = this.selected.dataEntity;
            }
            parentHidePopup.call(this, keycode);
            if(changed){
                var event = jQuery.Event("change");
                //Here we set an 'entity' property in our event so we can get it at our listener.
                event.entity = entity;
                $(this.input).trigger(event);
            }
        };
    }
});
```

And at our module we can use the autocomplete as usual:

my.module:

```php
<?php
function my_menu(){
    $routing["my/page_with_autocomplete"] = array(
        "page callback" => "my_do_something",
        "access callback" => true,
    );
    $routing["my/search_page"] = array(
        "page callback" => "my_search_nodes",
        "access callback" => true,
    );
    //And so on...
    return $routing;
}

function my_do_something(){
    // In theory, this should also work with more usual mechanisms, like form_alter or so.
    drupal_add_js($drupal_get_path('module', 'my') . 'drupal.enhanced_autocomplete.js');
    drupal_add_js($drupal_get_path('module', 'my') . 'my.custom_script.js');
    $output = "";
    //.... build your output ....
    //Add your autocomplete....
    $output .= drupal_render(array(
            "#id" => "id",
            '#type' => 'textfield',
            "#attributes" => array("class" => "my-autocomplete",),
            '#autocomplete_path' => 'my/search_page',
        ));
     return $output;
}

function my_search_nodes($search=""){
    $nodes = load_a_lot_of_nodes($search)
    //As this is usually an ajax search we just print the results.
    echo drupal_json($nodes);
}

?>
```
And finally we can manage our search results:

my.custom_script.js

```javascript
$(".my-autocomplete").live("change", function (e) {
    var entity = e.entity;
    if(entity != null && typeof entity == "object"){
        //Do something with our entity, like setting like
        //Setting a value on a hidden or dependant field.
    }
})
```
