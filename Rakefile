require 'rubygems'
require 'debian/build'
include Debian::Build

require 'debian/build/config'

namespace "package" do

  Package.new(:hpklinux) do |p|
    p.version = '3.08.05'
    p.debian_increment = 2

    p.source_provider = TarballSourceProvider.new('http://www.rivendellaudio.org/ftpdocs/audioscience/hpklinux-#{version}.tar.bz2')
  end

  main_source_provider = TarballSourceProvider.new('http://www.rivendellaudio.org/ftpdocs/#{name}/#{name}-#{version}.tar.gz')
  
  Package.new(:rivendell) do |p|
    p.version = '1.4.0'
    p.debian_increment = 1
    p.source_provider = main_source_provider
  end

  Package.new(:gpio) do |p|
    p.version = '1.0.0'
    p.debian_increment = 2
    p.source_provider = main_source_provider
  end

  ModulePackage.new('gpio-module')

  ModulePackage.new('hpklinux-module')

end

require 'debian/build/tasks'




