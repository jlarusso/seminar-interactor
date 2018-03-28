# Seminar: Monadic interactors make your life easier

### What are they?
Monadic interactors are something that we came up with on the platform team by combining the control flow benefits of [monads](https://codon.com/refactoring-ruby-with-monads) with the 'roles' aspect of [DCI architecture](http://www.artima.com/articles/dci_vision.html) (Both are previous seminar topics).

In a typical MVC application, lots of behavior-related code bloats the controllers and the models.  This makes code change a painful and error-prone experience. Using interactors is all about letting each component be responsible for one thing:

- The *controller* only deals with stuff related to the HTTP request.
- The *model* is all about fetching, persisting, and validating data.
- The *interactor* represents some kind of action like, "create a render for this project".  It may require different models and behavior to be composed together.

The way that we've decided to write these interactors is using a concept from functional programming called a monad. A monad is a really nice way to represent a chain of functions.  If we were writing an interactor for rendering a project, the chain of functions might look like:

1. validate that you have the params you will need
2. find the project
3. find or create the render
4. establish what resolution the video should be
5. update the project to be 'finalized' if the video is not a preview
6. enqueue the rendering job into redis

Monads wrap a value in some kind of an object, which is used to pass information to each link in the chain.  The value can contain anything, but there is the expectation that every one one of your functions takes this wrapper as an argument and also returns one.

We haven't even gotten to the good part yet.  My favorite thing about using monads is that error handling becomes pretty straightforward.  Instead of using conditionals or raise/rescue, you can declare that something has failed.  When this happens no subsequent functions in the chain are executed, and your value becomes an error object that you can use to deliver messaging to the caller.  Using the example above, this means that if the project is not found on step 2, steps 3-6 are never executed.  We exit straight out of the chain with an error object or something, which tells us what went wrong.

### Why are they good?
1. easy to test
2. easy to pick up what the code is doing / self documenting
3. errors are much more clear
4. avoids the pyramid of doom:

```
if first_thing
  if second_thing
    if third_thing
      if fourth_thing
        do_something
      end
    else
      do_another_thing
    end
  end
else
  do_something_else
end
```

### What do monadic interactors look like?
using the [deterministic gem](https://github.com/pzol/deterministic)...

```ruby
# Get a president's biography from wikipedia
class GetPresidentBioInteractor

  # Defining error codes makes it really easy for the caller to know what to expect.
  INVALID_FAILURE = "invalid_failure"
  SERVICE_FAILURE = "service_failure"

  # Here is our list of steps.  '.act' should be the only method a caller needs to know about.
  # Calling this method looks like:
  #   GetPresidentBioInteractor.act(:president_name => 'George Washington')
  def self.act(params)
    Success(params) >>
      method(:validate_params) >>
      method(:format_name) >>
      method(:scrape) >>
      method(:select_bio) >>
      method(:present_bio) >>
  end

  # Success on this step means that president_name is in the params.
  def self.validate_params(params)
    if params[:president_name].present?
      Success(params)
    else
      Failure(
        :code => INVALID_FAILURE,
        :message => "president_name is missing from the params."
      )
    end
  end

  # Here, we are adding new data to the params and passing it through.
  def self.format_name(params)
    params[:formatted_president_name] = params[:president_name].camelize
    Success(params)
  end

  # You should rescue non-exceptional errors that are raised and then return a Failure.
  # this makes it much simpler on the caller's side.  The caller doesn't have
  # to know how you made the request in order to rescue any errors.  All it
  # should care about is if it worked (and if not, why it didn't).
  def self.scrape(params)
    name = params[:formatted_president_name]
    params[:response] = Net::HTTP.get("wikipedia.org", "/wiki/#{name}")
    Success(params)
  rescue Timeout::Error, Errno::ECONNRESET => e
    Failure(
      :code => SERVICE_FAILURE,
      :message => e.message
    )
  end

  # Here, we are looking through the HTML in the response and trying to find the president's bio.
  def self.select_bio(params)
    params[:bio] = parse_through_html_and_select_bio(params[:response])
    if params[:bio].present?
      Success(params)
    else
      Failure(
        :code => SERVICE_FAILURE,
        :message => "The bio was not able to be parsed from the HTML response."
      )
    end
  end

  # The last step is to generate a string that contains the president's bio.
  def self.present_bio(params)
    Success("The bio for #{params[:president_name]}: #{params[:bio]}.")
  end
end



# OK so this is how you invoke the interactor:
# ------------------------------------------
result = GetPresidentBioInteractor.act(:president_name => 'Teddy Roosevelt')

if result.success? # => .success? is a helper method defined by the deterministic gem
  puts result.value
  # => "The bio for Teddy Roosevelt: Born a sickly child with debilitating asthma, Roosevelt..."
else
  case result.value[:code]
  when 'invalid_failure'
    puts "Sorry, there was something wrong with your request: #{result.value[:message]}."
  when 'service_failure'
    puts "Sorry, we could not get the president's bio: #{result.value[:message]}."
  end
end

```

Testing becomes really straightforward now.  Each chain in the monad can be tested independently and you can also test '.act' as a kind of integration.

# Activity: do the following and bring it to seminar
For this seminar, pick a small part of the codebase and implement a monadic interactor similar to the example above.  If you've done this before, feel free to challenge yourself by choosing to implement your interactor in a programming language you are less familiar with ([monet](https://cwmyers.github.io/monet.js/) is an example of a monad library for js).  You can also write it in pseudocode if you want.  Also, bonus points for specs.  On the day of the seminar class, we will go around and show what we have.  It doesn't need to be mergeable code; the idea is to just get some practice identifying refactor candidates and writing interactors.  Think about what you like/don't like about this approach.  What kind of situation does this work really well in?  Any suggestions on how to make our implementation of monadic interactors better?

### More examples
If you want more examples of how we've used these in real code, check out:

- [create project interactor](https://github.com/animoto/api/blob/master/app/interactors/create_project_interactor.rb)
- [create rendering job interactor](https://github.com/animoto/api/blob/master/app/interactors/create_rendering_job_interactor.rb)
- [exports interactors] (https://github.com/animoto/client_service/tree/master/app/interactors/exports)
- [asset library sync interactor] (https://github.com/animoto/api/blob/master/app/interactors/asset_library/sync_interactor.rb)

