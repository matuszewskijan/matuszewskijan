***
*Original available at: https://jmatuszewski.com/How-to-deploy-multiple-apps-on-single-server-with-Capistrano/*

*It's probably easier to read it on the blog than on GitHub.*
***

# How to deploy multiple Rails apps on a single server with Capistrano

Almost everyone uses cloud hosting solutions like Heroku or AWS nowadays(and I don't expect it to change even after [this DHH post](https://world.hey.com/dhh/why-we-re-leaving-the-cloud-654b47e0){:target="_blank"}). But if you still self-host your apps then [Capistrano](https://github.com/capistrano/capistrano){:target="_blank"} would be well known to you.

My client had an uncommon requirement - to deploy two versions of the same application twice on one server. The applications should use the same codebase but a different database, credentials and domain address as we wanted to perform some experiments.

Let me first show you my solution which wasn't ideal, then I'll try to cover more correct solution proposed to me after I first shared this article on [Reddit](https://www.reddit.com/r/ruby/comments/zboed1){:target="_blank"}.

## Naive approach

Knowing that `Capistrano` uses the `name` from `set :application, "#{name}"` to create a separate directory for application code on the server I thought that I could abuse this with such code:
```ruby
# config/deploy.rb
APPS = %w[first_app second_app]

raise 'Pass APP environment variable' unless APPS.include?(ENV['APP'])

set :application, ENV['APP']

append :linked_files, *%w[config/database.yml config/master.key config/credentials.yml.enc]
```

We're defining a list of application names that are allowed on our server. Users deploying the app will have to provide an `APP` environment variable, an error will be raised when it's missing.

As mentioned `set :application, ENV['APP']` is essential here. The change in the `application` directive is making Capistrano deploy all of the files to a *different directory*.

We also have to define `linked_files` with a list of the files that should be different between applications. In this case, it's a different set of database credentials, `master.key` file and different Rails encrypted credentials. If you're unfamiliar with `linked_files` then see [capistrano-linked-files](https://github.com/runar/capistrano-linked-files){:target="_blank"}.

The command that we'll use to perform the deployment:
```sh
APP=first_app bundle exec cap production deploy
```

From perspective, it is a **hacky** way to do it because:
1. We have to use only one deploy configuration for all tenants.
2. It could be unclear for other devs what's going on in this implementation.

The implementation worked as we expected. I was 100% sure that it was ideal as it took minutes to make it work, we reused our server = money saved. What might be wrong there?

## Capistrano approach

Instead of hacking the `deploy.rb` file we could just create stage-specific configuration in `config/deploy` folder:
```ruby
# production_second.rb

# We are running a multi-tenant application.
# It's the same server as in `production.rb` but we're deploying to a different directory.
server "example.com", user: "deploy", roles: %w{app db web}

set :deploy_to, "/var/www/sites/second_app"
```

Now we could use:
```sh
bundle exec cap production_second deploy
```

**It's clean, it's configurable, it's a better way to handle multiple apps on a single server with Capistrano.**

***

*You might say that it's still a little dirty. That Capistrano isn't designed to do such stuff and we should use a different tool.*

*I'd say: maybe? But switching to something else costs time, time is money. I am always trying to find the solution that will be the smallest trade-off. We were already using Capistrano and changing it would be a complete burden in this case.*

*If you start a new project now and you know you will have a such requirement(in 90% of scenarios you won't know that beforehand), then for sure go for the tool better suited for the task.*

<p class="centered">
Thank you for reading!
</p>
