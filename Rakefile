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
  fileparts = [ name.to_s ]
  case name
  when :hpklinux
    fileparts << 'libhpi'
  when :rivendell
    fileparts << 'librivendell'
  end

  fileparts.inject([]) do |files, filepart|
    files + Dir.glob("#{directory}/#{filepart}*#{version}-1*")
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

def package_tasks(name, version, options = {}) 
  options = { :debian_version => 1 }.update(options)

  debian_version = "#{version}-#{options[:debian_version]}"

  task "source_tarball_#{name}" do
    get source_tarball_url(name, version)
  end

  task "source_#{name}" => "source_tarball_#{name}" do
    uncompress source_tarball_name(name, version)
    rm_rf "#{name}-#{version}/debian"
    cp_r "#{name}/debian", "#{name}-#{version}/debian"
  end

  task "build_source_#{name}" => "source_#{name}" do
    orig_source_tarball_name = source_tarball_name(name, version).gsub("#{version}.tar","#{version}.orig.tar")
    sh "ln -fs #{source_tarball_name(name, version)} #{orig_source_tarball_name}"
    Dir.chdir("#{name}-#{version}") do  
      sh "dch --newversion #{debian_version} 'New upstream version' || true"
      sh "dpkg-buildpackage -rfakeroot -k9CC21102 -S"
      # -us -uc 
    end
  end

  task "pbuild_#{name}" => "build_source_#{name}" do
    pbuilder :build, "#{name}_#{debian_version}.dsc"

    package_files(name, version, "/var/cache/pbuilder/result/stable").each do |file|
      cp file, "."
    end
  end

  task "upload_#{name}" do
    sh "scp #{package_files(name, version).join(' ')} debian.tryphon.org:/var/lib/debarchiver/incoming/experimental"
  end

  task "clean_#{name}" do
    rm_f package_files(name, version)
    rm_rf "#{name}-#{version}"
  end
end

package_tasks :hpklinux, '3.08.05'
package_tasks :rivendell, '1.1.0'
package_tasks :gpio, '1.0.0'

task 'pbuilder_rivendell' => 'pbuilder_hpklinux'

packages = [ 'hpklinux', 'rivendell' ]

namespace "packages" do
  desc "Create packages source for all packages"
  task :sources

  packages.each do |package|
    task :sources => "build_source_#{package}"
    task :binaries => "pbuild_#{package}"
    task :upload => "upload_#{package}"
  end
  
end
