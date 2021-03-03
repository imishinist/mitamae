require 'fileutils'
require 'shellwords'

file :mruby do
  sh "git clone --branch=stable https://github.com/k0kubun/mruby"
  Dir.chdir('mruby') do
    IO.popen(['patch', '-p1'], 'w') do |io|
      io.write(<<-'EOS')
diff --git a/lib/mruby/build.rb b/lib/mruby/build.rb
index d6eabd79..a7973280 100644
--- a/lib/mruby/build.rb
+++ b/lib/mruby/build.rb
@@ -320,12 +320,16 @@ EOS
       return @mrbcfile if @mrbcfile

       gem_name = "mruby-bin-mrbc"
-      gem = @gems[gem_name]
-      gem ||= (host = MRuby.targets["host"]) && host.gems[gem_name]
-      unless gem
-        fail "external mrbc or mruby-bin-mrbc gem in current('#{@name}') or 'host' build is required"
+      if (gem = @gems[gem_name])
+        @mrbcfile = exefile("#{gem.build.build_dir}/bin/mrbc")
+      elsif !host? && (host = MRuby.targets["host"])
+        if (gem = host.gems[gem_name])
+          @mrbcfile = exefile("#{gem.build.build_dir}/bin/mrbc")
+        elsif host.mrbcfile_external?
+          @mrbcfile = host.mrbcfile
+        end
       end
-      @mrbcfile = exefile("#{gem.build.build_dir}/bin/mrbc")
+      @mrbcfile || fail("external mrbc or mruby-bin-mrbc gem in current('#{@name}') or 'host' build is required")
     end

     def mrbcfile=(path)
      EOS
    end
  end
end

DOCKCROSS_TARGETS = %w[
  linux-x86_64
  linux-i686
  linux-armhf
  linux-aarch64
  darwin-x86_64
  darwin-i386
]

STRIP_TARGETS = %w[
  linux-x86_64
  linux-i686
]

# avoid redefining constants in mruby Rakefile
mruby_root = File.expand_path(ENV['MRUBY_ROOT'] || "#{Dir.pwd}/mruby")
mruby_config = File.expand_path(ENV['MRUBY_CONFIG'] || 'build_config.rb')
ENV['MRUBY_ROOT'] = mruby_root
ENV['MRUBY_CONFIG'] = mruby_config
Rake::Task[:mruby].invoke unless Dir.exist?(mruby_root)
Dir.chdir(mruby_root)
load "#{mruby_root}/Rakefile"

desc 'run serverspec'
task 'test:integration' do
  Dir.chdir(__dir__) do
    sh 'bundle check || bundle install -j4'
    sh 'bundle exec rspec'
  end
end

desc 'compile binary'
task compile: :all

desc 'cleanup'
task :clean do
  sh 'rake deep_clean'
end

desc 'cross compile for release'
task 'release:build' => DOCKCROSS_TARGETS.map { |target| "release:build:#{target}" }

DOCKCROSS_TARGETS.each do |target|
  desc "Build for #{target}"
  task "release:build:#{target}" do
    sh [
      'docker', 'run', '--rm', '-e', "BUILD_TARGET=#{target}",
      '-v', "#{File.expand_path(__dir__)}:/home/mruby/code", '-w', '/home/mruby/code',
      "k0kubun/mitamae-dockcross:#{target}", 'rake', 'compile',
    ].shelljoin

    Dir.chdir(__dir__) do
      FileUtils.mkdir_p('mitamae-build')
      os, arch = target.split('-', 2)
      bin = "mitamae-build/mitamae-#{arch}-#{os}"
      sh "cp mruby/build/#{target.shellescape}/bin/mitamae #{bin.shellescape}"

      if STRIP_TARGETS.include?(target)
        sh "strip --strip-unneeded #{bin.shellescape}"
      end
    end
  end
end

desc 'compress binaries in mitamae-build'
task 'release:compress' do
  Dir.chdir(File.expand_path('./mitamae-build', __dir__)) do
    Dir.glob('mitamae-*').each do |path|
      sh "tar zcvf #{path}.tar.gz #{path}"
    end
  end
end
