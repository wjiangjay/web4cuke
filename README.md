The main filosophy behind this project is:
Separate the data from the logic as much as you possibly can. Have you had to
go through a tremendous ruby module with hundreds of methods which click
buttons, input text in form fields, randomly wait (probably, till the page is
loaded) and randomly take screenshots? Have you dreamed to have a way to store
test data in a separate simple text files and feed them to the cucumber? 
Or maybe you have hundreds of scenarios looking like this: 
``` 
 When I click the element ":id=>'click_me'"
 And I write "hello" in the form with ":id=>'topic'"
 And I write "Blahblahblah" in the form with ":id=>'body'"
 And I click the element ":id=>'submit'"
 Then the page shuold contain "Success"
```
And you sometimes ask yourselves "Is it a cucumber way to write scenarios like
this?" Well the answer is no. Probably, all you need is to write a couple of
steps instead:
```
 When I create a blogpost with:
  |title|body|
  |hello|Blahblahblah|
 Then the step should succeed
```
And have a simple yaml file describing the corresponding webpage, that would
look like this:
```
blogpost_create:
  pages:
    - 'blogpost_create_page'
blogpost_create_page:
  url: '/blogposts/new'
  expected_fields:
    title:
      type: 'textfield'
      selector:
        id: 'topic'
    body:
      type: 'textfield'
      selector:
        id: 'body'
  commit:
    selector:
      id: 'commit'
```
This library implements an abstraction layer between cucumber logic and Watir
webdriver. Using this library you no longer need to write the Watir-aware Ruby
code for low-level browser interaction. Instead you describe your pages and
actions to be performed in Web UI of your product in simple yaml-formatted
files. This approach allows you to separate the application data (html
properties of page elements) from test logic. 

How to use this? 

- Add `gem 'web4cuke'` to you Gemfile and run `bundle install`
- In your project in one of your lib/*.rb files add `require "web4cuke"` and inherit your own class from Web4Cuke one, like this:
```
class Web < Web4Cuke
  def initialize(options)
  super(options)
  # Add here some code specific to your project, like 
  # @@logged_in = false
end
```

- In your features/support/env.rb in Before hook instatiate your beautiful class:
```
  options = {
    :base_url => "http://base_url_of_your_project",
    :browser => :firefox, # or :chrome. Other browsers are not supported yet
    :rules_path => "path_to_the_folder_with_your_yaml_files",
    :logger => @logger 
    # You need to pass an object that will do the logging for you. I believe you have it implemented.
    # If not, please take a look in the examples/testproject folder.
  }
  @web = Web.new(options)

```
From now on you will have @web, an instance of Web4Cuke class with a bunch of convenient methods for high-level interactions with the browser, and, a browser running by default in the headless mode. For development and debugging purposes, however I recommend setting DEBUG_WEB environmental variable to *true* so that you will be presented with a visible browser window.

- Once this preparation is done, let's try to understand how to write yaml files. The key concept web4cuke is built around is an *action*. 
Action is any set of user actions you want to automate, it could be web search, form submissions, data upload etc. An action is described in a yaml file with the following structure:

```
our_test_action:
  pages:
    - 'first_page'
    - 'second_page'
  final_checkpoints:
    alert_success:
      selector:
        text: 'You have successfully performed whatever you intended'

first_page:
  url: '/some_relative_path/testme'
  expected_fields:
     field_one_on_page_one:
       type: textfield
       selector:
         id: 'i-am-field-1'
     field_two_on_page_one:
       type: textfield
       selector:
         class: 'generic-field-2-class'
  checkpoints:
    sometext:
      selector:
        text: 'I am text one on page one'
    someothertext:
      selector:
        text: 'I am text two on page one'
  commit:
    selector:
      text: 'Click me'

second_page:
  sleep 2 # wait 2 seconds till the page is loaded
  expected_fields:
    field_one_on_page_two:
      type: filefield
      selector:
        id: 'upload-something'
    field_two_on_page_two:
      type: textfield
      selector:
        xpath '//*[@id="fancy_something"]/span'
      def_value: 'If you dont pass :field_two_on_page_two value, this text will go there'
  checkpoints:
    sometext:
      selector:
        text: 'I am text one on page two'
    someothertext:
      selector:
        text: 'I am text two on page two'
  negative_checkpoints:
    error_message:
      selector:
        text: 'I am a critical error message! Wish you never see me on this page!'
  commit:
    scroll: true 
    # sometimes the element you need is outside the area of the virtual viewport
    # so webdriver is unable to interact with it. Use this keyword to execute a simple 
    # javascript *scroll_into_view* function
    selector:
      text: 'Click me too'
``` 
If you take a closer look at this yaml structure, you'll notice that it describes describes two web pages pages. The first one is accessed through relative url "/some_relative_path/testme" that is being appended to your project's base_url (used during Web initialization).
The second page is accessed by clicking the "Click me" button on the first page. The yaml file describes 2 textfields on first page and one textfield and one filefield on the second page. It also describes checkpoints - elements whose presence will be asserted during the action execution. We can provide default values for textfields right inside the yaml with the *def_value* keyword. Now the most interesting question: how do we use that?

- The whole workflow would look like this. You would create your step:

```
When I run my beautiful action with:
  |option                |value           |
  |field_one_on_page_one |some text       |
  |field_two_on_page_one | some other text|
  |field_one_on_page_two| lib/files/myfile| 
  # We remember that field_one_on_page_two is a filefield
  # And we leave default value for second field on the page two.
```
Then you would write your step definition:
```
When /^I run my beautiful action with:$/ do |table|
  options = {}
  table.rows.each do |row|
    options[row[0].to_sym] = row[1]
  end
  @result = @web.run_action(:our_test_action, options)
end
```

If you then inspect the @result object then ideally you would get something like this:
```
pp @result
{:result=>true,
 :failed_positive_checkpoints=>[],
 :failed_negative_checkpoints=>[],
 :errors=>[]}

```
If something went wrong and the webdriver was unable to find some checkpoints or other fields described in the yaml file, then the @result[:result] would be changed to *false* and @result[:failed_positive_checkpoints] array will be populated with the elements that were not found, @result[:failed_negative_checkpoints] - with elements you described in *negative_checkpoints* and @result[:errors] - with exception messages. Also, a *screenshots* folder will be created in your working directory, containing browser screenshot of the page that caught an error.