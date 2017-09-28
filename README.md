# hanami-architecture

Ideas and suggestions about architecture for hanami projects

## Table of Contents

* Application rules
  * Actions
  * View
  * API
    * Serializers
  * HTML
    * Forms
    * View objects
* IoC containers
  * How to load all dependencies
  * How to load system dependencies
  * `Import` object
  * Testing
* Interactors, operations and what you need to use
  * Hanami-Interactors
  * Dry-transactions
  * Testing
* Domain services
* Service objects, workers
* Models
  * Command pattern
  * Repository
  * Entity
  * Changesets
* Event sourcing

## Application rules
All logic for displaying data should be in applications.

If your application include custom middleware, it should be in apps/app_name/middlewares/ folder

### Actions
Actions it's just a transport layer of hanami projects. Here you can put:
1. request logic
2. call business logic (like services, interactors or operations)
3. Sereliaze response
4. Validate data from users
5. Call simple repository logic (but you need to understand that it'll create tech debt in your project)

```ruby
module Api::Controllers::Issue
  class Show
    include Api::Action
    include Import['tasks.interactors.issue_information']

    params do
      required(:issue_url).filled(:str?)
    end

    # bad, business logic here
    def call(params)
      if params[:action] == 'approve'
        TaskRepository.new.update(params[:id], { approved: true })
        ApproveTaskWorker.perform_async(params[:id])
      else
        TaskRepository.new.update(params[:id], { approved: false })
      end

      redirect_to routes.moderations_path
    end

    # good, we use intecator for updating task and sending some to background
    def call(params)
      TaskStatusUpdater.new(params[:id], params[:action]).call
      redirect_to routes.moderations_path
    end
  end
end
```

### API

#### Serializers

### HTML

#### Forms

#### View objects

## IoC containers
[IoC containers](https://gist.github.com/blairanderson/8072d951a480a590f0bd) is preferred way to work with project dependencies.

We suggest to use [dry-containers](http://dry-rb.org/gems/dry-container/) for working with containers:

```ruby
# in lib/container.rb
require 'dry-container'

class Container
  extend Dry::Container::Mixin

  register('core.http_request') { Core::HttpRequest.new }

  namespace('services') do
    register('analytic_reporter') { Services::AnalyticReporter.new }
    register('url_shortener') { Services::UrlShortener.new }
  end
end
```

Use string names as a keys, for example:
```ruby
Container['core.http_lib']
Container['repository.user']
Container['worders.approve_task']
```

You can initialize dependencies with different config:

```ruby
# in lib/container.rb
require 'dry-container'

class Container
  extend Dry::Container::Mixin

  register('events.memory_sync') { Hanami::Events.initialize(:memory_sync) }
  register('events.memory_async') { Hanami::Events.initialize(:memory_async) }
end
```

### How to load all dependencies

### How to load system dependencies
For loading system dependencies you can use 2 ways:
1. put all this code to `config/initializers/*`
2. use [dry-system](http://dry-rb.org/gems/dry-system/)

#### Dry-system
This libraty provide a simple way to load your dependency to container. For example you can load redis client or API clients here. Check this links as a example:
* https://github.com/ossboard-org/ossboard/tree/master/system
* https://github.com/hanami/contributors/tree/master/system

After that you can use container for other classes.

### `Import` object
For loading dependencies to other classes use `dry-auto_inject` gem. For this you need to create `Import` object:

```ruby
# in lib/container.rb
require 'dry-container'
require 'dry-auto_inject'

class Container
  extend Dry::Container::Mixin

  # ...
end

Import = Dry::AutoInject(Container)
```

After that you can import any dependency in to other class:

```ruby
module Admin::Controllers::User
  class Update
    include Admin::Action
    include Import['repositories.user']

    def call(params)
      user = user.update(params[:id], params[:user])
      redirect_to routes.user_path(user.id)
    end
  end
end
```

### Testing
For testing your code with dependencies you can use two ways.

The first, DI:
```ruby
let(:action) { Admin::Controllers::User::Update.new(user: MockUserRepository.new) }

it { expect(action.call(payload)).to be_success }
```

The second, mock:
```ruby
require 'dry/container/stub'

Container.enable_stubs!
Container.stub('repositories.user') { MockUserRepository.new }

let(:action) { Admin::Controllers::User::Update.new }

it { expect(action.call(payload)).to be_success }
```

We suggest using mocks only for not DI dependencies like persistent connections.

## Interactors, operations and what you need to use

Interactors, operations and other "functional objects" needs for saving your buisnes logic and they provide publick API for working with domains from other parts of hanami project. Also, from this objects you can call other "private" objects like service or lib.

### Hanami-Interactors
Interactors returns object with state and data:
```ruby
# in lib/users/interactors/signup
require 'hanami/interactor'

class Users::Intecators::Signup
  include Hanami::Interactor
  expose :user

  def initialize(params)
    @params = params
  end

  def call
    find_user!
    singup!
  end

  private

  def find_user!
    @user = UserRepository.new.create(@params)
    error "User not foound" unless @user
  end

  def singup!
    Users::Services::Signup.new.call(@user)
  end
end

result = User::Intecators::Signup.new(login: 'Anton').call
result.successful? # => true
result.errors # => []
```

Links:
* https://github.com/hanami/utils/blob/master/lib/hanami/interactor.rb

### Dry-transactions

## Domain services
We have applications for different logic. That's why we suggest using DDD and split you logic to separate domains. All these domains should be in `/lib` folder and looks like:

```
/lib
  /users
    /interactors
    /libs

  /books
    /interactors
    /libs

  /orders
    /interactors
    /libs
```

Each domain have "public" and "private" classes. Also, you can call "public" classes from apps and core finctionality (`lib/project_name/**/*.rb` folder) from domains.

![hanami-project](https://github.com/davydovanton/hanami-architecture/blob/master/images/project.png?raw=true)

Each domain should have a specific namespace in a container:

```ruby
# in lib/container.rb
require 'dry-container'

class Container
  extend Dry::Container::Mixin

  namespace('users') do
    namespace('interactors') do
      # ...
    end

    namespace('services') do
      # ...
    end

    # ...
  end
end
```

Each domain should have public interactor objects for calling from apps or other places (like workers_) and private objects as libraries:

```ruby
module Admin::Controllers::User
  class Update
    include Admin::Action
    # wrong, private object
    include Import['users.services.calculate_something']

    # good, public object
    include Import['users.interactor.update']

    def call(params)
      # ...
    end
  end
end
```

## Service objects, workers

## Event sourcing
