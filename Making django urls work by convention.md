Making django urls work by convention
=====================================

These days I've been looking at django framework, I need to learn it
for my new job, so, while following the django tutorials I realized
that for each method in my solution is necesary to configure a URL. That's
not bad at all as it allows the creation of user-friendly urls, but, I'm too
lazy to spend my time creating a new line which almost certainly will be a copy
of the one on top of it for each module and public method I make. 

``` python
urlpatterns = patterns('',
	url(r'^/somewhere/over/the-rainbow/', 'my_app.views.some_function')
	url(r'^/somewhere/over/the-rainbow/', 'my_app.views.some_function')
	url(r'^/somewhere/over/the-rainbow/', 'my_app.views.some_function')
	url(r'^/somewhere/over/the-rainbow/', 'my_app.views.some_function')
	# ... whatch out! this can become kilometric
)
```

So I thought: wouldn't it be cool to just have my project with a certain layout
and that urls get dealt by conventions? then I remembered an old known pattern:
the front controller pattern.

The idea is simple: all you need is to have a single entry point which deals with all
requests and call the adequate object to respond for each request. Besides, we count
with the posibility of defining urls using regular expressions, that way we can easily
capture all requests and send them to our front controller.

``` python
urlpatterns = patterns('',
	url(r'^(?P<app_name>[a-zA-Z_]\w*)/(?P<method_name>[a-zA-Z_]\w*)(/[a-zA-Z_0-9]\w*)', 'call.to.my_front_controller')
	url(r'^(?P<app_name>[a-zA-Z_]\w*)/(?P<method_name>[a-zA-Z_]\w*)', 'call.to.my_front_controller')
	url(r'^(?P<app_name>[a-zA-Z_]\w*)', 'call.to.my_front_controller')
)
```

That way we cover a lot of possible urls, all of them dealt by a single function. But we need our function
to call the adequate module based on the given url, so we can define our function this way:

``` python
def my_front_controller(request, app_name, method_name = "index", *args, **kwargs):
	#Get the fully qualified module name and import it
	module_name = "my_root_namespace.%s.views" % app_name
	module = __import__( module_name )
	#Actually get the module object.
	module = getattr(getattr(module, app_name), "views")

	#Get the method that will actually handle the request.
	responder = getattr(module, method_name, controller.index)
	#Return its result
	return responder(request, *args, **kwargs)
```

That way is possible to have a typical django project where, to have an application registered, is only 
necesary to stick to the directory conventions, views are resolved by the front controller.

Having this done, you may see a cool posibility, with this mechanism is possible to have any kind of 
conventions, from .NET MVC to Hirudo project layout styles, is all up to your imagination. Furthermore,
you can create functions that are "context-aware" so, when they are imported they will have some knoledge
of the currently running view, with this in mind, we can create, for example, a really short shortcut for
rendering templates, like this:

``` python
def my_front_controller(request, app_name, method_name = "index", *args, **kwargs):
	module_name = "my_root_namespace.%s.views" % app_name
	module = __import__( module_name )
	module = getattr(getattr(module, app_name), "views")

	responder = getattr(module, method_name, controller.index)
	#make the view function aware of the current request:
	view.request = request
	return responder(request, *args, **kwargs)

def view(view_function, context = {}, template = None):
	#In this case we receive a reference to the view, the idea is to
	#extract the template name from the function name unless the caller
	#explicitly indicates the template to be rendered
    t = view_function.func_name if template is None else template;
	#Call a template the usual hard-coded way. Note that we are getting the
	#request from this function.
    return HttpResponse(select_template((
                                "%s.html" % t,
                            )).render(RequestContext(view.request, context)))
```

The usage of this function is like this:

my_module.py

``` python
#do the imports....

def my_view(request):
	#do your stuff...
	#Call my view, note that we are sending the current function
	view(my_view)
	
def other_view(request):
	#do your stuff...
	view(other_view, {"data":some_data})

def yet_another(request):
	#do your magic...
	#Here we are calling a template from another view
	view(yet_another, {"data":some_data}, template="other_view")
```

All this may open a lot of posibilities in order to reduce the code needed
to make django applications. 

I've made a project sample to demostrate the ideas exposed here: https://github.com/JeyDotC/DjangoCustomProjectLayoutSample 
The project has two custom layouts, one is the *net_mvc* layout an the other is the *traditional*
layout.

## What about using controller classes?

Well, that was my first approach, but I then realized that
in such case we lose the posibility to use the built-in django
decorators. At the writing of this article I haven't found a way to
use django decorators over class methods :( has anybody made it?
