# Rails Todo (with tests)

Shipping code without tests is like leaving the house without your wallet. At first it's fine, but sooner or later you are going to have to make a purchase and your life will stop working.

Tests don't make our code 100% reliable, but they help. The three things we want to do with our tests is make sure as best we can out code is free from:

* errors
* faults
* failures

There are three different kinds of tests that every developer should know how to write.

* unit
* intergration
* system

Unit tests prove a single function, operation works as expected. The word "unit" describes a small piece of code where we test that given a set of inputs the code produces the correct output/outputs.

Intergration testing is taking a few different parts of your program and proving that they all interact correctly. For example, does your controller create the correct record in the database? and return the correct template, with the correct data?

System testing is proving your whole system works. Because it is not possible to subject a system to every possible state it might encounter, we use "test cases" and establish that the main paths or most important parts of the system perform as expected.

We are going to create another todo list, but this time cover our asses by writing a few examples of each type of test.

## Unit Tests

Begin with a new rails project and create the model for our Todo instances.

```sh
rails new todo-with-tests
cd todo-with-tests
rails g model Todo task:string status:integer
rails db:migrate
```
Rails has actually generated a unit test boilerplate for us. Look in `/test/models/todo_test.rb`. Lets create a simple test:
```ruby
# /test/models/todo_test.rb

require 'test_helper'

class TodoTest < ActiveSupport::TestCase
  test "there are some todos" do
    assert_equal(Todo.all.count, 2)
  end
end
```
This is using a testing library called [minitest](https://github.com/seattlerb/minitest). How come there are 2 Todo's? To understand this you need to look in to the `/test/fixtures/todos.yml` file there you can see test data that seeds the test database. So each time we run the tests we can set them up with data from this file. You can write erb in this file too (follow the link for an example). To run the tests type the command `rails test`.
```yml
# /test/fixtures/todos.yml

one:
  task: Task one
  status: 0

two:
  task: And another task
  status: 0
```
Lets do a little bit of Test Driven Development (TDD). All tasks should be created with a default status of 0. Create the following failing test:
```ruby
# /test/models/todo_test.rb

require 'test_helper'

class TodoTest < ActiveSupport::TestCase
  test "todos are created with a default of status 0" do
    todo = Todo.create({task: "some task"})
    assert_equal(todo.status, 0)
  end
end
```
It fails because we don't pass in a value for "status" so it defaults to `nil`. Lets correct this in the model. Add the following line to the Todo model.
```ruby
# /app/models/todo.rb

class Todo < ApplicationRecord
    attribute :status, default: 0
end
```

Nice.

## Intergration Tests

Very valuable intergration tests. Here we test how different parts of our programme work together. To generate intergration tests we first need to build out a bit more of our app. We can do this by writing tests first.
```
rails g integration_test todo_flows
```
Now we have some bootstraped intergration files.
```ruby
# /test/integration/todo_flows_test.rb

require 'test_helper'

class TodoFlowsTest < ActionDispatch::IntegrationTest
  test "can create a todo" do
    post "/todos", params: {todo: {task: "integration todo test"}}
    assert_response :redirect
    follow_redirect!
    assert_select "main article:last-child", /integration todo test/
  end
end
```
Above we are posting to the `/todos` endpoint with our params. The controller should create a todo, then redirect us back to the "todos#index", we are then asserting that within the HTML being returned to the client, there is displayed our todo item. BUT of cause this will fail. We need to create our controller. `assert_select` is a method that will take a css selector, and a regexp. Its the last child because the fixtures contain two seeded todo items.
```sh
rails g controller todos index
```
then update the controller to look like this.
```ruby
# /app/controllers/todos_controller.rb

class TodosController < ApplicationController
  def index
    @todos = Todo.all
  end

  def create
    todo = Todo.create(todo_params)
    redirect_to todos_path
  end

  private

  def todo_params
    params.require(:todo).permit(:task)
  end
end
```
And lets make the view while we are here
```html
# /app/views/todos/index.html.erb

<main>
    <%@todos.each do |todo|%>
        <article>
            <p><%=todo.task%> <%=link_to todo.status == 0 ? "DONE" : "DELETE", todo_path(todo), method: todo.status == 0 ? :patch : :delete%>
            </p>
        </article>
    <%end%>
</main>
<hr />
<%=form_with model: Todo.new do |helper|%>
    <%=helper.text_field :task, placeholder: "add a task"%>
    <%=helper.submit "Add"%>
<%end%>
```
Now we know our controller is doing the right things. What if we send in the wrong params? Try adding the test below. We should not be able to send an `:sql` parameter to out controller. We are not error handling so I would expect to redirect to the index page, but not create that todo.
```ruby
# /test/integration/todo_flows_test.rb

require 'test_helper'

class TodoFlowsTest < ActionDispatch::IntegrationTest
  test "can create a todo" do
    post "/todos", params: {todo: {task: "integration todo test"}}
    assert_response :redirect
    follow_redirect!
    assert_select "main article:last-child", /integration todo test/
  end

  test "should only create a todo with the :task param" do
    post "/todos", params: {todo: {sql: "DROP TABLE todos;"}}
    assert_response :redirect
    follow_redirect!
    assert_not_equal "main article:last-child", /DROP TABLE todos;/
  end
end
```

## System Tests

The last kind of test we are going to look at is whole system testing. Lets start this off with the following generator:
```
rails g system_test todos
```
This is going to run our whole application. We can programatically visit pages and do things just like users do in an actual browser. One of the main flows through our app is the ability to create todos, so we should create a test that ensures this functionality.
```ruby
# /test/system/todos_test.rb

require "application_system_test_case"

class TodosTest < ApplicationSystemTestCase
  test "you can create todos" do
    visit todos_url
    
    assert_selector "article", {count: 2}

    fill_in 'todo_task', with: "new system test todo"
    click_button 'Add'

    assert_selector "article", {count: 3}
    assert_selector "article", {maximum: 1, text: "new system test todo"}
  end
end
```
You have to run this with the following:
```
rails test:system
```
Here we visit the `todos_url` "/todos" there are 2 todo's on the page. Then we `fill in` the task field with a string, and click the "Add" button. Then we assert that on the page there is now another todo. Finally we assert there is no more that 1 todo with the text string we set in our test. You should be able to see a browser pop up and quickly perform all your programmed actions.

Now we have an app here where we are covering our MVC (model, view, controller) with unit, integration and system tests. It takes extra time to write tests, but in most organisations you are unable to submit code that is not tested.
