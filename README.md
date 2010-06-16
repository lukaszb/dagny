# Dagny

Dagny is a [Django][] adaptation of [Ruby on Rails][]’s Resource-Oriented
Architecture (a.k.a. ‘RESTful Rails’). The name is [a reference][dagny taggart].

  [django]: http://djangoproject.com/
  [ruby on rails]: http://rubyonrails.org/
  [dagny taggart]: http://en.wikipedia.org/wiki/List_of_characters_in_Atlas_Shrugged#Dagny_Taggart

At present, this project is in an experimental phase. APIs are liable to change,
and code listings in this README may not work or may even be purely speculative
explorations of what the final API might look like. **You have been warned.**


## Resources

Dagny’s fundamental unit is the **resource**. A resource is identified by, and
can be accessed at, a single HTTP URI. A resource may be singular or plural; for
example, `/users` is a collection of users, and `/users/31215` is a single user.

In your code, resources are defined as classes with several **actions**. An
action is essentially a method, routed to based on the request path and method,
which is in charge of processing that particular request and returning a
response.


### URIs

For collections, the URIs and their interaction with the standard HTTP methods
follows the Rails convention:

    Method  Path            Action        Behavior
    --------------------------------------------------------------
    GET     /users          User.index    List all users
    GET     /users/new      User.new      Display new user form
    POST    /users          User.create   Create new user
    GET     /users/1        User.show     Display user 1
    GET     /users/1/edit   User.edit     Display edit user 1 form
    PUT     /users/1        User.update   Update user 1
    DELETE  /users/1        User.destroy  Delete user 1

Note that not all of these actions are required; for example, you may not wish
to provide `/users/new` and `/users/1/edit`, instead preferring to display the
relevant forms under `/users/` and `/users/1/`.

To work around the fact that `PUT` and `DELETE` are not typically supported in
browsers, you can add a `method` parameter over `POST` to override the request
method.

For singular resources, the URI scheme is similar:

    Method  Path            Action            Behavior
    -------------------------------------------------------------------
    GET     /account        Account.show      Display the account
    GET     /account/new    Account.new       Display new account form
    POST    /account        Account.create    Create the new account
    GET     /account/edit   Account.edit      Display edit account form
    PUT     /account        Account.update    Update the account
    DELETE  /account        Account.destroy   Delete the account

The same point applies here: you don’t need to specify all of these actions
every time.


### Defining Resources

Resources are subclasses of `dagny.Resource`. Actions are methods on these
subclasses, decorated with `@action`. Here’s a short example:

    from dagny import Resource, action
    from django.shortcuts import get_object_or_404, redirect
    
    from authapp import forms, models
    
    class User(Resource):
        
        @action
        def index(self):
            self.users = models.User.objects.all()
        
        @action
        def new(self):
            self.form = forms.UserForm()
        
        @action
        def create(self):
            self.form = forms.UserForm(request.POST)
            if self.form.is_valid():
                self.user = self.form.save()
                return redirect(self.user)
            
            response = self.new.render()
            response.status_code = 403 # Forbidden
            return response
        
        @action
        def show(self, username):
            self.user = get_object_or_404(models.User, username=username)
        
        @action
        def edit(self, username):
            self.user = get_object_or_404(models.User, username=username)
            self.form = forms.UserForm(instance=self.user)
            return self.new.render()
        
        @action
        def update(self, username):
            self.user = get_object_or_404(models.User, username=username)
            self.form = forms.UserForm(self.request.POST, instance=self.user)
            if self.form.is_valid():
                self.form.save()
                return redirect(self.user)
            
            response = self.edit.render()
            response.status_code = 403
            return response
        
        @action
        def destroy(self, username):
            self.user = get_object_or_404(models.User, username=username)
            self.user.delete()
            return redirect('/users')

`Resource` uses [django-clsview][] to define class-based views; the extensive
use of `self` is safe because a new instance is created for each request.

  [django-clsview]: http://github.com/zacharyvoase/django-clsview

You might notice that there are no explicit calls to `render_to_response()`;
most of the time you’ll want to render the same templates: `"user/index.html"`,
`"user/new.html"`, `"user/show.html"` and `"user/edit.html"`. Therefore, if your
action doesn’t return anything (i.e. returns `None`), a template corresponding
to the resource and action will be rendered. See the next section for a more
in-depth explanation.


### Renderers

An action actually comes in two parts: one part does the processing, the other
returns a response to the client. This allows for transparent content
negotiation, and means you never have to write a separate ‘API’ for your site,
or call `render_to_response()` at the bottom of every view function.

Usage is as follows:

    class User(Resource):
        
        @action
        def show(self, username):
            self.user = get_object_or_404(User, username=username)
        
        @show.render.json
        def show(self):
            return HttpResponse(content=simplejson.dumps(self.user.to_dict()),
                                mimetype='application/json')

Now, if you fetch `/users/zacharyvoase/`, you’ll see the result of rendering the
`"users/show.html"` template as a HTML page. If you fetch
`/users/zacharyvoase/?format=json`, however, you’ll get a JSON representation of
that user. Dagny also uses `webob.acceptparse`, so passing an
`Accept: application/json` header will result in JSON output, too.


#### How Renderers Work

When `show()` is called by a request, the main body of the action is first run.
If this does not return a `HttpResponse` outright, the action will perform
content negotiation, to decide which renderer to use. In most circumstances,
this will be `html`. The default behavior of the `html` renderer looks something
like this:

    class User(Resource):
        
        # ... snip! ...
        
        @show.render.html
        def show(self):
            return render_to_response("user/show.html", {
                'self': self
            }, context_instance=RequestContext(self.request))


#### Additional MIME types

Additional renderers for a single action are defined using the decoration syntax
(`@<action_name>.render.<format>`) as seen above, but since content negotiation
is based on mimetypes, Dagny keeps a global `dict` mapping these **shortcodes**
to full mimetype strings (as `dagny.conneg.MIMETYPES`). You can create your own
shortcodes, and use them in resource definitions:

    from dagny.conneg import MIMETYPES
    
    MIMETYPES['rss'] = 'application/rss+xml'
    MIMETYPES['png'] = 'image/png'
    MIMETYPES.setdefault('json', 'text/javascript')

There is already a relatively extensive list of types defined; see the
[`dagny.conneg` module][dagny.conneg] for more information.

  [dagny.conneg]: http://github.com/zacharyvoase/dagny/blob/master/src/dagny/conneg.py


#### Generic Renderers

Dagny also supports **generic renderers**; these are type-specific renderers
attached to the `dagny.renderer.Renderer` class (or a subclass thereof) which
will be available on *all* actions by default. These renderers are simple
functions which take both the action instance and the resource instance. For
example, the HTML renderer (which every action has as standard) looks like:

    from dagny.renderer import Renderer
    from dagny.utils import camel_to_underscore, resource_name
    
    from django.shortcuts import render_to_response
    from django.template import RequestContext
    
    @Renderer.generic('html')
    def render_html(action, resource):
        template_path_prefix = getattr(resource, 'template_path_prefix', "")
        resource_label = camel_to_underscore(resource_name(resource))
        template_name = "%s%s/%s.html" % (template_path_prefix, resource_label, action.name)
        
        return render_to_response(template_name, {
          'self': resource
        }, context_instance=RequestContext(resource.request))

You can also *remove* a generic renderer from a single action like so:

    class User(Resource):
    
        @action
        def show(self, username):
            self.user = get_object_or_404(User, username=username)
        
        del show.render['html']