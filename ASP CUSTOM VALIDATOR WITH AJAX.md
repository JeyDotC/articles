

# ASP CUSTOM VALIDATOR WITH AJAX

In ASP, there is a built-in validation system, it is flexible enough to provide a simple interface to create your own validators:

```asp
<asp:CustomValidator runat="server" 
                    ControlToValidate="MyValidatedControlID"
                    OnServerValidate="MyCodeBehindValidationEventHandlerName"
                    ClientValidationFunction="MyJavascriptValidationFunctionName"
                    ErrorMessage="You are wrong!" />
```

It is simple, is cool and allows you o do both, client and server side validation. But what if you need to do an AJAX validation? The obvious answer to this would be:

```javascript
function MyJavascriptValidationFunctionName(sender, args){
    $.ajax({
        url: ...
        success: function(data){
            args.IsValid = doSomethingWithTheResult(data);
        }
    });
}
```
Cool! but there is a problem, AJAX calls are asynchronous, thus, the postback will start before your validation is done :(

"But Jey, it can be easily solved by just making the AJAX request synchronous"

```javascript
function MyJavascriptValidationFunctionName(sender, args){
    $.ajax({
        url: ...
        async: false, //<-- Here is the trick! http://bit.ly/1jMKItD
        success: function(data){
            args.IsValid = doSomethingWithTheResult(data);
        }
    });
}
```
Well, indeed it does the thing, but it has a disadvantage, it locks the entire page until the validation is done. This might not be so bad, as long as the request is fast enough to keep the user unaware of it; the problem comes when the request is slow, then, your validation becomes as annoying as an alert.

What do we do then? Well, after some experimentation and a lot of StackOverflow research I came to a solution which involves these steps:

1. Say *"Invalid!"* immediately.
2. Show a *"processing..."* message instead of your *"invalid"* message.
3. Start your long-runing process, AKA your AJAX request.
4. As soon as your request ends, replace the ClientValidationFunction for a dummy function.
5. Reset the original message.
5. Update the validation state.
6. Reset the original validation function but only when the validated control changes.

Looks like a lot of things to do (°_°) but it is what it takes if we want some kind of generic solution for this problem. Ok, lets do it.

### Say *"Invalid!"* immediately.

The first thing we want is to stop the callback process in order to do our slow job without being interrupted, so, as soon as your validation function gets called, say *Invalid!*.

```javascript
function MyJavascriptValidationFunctionName(sender, args){
    args.IsValid = false;
}
```

### Show a *"processing..."* message instead of your *"invalid"* message.

Ok, that stopped the postback, but now we are telling the user that the field is invalid without actually validating it. To solve that, all we need to do is to alter the validator control(which happens to be `span` element with extra attributes) to show a more accurate message, but keeping a copy of the original values to restore them later.

```javascript
function MyJavascriptValidationFunctionName(sender, args){
    args.IsValid = false;
    
    //This is a reference to our validator control
    var $sender = $(sender);
    
    //Save the original message, color and validation function to restore them later.
    var originalMessage = $sender.text();
    var originalColor = $sender.css("color");
    var originalFunction = sender.clientvalidationfunction;
    var validatedControl = $("#" + sender.controltovalidate);

    //Change the error message for a friendlier one.
    $sender.text("Checking...").css({ color: "black" });
}
```

Note that we are taking the properties like `sender.clientvalidationfunction` or `sender.controltovalidate` those are  properties added to the `span` that represents our validator.

### Start your long-running process, AKA your AJAX request.

Now we start our slow process. As AJAX calls are usually the slowest thing in a web page, we will take that for our experiment.

```javascript
function MyJavascriptValidationFunctionName(sender, args){
    args.IsValid = false;
    
    //This is a reference to our validator control
    var $sender = $(sender);
    
    //Save the original message, color and validation function to restore them later.
    var originalMessage = $sender.text();
    var originalColor = $sender.css("color");
    var originalFunction = sender.clientvalidationfunction;
    var validatedControl = $("#" + sender.controltovalidate);

    //Change the error message for a friendlier one.
    $sender.text("Checking...").css({ color: "black" });
    
    //Start the AJAX call..
    $.ajax({
        url: ...
        success: function (data) {
            //Here we got an answer...
        }
    });
}
```

### As soon as your request ends, replace the ClientValidationFunction with a dummy function.

Well, after a long journey, our message is responded, but now, we need to notify the validation system about the news. The problem here is that we don't want the AJAX call to be triggered again for now as we already know the answer for the validation. 

To avoid triggering the AJAX request but notify the validation system that the control has just been validated, we need to replace the original validation function with one that just says "yes, the control is valid" or "no, try again".

```javascript
//Create our respond functions...
var Respond_True = function (sender, args) { args.IsValid = true; };
var Respond_False = function (sender, args) { args.IsValid = false; };

function MyJavascriptValidationFunctionName(sender, args){
    ...content ommited...
    //Start the AJAX call..
    $.ajax({
        url: ...
        success: function (data) {
            //Here we got an answer...
            sender.clientvalidationfunction = /*Is valid*/ ? "Respond_True" : "Respond_False";
        }
    });
}
```

With this we ensure that if validation is triggered again (e.g. the submit button is clicked) we will have an immediate answer.

## Reset the original message.

In a previous step, we modified the original message to tell the user that we were in a validation process, but now we need the original one to the case when the validation gets triggered again.

```javascript
//Create our respond functions...
var Respond_True = function (sender, args) { args.IsValid = true; };
var Respond_False = function (sender, args) { args.IsValid = false; };

function MyJavascriptValidationFunctionName(sender, args){
    ...content ommited...
    //Start the AJAX call..
    $.ajax({
        url: ...
        success: function (data) {
            //Here we got an answer...
            sender.clientvalidationfunction = /*Is valid*/ ? "Respond_True" : "Respond_False";
            
            //Reconstitute original styles.
            $sender.text(originalMessage).css({ color: originalColor });
        }
    });
}
```
Now if validation for this field failed and the submit button gets clicked, the right error message will appear immidiately.

### Update the validation state.

But we don't want the user to wait until he triggers the validation event again, we want to feed him back as soon as the answer to the validation is ready, and also, remove the "checking..." message which otherwise would remain there forever. 

To accomplish that, we just call the validation but only for our control.

```javascript
function MyJavascriptValidationFunctionName(sender, args){
    ...content ommited...
    //Start the AJAX call..
    $.ajax({
        url: ...
        success: function (data) {
            //Here we got an answer...
            sender.clientvalidationfunction = /*Is valid*/ ? "Respond_True" : "Respond_False";
            
            //Reconstitute original styles.
            $sender.text(originalMessage).css({ color: originalColor });
            
            //Re-validate our control
            ValidatorValidate(sender, null, null);
            ValidatorUpdateIsValid();
        }
    });
}
```

With that, we feed the user with an appropiate message and set the correct status for our control.

### Reset the original validation function but only when the validated control changes.

Earlier in this post, we stored the original validation function name to set it back in our control, but we don't just set it again immediately, is necessary to do it in the right moment, that is, when the control changes.

To do it, we listen to the "change" event.

```javascript
function MyJavascriptValidationFunctionName(sender, args){
    ...content ommited...
    //Start the AJAX call..
    $.ajax({
        url: ...
        success: function (data) {
            ...content ommited...
            //Re-validate our control
            ValidatorValidate(sender, null, null);
            ValidatorUpdateIsValid();
            var onChange = function(){
                //Reset the original validation function
                sender.clientvalidationfunction = originalFunction;
                //Re-validate to ensure the original validation function gets called
                ValidatorValidate(sender, null, null);
                ValidatorUpdateIsValid();
                //Ensure the validation function is called just once.
                validatedControl.unbind("change", onChange);
            };
            
            validatedControl.on("change", onChange);
        }
    });
}
```

Note that our listener does three things:

* Resets the original validation function.
* Invokes the validation for the control. Otherwise, the correct validation wouldn't be called.
* Unbinds itself. As the validation process may be called more than once, forgetting to unbind the listener would end in a catastroffic accumulation of AJAX calls!

### Putting all togather

Here is the complete function that validates with an ajax call:

```javascript
//Create our respond functions...
var Respond_True = function (sender, args) { args.IsValid = true; };
var Respond_False = function (sender, args) { args.IsValid = false; };

function MyJavascriptValidationFunctionName(sender, args){
    args.IsValid = false;
    
    //This is a reference to our validator control
    var $sender = $(sender);
    
    //Save the original message, color and validation function to restore them later.
    var originalMessage = $sender.text();
    var originalColor = $sender.css("color");
    var originalFunction = sender.clientvalidationfunction;
    
    //Change the error message for a friendlier one.
    $sender.text("Checking...").css({ color: "black" });
    
    //Start the AJAX call..
    $.ajax({
        url: ...
        success: function (data) {
            //Here we got an answer...
            sender.clientvalidationfunction = /*Is valid*/ ? "Respond_True" : "Respond_False";
            
            //Reconstitute original styles.
            $sender.text(originalMessage).css({ color: originalColor });
            
            //Re-validate our control
            ValidatorValidate(sender, null, null);
            ValidatorUpdateIsValid();
            
            var onChange = function(){
                //Reset the original validation function
                sender.clientvalidationfunction = originalFunction;
                //Re-validate to ensure the original validation function gets called
                ValidatorValidate(sender, null, null);
                ValidatorUpdateIsValid();
                //Ensure the validation function is called just once.
                validatedControl.unbind("change", onChange);
            };
            $("#" + sender.controltovalidate).on("change", onChange);
        }
    });
}
```

### Divide and conquer!

Too much work for a single function, isn't it? Fortunately we are programmers and know how to encapsulate repetitive behavior, so, lets reorganize code into something reusable :)

```javascript
//Create our respond functions...
var Respond_True = function (sender, args) { args.IsValid = true; };
var Respond_False = function (sender, args) { args.IsValid = false; };

function AjaxValidator(sender, args, ajaxSettings){
    args.IsValid = false;
    
    //This is a reference to our validator control
    var $sender = $(sender);
    
    //Save the original message, color and validation function to restore them later.
    var originalMessage = $sender.text();
    var originalColor = $sender.css("color");
    var originalFunction = sender.clientvalidationfunction;
    var validatedControl = $("#" + sender.controltovalidate);
    
    //Change the error message for a friendlier one.
    $sender.text("Checking...").css({ color: "black" });
    
    var setRespondFunction = function (respondFunction) {
        sender.clientvalidationfunction = respondFunction;
            
        //Reconstitute original styles.
        $sender.text(originalMessage).css({ color: originalColor });
        
        //Re-validate our control
        ValidatorValidate(sender, null, null);
        ValidatorUpdateIsValid();
        
        var onChange = function(){
            //Reset the original validation function
            sender.clientvalidationfunction = originalFunction;
            //Re-validate to ensure the original validation function gets called
            ValidatorValidate(sender, null, null);
            ValidatorUpdateIsValid();
            //Ensure the validation function is called just once.
            validatedControl.unbind("change", onChange);
        };
        validatedControl.on("change", onChange);
    }
    
    var originalSuccessFunction = ajaxSettings.success;
    //Start the AJAX call..
    $.ajax($.extend(ajaxSettings, {
        success: function(data){
            setRespondFunction(originalSuccessFunction(data) ? "Respond_True" : "Respond_False");
        }
    }));
}
```

Having this, validating controls with AJAX becomes as simple as this:

```javascript
function MyJavascriptValidationFunctionName(sender, args){
    AjaxValidator(sender, args, {
        url: ...,
        type: ...,
        data: ...,
        success: function(data){
            return /*True or false*/;
        }
    });
}
```

# Conclusions

This is just an experiment, there are lots of things to polish, but I think it is a good starting point for a full fledged extension for client-side ASP validation.

> Written with [StackEdit](https://stackedit.io/).