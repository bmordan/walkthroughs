# Rails React with Redux

Having an application with lots of different components is to be expected. How can we share state between components? Often a few different components are going to have their state shared. For example is a user logged in or out? Redux gives our react an application wide state, we can then connect our different components to that shared state (redux calls it the "store").

We are going to make a simple tamagochi. Its a little browser pet that we will have to play with and feed. Let us create the rails project, add react and redux.

```sh
rails new tamagochi-redux --webpack=react
```
cd into the project then add react and redux.
```
bundle add react-rails
rails g react:install
rails webpacker:install:react
```
Lets create an App component and make sure we can server that to our frontend.
```
rails g react:component Pet
rails g controller pets
```
Now for some wiring. Update the routes file
```ruby
# /config/routes.rb
Rails.application.routes.draw do
  root "pets#index"
end
```
and the controller
```ruby
# /app/controllers/pets_controller.rb
class PetsController < ApplicationController
  def index
    render component: "Pet"
  end
end
```
and finally put a little pet onto your page.
```js
// /app/javascript/components/Pet.js
import React from "react"

export default function () {
  return (
    <h1 style={{writingMode: "vertical-rl;"}}>:)</h1>
  )  
}
```
cute. Now we are going to create a series of react components and have them all share the redux store's state.