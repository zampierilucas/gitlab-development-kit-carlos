# frozen_string_literal: true

$LOAD_PATH.unshift('.')

require 'fileutils'
require 'lib/gdk'
require 'lib/git/configure'
require 'rake/clean'

Dir.glob('lib/tasks/**/*.rake') { |task| import task }

CONFIGS = FileList['Procfile', 'nginx/conf/nginx.conf', 'gitlab/config/gitlab.yml']
CLOBBER.include *CONFIGS, 'gdk.example.yml'

def config
  @config ||= GDK::Config.new
end

desc 'Dump the configured settings'
task 'dump_config' do
  GDK::Config.new.dump!(STDOUT)
end

desc 'Generate an example config file with all the defaults'
file 'gdk.example.yml' => 'clobber:gdk.example.yml' do |t|
  File.open(t.name, File::CREAT|File::TRUNC|File::WRONLY) do |file|
    config = Class.new(GDK::Config)
    config.define_method(:gdk_root) { Pathname.new('/home/git/gdk') }
    config.define_method(:username) { 'git' }
    config.define_method(:read!) { |_| nil }

    config.new(yaml: {}).dump!(file)
  end
end

desc 'Regenerate all config files from scratch'
task reconfigure: [:clobber, :all]

desc 'Generate all config files'
task all: CONFIGS

task 'clobber:gdk.example.yml' do |t|
  Rake::Cleaner.cleanup_files([t.name])
end

desc 'Generate Procfile for Foreman'
file 'Procfile' => ['Procfile.erb', GDK::Config::FILE] do |t|
  GDK::ErbRenderer.new(t.source, t.name).safe_render!
end

namespace :git do
  desc 'Configure your Git with recommended settings'
  task :configure, :global do |_t, args|
    global = args[:global] == "true"

    Git::Configure.new(global: global).run!
  end
end

desc 'Generate nginx configuration'
file 'nginx/conf/nginx.conf' => ['nginx/conf/nginx.conf.erb', GDK::Config::FILE] do |t|
  GDK::ErbRenderer.new(t.source, t.name).safe_render!
end

desc 'Generate the gitlab.yml config file'
file 'gitlab/config/gitlab.yml' => ['support/templates/gitlab.yml.erb'] do |t|
  GDK::ErbRenderer.new(t.source, t.name).render!
end

desc "Generate gitaly config toml"
file "gitaly/gitaly.config.toml" => ['support/templates/gitaly.config.toml.erb'] do |t|
  GDK::ErbRenderer.new(
    t.source,
    t.name,
    path: config.repositories_root,
    storage: 'default',
    socket_path: config.gitaly.address
  ).render!
  FileUtils.mkdir_p(config.repositories_root)
end

file 'gitaly/praefect.config.toml' => ['support/templates/praefect.config.toml.erb'] do |t|
  GDK::ErbRenderer.new(t.source, t.name).render!
end

config.praefect.nodes.each_with_index do |node, index|
  desc "Generate gitaly config #{index} toml"
  file "gitaly/gitaly-#{index}.praefect.toml" => ['support/templates/gitaly.config.toml.erb'] do |t|
    GDK::ErbRenderer.new(
      t.source,
      t.name,
      path: File.join(config.repositories_root, node[:storage]),
      storage: node[:storage],
      socket_path: node[:address]).render!
  end
  FileUtils.mkdir_p(File.join(config.repositories_root, "praefect-internal-#{index}"))
end
