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

def package_files(name, version, directory = '.')
  [ name.to_s, 'libhpi' ].inject([]) do |files, filepart|
    files + Dir.glob("#{directory}/#{filepart}*#{version}*")
  end
end

def package_deb_files(name, version, directory = '.')
  package_files(name, version, directory).find_all { |file| file.match(/.deb$/) }
end

def build_dependencies(name)
  case name
  when :rivendell
    %w{automake1.9 libjack0.100.0-dev libqt3-mt-dev libasound2-dev libsamplerate0-dev libvorbis-dev libtag1-dev libid3-dev libflac-dev libpam0g-dev}  
  else
    []
  end
end

def source_tarball_url(name, version)
  remote_directory = 
    unless name == :hpklinux 
      name.to_s
    else
      "audioscience" 
    end

  "http://www.rivendellaudio.org/ftpdocs/#{remote_directory}/#{source_tarball_name(name, version)}"
end

def source_tarball_name(name, version)
  case name
  when :rivendell 
    "rivendell-#{version}.tar.gz"
  when :hpklinux 
    "hpklinux-#{version}.tar.bz2"
  when :gpio
    "gpio-#{version}.tar.gz"
  end
end

def package_tasks(name, version) 

  task "source_tarball_#{name}" do
    get source_tarball_url(name, version)
  end

  task "source_#{name}" => "source_tarball_#{name}" do
    uncompress source_tarball_name(name, version)
    rm_rf "#{name}-#{version}/debian"
    cp_r "#{name}/debian", "#{name}-#{version}/debian"
  end

  task "buildpackage_#{name}" => [ "source_#{name}", "install_build_depencies_#{name}" ] do
    unless File.exists?("#{name}_#{version}-0.dsc") 
      Dir.chdir("#{name}-#{version}") do  
        sh "dch --newversion #{version}-0 'New upstream version' || true"
        sh "dpkg-buildpackage -rfakeroot -us -uc"
        # "-k9CC21102"
      end
    end
  end

  case name
  when :rivendell
    task "buildpackage_rivendell" => "install_dev_packages_hpklinux"
  end

  task "package_#{name}" => "buildpackage_#{name}" do
    pbuilder :build, "#{name}_#{version}-0.dsc"

    package_files(name, version, "/var/cache/pbuilder/result/stable").each do |file|
      cp file, "."
    end
  end

  task "install_dev_packages_#{name}" do
    sudo "dpkg -i #{package_deb_files(name,version).join(' ')}"
  end

  task "install_build_depencies_#{name}" do
    dependencies = build_dependencies(name)
    sudo "apt-get install #{dependencies.join(' ')}" unless dependencies.empty?
  end

  task "clean_#{name}" do
    rm_f package_files(name, version)
    rm_rf "#{name}-#{version}"
  end
end

package_tasks :hpklinux, '3.08.05'
package_tasks :rivendell, '1.1.0'
package_tasks :gpio, '1.0.0'

# http://www.rivendellaudio.org/ftpdocs/rivendell/rivendell-1.1.0.tar.gz"
# http://www.rivendellaudio.org/ftpdocs/audioscience/hpklinux-3.08.05.tar.bz2
# http://www.rivendellaudio.org/ftpdocs/gpio/gpio-1.0.0.tar.gz
