$:.unshift << "/home/alban/share/projects/tryphon.org/rake-debian-build/lib"

#require 'rubygems'
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
    p.version = '1.5.0'
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

  class SoxSourceProvider

    def retrieve(package)
      %w{sox_12.17.9-1.dsc sox_12.17.9.orig.tar.gz sox_12.17.9-1.diff.gz}.each do |file|
        get "http://ftp.debian.org/debian/pool/main/s/sox/#{file}"
      end
      sh "dpkg-source -x sox_12.17.9-1.dsc"
      sh "mv sox-12.17.9 sox-12-12.17.9"
    end

  end

  Package.new(:'sox-12') do |p|
    p.version = '12.17.9'
    p.debian_increment = 1
    p.source_provider = SoxSourceProvider.new
  end

end

require 'debian/build/tasks'




