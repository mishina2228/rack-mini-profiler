# frozen_string_literal: true

require 'rubygems'
require 'bundler'
require 'bundler/gem_tasks'

Bundler.setup(:default, :test)

require 'rubocop/rake_task'

RuboCop::RakeTask.new

task default: [:rubocop, :spec]

require 'rspec/core'
require 'rspec/core/rake_task'
RSpec::Core::RakeTask.new(:spec) do |spec|
  pattern = ARGV[1] || 'spec/**/*_spec.rb'
  excluded = 'spec/support/*.rb'
  spec.pattern = FileList[pattern] - FileList[excluded]
  spec.verbose = false
  # spec.rspec_opts = ["-p"] # turns on profiling
end

desc "builds a gem"
task build: :update_asset_version do
  `gem build rack-mini-profiler.gemspec 1>&2`
end

desc "compile sass"
task :compile_sass do
  require "sassc"
  scss = File.read("lib/html/includes.scss")
  css = SassC::Engine.new(scss).render
  File.write("lib/html/includes.css", css)
end

desc "update asset version file"
task update_asset_version: [:compile_sass, :write_vendor_js] do
  require 'digest/md5'
  h = []
  Dir.glob('lib/html/*.{js,html,css,tmpl}').each do |f|
    h << Digest::MD5.hexdigest(::File.read(f))
  end
  File.open('lib/mini_profiler/asset_version.rb', 'w') do |f|
    f.write \
"# frozen_string_literal: true
module Rack
  class MiniProfiler
    ASSET_VERSION = '#{Digest::MD5.hexdigest(h.sort.join(''))}'
  end
end\n"
  end
end

@mini_racer_context = nil
desc "generate vendor asset file"
task :write_vendor_js do
  require 'mini_racer'
  require 'nokogiri'

  dot_js = File.read(File.expand_path("../lib/html/dot.1.1.2.min.js", __FILE__))
  html = File.read(File.expand_path("../lib/html/includes.tmpl", __FILE__))

  templates = {}
  Nokogiri::HTML(html).css('script[type="text/x-dot-tmpl"]').each do |node|
    id = node["id"]
    raise "Each template must have a unique id" if !id || id.size == 0 || templates.key?(id)
    templates[id] = node.content
  end

  @mini_racer_context ||= MiniRacer::Context.new
  @mini_racer_context.eval(dot_js)
  templates_js = "MiniProfiler.templates = {};\n"

  templates.each do |k, v|
    template = v.gsub('`', '\\`')
    compiled = @mini_racer_context.eval <<~JS
      doT.compile(`#{template}`).toString()
    JS
    templates_js += <<~JS
      MiniProfiler.templates["#{k}"] = #{compiled}
    JS
  end

  pretty_print = File.read(File.expand_path("../lib/html/pretty-print.js", __FILE__))
  content = <<~JS
    /**
      THIS FILE IS AUTOMATICALLY GENERATED BY THE `write_vendor_js` RAKE TASK.
      DON'T EDIT THIS FILE BY HAND; CHANGES WILL BE OVERRIDEN.
    **/

    "use strict";
    #{templates_js}
    #{pretty_print}
    MiniProfiler.loadedVendor = true;
  JS
  path = File.expand_path("../lib/html/vendor.js", __FILE__)
  FileUtils.touch(path)
  File.write(path, content)
end

desc "Start Sinatra server for client-side development"
task :client_dev do
  require 'listen'

  regexp = /(vendor\.js|includes\.css)$/
  listener = Listen.to(File.expand_path("lib/html", __dir__)) do |modified|
    next if modified.all? { |m| m =~ regexp }
    print("Assets change detected; updating ASSET_VERSION constant and recompiling templates... ")
    rake_task = Rake.application[:update_asset_version]
    rake_task.all_prerequisite_tasks.each(&:reenable)
    rake_task.reenable
    rake_task.invoke
    puts "Done"
  rescue => err
    puts "\nError occurred: #{err.inspect}"
  end
  listener.start
  pid = spawn("cd website && BUNDLE_GEMFILE=Gemfile bundle exec rackup")
  Process.wait(pid)
rescue Interrupt
  listener.stop
end

desc "Upgrade Speedscope to the latest version"
task :speedscope_upgrade do
  require 'net/http'
  require 'json'
  require 'tmpdir'
  require 'zip'

  puts "Checking GitHub for the latest version..."
  releases_uri = URI('https://api.github.com/repos/jlfwong/speedscope/releases/latest')
  req = Net::HTTP::Get.new(releases_uri, { 'Accept' => 'application/vnd.github.v3+json' })
  http = Net::HTTP.new(releases_uri.hostname, releases_uri.port)
  http.use_ssl = true
  res = http.request(req)
  if res.code.to_i != 200
    puts "ERROR: GitHub responded with an unexpected status code: #{res.code.to_i}."
    exit
  end
  latest_release_info = JSON.parse(res.body)
  latest_version = latest_release_info['name'].sub('v', '')

  speedscope_dir = File.expand_path('./lib/html/speedscope', __dir__)
  current_version = File.read(File.join(speedscope_dir, 'release.txt')).split("\n")[0].split("@")[-1]
  if latest_version == current_version
    puts "Speedscope is already on the latest version (#{current_version.inspect})."
    exit
  end
  puts "Current version is #{current_version.inspect} and latest version is: #{latest_version.inspect}"
  asset = latest_release_info['assets'].find { |ast| ast['content_type'] == 'application/zip' }
  asset_id = asset && asset['id']
  if !asset_id
    puts "ERROR: Couldn't find any zip files in the #{latest_version.inspect} release. "\
         "Maybe the maintainer forgot to add one or the content type has changed. "\
         "Please the check the releases page of the repository and/or contact the maintainer."
    exit
  end
  Dir.mktmpdir do |temp_dir|
    puts "Downloading zip file of latest release to #{temp_dir}..."
    download_uri = URI("https://api.github.com/repos/jlfwong/speedscope/releases/assets/#{asset_id}")
    req = Net::HTTP::Get.new(download_uri, { 'Accept' => 'application/octet-stream' })
    http = Net::HTTP.new(download_uri.hostname, download_uri.port)
    http.use_ssl = true
    res = http.request(req)
    if res.code.to_i != 302
      puts "ERROR: Expected a 302 status code from GitHub download URL but instead got #{res.code.inspect}."
      exit
    end
    aws_uri = URI(res['Location'])
    http = Net::HTTP.new(aws_uri.host, aws_uri.port)
    http.use_ssl = true
    request = Net::HTTP::Get.new(aws_uri)
    temp_zip_file = File.join(temp_dir, "speedscope-v#{latest_version}.zip")
    http.request(request) do |response|
      if response.code.to_i != 200
        puts "ERROR: Expected a 200 status code from download URL but instead got #{res.code.inspect}."
        exit
      end
      File.open(temp_zip_file, 'w') do |io|
        response.read_body do |chunk|
          io.write(chunk)
        end
      end
    end
    puts "Download completed."
    kept_files = File.read(File.join(speedscope_dir, '.kept-files')).split("\n").reject { |n| n.strip.start_with?('//') }
    puts "Deleting existing speedscope files..."
    Dir.foreach(speedscope_dir) do |name|
      next if name == '.' || name == '..'
      next if kept_files.include?(name)
      full_path = File.join(speedscope_dir, name)
      File.delete(full_path)
      msg = "Deleted #{full_path}"
      puts msg.rjust(msg.size + 4, ' ')
    end
    puts "Extracting zip files..."
    Zip::File.open(temp_zip_file) do |zip_file|
      zip_file.each do |entry|
        next if !entry.name.start_with?('speedscope/')
        next if entry.name =~ /perf-vertx-stacks/
        next if entry.name =~ /README/
        dest_path = File.join(File.dirname(speedscope_dir), entry.name)
        entry.extract(dest_path)
        msg = "Extracted #{entry.name} to #{dest_path}"
        puts msg.rjust(msg.size + 4, ' ')
      end
    end
    new_version = File.read(File.join(speedscope_dir, 'release.txt')).split("\n")[0].split("@")[-1]
    if new_version != latest_version
      puts "ERROR: Something went wrong. Expected the zip file to contain release #{latest_version.inspect}, "\
           "but instead it contained #{new_version.inspect}. You'll need to investigate what went wrong."
      exit
    end
    puts "Replacing Google Fonts stylesheet URL with the URL of the local copy in index.html..."
    index_html_content = File.read(File.join(speedscope_dir, 'index.html'))
    index_html_content.sub!('https://fonts.googleapis.com/css?family=Source+Code+Pro', 'fonts/source-code-pro-regular.css')
    File.write(File.join(speedscope_dir, 'index.html'), index_html_content)
    puts "All done!"
  end
end
