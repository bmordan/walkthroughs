# Rails Upload a File

It is very useful to allow users to transfer files into your app. For example this functionality might enable a user to upload a photo for their avatar, or an image of a property they are selling. Rails gives use a module to deal with this feature. It's called ActiveStorage.

## Set up

In your rails project add ActiveStorage.

```
rails active_storage:install
```
This is going to add a couple of tables to your database (active_storage_blobs, active_storage_attachments). So you are going to have to run a migration.
```
rails db:migrate
```
So we are going to store image files in our example. That means we are going to have to update our model. So make sure your user model (username:string, password_digest:string) has this in it's model class definition.
```ruby
class User < ApplicationRecord
    has_secure_password
    has_one_attached :avatar
end
```
and that is it for a simple upload to your server. There is more you can do (see the documentation) for example you can connect cloud storage services (AWS, Azure, Google).

## Uploading and attaching files to instances

We need to present to the user a way to pick the file they want to use as an image. Lets create a simple form to create a user with an avatar.
```ruby
<%=form_with model: User.new, local: true, enctype: "multipart/form-data" do |h|%>
    <%=h.text_field :username, placeholder: "username" %>
    <%=h.password_field :password, placeholder: "password"%>
    <%=h.file_field :avatar%>
    <%=h.submit "Create"%>
<%end%>
```
Notice you have to add `enctype: "multipart/form-data"` to the form so you can send a file binary with the other form values. Also can you see the field for uploading a file has the type `file`. An example of a file field in plain html is below
```html
<input type="file" name="avatar" />
```
Now lets have a look at the controller code to deal with this incoming payload.
```ruby
  def create
    user = User.create(user_params)
    user.avatar.attach(params[:user][:avatar])
    session[:user_id] = user.id
    redirect_to user_url(user)
  end
  
  private

  def user_params
    params.require(:user).permit(:username, :password)
  end
```
We are creating a new `User` with the `:username` and `:password` fields only, so we just permit these in the validation function called `user_params`, however you should be aware we have still sent the file information, and we use that in the next step. We are not saving the file data in the `users` table.

In a separate step we attach the image data to the user property we named in the User model (see below).
```ruby
class User < ApplicationRecord
    has_secure_password
    has_one_attached :avatar
end
```
This macro `has_one_attached` is adding another property to the model. That property is not a column in that model's table, it is extending instances of the user with a `user#avatar` property. Rails is storing your image elsewhere, then attaching it instances of a user when they are created.

## Using the image

We can upload files, now how do we reference them? Its very easy. We have a template helper we can use called `url_for` so displaying a users avatar in the browser is as easy as:
```html
<article>
    <img src="<%=url_for(user.avatar)%>%" />
    <h1><%=user.username%></h1>
</article>
```
That is going to generate some thing like this in your html
```
http://localhost:3000/rails/active_storage/blobs/eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaHBCZz09IiwiZXhwIjpudWxsLCJwdXIiOiJibG9iX2lkIn19--b165c4edceab34a586d6fb7c02d2b4720601bdf9/bobthebuilderweb_1663316a.jpg%
```
Maybe you want to list a series of file you can download. For that you can generate a download link like this:
```
<a href="<%=rails_blob_path(user.avatar, disposition: "attachment")%>">Download</a>
```
That is going to give you something like this:
```
/rails/active_storage/blobs/eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaHBCZz09IiwiZXhwIjpudWxsLCJwdXIiOiJibG9iX2lkIn19--b165c4edceab34a586d6fb7c02d2b4720601bdf9/bobthebuilderweb_1663316a.jpg?disposition=attachment
```
There is more you can do with [ActiveStorage](https://guides.rubyonrails.org/v5.2/active_storage_overview.html) but this is enough to get started.




