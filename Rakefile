def sudo(*args)
  sh (["sudo"] + args).join(' ')
end

def pbuilder(command, *args)
  sudo "pbuilder #{command} --configfile /etc/pbuilder/pbuilderrc.stable #{args.join(' ')}"
end

namespace :pbuilder do
  desc "Update pbuilder"
  task :update do
    pbuilder :update
  end

  desc "Update pbuilder by overriding config"
  task :update_config do
    pbuilder :update, "--override-config"
  end
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

require 'rake/tasklib'

class Package < Rake::TaskLib

  attr_reader :name
  attr_accessor :package, :version, :signing_key, :debian_increment

  def initialize(name)
    @name = @package = name
    @debian_increment = 1

    @signing_key = ENV['KEY']

    yield self if block_given?
    raise "Version must be set" if version.nil?

    define
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
    "#{name}-#{version}"
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

  def define
    namespace @name do
      namespace "source" do
        desc "Create source directory for #{package} #{version}"
        task "tarball" do
          get source_tarball_url
        end

        desc "Retrieve source tarball for #{package} #{version}"
        task "directory" => "tarball" do
          uncompress source_tarball_name
          rm_rf "#{source_directory}/debian"
          cp_r "#{package}/debian", "#{source_directory}/debian"
        end
      end

      desc "Build source package for #{package} #{version}"
      task "source" => "source:directory" do
        sh "ln -fs #{source_tarball_name} #{orig_source_tarball_name}"
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

      desc "PBuild binary package for #{package} #{version}"
      task "pbuild" do
        pbuilder :build, "#{name}_#{debian_version}.dsc"

        mkdir_p "build"
        package_files("/var/cache/pbuilder/result/stable").each do |file|
          cp file, "build"
        end
      end

      desc "Upload packages for #{package} #{version}"
      task "upload" do
        changes_files = "#{package}_#{debian_version}*.changes"
        sh "dupload -t tryphon #{changes_files}"
        sh "scp #{package_files("build").join(' ')} debian.tryphon.org:/var/lib/debarchiver/incoming/experimental"
      end

      desc "Clean files created for #{package} #{version}"
      task "clean" do
        rm_f package_files
        rm_f source_tarball_name
        rm_f orig_source_tarball_name
        rm_rf "#{source_directory}"
      end
    end
  end


end

packages = []

namespace "package" do

  packages << :hpklinux
  Package.new(:hpklinux) do |t|
    t.version = '3.08.05'
  end
  
  packages << :rivendell
  Package.new(:rivendell) do |t|
    t.version = '1.1.0'
  end

  packages << :gpio
  Package.new(:gpio) do |t|
    t.version = '1.0.0'
  end

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
    task :binaries => "package:#{package}:binary"
    task :upload => "package:#{package}:upload"
    task :clean => "package:#{package}:clean"
  end
  
end

task "clean" => "packages:clean" do
  rm_rf "build"
end
