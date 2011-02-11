require 'rake/rdoctask'
require 'rake/packagetask'
require 'rake/clean'
require 'find'

PROJ_DOC_TITLE = "MCollective Server Provisioner"
PROJ_VERSION = "2.0"
PROJ_RELEASE = "1"
PROJ_NAME = "mcollective-server-provisioner"
PROJ_URL = "http://marionette-collective.org/"
PROJ_FILES = ["provisioner.rb", "COPYING", "README.markdown", "doc", "etc", "lib", "ext", "agent"]

ENV["PKG_VERSION"] ? CURRENT_VERSION = ENV["PKG_VERSION"] : CURRENT_VERSION = PROJ_VERSION
ENV["BUILD_NUMBER"] ? CURRENT_RELEASE = ENV["BUILD_NUMBER"] : CURRENT_RELEASE = PROJ_RELEASE

CLEAN.include(["build", "doc"])

def announce(msg='')
    STDERR.puts "================"
    STDERR.puts msg
    STDERR.puts "================"
end

def init
    FileUtils.mkdir("build") unless File.exist?("build")
end

def safe_system *args
    raise RuntimeError, "Failed: #{args.join(' ')}" unless system *args
end

desc "Build documentation, tar balls and rpms"
task :default => [:clean, :doc, :package]

# task for building docs
rd = Rake::RDocTask.new(:doc) { |rdoc|
    rdoc.rdoc_dir = 'doc'
    rdoc.template = 'html'
    rdoc.title    = "#{PROJ_DOC_TITLE} version #{CURRENT_VERSION}"
    rdoc.options << '--line-numbers' << '--inline-source' << '--main' << 'MCollective'
}

#desc "Run spec tests"
#task :test do
#    sh "cd spec;rake"
#end

desc "Create a tarball for this release"
task :package => [:clean, :doc] do
    announce "Creating #{PROJ_NAME}-#{CURRENT_VERSION}.tgz"

    FileUtils.mkdir_p("build/#{PROJ_NAME}-#{CURRENT_VERSION}")
    safe_system("cp -R #{PROJ_FILES.join(' ')} build/#{PROJ_NAME}-#{CURRENT_VERSION}")

    announce "Setting MCollective.version to #{CURRENT_VERSION}"
    safe_system("cd build/#{PROJ_NAME}-#{CURRENT_VERSION}/lib && sed --in-place -e s/@DEVELOPMENT_VERSION@/#{CURRENT_VERSION}/ mcprovision.rb")
    safe_system("cd build/#{PROJ_NAME}-#{CURRENT_VERSION}/agent && sed --in-place -e s/@DEVELOPMENT_VERSION@/#{CURRENT_VERSION}/ provision.rb")
    safe_system("cd build/#{PROJ_NAME}-#{CURRENT_VERSION}/agent && sed --in-place -e s/@DEVELOPMENT_VERSION@/#{CURRENT_VERSION}/ provision.ddl")

    safe_system("cd build && tar --exclude .svn -cvzf #{PROJ_NAME}-#{CURRENT_VERSION}.tgz #{PROJ_NAME}-#{CURRENT_VERSION}")
end

namespace :package do
    desc "Create .deb packages"
    task :deb => [:clean, :doc, :package] do
        announce("Building debian packages")

        FileUtils.mkdir_p("build/deb")
        Dir.chdir("build/deb") do
            safe_system %{tar -xzf ../#{PROJ_NAME}-#{CURRENT_VERSION}.tgz}
            safe_system %{cp ../#{PROJ_NAME}-#{CURRENT_VERSION}.tgz #{PROJ_NAME}_#{CURRENT_VERSION}.orig.tar.gz}

            Dir.chdir("#{PROJ_NAME}-#{CURRENT_VERSION}") do
                safe_system %{cp -R ext/debian .}

                File.open("debian/changelog", "w") do |f|
                    f.puts("#{PROJ_NAME} (#{CURRENT_VERSION}-#{CURRENT_RELEASE}) unstable; urgency=low")
                    f.puts
                    f.puts("  * Automated release for #{CURRENT_VERSION}-#{CURRENT_RELEASE} by rake package:deb")
                    f.puts
                    f.puts("    See #{PROJ_URL} for full details")
                    f.puts
                    f.puts(" -- Kiall Mac Innes <kiall@managedit.ie>  #{Time.new.strftime('%a, %d %b %Y %H:%M:%S %z')}")
                end

                if ENV['SIGNED'] == '1'
                    safe_system %{debuild -i -b}
                else
                    safe_system %{debuild -i -us -uc -b}
                end
            end

            safe_system %{cp *.deb ..}
        end
    end

    desc "Create RPM packages"
    task :rpm => [:clean, :doc, :package] do
        announce("Building RPM for #{PROJ_NAME}-#{CURRENT_VERSION}-#{CURRENT_RELEASE}")

        sourcedir = `rpm --eval '%_sourcedir'`.chomp
        specsdir = `rpm --eval '%_specdir'`.chomp
        srpmsdir = `rpm --eval '%_srcrpmdir'`.chomp
        rpmdir = `rpm --eval '%_rpmdir'`.chomp
        lsbdistrel = `lsb_release -r -s | cut -d . -f1`.chomp
        lsbdistro = `lsb_release -i -s`.chomp

        case lsbdistro
            when 'CentOS'
                rpmdist = ".el#{lsbdistrel}"
            else
                rpmdist = ""
        end

        safe_system %{cp build/#{PROJ_NAME}-#{CURRENT_VERSION}.tgz #{sourcedir}}
        safe_system %{cat #{PROJ_NAME}.spec|sed -e s/%{rpm_release}/#{CURRENT_RELEASE}/g | sed -e s/%{version}/#{CURRENT_VERSION}/g > #{specsdir}/#{PROJ_NAME}.spec}

        if ENV['SIGNED'] == '1'
            safe_system %{rpmbuild --sign -D 'version #{CURRENT_VERSION}' -D 'rpm_release #{CURRENT_RELEASE}' -D 'dist #{rpmdist}' -D 'use_lsb 0' -ba #{PROJ_NAME}.spec}
        else
            safe_system %{rpmbuild -D 'version #{CURRENT_VERSION}' -D 'rpm_release #{CURRENT_RELEASE}' -D 'dist #{rpmdist}' -D 'use_lsb 0' -ba #{PROJ_NAME}.spec}
        end

        safe_system %{cp #{srpmsdir}/#{PROJ_NAME}-#{CURRENT_VERSION}-#{CURRENT_RELEASE}#{rpmdist}.src.rpm build/}

        safe_system %{cp #{rpmdir}/*/#{PROJ_NAME}*-#{CURRENT_VERSION}-#{CURRENT_RELEASE}#{rpmdist}.*.rpm build/}
    end
end
