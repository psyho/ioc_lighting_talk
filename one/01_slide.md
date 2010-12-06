!SLIDE
# Inversion of Control #
## the forgotten design pattern ##

!SLIDE
# What's wrong with this code? #

    @@@Ruby
    class UserPreferences
      def initialize
        @session_store  = SomeSessionStore.new
      end

      # reads and writes to session store...
    end

!SLIDE smaller
# What if I want to chage the type of session store? #

    @@@Ruby
    class UserPreferences
      def initialize
        @session_store  = MysqlSessionStore.new($db_connection)
        # @session_store  = CookieSessionStore.new
      end

      # reads and writes to session store...
    end

!SLIDE smaller
# What if I want to make the session store configurable? #

    @@@Ruby
    class UserPreferences
      def initialize
        case SESSION_STORE
        when :mysql
          @session_store  = MysqlSessionStore.new($db_connection)
        when :cookie
          @session_store  = CookieSessionStore.new
        end
      end

      # reads and writes to session store...
    end

!SLIDE smaller
# Can I even fake session_store? #

    @@@Ruby
    it "should do something" do
      # this assumes that UserPreferences only accesses session_store
      # through an accessor
      fake = FakeSessionStore.new
      UserPreferences.any_instance.stubs(:session_store => fake)

      SomeClassThatInstantiatesUserPrefferences.new.do_something

      # ...
    end

!SLIDE bullets incremental
# The solution #
* Don't call us, we'll call you

!SLIDE bullets incremental
# Two main ways to implement IoC #
* Service Locator
* Dependency Injection

!SLIDE smaller
# Classical ServiceLocator #

    @@@Ruby
    # user_preferences.rb
    class UserPreferences
      def initialize
        @session_store = ServiceLocator.get(:session_store)
      end

      # reads and writes to session store...
    end

    # mysql_session_store.rb
    class MysqlSessionStore
      def initialize
        @db_connection = ServiceLocator.get(:db_connection)
      end

      # ...
    end

    # somewhere else
    ServiceLocator.register(:db_connection, ...)
    ServiceLocator.register(:session_store, MysqlSessionStore)

!SLIDE smaller
# More Ruby-ish ServiceLocator #

    @@@Ruby
    # user_preferences.rb
    class UserPreferences
      include SessionStore

      # reads and writes to session store...
    end

    # mysql_session_store.rb
    class MysqlSessionStore
      include DbConnection

      # ...
    end

!SLIDE smaller
# More Ruby-ish ServiceLocator contd. #

    @@@Ruby
    # somewhere else
    module SessionStore
      def session_store
        @session_store ||= MysqlSessionStore.new
      end
    end

    module DbConnection
      def db_connection
        @db_connection ||= ...
      end
    end

!SLIDE bullets incremental
# ServiceLocator #

* not context aware
* faking only on a class level
* makes it easy to have many, many dependencies per class
* easy to add to an existing system

!SLIDE
# Manual Dependency Injection #

    @@@Ruby
    class UserPreferences
      def initialize(session_store)
        @session_store = session_store
      end
      # ...
    end

    class MysqlSessionStore
      def initialize(db_connection)
        @db_connection = db_connection
      end
    end

!SLIDE smaller
# Manual Dependency Injection contd. #

    @@@Ruby
    class Injector
      def initialize(config)
        @config = config
      end

      def user_preferences
        @user_preferences ||= UserPreferences.new(session_store)
      end

      def session_store
        @session_store ||= MysqlSessionStore.new(db_connection)
      end

      def db_connection
        @db_connection ||= Mysql::Connection.new(config['server'])
      end

      # ...
    end

!SLIDE
# Manual Dependency Injection contd. #

    @@@Ruby
    # config.ru
    config = YAML.parse_file('config.yml')
    injector = Injector.new(config[ENV_NAME])
    run injector.http_router

!SLIDE smaller
# Setter injection #

    @@@Ruby
    class Injector
      # ...

      def inject(object, dependencies = {})
        return object if injected[object]
        injected[object] = true
        dependencies.each do |attr, value|
          object.send("#{attr}=", value)
        end
        object
      end

      def injected
        @injected ||= {}
      end

      def session_store
        @session_store ||= MysqlSessionStore.new
        inject(@session_store) do |o|
          o.db_connection = db_connection
        end
      end

      # ...
    end

!SLIDE bullets incremental
# Dependency Injection #

* fully configurable
* context aware
* mocking can be done per component
* you can have an injector hierarchy to achieve different object lifetime scopes (singleton/application, session, request)

!SLIDE small
# Dependency Injection in Aggregatr #

    @@@Ruby
    class SomeComponentClass
      # almost always only :depends_on is used
      include Component(:depends_on => [:foo, :bar, :baz],
                        :autowire => true,
                        :scope => :application,
                        :define_constructor => true,
                        :define_accessors => true,
                        :group => :some_list_of_components,
                        :provider => false)

      def some_method
        # ...
      end
    end

!SLIDE small
# Dependency Injection in Aggregatr contd. #

    @@@Ruby
    context = ComponentContext.new(parent_context = nil) do
      autowire_scoped :application
      autowire SomeClassName

      failures_redis(:db => 1) do |c|
        c.class = Redis
      end

      failed_jobs_dao.redis = failures_redis

      view_path.set(RAILS_ROOT + "/app/views")
    end

!SLIDE small
# Dependency Injection in Aggregatr contd. #

    @@@Ruby
    it "should do something" do
      something = SomeComponentClass.new(MockFoo.new,
                                         MockBar.new,
                                         MockBaz.new)

      something.some_method

      # ...
    end

!SLIDE bullets incremental smaller
# Our Dependency Injection Framework #

* uses only constructor injection
* can handle circular dependencies
* can handle different object lifetimes
* allows us to have very fast (and thread-safe) tests
* can do constructor currying
* forces us to obey the Single Responsibility Principle (and probably the rest of SOLID principles as well)

!SLIDE
# Questions? #

!SLIDE
# Thank You #
