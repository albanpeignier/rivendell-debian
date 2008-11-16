require 'rake/tasklib'

module HelperMethods

  def sudo(*args)
    sh (["sudo"] + args).join(' ')
  end

  def uncompress(archive)
    options = 
      case archive
      when /bz2$/
        "-j"
      when /gz/
        "-z"
      end

    sh "tar -xf #{archive} #{options}"
  end

  def get(url)
    sh "wget -m --no-directories #{url}"  
  end

end

module BuildDirectoryMethods

  @@build_directory = '/var/tmp/rivendell-debian'

  def build_directory=(directory)
    @@build_directory = directory
  end

  def build_directory
    @@build_directory or '/var/tmp/rivendell-debian'
  end

end

class PBuilder
  extend BuildDirectoryMethods
  include HelperMethods

  @@default_build_host = nil

  def self.default_build_host=(host)
    @@default_build_host = host
  end

  def initialize
    @options = {}
    @before_exec_callbacks = []

    yield self if block_given?
  end

  def build_host
    @@default_build_host
  end

  def [](option)
    @options[option.to_sym]
  end

  def []=(option, value)
    @options[option.to_sym] = value
  end

  def options=(options)
    @options = @options.update options
  end

  def options_arguments
    @options.collect do |option, argument| 
      command_option = "--#{option}"
      case argument
      when true
        command_option        
      when Array
        argument.collect { |a| "#{command_option} #{a}" }
      else 
        "#{command_option} #{argument}" 
      end
    end.flatten
  end

  def before_exec(proc)
    @before_exec_callbacks << proc
  end

  def exec(command, *arguments)
    @before_exec_callbacks.each { |c| c.call }

    if build_host
      remote_exec command, *arguments
    else
      sudo "pbuilder", command, *(options_arguments + arguments)
    end
  end

  private 

  def remote_exec(command, *arguments)
    local_build_directory = PBuilder.build_directory
    remote_build_directory = "/var/tmp/rivendell-debian"

    sh "rsync -av --no-owner --no-group --no-perms --cvs-exclude --exclude=Rakefile --delete #{local_build_directory}/ #{build_host}:#{remote_build_directory}/"

    quoted_arguments = (options_arguments + arguments).join(' ').gsub("'","\\\\\"")
    quoted_arguments = quoted_arguments.gsub(local_build_directory, remote_build_directory)

    sh "ssh #{build_host} \"cd #{remote_build_directory}; sudo pbuilder #{command} #{quoted_arguments}\""
    sh "rsync -av --no-owner --no-group --no-perms #{build_host}:#{remote_build_directory}/ #{local_build_directory}"
  end

end

class Platform
  extend BuildDirectoryMethods

  attr_reader :flavor, :distribution, :architecture

  def initialize(flavor, distribution, architecture)
    @flavor = flavor
    @distribution = distribution.to_sym
    @architecture = architecture
  end

  def self.supported_architectures
    %w{i386 amd64}
  end

  def self.all
    @@all ||= debian_platforms + ubuntu_platforms
  end

  def self.debian_platforms
    @@debian_platforms ||= 
      %w{stable testing unstable}.collect do |distribution|
      supported_architectures.collect { |architecture| Platform.new(:debian, distribution, architecture) }
    end.flatten
  end

  def self.ubuntu_platforms
    @@ubuntu_platforms ||= 
      %w{hardy intrepid}.collect do |distribution|
      supported_architectures.collect { |architecture| Platform.new(:ubuntu, distribution, architecture) }
    end.flatten
  end

  def mirror
    case flavor
    when :debian
      "http://ftp.fr.debian.org/debian"
    when :ubuntu
      'http://fr.archive.ubuntu.com/ubuntu'
    end
  end

  def self.each(&block)
    all.each &block
  end

  def build_result_directory
    File.expand_path "#{Platform.build_directory}/binaries/#{distribution}/#{architecture}"
  end

  def stable?
    distribution == :stable
  end

  def pbuilder_base_file
    "/var/cache/pbuilder/base-#{distribution}-#{architecture}.tgz"
  end

  def pbuilder_enabled?
    File.exists? pbuilder_base_file
  end

  def pbuilder(options = {})
    PBuilder.new do |p|
      p[:basetgz] = pbuilder_base_file
      p[:othermirror] = "'deb file:#{build_result_directory} ./'"
      p[:bindmounts] = p[:buildresult] = build_result_directory
      p[:distribution] = distribution
      p[:hookdir] = default_hooks_directory

      # to use i386 on amd64 architecture
      p[:debootstrapopts] ||= []
      p[:debootstrapopts] << "--arch=#{architecture}"
      
      p[:debbuildopts] = "-a#{architecture}"

      p[:mirror] = mirror

      if flavor == :ubuntu
        # cdebootstrap fails with ubuntu
        p[:components] = "'main universe'"
        p[:debootstrap] = 'debootstrap'
      end

      p.options = options

      p.before_exec Proc.new { 
        prepare_build_result_directory 
        prepare_default_hooks
      }
    end
  end

  def default_hooks_directory
    "#{Package.build_directory}/tmp/hooks"
  end

  def prepare_build_result_directory
    mkdir_p build_result_directory
    
    Dir.chdir(build_result_directory) do 
      sh "/usr/bin/dpkg-scanpackages . /dev/null > Packages"
    end
  end

  def prepare_default_hooks
    mkdir_p default_hooks_directory

    apt_update_hook_file = "#{default_hooks_directory}/D80apt-get-update"
    unless File.exists?(apt_update_hook_file)    
      File.open(apt_update_hook_file, "w") do |f|
        f.puts "#!/bin/sh"
        f.puts "apt-get update"
      end 

      FileUtils.chmod 0755, apt_update_hook_file
    end
  end

  def to_s(separator = '/')
    "#{distribution}#{separator}#{architecture}"
  end

  def task_name
    to_s '_'
  end

end

class AbstractPackage < Rake::TaskLib
  include HelperMethods
  extend BuildDirectoryMethods

  attr_reader :name
  attr_accessor :package, :exclude_from_build

  def initialize(name)
    @name = @package = name

    init
    yield self if block_given?
    define
  end
  
  def init
  end

  def default_platforms
    unless exclude_from_build
      Platform.all
    else
      Platform.all.reject do |platform|
        platform.to_s.match exclude_from_build
      end
    end
  end

end

class Package < AbstractPackage

  attr_reader :name
  attr_accessor :package, :version, :signing_key, :debian_increment, :exclude_from_build

  def init
    @debian_increment = 1
    # TODO use a class attribute
    @signing_key = ENV['KEY']
  end

  def define
    raise "Version must be set" if version.nil?

    namespace @name do
      namespace "source" do
        desc "Create source directory for #{package} #{version}"
        task "tarball" do
          mkdir_p sources_directory
          Dir.chdir(sources_directory) do
            get source_tarball_url
          end
        end

        desc "Retrieve source tarball for #{package} #{version}"
        task "directory" => "tarball" do
          Dir.chdir(sources_directory) do
            uncompress source_tarball_name
          end

          rm_rf "#{source_directory}/debian"
          cp_r "#{package}/debian", "#{source_directory}/debian"
        end
      end

      desc "Build source package for #{package} #{version}"
      task "source" => "source:directory" do
        Dir.chdir(sources_directory) do  
          sh "ln -fs #{source_tarball_name} #{orig_source_tarball_name}"
        end
        Dir.chdir(source_directory) do  
          sh "dch --newversion #{debian_version} 'New upstream version' || true"

          signing_options =
            if signing_key
              "-k#{signing_key}"
            else
              "-us -uc"
            end
          sh "dpkg-buildpackage -rfakeroot #{signing_options} -S"
        end
      end

      namespace :pbuild do
        Platform.each do |platform|
          desc "Pbuild #{platform} binary package for #{package} #{version}"
          task platform.task_name do |t, args|
            pbuilder_options = {
              :logfile => "#{platform.build_result_directory}/pbuilder-#{package}-#{debian_version}.log"
            }

            platform.pbuilder(pbuilder_options).exec :build, "#{sources_directory}/#{package}_#{debian_version}.dsc"

            changes_file = "#{platform.build_result_directory}/#{package}_#{debian_version}_#{platform.architecture}.changes"
            # force target distribution
            sh "sed -i 's/Distribution: .*/Distribution: #{platform.distribution}/' #{changes_file}"
          end
        end

        desc "Pbuild binary package for #{package} #{version} (on #{default_platforms.join(', ')})"
        task :all => default_platforms.collect { |platform| platform.task_name }

      end

      desc "Upload packages for #{package} #{version}"
      task "upload" do
#         changes_files = "#{package}_#{debian_version}*.changes"
#         sh "dupload -t tryphon #{changes_files}"
#         sh "scp #{package_files("build").join(' ')} debian.tryphon.org:/var/lib/debarchiver/incoming/experimental"
        Platform.each do |platform|
          deb_files = package_deb_files(platform.build_result_directory)
          unless deb_files.empty?
            sh "rsync -av #{deb_files.join(' ')} debian.tryphon.org:/var/lib/debarchiver/incoming/#{platform.distribution}"
          end
        end
      end

      desc "Clean files created for #{package} #{version}"
      task "clean" do
        Platform.each do |platform|
          rm_f package_files(platform.build_result_directory)
        end

        rm_f package_source_files

        rm_f "#{sources_directory}/#{source_tarball_name}"
        rm_f "#{sources_directory}/#{orig_source_tarball_name}"
        rm_rf "#{source_directory}"
      end
    end
  end

  def sources_directory
    "#{Package.build_directory}/sources"
  end

  def source_tarball_url
    remote_directory = 
      unless package == :hpklinux 
        package.to_s
      else
        "audioscience" 
      end

    "http://www.rivendellaudio.org/ftpdocs/#{remote_directory}/#{source_tarball_name}"
  end

  def source_tarball_name
    case package
    when :rivendell 
      "rivendell-#{version}.tar.gz"
    when :hpklinux 
      "hpklinux-#{version}.tar.bz2"
    when :gpio
      "gpio-#{version}.tar.gz"
    end
  end

  def orig_source_tarball_name
    source_tarball_name.gsub "#{version}.tar","#{version}.orig.tar"
  end

  def source_directory
    "#{sources_directory}/#{name}-#{version}"
  end

  def debian_version
    "#{version}-#{debian_increment}"    
  end

  def package_files(directory = '.')
    fileparts = [ name.to_s ]
    case name
    when :hpklinux
      fileparts << 'libhpi'
    when :rivendell
      fileparts << 'librivendell'
    end

    fileparts.inject([]) do |files, filepart|
      files + Dir.glob("#{directory}/#{filepart}*#{debian_version}*")
    end
  end

  def package_source_files
    %w{.dsc .tar.gz _source.changes}.collect do |extension|
      "#{sources_directory}/#{package}_#{debian_version}#{extension}"
    end
  end

  def package_deb_files(directory = '.')
    package_files(directory).find_all { |f| f.match /\.deb$/ }
  end

end

class ModulePackage < AbstractPackage

  def module_name
    package.gsub(/-module$/,'')
  end

  def define
    namespace @name do
      namespace :pbuild do
        Platform.each do |platform|
          desc "Pbuild #{platform} binary package for #{package}"
          task platform.task_name do |t, args|
            cp "execute-module-assistant", "#{AbstractPackage.build_directory}/tmp"

            pbuilder_options = {
              :logfile => "#{platform.build_result_directory}/pbuilder-#{package}.log"
            }

            platform.pbuilder(pbuilder_options).exec :execute, "-- #{AbstractPackage.build_directory}/tmp/execute-module-assistant #{module_name} #{platform.architecture} #{platform.build_result_directory}"
          end
        end

        desc "Pbuild binary package for #{package} (on #{default_platforms.join(', ')})"
        task :all => default_platforms.collect { |platform| platform.task_name }

      end

      desc "Upload packages for #{package}"
      task "upload" do
        Platform.each do |platform|
          sh "rsync -av #{platform.build_result_directory}/#{package}-*.deb debian.tryphon.org:/var/lib/debarchiver/incoming/#{platform.distribution}"
        end
      end

    end
  end

end

require 'config' if File.exists? 'config.rb'

packages = []

namespace "package" do

  packages << :hpklinux
  Package.new(:hpklinux) do |t|
    t.version = '3.08.05'
  end
  
  packages << :rivendell
  Package.new(:rivendell) do |t|
    t.version = '1.1.1'
  end

  packages << :gpio
  Package.new(:gpio) do |t|
    t.version = '1.0.0'
  end

  packages << 'gpio-module'.to_sym
  ModulePackage.new('gpio-module')

  packages << 'hpklinux-module'.to_sym
  ModulePackage.new('hpklinux-module')

end

namespace "packages" do
  desc "Create packages source for all packages"

  desc "Build source packages for #{packages.join(' ')}"
  task :sources

  desc "Build binary packages for #{packages.join(' ')}"
  task :binaries

  desc "Upload packages for #{packages.join(' ')}"
  task :upload

  packages.each do |package|
    task :sources => "package:#{package}:source"
    task :binaries => "package:#{package}:pbuild:all"
    task :upload => "package:#{package}:upload"
    task :clean => "package:#{package}:clean"
  end
  
end

task :packages => [ "packages:sources", "packages:binaries", "packages:upload" ]

task :setup => "pbuilder:setup" do
  sudo "apt-get install devscripts"
end

namespace :setup do

  task :ubuntu do
    get 'http://fr.archive.ubuntu.com/ubuntu/pool/main/u/ubuntu-keyring/ubuntu-keyring_2008.03.04_all.deb'
    sudo "dpkg -i ubuntu-keyring_2008.03.04_all.deb"
  end

end

task "clean" => "packages:clean" do
  rm_rf "build"
end

namespace :pbuilder do
  include HelperMethods

  desc "Install pbuilder"
  task :setup do 
    sudo "apt-get install pbuilder"
  end

  namespace "create" do
    Platform.each do |platform| 
      desc "Create pbuilder base image for #{platform}"
      task platform.task_name do
        platform.pbuilder.exec :create
      end
    end
  end

  desc "Update pbuilder"
  task :update do
    Platform.each do |platform| 
      platform.pbuilder.exec :update
    end
  end

  desc "Update pbuilder by overriding config"
  task :update_config do
    Platform.each do |platform| 
      platform.pbuilder("override-config" => true).exec :update  
    end
  end

end
