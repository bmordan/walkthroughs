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
Rails has actually generated a unit test boilerplate for us. Look in