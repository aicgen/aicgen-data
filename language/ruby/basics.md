# Ruby Fundamentals

## Project Structure (Rails)

```
myapp/
├── app/
│   ├── controllers/
│   ├── models/
│   ├── services/
│   ├── views/
│   └── jobs/
├── config/
├── db/
│   └── migrate/
├── lib/
├── spec/ or test/
├── Gemfile
└── Gemfile.lock
```

## Naming Conventions

```ruby
# Classes/Modules: PascalCase
class UserService
end

module Authentication
end

# Methods/variables: snake_case
def create_user(email)
  user_name = "test"
end

# Constants: SCREAMING_SNAKE_CASE
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30

# Predicate methods: end with ?
def valid?
  @errors.empty?
end

# Dangerous methods: end with !
def save!
  raise Error unless save
end

# Private attr: prefix with _
attr_reader :_internal_state
```

## Idiomatic Ruby

```ruby
# Blocks
users.each do |user|
  puts user.name
end

# Short blocks with &:method
emails = users.map(&:email)
active = users.select(&:active?)

# Safe navigation
user&.profile&.avatar_url

# Default values
def greet(name = "World")
  "Hello, #{name}!"
end

# Keyword arguments
def create_user(email:, name: nil, role: :user)
  User.new(email: email, name: name, role: role)
end

# Multiple return values
def parse(input)
  [result, errors]
end

result, errors = parse(input)
```

## Error Handling

```ruby
# Begin/rescue/ensure
begin
  risky_operation
rescue NetworkError => e
  logger.error("Network failed: #{e.message}")
  retry if should_retry?
rescue StandardError => e
  logger.error("Unexpected error: #{e.message}")
  raise
ensure
  cleanup
end

# Custom errors
class NotFoundError < StandardError
  attr_reader :id

  def initialize(id)
    @id = id
    super("Not found: #{id}")
  end
end

# Inline rescue (use sparingly)
value = risky_call rescue default_value
```

## Collections

```ruby
# Map/Select/Reduce
emails = users.map { |u| u.email }
active = users.select { |u| u.active? }
total = orders.reduce(0) { |sum, o| sum + o.total }

# Chaining
users
  .select(&:active?)
  .map(&:email)
  .uniq
  .sort

# Hash operations
counts = users.group_by(&:department)
               .transform_values(&:count)

# Find
user = users.find { |u| u.id == target_id }
```

## Classes and Modules

```ruby
# Service object pattern
class CreateUser
  def initialize(repository:, notifier:)
    @repository = repository
    @notifier = notifier
  end

  def call(email:, name:)
    user = User.new(email: email, name: name)
    @repository.save(user)
    @notifier.welcome(user)
    user
  end
end

# Module for shared behavior
module Timestampable
  def created_at
    @created_at ||= Time.now
  end
end

class User
  include Timestampable
end
```

## Testing (RSpec)

```ruby
RSpec.describe UserService do
  let(:repository) { instance_double(UserRepository) }
  let(:service) { described_class.new(repository: repository) }

  describe "#create" do
    context "with valid email" do
      it "creates a user" do
        allow(repository).to receive(:save)

        user = service.create(email: "test@example.com")

        expect(user.email).to eq("test@example.com")
        expect(repository).to have_received(:save).with(user)
      end
    end

    context "with invalid email" do
      it "raises ValidationError" do
        expect { service.create(email: "invalid") }
          .to raise_error(ValidationError)
      end
    end
  end
end
```

## Rails Conventions

```ruby
# Controller
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user, notice: "User created"
    else
      render :new, status: :unprocessable_entity
    end
  end

  private

  def user_params
    params.require(:user).permit(:email, :name)
  end
end

# Model
class User < ApplicationRecord
  validates :email, presence: true, uniqueness: true
  has_many :posts, dependent: :destroy
  scope :active, -> { where(active: true) }
end
```
