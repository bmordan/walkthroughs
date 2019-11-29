# Rails with React User Accounts

We are going to create a simple app in which you can save your favorite links. To start collecting links we need to create user accounts secured by a password. There are three things that form the components of our implementation of user authentication: sessions, middleware and hashing.

* Sessions - this is where we store our user id and it is createing and destroying a session that constitutes login in and login our
* Middleware - we need to check each request, are you logged in? If you are not logged in we are going to redirect you so you can't see stuff you shouldn't. We will do this in the middle of every request.
* Hashing - you create an account an provide a password, we are not going to store this in plain text for all to see and access in the database. We are going to save it in an encrypted for called a digest. That way we can keep your password secret.

Good to go?

```
rails new save-my-links --webpack=react 
```
```
bundle add react-rails bcrypt
```
```
rails g react:install
```
```
rails webpacker:install:react
```
lets generate our models [User, Links] and establish their relationships
```
rails g model User name:string password_digest:string
```
Thats our user we need that field labeled password_digest to benefit from rails helpers
```
rails g model Link url:string user_id:integer
```
Lets edit the models a little before running our migration
```ruby
# /app/models/user.rb
class User < ApplicationRecord
    has_secure_password
    has_many :links
end
```
the user has many links, and we set up password hashing on save, so a hash of the password is saved when a user is created.
```ruby
# /app/models/link.rb
class Link < ApplicationRecord
  belongs_to :user
end
```
Before we spin up for the first time we should create some routes and a couple of controllers.
```
rails g controller sessions
rails g controller users
rails g controller links
```
and in `/config/routes.rb`
```ruby
Rails.application.routes.draw do
  root "sessions#index"
  post "/login", to: "sessions#create", as: "login"
  get "/logout", to: "session#destroy", as: "logout"

  resources :users do
    resources :links
  end
end
```
Check with `rails routes` that you have routes like this `user_link GET /users/:user_id/links/:id(.:format)` with the `:user_id` parameter. Finally to see something we need a user account create form so make that.
```html
<!-- /app/views/users/new.html.erb -->
<%=form_with model: @user, local: true do |f|%>
    <%=f.text_field :name, placeholder: "Username"%>
    <%=f.password_field :password, placeholder: "password"%>
    <%=f.submit%>
<%end%>
```
Its ok this is a erb template. To see the react component you need to be logged in. We need to complement this with some additions to our controller so update it like this.
```ruby
# /app/controllers/user_controller.rb
class UsersController < ApplicationController
    def new
        @user = User.new
    end

    def show
        render component: "User", props: {user: User.find(params[:id])}
    end

    def create
        user = User.create(user_params)
        redirect_to user_path(user)
    end

    private

    def user_params
        params.require(:user).permit(:name, :password)
    end
end
```
Here in the `new` method we pass down our empty model for the user to add data to. Then in create we create the user (hash their password) and redirect to `"users#show"` and render our React component with a user in the props. Lets make the react component.
```
rails g react:component User user:object
```
and then open the generated file and just alter the component to return the users, name and hashed password
```js
class User extends React.Component {
  render () {
    const {name, password_digest} = this.props.user
    return (
      <div>
        User: {name} {password_digest}
        <a href="/logout">Logout</a>
      </div>
    )
  }
}
User.propTypes = {
  user: PropTypes.object
}
export default User
```
You can stop here and spin up your rails server. Make sure nothing is out of place. All ok? At this point we can create a user and show them on a page. Lets only show that page when you are logged in. We'll need a login form. Please create one.
```html
<!-- /app/views/sessions/index.html.erb -->
<%=form_with scope: :session, url: login_path, local: true do |f|%>
    <%=f.text_field :name, required: true, placeholder: "username"%>
    <%=f.text_field :password, type: :password, required: true, placeholder: "password"%>
    <%=f.submit%>
<%end%>
<%=link_to "create an account", new_user_url%>
```
Notice how here in the form we are using `scope: :session` as sessions are not backed by a model, they just exist in memory for the life of the application. You should add the `index` method in the sessions controller. We set this as our root path so you should be able to reach this route on `http://localhost:3000/`. Look at the form's post route, where will we handle this?

In the sessions controller we are going to receive a payload of username and password. We are going to check these then if all is well add them to the session, if they are not ok we wont. Add the following to your session controller.
```ruby
# /app/controllers/session_controller.rb
class SessionsController < ApplicationController
    def index
    end

    def create
        user = User.find_by(name: params[:session][:name])

        if user && user.authenticate(params[:session][:password])
            session[:user_id] = user.id.to_s
            redirect_to user_path(user)
        else
            redirect_to root_path
        end
    end

    def destroy
        session.delete(:user_id)
        redirect_to root_path
    end
end
```
`has_secure_password` and the bcrypt gem are adding the "user#authenticate" method (can you see that in the code above) which will return `true` if the text string password matches the hashed password. If there is a match we put the user's id (as a string) in the session. Finally we can remove the user from the session by creating the destroy method that removes the user's id from the session. Try this out login and logout.

This doesn't really work yet. You can visit any of the routes. Nothing is protected. We can still visit the users pages regardless of being logged in or out. To protect routes we want to interupt each request and check that the user is logged in. Goto the `applications_controller.rb` and add a middleware function that will run on each request.
```ruby
# /app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  helper_method :current_user

  def current_user
    @current_user ||= User.find(session[:user_id]) if session[:user_id]
  end

  def authorise
    redirect_to root_path if current_user.nil?
  end
end
```
Here we create a helper method that is going to cache the logged in user to `@current_user` which our templates and other controllers can access. We are also checking this for each request and will redirect if there is no user. Before you try this. We'll be in a auth loop if we dont allow the creation of a user so we are going to skip this authorise call in the users_controller add the following line at the top of the class before all the methods.
```ruby
  before_action :authorise, except: [:new, :create]
```
As it says we are going to make an exception and not authorise the `new` method and the `create` method. What will happen if we don't do this? Now try this out.

One final problem, if you have a couple of users, you can swap out their id in the URL and see their page. Stop this. Add another condition to the `authorise` method, something like this:
```ruby
  def authorise
    return redirect_to root_path if current_user.nil?
    return redirect_to user_path(current_user) if (params[:id] && params[:id] != session[:user_id])
  end
```
I'm returning in the method because if one of these is true I want the method to stop executing.
This secures our app with a very basic login logout. You can extend this by adding the ability to add links to the users page. Maybe some UI to show when you are logged in or logged out.
