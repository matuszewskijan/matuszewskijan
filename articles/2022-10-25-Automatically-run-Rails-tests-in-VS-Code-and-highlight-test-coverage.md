***
*Original available at: https://jmatuszewski.com/Automatically-run-Rails-tests-in-VS-Code-and-highlight-test-coverage/*

*It's probably easier to read it on the blog than on GitHub.*
***

# Automatically Run Rails tests in VS Code and highlight test coverage

> Empower your IDE to automatically run RSpec/Minitest tests. Get code coverage highlights with VS Code extension and a few lines of code changed.

***

Man, I love TDD... but once every few times I forget to run my tests after doing changes.

> My brain resists typing `rspec spec/models/changed_spec.rb` as finding the right test file and typing its path takes too much time and energy, the same applies to running the whole test suite.


What happens next is - I push the untested code to Github just to realize after a few minutes that continuous integration is failing. At this moment I've likely already switched context to something else so I need to spend a lot of energy to understand the pipeline output just to discover the fix might took me only a few seconds to fix if I'd done it in the first place. Overall my feedback loop takes minutes when it could definitely be seconds.

Is it happening only to me?

Even if the answer is yes - the typical flow of starting test files from the command line seems simply too slow for me. Maybe there is a way to make it automatically?

**And to answer this question with yes...**

### I wrote my own VS Code extension

The idea was simple: "if all test files follow a strict naming convention then it should be possible to find corresponding test files for all entries from the `/app` directory". Leveraging this I should be able to run `rspec spec/models/user_spec.rb` command automatically after saving `app/models/user.rb`.

#### How I'd know if the tests are passing?
That's why I decided to use a VS Code extension instead of `Guard` script. The extension will automatically switch VS Code terminal to the one named `Rails Test Runner`. The results will be displayed once the command is executed. It's also an interactive terminal so you should be able to use debugger inside the code.

![Rails Automatic Test Runner](/images/VS-Code-Rails-Automatic-Test-Runner.gif)

**Save your source file to automatically run its test.** The extension is available on Visual Studio Marketplace: [Rails Automatic Test Runner.](https://marketplace.visualstudio.com/items?itemName=jmatuszewski.rails-automatic-test-runner){:target="_blank"}

I am using it for a few days already, tried it with both `rspec` and `minitest` projects, and I am very happy with the results so far!

I have some ideas for improvements, mostly adding more configuration options to ensure it could work with any project. But before I would do so...

### I encourage you to [test the extension](https://marketplace.visualstudio.com/items?itemName=jmatuszewski.rails-automatic-test-runner){:target="_blank"} and leave feedback. Do you find it useful? Would you incorporate it to your everyday workflow? If not, why not?
Share your thoughts in the comments, [marketplace review](https://marketplace.visualstudio.com/items?itemName=jmatuszewski.rails-automatic-test-runner&ssr=false#review-details){:target="_blank"} or simply drop me a [message](mailto:matuszewski.jan@hotmail.com).

***

<div class="" markdown="1" style="margin-left: 15pt;margin-right: 15pt;">
<p class="text-grey font-small">Advertisment</p>
Every few seconds spent on repetitive tasks are hours in the long run. Hours that could be spent on generating value for your business.

My passion is to locate and optimize any pitfalls so my clients could focus fully on their growth. Sounds interesting?

<p class="centered">
<a href="/contact" class="button button-red centered">Let's talk!</a>
</p>
</div>

***

## Code Coverage highlighting
Having the tests running automatically I thought it'd be cool to also highlight line coverage in my IDE, as I saw that Rubymine could do it and I also wanted such a feature. Luckily for VS Code there is the [Coverage Gutters](https://marketplace.visualstudio.com/items?itemName=ryanluker.vscode-coverage-gutters){:target="_blank"} extension that should help to achieve exactly that.

All actions needed are:
1. Install the extension and click the **Watch** icon added to the footer
2. Add `Simplecov` and `Simplecov-lcov` to Gemfile:
```ruby
group :test do
  gem 'simplecov'
  gem 'simplecov-lcov'
end
```

3. Add the `lcov` formatter to SimpleCov initialization:
```ruby
# spec/test_helper.rb
SimpleCov::Formatter::LcovFormatter.config do |c|
  c.report_with_single_file = true
  c.output_directory = 'coverage'
  c.lcov_file_name = 'lcov.info'
end
SimpleCov.start do
  enable_coverage :branch
  formatter SimpleCov::Formatter::MultiFormatter.new([
    SimpleCov::Formatter::LcovFormatter,
    SimpleCov::Formatter::HTMLFormatter
  ])
end
```

Voila! Now when hitting `CTRL + S` on any file from `/app` folder I do get coverage higlights in gutter:
![VS Code coverage gutters for Rails application](/images/VS-Code-coverage-gutters-for-rails-application.png)

### Extension Configuration
Every project is different so it might be that you have to change some configuration options as it's set up to support RSpec by default:
-   `railsAutomaticTestRunner.framework`: Specifies which framework is used to execute test commands: `rspec` or `minitest`.
-   `railsAutomaticTestRunner.testsDirectory`: Specifies the folder where your test files are located, the default directory is `spec`.
-   `railsAutomaticTestRunner.bundleExec`: Append `bundle exec` before test command?
-   `railsAutomaticTestRunner.envVariables`: Pass any environment variables as a string there, eg: `CRUCIAL_API_KEY=test`
-   `railsAutomaticTestRunner.args`: Pass any arguments like `--fail-fast` as a string there
-   `railsAutomaticTestRunner.automaticOutputDisplay`: Automatically switch to the output tab, when enabled might interrupt terminal work.
