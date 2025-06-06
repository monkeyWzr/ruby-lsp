---
layout: default
title: Troubleshooting
nav_order: 20
---

# Troubleshooting

## How the Ruby LSP activation works

The Ruby LSP extension runs inside VS Code's NodeJS runtime, like any other VS Code extension. This means that
environment variables that are properly set in your shell may not exist inside the NodeJS process. To run the LSP server
with the same Ruby version as your projects, we need to properly set these environment variables, which is done by
invoking your Ruby version manager.

The extension runs a command using your shell's interactive mode, so that version managers configured in files such as
`~/.zshrc` are picked up automatically. The command then exports the environment information into JSON, so that we can
inject it into the NodeJS process and appropriately set the Ruby version and gem installation paths.

As an example, the activation script for `zsh` using `rbenv` as a version manager will look something like this:

```shell
# Invoke zsh using interactive mode (loads ~/.zshrc) to run a single command
# The command is `rbenv exec ruby`, which automatically sets all relevant environment variables and selects the
# specified Ruby version
# We then print the activated environment as JSON. We read that JSON from the NodeJS process to insert the needed
# environment variables in order to run Ruby correctly
/bin/zsh -ic 'rbenv exec ruby -rjson -e "puts JSON.dump(ENV.to_h)"'
```

After activating the Ruby version, we then proceed to boot the server gem (`ruby-lsp`). To avoid having users include
the `ruby-lsp` in their `Gemfile`, we create a [composed
bundle](composed-bundle) under the `.ruby-lsp` directory inside your project.

## Common issues

There are several main sources of issues users typically face during activation: outdated version, shell problems, or Bundler related problems.

### Outdated Version

{: .note }
> If using VS Code, the version of the extension is distinct from that of the server (the `ruby-lsp` gem). You can check the server version in the [status center](https://github.com/Shopify/ruby-lsp/blob/main/vscode/extras/ruby_lsp_status_center.png).

In most cases, the server gem will be automatically updated. You can also trigger a manual update with the "Update language server gem" command in VS Code.

You can also attempt an update from the command line with `BUNDLE_GEMFILE=.ruby-lsp/Gemfile bundle update ruby-lsp`

{: .note }
If you're using any add-on gem, such as `ruby-lsp-rspec`, then `ruby-lsp` will also be present in your `Gemfile.lock` and it's possible that an outdated add-on could prevent `ruby-lsp` from updating.

Another possible scenario where the `ruby-lsp` gem cannot be updated is when one of its runtime dependencies are
constrained by another gem in the application. For example, Ruby LSP has a dependency on
[RBS](https://github.com/ruby/rbs) v3. If another gem constrains the version of RBS to an older release, it will not be
possible to use newer versions of Ruby LSP.

The 3 runtime dependencies of the Ruby LSP are `rbs`, `prism` and `sorbet-runtime`. If any of them are being constrained
by the application, the Ruby LSP may fail to update.

Running `BUNDLE_GEMFILE=.ruby-lsp/Gemfile bundle outdated` may help with understanding what is being constrained.

### Shell issues

When the extension invokes the shell and loads its config file (`~/.zshrc`, `~/.bashrc`, etc), it is susceptible to
issues that may be caused by how the shell or its plugins interact with the NodeJS process. For example

- Some plugins completely redirect the stderr pipe to implement their functionality (fixed on the Ruby LSP side by
  https://github.com/Shopify/vscode-ruby-lsp/pull/918)
- Some plugins fail immediately or end up in an endless loop if they detect there's no UI attached to the shell process.
  In this case, it's not possible to fix from the Ruby LSP side since a shell invoked by NodeJS will never have a UI

Additionally, some users experience an issue where VS Code selects the wrong shell, not respecting the `SHELL`
environment variable. This usually ends up in having `/bin/sh` selected instead of your actual shell. If you are facing
this problem, please try to

- Update VS Code to the latest version
- Completely close VS Code and launch it from the terminal with `code .` (instead of opening VS Code from the launch
  icon)

More context about this issue on https://github.com/Shopify/vscode-ruby-lsp/issues/901.

### Bundler issues

Firstly, ensure you are using the latest release of Bundler (run `bundle update --bundler`).

If the extension successfully activated the Ruby environment, it may still fail when trying to compose the composed bundle
to run the server gem. This could be a regular Bundler issue, like not being able to satisfy dependencies due to a
conflicting version requirement, or it could be a configuration issue.

For example, if the project has its linter/formatter put in an optional `Gemfile` group and that group is excluded in
the Bundler configuration, the Ruby LSP will not be able to see those gems.

```ruby
# Gemfile

# ...

# If Bundler is configured to exclude this group, the Ruby LSP will not be able to find `rubocop`
group :optional_group do
  gem "rubocop"
end
```

If you experience Bundler related issues, double-check both your global and project-specific configuration to check if
there's anything that could be preventing the server from booting. You can print your Bundler configuration with

```shell
bundle config
```

### Format on save dialogue won't disappear

When VS Code requests formatting for a document, it opens a dialogue showing progress a couple of seconds after sending
the request, closing it once the server has responded with the formatting result.

If you are seeing that the dialogue is not going away, this likely doesn't mean that formatting is taking very long or
hanging. It likely means that the server crashed or got into a corrupt state and is simply not responding to any
requests, which means the dialogue will never go away.

This is always the result of a bug in the server. It should always fail gracefully without getting into a corrupt state
that prevents it from responding to new requests coming from the editor. If you encounter this, please submit a bug
report [here](https://github.com/Shopify/ruby-lsp/issues/new/choose) including the
steps that led to the server getting stuck.

### Missing Features

If you find that some features are working (such as formatting), but others aren't (such as go to definition),
and are working on a codebase that uses Sorbet, then this may indicate the
[Sorbet LSP isn't running](https://sorbet.org/docs/lsp#instructions-for-specific-language-clients).
To avoid duplicate/conflicting behavior, Ruby LSP disables some features when a Sorbet codebase is detected, with the
intention that Sorbet can provide better accuracy.

### Gem installation locations and permissions

To launch the Ruby LSP server, the `ruby-lsp` gem must be installed. And in order to automatically index your project's
dependencies, they must also be installed so that we can read, parse and analyze their source files. The `ruby-lsp` gem
is installed via `gem install` (using RubyGems). The project dependencies are installed via `bundle install` (using
Bundler).

If you use a non-default path to install your gems, please remember that RubyGems and Bundler require separate
configurations to achieve that.

For example, if you configure `BUNDLE_PATH` to point to `vendor/bundle` so that gems are installed inside the same
directory as your project, `bundle install` will automatically pick that up and install them in the right place. But
`gem install` will not as it requires a different setting to achieve it.

You can apply your preferred installed locations for RubyGems by using the `~/.gemrc` file. In that file, you can decide
to either install it with `--user-install` or select a specific installation directory with `--install-dir`.

```yaml
gem: --user-install
# Or
gem: --install-dir /my/preferred/path/for/gem/install
```

One scenario where this is useful is if the user doesn't have permissions for the default gem installation directory and
`gem install` fails. For example, when using the system Ruby on certain Linux distributions.

{: .note }
> Using non-default gem installation paths may lead to other integration issues with version managers. For example, for
> Ruby 3.3.1 the default `GEM_HOME` is `~/.gem/ruby/3.3.0` (without the patch part of the version). However, `chruby`
> (and potentially other version managers) override `GEM_HOME` to include the version patch resulting in
> `~/.gem/ruby/3.3.1`. When you install a gem using `gem install --user-install`, RubyGems ignores the `GEM_HOME`
> override and installs the gem inside `~/.gem/ruby/3.3.0`. This results in executables not being found because `chruby`
> modified the `PATH` to only include executables installed under `~/.gem/ruby/3.3.1`.
>
> Similarly, the majority of version managers don't read your `~/.gemrc` configurations. If you use a custom
> installation with `--install-dir`, it's unlikely that the version manager will know about it. This may result in the
> gem executables not being found.
>
> Incompatibilities between RubyGems and version managers like this one are beyond the scope of the Ruby LSP and should
> be reported either to RubyGems or the respective version manager.

### Developing on containers

See the [documentation](vscode-extension#developing-on-containers).

## Diagnosing the problem

Many activation issues are specific to how your development environment is configured. If you can reproduce the problem
you are seeing, including information about these steps is the best way to ensure that we can fix the issue in a timely
manner. Please include the steps taken to diagnose in your bug report.

### Check if the server is running

Check the [status center](https://github.com/Shopify/ruby-lsp/blob/main/vscode/extras/ruby_lsp_status_center.png).
Does the server status say it's running? If it is running, but you are missing certain features, please check our
[features documentation](index#general-features) to ensure we already added support for it.

If the feature is listed as fully supported, but not working for you, report [an
issue](https://github.com/Shopify/ruby-lsp/issues/new/choose) so that we can
assist.

### Check the VS Code output tab

Many of the activation steps taken are logged in the `Ruby LSP` channel of VS Code's `Output` tab. Check the logs to see
if any entries hint at what the issue might be.

Did the extension select your preferred shell?

Did it select your preferred version manager? You can define which version manager to use with the
`"rubyLsp.rubyVersionManager"` setting.

No output in the `Ruby LSP` channel? Check the `Extension Host` channel for any errors related to extension startup.

### Enable logging

You can enable logging to the VS Code output tab,
[as described in the Contributing](contributing#tracing-lsp-requests-and-responses) docs.

### Environment activation failed

We listed version manager related information and tips in this [documentation](version-managers).

### My preferred version manager is not supported

We default to supporting the most common version managers, but that may not cover every single tool available. For these
cases, we offer custom activation support. More context in the version manager
[documentation](version-managers#custom-activation).

### Try to run the Ruby activation manually

If the extension is failing to activate the Ruby environment, try running the same command manually in your shell to see
if the issue is exclusively related with the extension. The exact command used for activation is printed to the output
tab.

### Try booting the server manually

If the Ruby environment seems to activate properly, but the server won't boot, try to launch is manually from the
terminal with

```shell
# Do not use bundle exec
ruby-lsp
```

Is there any extra information given from booting the server manually? Or does it only fail when booting through the
extension?

## Indexing

When Ruby LSP starts, it attempts to index your code as well as your dependencies as described in [Configuring code indexing](index#configuring-code-indexing).

In rare cases, Ruby LSP will encounter an error which prevents indexing from completing, which will result in incomplete information in the editor.

Firstly, ensure that you are using the latest release of the `ruby-lsp` gem, as the problem may have been already fixed.

To diagnose the particular file(s) causing a problem, run `ruby-lsp-check`. Please log an issue so that we can address it. If the code is not open source then please provide a minimal reproduction.

In the meantime, you can [configure Ruby LSP to ignore a particular gem or file for indexing](index#configuring-code-indexing).

## After troubleshooting

If after troubleshooting the Ruby LSP is still not initializing properly, please report an issue
[here](https://github.com/Shopify/ruby-lsp/issues/new/choose) so that we can assist
in fixing the problem. Remember to include the steps taken when trying to diagnose the issue.
