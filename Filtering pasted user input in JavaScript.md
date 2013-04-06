Filtering pasted user input in JavaScript
=========================================

In some rare cases is necessary to filter what users paste
in the exact moment they do it, one reason is to allow users to 
paste URLs with space characters which otherwise would be ignored 
by any script with the responsibility of converting URLs into 
links.

But here comes the problem, filtering what users paste is kind of
complicated or directly impossible as the paste event doesn't provide
the pasted string, neither for reading.

So, what can we do? the solution came from [this](http://www.mattbenton.net/2012/01/jquery-plugin-paste-events/)
post. It made clearer something I saw in various answers from stackOverflow: You can
listen to the paste event before and after the text is pasted and thus you have the old
and the new versions of the text. 

```javascript

$.fn.pasteEvents = function( delay ) {
    if (delay == undefined) delay = 20;
    return $(this).each(function() {
        var $el = $(this);
        $el.on("paste", function() {
            $el.trigger("prepaste");
            setTimeout(function() { $el.trigger("postpaste"); }, delay);
        });
    });
};

var oldValue;

$(selector).bind("prepaste", function (e) {
	// Retrieve the old version of the string.
	oldValue = e.target.value;
}).bind("postpaste", function (e) {
	// Get the new one.
	var newValue = e.target.value;
}).pasteEvents();

```

So, obviously, as we are speaking about a paste process the difference between the two texts
should be the pasted text! but then we remember that there is no built in diff function in JavaScript
and... man! we are too lazy to make it, so, searching here and there I found [this](https://code.google.com/p/google-diff-match-patch/)
neat library from google that does exactly what we need, diff and patch (among other things). That way
we can get our precious pasted text:

```javascript
//Creating the diff object.
var diff = new diff_match_patch();
//Get the differences
var diffs = diff.diff_main(oldValue, newValue);

$.each(diffs, function (k, v) {
	// In this case 'v' is an array where the first element is an int indicating
	// the type of difference: 1 is added, -1 is removed and 0 is unchanged.
	// The seccond elemnt is the string in question.
	var text = v[1];
	// Do your filtering with the text...
});
// Apply changes...
var newText = diff.patch_apply(diff.patch_make(diffs), oldValue)[0];
```

Now that we can get the pasted text, modify it and apply the changes to the final value, all we need is
to wrap it all in a jQuery plugin:

```javascript

//From http://www.mattbenton.net/2012/01/jquery-plugin-paste-events/
$.fn.pasteEvents = function (delay) {
    if (delay === undefined) delay = 20;
    return $(this).each(function () {
        var $el = $(this);
        $el.bind("paste", function () {
            $el.trigger("prepaste");
            setTimeout(function () {
                $el.trigger("postpaste");
            }, delay);
        });
    });
};

//The cool plugin
/**
* Intercepts the value pasted by the user and allows a callback to filter it before
* it gets to the target input.
*
* @param paste_modifier a callback that modifies the value pasted 
* by user.
*/
$.fn.pasteFilter = function (paste_modifier) {
    var oldValue;
	// This needs some optimization.
    var diff = new diff_match_patch();

    return $(this).each(function () {
        var $el = $(this);
        $el.bind("prepaste", function (e) {
            oldValue = e.target.value;
        }).bind("postpaste", function (e) {
            var diffs = diff.diff_main(oldValue, e.target.value);
            $.each(diffs, function (k, v) {
                var text = v[1];
                if (v[0] === 1) {
                    v[1] = paste_modifier(text);
                }
            });
            var newText = diff.patch_apply(diff.patch_make(diffs), oldValue)[0];
            e.target.value = newText;
        }).pasteEvents();
    });
};
```

Now we have a *really* simple way of filtering the just pasted text. Here is the plugin usage:

```javascript
$(selector).pasteFilter(function (text) {
    // Do whatever cool thing with your string
	// and return the modified version...
    return text;
});
```

Here is a [jsFiddle](http://jsfiddle.net/JeyDotC/7SVbN/3/) to prove the idea and remember that it depends
on [google-diff-patch-match](https://code.google.com/p/google-diff-match-patch/)

The plugin is far from finished, as you may improve it by adding a check to allow filters to return 
boolean false when those aren't willing to modify an specific pasted string or by optimizing it. 
But I gess it's a good start.

Thanks for reading!
