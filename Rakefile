# frozen_string_literal: true

# Define a base default task early so other files can enhance it.
desc 'Default tasks aggregator'
task :default do
  puts 'Default task complete.'
end

# External gems that define tasks - add here!
begin
  require 'rspec/core/rake_task'

  # Acceptance tests - run AFTER the action has installed Ruby
  # These verify the action worked correctly
  RSpec::Core::RakeTask.new(:acceptance) do |task|
    task.pattern = 'spec/acceptance/*_spec.rb'
    task.rspec_opts = '--tag acceptance'
  end
rescue LoadError
  # Insulating for arbitrary Gemfiles in CI that add rake, but not the other gems.
  desc 'Run specs (stub)'
  task :spec do
    # Stub
  end

  desc 'Run acceptance tests (stub)'
  task :acceptance do
    # Stub
  end
end

begin
  require 'rubocop/rake_task'

  RuboCop::RakeTask.new(:lint)
rescue LoadError
  desc 'Run RuboCop linting (stub)'
  task :lint do
    # Stub
  end
  # Insulating for arbitrary Gemfiles in CI that add rake, but not the other gems.
end

begin
  require 'kettle/dev'
rescue LoadError
  # Insulating for arbitrary Gemfiles in CI that add rake, but not the other gems.
end

desc 'Run all checks (lint + acceptance tests)'
task ci: %i[lint spec]
