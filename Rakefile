# encoding: utf-8
require 'rubygems'
require 'rake'
require 'date'

# Defines gem name.
def repo_name; 'appium_lib' end # ruby_lib published as appium_lib
def gh_name; 'ruby_lib' end # the name as used on github.com
def version_file; "lib/#{repo_name}/common/version.rb" end
def version_rgx; /VERSION = '([^']+)'/m end

def version
  @version = @version || File.read(version_file).match(version_rgx)[1]
end

def bump
  data = File.read version_file

  v_line = data.match version_rgx
  d_line = data.match /DATE = '([^']+)'/m

  old_v = v_line[0]
  old_d = d_line[0]

  old_num = v_line[1]
  new_num = old_num.split('.')
  new_num[-1] = new_num[-1].to_i + 1
  new_num = new_num.join '.'

  new_v = old_v.sub old_num, new_num
  puts "#{old_num} -> #{new_num}"

  old_date = d_line[1]
  new_date = Date.today.to_s
  new_d = old_d.sub old_date, new_date
  puts "#{old_date} -> #{new_date}" unless old_date == new_date

  data.sub! old_v, new_v
  data.sub! old_d, new_d

  File.write version_file, data
end

desc 'Bump the version number and update the date.'
task :bump do
  bump
end

desc 'Install gems required for release task'
task :dev do
  sh 'gem install --no-rdoc --no-ri yard'
  sh 'gem install --no-rdoc --no-ri redcarpet'
end

# Inspired by Gollum's Rakefile
desc 'Build and release a new gem to rubygems.org'
task :release => :gem do
  unless `git branch`.include? '* master'
    puts 'Master branch required to release.'
    exit!
  end

  # Commit then pull before pushing.
  sh "git commit --allow-empty -am 'Release #{version}'"
  sh 'git pull'
  sh "git tag v#{version}"
  # update notes and docs now that there's a new tag
  Rake::Task['notes'].execute
  Rake::Task['docs'].execute
  sh "git commit --allow-empty -am 'Update release notes'"
  sh 'git push origin master'
  sh "git push origin v#{version}"
  sh "gem push #{repo_name}-#{version}.gem"
end

desc 'Build and release a new gem to rubygems.org (same as release)'
task :publish => :release do
end

desc 'Build a new gem'
task :gem do
  `chmod 0600 ~/.gem/credentials`
  sh "gem build #{repo_name}.gemspec"
end

desc 'Build a new gem (same as gem task)'
task :build => :gem do
end

desc 'Uninstall gem'
task :uninstall do
  cmd = "gem uninstall -aIx #{repo_name}"
  puts cmd
  # rescue on gem not installed error.
  begin; `cmd`; rescue; end
end

desc 'Install gem'
task :install => [ :gem, :uninstall ] do
  sh "gem install --no-rdoc --no-ri --local #{repo_name}-#{version}.gem"
end

desc 'Update android and iOS docs'
task :docs do
  sh "ruby docs_gen/make_docs.rb"
end

desc 'Update release notes'
task :notes do
  tag_sort = ->(tag1,tag2) do
    tag1_numbers = tag1.match(/\.?v(\d+\.\d+\.\d+)$/)[1].split('.').map! { |n| n.to_i }
    tag2_numbers = tag2.match(/\.?v(\d+\.\d+\.\d+)$/)[1].split('.').map! { |n| n.to_i }
    tag1_numbers <=> tag2_numbers
  end

  tags = `git tag`.split "\n"
  tags.sort! &tag_sort
  pairs = []
  tags.each_index { |a| pairs.push tags[a] + '...' + tags[a+1] unless tags[a+1].nil? }

  notes = ''

  dates = `git log --tags --simplify-by-decoration --pretty="format:%d %ad" --date=short`.split "\n"

  pairs.sort! &tag_sort
  pairs.reverse! # pairs are in reverse order.

  tag_date = []
  pairs.each do |pair|
    tag = pair.split('...').last
    dates.each do |line|
      # regular tag, or tag on master.
      if line.include?('(' + tag + ')') || line.include?(tag + ',')
        tag_date.push tag + ' ' + line.match(/\d{4}-\d{2}-\d{2}/)[0]
        break
      end
    end
  end

  pairs.each_index do |a|
    data =`git log --pretty=oneline #{pairs[a]}`
    new_data = ''
    data.split("\n").each do |line|
      hex = line.match(/[a-zA-Z0-9]+/)[0]
      # use first 7 chars to match GitHub
      comment = line.gsub(hex, '').strip
      next if comment == 'Update release notes'
      new_data += "- [#{hex[0...7]}](https://github.com/appium/#{gh_name}/commit/#{hex}) #{comment}\n"
    end
    data = new_data + "\n"

    # last pair is the released version.
    notes += "#### #{tag_date[a]}\n\n" + data + "\n"
  end

  File.open('release_notes.md', 'w') { |f| f.write notes.to_s.strip }
end
