# hanami-architecture

Ideas and suggestions about architecture for hanami projects

## Table of Contents

* Application rules
  * API
    * Serializers
  * HTML
    * Forms
    * View objects
* IoC containers
  * How to load all dependencies
  * `Import` object
  * Testing
* Interactors, operations and what you need to use
  * Hanami-Interactors
  * Dry-transactions
  * Testing
* Domain services
* Service objects, workers
* Event sourcing

## Application rules
All logic for displaing data should be in applications.

If you alllication include custom middleware it should be in apps/app\_name/middlewares/ folder

### API

#### Serializers

### HTML

#### Forms

#### View objects

## IoC containers
[IoC containers](https://gist.github.com/blairanderson/8072d951a480a590f0bd) is prefered way to work with project dependencies.

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

### `Import` object
For loading dependencies to other classes use `dry-auto\_inject` gem. For this you need to create `Import` object:

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
For testing you code with dependencies you can use two ways.

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

We suggest to use mocks only for not DI dependencies like persistance connections.

## Interactors, operations and what you need to use

### Hanami-Interactors

### Dry-transactions

## Domain services

## Service objects, workers

## Event sourcing
