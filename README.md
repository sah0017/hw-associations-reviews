# Associations

**This CHIPS is under construction.**

In this assignment you'll add the ability for people to leave reviews
about movies in RottenPotatoes. 
Of course, the review has to be written by someone, and we'd like
people to be able to track their reviews, so we need to add the idea
of a user, and make it possible for a user to sign in to keep track of
their reviews.  To avoid the hassles of managing passwords ourselves,
we will allow users to use Single Sign-on (SSO) to sign in with
third-party credentials.  We will use GitHub in this example, but with
minimal changes you can enable SSO sign-in for any third-party auth
provider supported by [OmniAuth](https://github.com/omniauth/omniauth).

Here is the set of steps:

1. Set up the app for SSO sign-in using the OmniAuth gem.

2. Add a `Moviegoer` model, and arrange for a successful sign-in to
associate the authenticated Moviegoer's ID with the current session
(to indicate they are signed in).

3. Add a `Review` model set up in such a way that a review belongs to
a `Movie` and a `Moviegoer`.

4. Create basic routes, controller actions, and views to allow a moviegoer
to create, edit, update, or delete their reviews.

You can start with working RottenPotatoes code from any previous
CHIPS (it doesn't matter if it includes the additional "filter movie
list" or "find by same director" features of some of the other CHIPS).

As usual for any new Rails project, get your starter code ready,
run `bundle install --without production`, ensure the app runs
locally, and use `git commit` to take a snapshot of this "safe" state.

Now you're ready to start!

# Prerequisite skills and knowledge

* You should be comfortable with the workings of a simple one-model
Rails app like RottenPotatoes.

* You should know how to create and apply migrations to add models to
a Rails app.

* You should be familiar with the concepts of how SSO works
ESaaS section 5.2).

* You should be familiar with the concepts and basic mechanics
(`has_many`, `belongs_to`, and so on) of ActiveRecord Associations
ESaaS sections 5.4-5.6).


# 1. Set up for SSO using OmniAuth for "Login with GitHub"

Before we begin, you need to deploy your Rotten Potatoes app to heroku
so it will have an application URL.  You will need this URL in a future 
step.

First we'll add a simple model to represent a moviegoer and allow
sign-in via SSO,
so that reviews can be linked to moviegoers as well as movies.  We
will use GitHub as our SSO provider, but with minimal
changes you can use a different provider as well.

We will use the [OmniAuth gem](https://github.com/omniauth/omniauth)
to handle SSO.  OmniAuth is an excellent example of the Strategy
design pattern, which is covered later in the course.  In a nutshell,
OmniAuth abstracts away the details of the specific API calls required
to authenticate with an SSO identity provider, and replaces them with
three routes your app must support (where `:provider` is the SSO
identity provider, e.g. `github`):

* `GET /auth/:provider`: when a user visits this route, OmniAuth will
intercept the request (i.e. your app's controllers won't see it at
all) and send the user over to the SSO provider's login flow.

* `GET /auth/failure`: if the user is unable to sign in via SSO (wrong
password, etc.), your app will receive a request to this route.  You
should arrange to have it map to a controller action that tells the
user their sign-in was unsuccessful, or perhaps redirect them back to
the sign-in page.

* `(GET|POST) /auth/:provider/callback`: if the user successfully signs 
in via SSO, your app will receive a request to this route.  You should
therefore arrange to have this route map to a controller action in
which you take steps to "sign the user in", such as remembering
the user's ID in the `session[]`.  When this route is called, OmniAuth
makes available a hash called `auth_hash[]`  containing information about 
the user, provided by the SSO provider.  The specific information varies 
depending on the provider, but there are a couple of hash keys that are 
guaranteed to be present, which we will use.  The route should be 
accessible via either `GET` or `POST` due to differences in how SSO 
providers work.

Consult the OmniAuth project's wiki (on [GitHub](https://github.com/omniauth/omniauth) to answer the
following questions in preparation to add OmniAuth via GitHub:

* What keys of `auth_hash` are **always** guaranteed to be present
after a successful auth, regardless of which provider is used?

((Answer: `auth[:provider]`, `auth[:uid]`, `auth[:info][:name]`))

* The `omniauth-github` Gem page on GitHub (**find it**) specifies that
you must add the following line in your Gemfile:

`gem 'omniauth-github', github: 'omniauth/omniauth-github', branch: 'master'`

* Why don't you have to add the `omniauth` gem itself?

((Answer: since every strategy gem depends on it, it will be pulled in
by default by Bundler.))

Follow the instructions for `omniauth-github` to setup your Rack
middleware to intercept the above routes.  Whereas the generic omniauth has
you create an omniauth.rb file, for this exercise you will create a github.rb
file per the `omniauth-github` instructions.  You will need to set up
environment variables with your github developer keys.  In order for
your app to authenticate with Github, you must register your app and obtain
a Github key and a Github secret.  Follow these [instructions](https://docs.github.com/en/enterprise-server@3.4/developers/apps/building-oauth-apps/creating-an-oauth-app)
to set up the key and secret.  Creating the OAuth app requires a Homepage URL.
This is where you will enter your Heroku address for your application.
The Authorization callback URL will be your Heroku address + auth/github/callback.
This will allow Github to send the HTTP request that we defined in routes.rb.
Github calls the key a Client ID, and you'll need to press a button to Generate
a Client Secret.  Make sure you copy the Client Secret because it will not
allow you to see it or retrieve it again.

Now you'll want to put your Github key and secret into Linux
environment variables.  The ENV['GITHUB_KEY'] and ENV['GITHUB_SECRET']
in github.rb will read the key and secret from the environment variables 
and pass them to Github.  If you're unfamiliar with environment variables, 
this tutorial explains how to set up [environment variables in Rails.](https://blog.devgenius.io/what-are-environment-variables-in-rails-6f7e97a0b164)

* At this point, you should be committing three new or modified files---what are they?

((Answer: `config/initializers/github.rb`, `Gemfile`, `Gemfile.lock`))

Next, we have to decide on a controller that will handle
authentication actions.  

# Handling Authentication actions

In the next section we will add a Moviegoers model and controller, and
you might think that would be the place to handle SSO, since it is the
Moviegoer who will be logging in.  But the Single Responsibility
Principle, which we'll learn about when we cover design patterns and
refactoring, says that any given class should have only one
responsibility, and in a MVC app, a resource's controller is already
responsible for handling the basic CRUD actions on the resource.

So, to keep concerns separate, we will create a `SessionController`
whose sole responsibility is to handle sign-in and sign-out.  In
particular, we will use it to abstract the concept of a _session_ (not
to be confused with Rails' `session[]` hash!),
which is created when the moviegoer signs in and destroyed when they
sign out.

* Why `SessionController` and not `SessionsController`?  (Look it up
in the Rails documentation.)

((Answer: A singular controller name tells Rails---and other
developers---that only a single instance of this resource will exist
within the scope of a particular HTTP request.  That is, while a single
request might refer to multiple `Movies` or `Moviegoers` (or
later, `Reviews`), a single request will refer to at most one `Session`.

A session cannot be edited or updated like a table-backed
resource, so we don't really need a separate session model: later we
will see that we can just set a particular key in the Rails
`session[]` when a session begins (user signs in), and delete that key when the
session ends (user signs out).  Any controller action can then check
for the presence of that key to condition an action on whether someone
is signed in.

1. Add a `SessionController < ApplicationController` in the right
place, define some  (empty) controller actions for `create`,
`destroy`, and `fail`.  (The latter will be used when the user is
unable to successfully sign in, e.g. forgotten password.) 

2. Modify `routes.rb` so that `/auth/:provider/callback` (whether via 'GET' 
or `POST`) maps to the `create` action, and `/auth/failure` maps to the 
`fail` action.  Also create a route for `GET /session/destroy`, which we'll 
use later.

* Why don't you need to map `GET /auth/provider`?

((Answer: OmniAuth intercepts that route automatically and redirects
the user to the SSO provider's login flow, so there is nothing for
your app to do.))

* Why isn't there an OmniAuth intercepted route that you should map to
the `SessionController#destroy` action?

((Answer: SSO providers don't always provide a way to sign out, but we
will sign out the user by deleting the appropriate session hash key.))

3. Modify the application view template `application.html.erb` to
include simple Sign In and Sign Out buttons.  Here's some recommended
code for placing this at the top of every page using Bootstrap:

In the `head` part of your layout:

```html
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-EVSTQN3/azprG1Anm3QDgpJLIm9Nao0Yz1ztcQTwFspd3yD65VohhpuuCOmLASjC" crossorigin="anonymous">
```

Near the top of the body of your layout:

```html
<div id="login" class="container" width="100%">
  <div class="row bg-light border-bottom">
    <div class="col-sm">
      <%= link_to "Sign In", href="LINK HERE", method: :post, class: 'btn btn-primary text-center' %>
    </div>
    <div class="col-sm">
      <a class="btn btn-danger  text-center" href="LINK HERE">Sign Out</a>
    </div>
  </div>
</div>
```

* What links should replace `LINK HERE` in the code above?

((Answer: for sign-in, `/auth/github`; for sign-out,
`/session/destroy`))

Let's test-drive the login flow.  In each of the `create` and `fail`
actions, force a bogus exception and show the contents of the auth
hash:

```ruby
raise StandardError.new(ENV["omniauth.auth"])
```

You should be able to verify that:

* clicking the Sign In button redirects you to a GitHub login flow;

* after successful GitHub sign in, you are redirected back to your
app's `SessionController#create` action;

* after a failed login with GitHub, you are redirected back to your
apps' `SessionController#fail` action.

Now we just need a model to represent someone who can login.

# 2. Add a Moviegoer model

To move things along quickly, we can _scaffold_ the Moviegoer
resource.

Consulting the official [Rails guides](https://guides.rubyonrails.org/v3.2/getting_started.html#creating-a-resource),
when you scaffold a resource, what is created for you?

((Answer: A migration to add the model, with columns you specify;
routes, controller actions, and views for the CRUD actions on the
resource.))

Since we will be using single sign on, at a minimum we need to track
the moviegoer's UID and "nickname" as provided by GitHub.  **Side
note:** If we were building an app that allows login via a variety of
different providers, a better approach would probably be to model user
identity as a separate entity, say an Identities table, that tracks
the provider, name, and UID for each SSO provider.  We'd then establish
a has-many relationship from users to identities.  That way the same
user can sign in using different SSO providers at different times but
still be in the same account.

To keep things simple, in this example app we will not collect or
manage any other info about the user.

1.  Scaffold the Moviegoer resource and specify `name` and `uid` as
the attributes of the moviegoer.  Run rake db:migrate to create the
new database table MovieGoer.

2.  We have to be careful with the Create and Update actions for
users.  Specifically, since we're using SSO exclusively (rather than
having users sign up manually with a username and password), a user
will only be Created the first time they try to sign in with SSO, and
cannot really be edited or updated since the only fields (name and
UID) are SSO-controlled.  So, get rid of the extraneous views and routes.

* What keywords can we use in `routes.rb` to restrict which routes are
created with resources :movie_goers

((Answer:  :only or :except))

3.  Add the following code to the MovieGoer model:
```ruby
    def self.from_omniauth(auth)
        where(uid: auth.uid).first_or_create do |moviegoer|
            moviegoer.uid = auth.uid,
            moviegoer.name = auth.info.name
        end
    end
```

This code will allow us to link the user's ID in our own app with the
provider's ID.  Now we need to modify our session_controller to call
this method.  Previously, we forces a bogus exception to throw an error
in the create method.  It's time to modify the create method to link
the different ID's and "login" the user to Rotten Potatoes.

Add the following code to the create method in `SessionController`:
```ruby
	auth = request.env["omniauth.auth"]
	user = MovieGoer.from_omniauth(auth)
	session[:user_id] = user.id
	flash[:notice] = "Logged in successfully."
	redirect_to movies_path
```
The first line will get the hash returned by omniauth so we can retrieve
information about the user.  Then we'll call the from_omniauth code that we 
previously placed in the MovieGoer model.  This will allow us to create a 
session when a user is successfully authenticated via the external provider.

In order to keep track of the authenticated user, we will add some code to 
the ApplicationController, which will be inerited by all controllers.  In
this code, we will establish the variable `@current_user` so that controller 
methods and views can just look at `@current_user` without being coupled to 
the details of how the user was authenticated.  `ApplicationController`
should contain the following code:
```ruby
class ApplicationController < ActionController::Base
	before_filter :set_current_user
	protected  # prevents method from being invoked by a route
	def set_current_user
		# we exploit the fact that the below query may return nil
		@current_user ||= MovieGoer.where(:id => session[:user_id])
		redirect_to movies_path and return unless @current_user
	end
end
```
The before_filter will fire before an action is run in the controllers,
and this filter will enforce that a user is logged in or else send the 
user back to the movies index page.
    
# 3. Add a Review model

As with the MovieGoer above, use the scaffolding method to create the 
Review resource.  

1.  You will need to edit the migration to contain the following code:
```ruby
class CreateReviews < ActiveRecord::Migration
  def change
    create_table 'reviews' do |t|
      t.integer    'potatoes'
      t.text       'comments'
      t.references 'moviegoer'
      t.references 'movie'
    end
  end
end
```
This code will create the new table reviews and add the necessary fields.
Note the two types called references.  These fields allow us to associate 
a review with the correct moviegoer and movie.

2.  Next, edit the review.rb model and add the following code:
```ruby
  belongs_to :movie
  belongs_to :moviegoer
```
3.  To complete the code required to establish the association, you will 
need to edit both the Movie and the Moviegoer class and add the following 
field (idiomatically, it should go right after 'class Movie' or 'class Moviegoer'):
```ruby
  has_many :reviews
```

# 4. Create the basic CRUD actions for reviews using nested routes
Now that the models are set up to represent the relationships between a Movie, a
MovieGoer, and a Review, we need a RESTful want to refer to actions associated
with movie reviews.  When creating or updating a review, how do we link it
to the moviegoer and movie?  Presumably the moviegoer will be the `@current_user`, 
but what about the movie?

It only makes sense to create a review when you have a movie in mind, therefore the 
most likely approach is for the "Create Review" functionality to be accessible from
a button or link on the Show Movie Details page for a specfic movie.  So at the
moment we display this control, we know what movie the review is going to be
associated with.  The question is how to get this information to the _new_ or
_create_ method in the `ReviewsController`.

We can create RESTful routes that will reflect the logical "nesting" of Reviews
inside of Movies, and this method will make the Movie ID explicit.  In `routes.rb`,
change the line _resources :movies_ to:
```ruby
resources :movies do
	resources :reviews
end
```
Since _Movie_ is the "owning" side of the association, it's the outer resource.
Just as the original resources `:movies` provided a set of RESTful URI helpers for
CRUD actions on movies, this _nested resource_ route specification provides a 
set of RESTful URI helpers for CRUD acions on _reviews that are owned by a movie_.  
Run _rake routes_ to see the additional routes that have been created.

Note that via convention over configuration, the URI wildcard `:id` will match the ID 
of the resource itself - that is, the ID of a review - and Rails chooses the 
"outer" resource name to make `:movie_id` capture the ID of the "owning" resource.
The ID values will therefore be available in controller actions as `params[:id]`
(the review) and `params[:movie_id]` (the movie with which the review will be
associated).

We are finally ready to create the views and actions associated with a new
review.  In the `ReviewsController`, we will add a before-filter that will check 
for two conditions before a review can be created:

1.  @current_user is set (that is, someone is logged in and will "own the new review).
2.  The movie captures from the route as params[:movie_id] exists in the database.

Add the following code to `ReviewsController`:
```ruby
before_filter :has_moviegoer_and_movie, :only => [:new, :create]
protected
def has_moviegoer_and_movie
	unless @current_user
		flash[:warning] = 'You must be logged in to create a review.'
		redirect_to movies_path
	end
	unless (@movie = Movie.where(:id => params[:movie_id]).first)
		flash[:warning] = 'Review must be for an existing movie.'
		redirect_to movies_path
	end
end
```
The current movie_id is passed in via params.  When we validate that the movie 
is present in the database, even though we are searching by id, the where clause
will return a hash.  So we need to add the _.first_ method to the results of
Movie.where so we only have a singular movie in the @movie variable.

The view uses the `@movie` variable to create a submission path for the form using 
the _movie_review_path_ helper.  When that form is submitted, once again _movie_id_ 
is parsed from the route and checked by the before-filter prior to calling the 
_create_ action.  Similarly, we could link to the page for creating a new review by
calling _link_to_ with the route helper _new_movie_review_path(@movie)_ as its
URI argument.

Add the following code to app/views/reviews/new.html.erb:
```html
<h1> New Review for <%= @movie.title %> </h1>

<%= form_tag movie_reviews_path(@movie), class: 'form' do %>
	<label class="col-form-label"> How many potatoes: </label>
	<%= select_tag 'review[potatoes]', options_for_select(1..5), class: 'form-control' %>
	<%= submit_tag 'Create Review', :class => 'btn btn-success' %>
<% end %> 
```
And to access the Create Review form while viewing the details of a specific movie, we
need to add a button to the Show Movie form.  Add the following line to the row of
buttons at the bottom of app/views/movies/show.html.erb:
```html
<%= link_to 'Review the movie', new_movie_review_path(@movie), :class 'btn btn-primary col-2' %>
```
Pressing this button should take you to the Create Review page.
