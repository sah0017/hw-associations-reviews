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
provider supported by [OmniAuth]().

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
([ESaaS](http://www.saasbook.info) section 5.2).

* You should be familiar with the concepts and basic mechanics
(`has_many`, `belongs_to`, and so on) of ActiveRecord Associations
([ESaaS](http://www.saasbook.info) sections 5.4-5.6).


# 1. Set up for SSO using OmniAuth for "Login with GitHub"

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

* `(GET|POST) /auth/:provider/callback`: if the user successfully signs in
via SSO, your app will receive a request to this route.  You should
therefore arrange to have this route map to a controller action in
which you take steps to "sign the user in", such as remembering
the user's ID in the `session[]`.  When this route is called, OmniAuth
makes available a hash called `auth_hash[]`  containing information about the user, provided
by the SSO provider.  The specific information varies depending on the
provider, but there are a couple of hash keys that are guaranteed to
be present, which we will use.  The route should be accessible via
either `GET` or `POST` due to differences in how SSO providers work.

Consult the OmniAuth project's wiki (on GitHub) to answer the
following questions in preparation to add OmniAuth via GitHub:

* What keys of `auth_hash` are **always** guaranteed to be present
after a successful auth, regardless of which provider is used?

((Answer: `auth[:provider]`, `auth[:uid]`, `auth[:info][:name]`))

* The `omniauth-github` Gem page on GitHub (find it) specifies that
you must add the following line in your Gemfile:

`gem 'omniauth-github', github: 'omniauth/omniauth-github', branch: 'master'`

* Why don't you have to add the `omniauth` gem itself?

((Answer: since every strategy gem depends on it, it will be pulled in
by default by Bundler.))

Follow the instructions for `omniauth-github` to setup your Rack
middleware to intercept the above routes.  

* At this point, you should be committing three new or modified files---what are they?

((Answer: `config/initializers/github.rb`, `Gemfile`, `Gemfile.lock`))

Next, we have to decide on a controller that will handle
authentication actions.  
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

2. Modify `routes.rb` so that `/auth/:provider/callback` (using `POST`) maps 
to the `create` action, and `/auth/failure` maps to the `fail` action.  Also 
create a route for `GET /session/destroy`, which we'll use later.

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
    <div class="col-2">
      <a class="btn btn-primary text-center" href="LINK HERE">Sign In</a>
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

Consulting the official [Rails
guides](https://guides.rubyonrails.org/v3.2/getting_started.html#creating-a-resource),
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

* What keywords can we use in routes.rb to restrict which routes are
created with resources :movie_goers

((Answer:  :only or :except))

3.  Add the following code to the MovieGoer model:
    def self.create_with_omniauth(auth)
        Moviergoer.create!(
            :uid => auth["uid"],
            :name => auth["info"]["name"]
        )

    
# 3. Add a Review model

As with the MovieGoer above, use the scaffolding method to create the 
Review resource.  
1.  You will need to edit the migration to contain the following code:
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

This code will create the new table reviews and add the necessary fields.
Note the two types called references.  These fields allow us to associate 
a review with the correct moviegoer and movie.

2.  Next, edit the review.rb model and add the following code:
  belongs_to :movie
  belongs_to :moviegoer

3.  To complete the code required to establish the association, you will 
need to edit both the Movie and the Moviegoer class and add the following 
field (idiomatically, it should go right after 'class Movie' or 'class Moviegoer'):
  has_many :reviews

# 4. Create the basic CRUD actions for reviews using nested routes
