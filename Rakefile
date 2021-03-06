require 'fileutils'
require 'find'
require 'rake/clean'
require 'rubygems'
require 'simp/rake'
require 'simp/rake/beaker'
require 'erb'
require 'yaml'

pkg = Simp::Rake::Pkg.new(File.dirname(__FILE__))
# Simp::RakePkg decides where the RPM spec file for a project is in
# its constructor.  However, for this project, the spec file will be
# generated.  So need to tell it where the spec file will be found.
pkg.spec_file = File.join(File.dirname(__FILE__), 'build', 'simp-vendored-r10k.spec')

@rakefile_dir=File.dirname(__FILE__)
deps = YAML.load_file('build/sources.yaml')

CLEAN.include 'dist'
CLEAN.include 'pkg'
CLEAN.include 'src'
CLEAN.include 'build/simp-vendored-r10k.spec'
Find.find( @rakefile_dir ) do |path|
  if File.directory? path
    CLEAN.include path if File.basename(path) == 'tmp'
  else
    Find.prune
  end
end

task :checkout do
  deps['gems'].each do |gem,data|
    FileUtils.mkdir_p 'src/gems'
    puts "Checkout out gem #{gem} #{data['version']}"
    Dir.chdir('src/gems') do
      unless File.exist?(File.join(gem, "#{gem}.gemspec"))
        if data['repo']
          `git clone -q #{data['repo']} #{gem}`
        else
          `git clone -q #{data['url']} #{gem}`
        end
      end

      Dir.chdir(gem) do
        `git fetch -q`
        `git checkout -q #{data['version']} 2> /dev/null`
        `git checkout -q v#{data['version']} 2> /dev/null`
      end
    end
  end
end

task :gem_update do
  deps['gems'].each do |gem,data|
    data['version'] = Gem.latest_spec_for(gem).version.to_s
  end
  File.write('build/sources.yaml', deps.to_yaml)
end

task :rpms_present do
  rpms=Dir.glob('dist/simp-vendored-r10k-*.noarch.rpm')
  if rpms.empty?
    Rake::Task['pkg:rpm'].invoke
  end
end

namespace :pkg do
  directory 'dist'

  desc 'build rubygem sub-packages'
  task :gem => ['dist', :checkout] do
    deps['gems'].each do |gem,data|
      puts "Building gem #{gem} #{data['version']}"
      Dir.chdir("src/gems/#{gem}") do |d|
        # multi_json requires a signing key, we don't have one
        sh "sed -i '/signing_key/d' #{gem}.gemspec" if gem == 'multi_json'
        # this is a very old gem and newer versions of rubygems require a LICENSE
        # the master branch has one but there will be no new release
        sh 'curl -s https://raw.githubusercontent.com/defunkt/colored/master/LICENSE > LICENSE' if gem == 'colored'

        `gem build --silent #{gem}.gemspec`
        FileUtils.mkdir_p 'dist'
        FileUtils.mv Dir.glob('*.gem'), File.join(@rakefile_dir, 'dist')
      end
    end
  end

  desc 'Generate the rpm spec file using build/sources.yaml'
  task :rpmspec do
    changelog = File.read('CHANGELOG')

    f = File.open('build/simp-vendored-r10k.spec', 'w')
    f << ERB.new(File.read('build/simp-vendored-r10k.spec.erb'), nil, '-').result(binding)
    f.close
  end

  Rake::Task['pkg:rpm'].clear
  desc 'Build the rpms for all gems in the build/sources.yaml file'
  task :rpm => [:rpmspec,:gem] do
    version  = deps['version']
    release  = deps['gems']['r10k']['release'].nil? ? 0 : deps['gems']['r10k']['release']
    builddir = File.join('dist','RPMBUILD')

    FileUtils.mkdir_p 'dist'
    FileUtils.mkdir_p ['BUILD','BUILDROOT','RPMS','SOURCES','SPECS','SRPMS'].map { |e| File.join(builddir,e) }
    FileUtils.cp Dir.glob('dist/*.gem'), File.join(builddir, 'SOURCES')

    Dir.chdir(File.join('dist','tmp')) do
      sh "[ -e simp-vendored-r10k-#{version} ] || ln -s ../.. simp-vendored-r10k-#{version}"
      tar_cmd  = []
      tar_cmd << 'tar --dereference'
      tar_cmd << "--exclude=simp-vendored-r10k-#{version}/dist"
      tar_cmd << "--exclude=simp-vendored-r10k-#{version}/vendor"
      tar_cmd << "--exclude=simp-vendored-r10k-#{version}/.vendor"
      tar_cmd << "--exclude=simp-vendored-r10k-#{version}/.bundle"
      tar_cmd << "--exclude=simp-vendored-r10k-#{version}/spec/fixtures/modules"
      tar_cmd << '-czf'
      tar_cmd << "../RPMBUILD/SOURCES/simp-vendored-r10k-#{version}-#{release}.tar.gz"
      tar_cmd << "simp-vendored-r10k-#{version}"
      sh tar_cmd.join(' ')
    end
    FileUtils.rm_rf File.join('dist','tmp')

    buildroot  = File.join(Dir.pwd,builddir)
    src_cmd  = []
    src_cmd << 'rpmbuild'
    src_cmd << "-D '_topdir #{buildroot}'"
    src_cmd << '-v -bs build/simp-vendored-r10k.spec'
    sh src_cmd.join(' ')

    rpm_cmd  = []
    rpm_cmd << 'rpmbuild'
    rpm_cmd << "-D '_topdir #{buildroot}'"
    rpm_cmd << '-v -ba build/simp-vendored-r10k.spec'
    sh rpm_cmd.join(' ')

    FileUtils.cp Dir.glob('dist/RPMBUILD/RPMS/noarch/*.rpm'), 'dist'
  end

  Rake::Task[:rpm].prerequisites.unshift(:rpmspec)
  Rake::Task[:check_rpm_changelog].enhance [:rpmspec]
  Rake::Task[:create_tag_changelog].enhance [:rpmspec]

  # CAUTION: pkg:compare_latest_tag doesn't work with release-only version
  # differences in this project...
  Rake::Task[:compare_latest_tag].enhance [:rpmspec]
end

task :default do
  system('rake -T')
end
# Acceptance Tests
Simp::Rake::Beaker.new(File.dirname(__FILE__))

# make sure pkg:rpm is a prerequisite for beaker:suites
Rake::Task['beaker:suites'].enhance ['pkg:rpm']

# vim: syntax=ruby
